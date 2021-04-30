# Cross-validate your machine-learning model with SageMaker and Step Functions

> Automatize cross-validated machine-learning training jobs on AWS infrastructure

![Antelopes on the savanna](images/springbok-antelope.jpg)  
*Image by [TeeFarm](https://pixabay.com/users/teefarm-199315/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=3758346) from [Pixabay](https://pixabay.com/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=3758346)*

Cross-validation is a powerful technique to build  machine learning models that perform well on unseen data. However it can be also time consuming as it includes training multiple models. In this post I will show you how to automatically k-fold cross-validate a machine-learning model using several services of Amazon Web Services (AWS) including SageMaker, Step Functions and Lambda.

## Why do you need cross-validation?

> If you know the concept of cross-validation feel free to jump directly to section introducing [SMX-Validator](#smx-validator).

### Small datasets and sample distribution

Imagine the antelopes of the savanna entrust you to train an image classifier model that helps them to decide if there is a jaguar in a picture or not. They give you 50 photos of a jaguar and 50 photos of the savanna with no jaguars. You divide the dataset into a training set of 80 photos and a test set of 20 photos, taking care that in each set there would be equal number of jaguar and non-jaguar photos. You train your model with your favorite image classifier algorithm and get an impressive validation accuracy of 95%. 

You decide to at some correctly classified photos in the test set:

![Amur Leopard](images/amur-leopard.jpg)  
*Image by [Mark Murphy](https://pixabay.com/users/markmurphy-10772193/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=4112011) from [Pixabay](https://pixabay.com/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=4112011)*

Some time later you decide to retrain your model with the same parameters. You split your dataset again into 80/20 train/test sets, use exactly the same hyperparameter of the first model, and get a validation accuracy of 80%, with a couple of false negatives (lethal for the antelopes!). So what has happened?

You look at the false negatives in the test set and find photos like this:

![Camouflaged Sri Lankan Leopard](images/Camouflaged_Sri_Lankan_leopard.jpg)  
*Photo by [Senthiaathavan](https://commons.wikimedia.org/wiki/File:Camouflaged_Sri_Lankan_leopard_(Panthera_pardus_kotiya).jpg) on [Wikimedia Commons](https://commons.wikimedia.org/wiki/Main_Page)*

It is obvious that there are "easier" and "harder" examples in your dataset. When you split your dataset into the train/test sets, your goal is always that the *test set would be a good representative of the whole data*. Talking about class distribution you can enforce this using stratified split strategy, but what about sample "hardness" it is much more difficult task. Especially if you have a small dataset you could easily get a split where all "hard" samples finish in the training set, so your test set will contain only "easy" samples and thus lead to a good test score. On the other hand if most "hard" examples will be assigned to the test set, you will get a worse result using a model trained with exactly same hyperparameters.

### Hyperparameter optimization

An other very important case is when you are tuning the hyperparameters of your model. In case of [*hyperparameter optimization*](https://en.wikipedia.org/wiki/Hyperparameter_optimization) you iteratively modify some hyperparameter of the model and retrain it with the same training set and check the performance on the validation set. If the model performs better since the last training, you know that the hyperparameter adjustment was likely in the correct direction so you have a clue how to continue with the tuning. 

![Sound Mixer Image](images/mixer.jpg)  
*Image by [Niklas Ahrnke](https://pixabay.com/users/niklas_ahrnke-10296613/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=3721929) from [Pixabay](https://pixabay.com/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=3721929)*

The problem with this approach is that there is some information leaking from the validation set to the training process. As the tuning process depends on how the model performs on the validation set, you might end up with a hyperparameter set (and model) that is optimized for *that particular validation set* and not for the generic case. The simplest solution to this problem is to divide your dataset to three partitions: [training, validation and test sets](https://en.wikipedia.org/wiki/Training,_validation,_and_test_sets), tune your model with the training and validation set, and get the final performance metrics with the completely unseen test set. Unfortunately this means even less samples that you can use for training as you would split your dataset for example into 60-20-20 sized train-validation-test splits.

### Other reasons

There are some other scenarios when cross-validation can be particular useful, for example when your dataset contains interdependent data points or when you plan to stack up machine learning models so that one model uses the prediction of the previous model as input. For a detailed discussion of these cases check out [this article](https://towardsdatascience.com/5-reasons-why-you-should-use-cross-validation-in-your-data-science-project-8163311a1e79).

## How can cross-validation help?

[*Cross-validation*](https://en.wikipedia.org/wiki/Cross-validation_(statistics)) is a group of techniques to asses how well your model can generalize for unseen data. The standard train-test split of the dataset can be thought as one round of a cross-validation process. Indeed most cross-validation strategies use multiple rounds to split the dataset into different partitions, train different models and then assess their combined performance.

One of the most used cross-validation strategy is called *k-fold cross-validation*. Wikipedia gives a [precise definition](https://en.wikipedia.org/wiki/Cross-validation_(statistics)#k-fold_cross-validation) of what k-fold cross-validation is:

> In k-fold cross-validation, the original sample is randomly partitioned into k equal sized subsamples. Of the k subsamples, a single subsample is retained as the validation data for testing the model, and the remaining *k&nbsp;−&nbsp;1* subsamples are used as training data. The cross-validation process is then repeated k times, with each of the k subsamples used exactly once as the validation data. The k results can then be averaged to produce a single estimation.

For example, if you use 5-fold cross-validation, you will train 5 different model using the following splits:

![5-fold cross-validation split scheme](images/crossvalidation-5fold.png)  
*Illustration by the author*

In the jaguar classification example above the "hard" sample will turn up in the test set of on training round and in the training set of the other four rounds. Examining the accuracy of all five models will give you a better overall picture of how your model will perform on unseen dataset. At the same time you've used all available data to train and validate your model.

This all sounds good but this means that you have to split the dataset into 5 folds, assemble them into the training sets and test sets of the 5 training rounds, schedule the training of each machine learning models on the available hardware and collect the training metrics from each training jobs. This is a considerable overhead to deal with, especially if you need to train your model automatically, on a regular bases.

Here it comes into play SMX-Validator.

## SMX-Validator

SMX-Validator is an application directly deployable to AWS infrastructure that manages the cross-validated training of almost any supervised machine learning model that you can train with SageMaker.

The application is built upon several AWS services:

 - [Amazon SageMaker](https://aws.amazon.com/sagemaker/) is a cloud machine-learning platform that enables developers to create, train, and deploy machine-learning (ML) models in the AWS cloud. 
 - [AWS Step Functions](https://aws.amazon.com/step-functions/) is a serverless function orchestrator and state machine implementation that can be used to sequence AWS Lambda functions and other AWS services, including SageMaker.
 - [AWS Lambda](https://aws.amazon.com/lambda/) is a serverless computing platform that can be used to run code in response to events, for example in response to a SageMaker "training finished" event. It also automatically manages the computing resources to run that code.
 - [Amazon S3](https://aws.amazon.com/s3/) is an object storage service that allows storing data in the cloud with a filesystem-like interface.

 After deploying SMX-Validator to your AWS account, you specify a machine learning model training template, an input dataset and the number of folds you wish to train. The application will automatically orchestrate all training jobs, execute them possibly in parallel and report you back the results and performance of each training job. Follow along to train your first cross-validated job with SMX-Validator.

 ### Prerequisites

 1. An AWS account. If you don't have one yet, open a [new one](https://aws.amazon.com/premiumsupport/knowledge-center/create-and-activate-aws-account/).
 2. AWS offers a [free tier](https://aws.amazon.com/free/) for new account. However GPU instances (highly recommended for training image classifiers) are not included so you should expect some training costs, in the order of magnitude of a couple of USD.
 3. A dataset that you want to train on. In the [Prepare your data](#prepare-your-data) section below I will describe how to get a dataset if you don't have already one.
 4. SMX-Validator is a [Serverless Application Model (SAM)](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/what-is-sam.html) application. You will need the the [SAM Command Line Interface (CLI)](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html) to deploy it to your AWS account. Install the SAM CLI with Docker and setup your AWS credentials as described in the documentation.

### Supported SageMaker algorithms and containers

You can use any supervised training container with SMX-Validator that accepts a newline separated file as input, for example:
 - Linear Learner with [CSV input format](https://docs.aws.amazon.com/sagemaker/latest/dg/linear-learner.html#ll-input_output)
 - K-Nearest Neighbors Algorithm with [CSV input format](https://docs.aws.amazon.com/sagemaker/latest/dg/k-nearest-neighbors.html#kNN-input_output)
 - Image Classification Algorithm with [Augmented Manifest Image Format](https://docs.aws.amazon.com/sagemaker/latest/dg/image-classification.html#IC-augmented-manifest-training)
 - XGBoost Algorithm with [CSV input format](https://docs.aws.amazon.com/sagemaker/latest/dg/xgboost.html#InputOutput-XGBoost)
 - BlazingText in Text Classification mode with [File Mode or Augmented Manifest Text Format](https://docs.aws.amazon.com/sagemaker/latest/dg/blazingtext.html#blazingtext-data-formats-text-class)
 - your custom training container that accepts newline separated files as input.

### Prepare your data

In this tutorial we will use a binary image classification dataset. If you don't have one, you can download the famous [dogs-vs-cats](https://www.kaggle.com/c/dogs-vs-cats) dataset. Download the dataset from Kaggle to a directory on your workstation. Create also an S3 bucket and upload all images files to the bucket.

To be able to use SMX-Validator with SageMaker Image Classification we'll have to create a [Augmented Manifest Image Format](https://docs.aws.amazon.com/sagemaker/latest/dg/image-classification.html#IC-augmented-manifest-training) file. Assuming you use the Dogs vs Cats dataset, this python script might help you to create one:

```python
import glob
remote_s3_prefix = 's3://my_bucket/dogs_vs_cats/'
with open('manifest.jsonl', 'w') as f:
    for img in glob.glob('*.jpg'):
        lbl = 0 if img.startswith('dog') else 1
        f.write('{{"source-ref":"s3://my_bucket/dogs_and_cats/{}", "class":"{}"}}\n'\
            .format(img, lbl))
```

Remember to change `my_bucket/dogs_vs_cats` to the bucket name and prefix of your input dataset location. Upload also the created `manifest.jsonl` to the input bucket. This script will assign the class id `0` for dogs and `1` for cats.

### Deploy SMX-Validator

Clone out the git repository of SMX-Validator to your workstation.

```bash
$ git clone [sm-cross-validator-repo-url]
$ cd sm-cross-validator
```

Build and deploy the application with the SAM CLI.

```bash
$ sam build --use-container
$ sam deploy --guided
```

The SAM CLI will guide you through the deployment process. It will also ask you the value of some parameters specific to SMX-Validator:

 - `InputBucketName`: the name of the S3 bucket where you uploaded your image dataset. At deployment time an [IAM Role](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html) will be created that allows the application to read data from this bucket.
 - `OutputBucketName`: the name of an S3 bucket where the application can write partial results and working data. At deployment time an IAM Role will be created that allows the application to read and write data to this bucket.

### Start a cross-validated training

SM-Cross-Validator deploys a Step Functions state machine that orchestrates the training of the cross-validated folds. You can start a cross-validated job launching a new execution of the state machine.

You should specify some input parameters of the cross-validated training job in a json file, like the S3 path of the input `manifest.jsonl`, the output path (these paths should point to the buckets you specified at application deploy time), the name of the cross-validated training job, the number of folds, and a SageMaker training job template including training hyperparameters and resource configuration. You can find the input schema specification, a detailed documentation and a complete example of an input file in the documentation of SMX-Validator.

Once you created your input json file, you have different options to start the execution of the state machine:

1. Using the the [AWS CLI](https://docs.aws.amazon.com/cli/latest/reference/stepfunctions/start-execution.html):
    ```bash
    aws stepfunctions start-execution \
        --state-machine-arn {CrossValidatorStateMachineArn} \
        --input file://my_input.json
    ```
2. From the [AWS Step Functions web console](https://console.aws.amazon.com/states/home#/statemachines) select the `CrossValidatorStateMachine-*` and on the state machine page click the "Start execution" button. Copy the contents of your input json into the Input text area.

3. Using the [AWS SDKs](https://aws.amazon.com/tools/), from a local script or from a lambda function.

SMX-Validator will launch training jobs in parallel. The number of concurrently executed jobs is specified in the state machine specification file (by default two) and can be adjusted to the number of available training ml-instances in your AWS account. The steps in the three light green boxes in the state machine diagram are executed concurrently.

![SMX Cross-Validator State Machine](images/smx-statemachine.png)  
*Illustration by the author*

### Training jobs

SMX-Validator will launch a SageMaker training job for each cross-validated splits. The number of splits you define in the `crossvalidation.n_splits` parameter. The training jobs will be named based on the following template:

```
crossvalidator-{input.job_config.name}-{YYYYMMDD}-{HHMMSS}-fold{fold_idx}
```

where `{input.job_config.name}` is the job name from the input configuration, `{YYYYMMDD}-{HHMMSS}` is the timestamp of the time at the start job and `{fold_idx}` is the zero-based index of the fold.

You can find all training jobs launched by SMX-Validator in the [SageMaker Training Jobs](https://console.aws.amazon.com/sagemaker/home#/jobs) console. On the page of a single training job you can find the training and validation metrics of the job, as well as the final accuracy metrics.

## Conclusion

Cross-validating a machine learning model is a powerful tool in the toolset of every data scientist. It can help you to efficiently assess models trained on a small dataset, in the case the dataset contains variable samples of variable "difficulty", if you are optimizing model hyperparameters. However training multiple models on limited hardware resources can be time consuming and needs manual orchestration. SMX-Validator helps to train k-fold cross-validated machine learning models on AWS infrastructure, taking care of the heavy-duty task orchestration of the training jobs.