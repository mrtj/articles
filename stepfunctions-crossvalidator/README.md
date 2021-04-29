# Cross-validate your machine-learning model with AWS SageMaker and Step Functions

## Build robust ML models with a single click!

Cross-validation is a powerful technique to build  machine learning models that perform well on unseen data. However it can be also time consuming as it includes training multiple models. In this post I will show you how to automatically k-fold cross-validate a machine-learning model using several services of Amazon Web Services (AWS) including SageMaker, Step Functions and Lambda.

## Why do you need cross-validation?

### Small datasets and sample distribution

Imagine the antelopes of the savanna entrust you to train an image classifier model that helps them to decide if there is a jaguar in a picture or not. They give you 50 photos of a jaguar and 50 photos of the savanna with no jaguars. You divide the dataset into a training set of 80 photos and a test set of 20 photos, taking care that in each set there would be equal number of jaguar and non-jaguar photos. You train your model with your favorite image classifier algorithm and get an impressive validation accuracy of 95%. 

You decide to at some correctly classified photos in the test set:

![Amur Leopard](images/amur-leopard.jpg)
*Photo by [MarkMurphy](https://pixabay.com/users/markmurphy-10772193/) on [Pixabay](https://pixabay.com)*

Some time later you decide to retrain your model with the same parameters. You split your dataset again into 80/20 train/test sets, use exactly the same hyperparameter of the first model, and get a validation accuracy of 80%, with a couple of false negatives (lethal for the antelopes!). So what has happened?

You look at the false negatives in the test set and find photos like this:

![Camouflaged Sri Lankan Leopard](images/Camouflaged_Sri_Lankan_leopard.jpg)
*Photo by [Senthiaathavan](https://commons.wikimedia.org/wiki/File:Camouflaged_Sri_Lankan_leopard_(Panthera_pardus_kotiya).jpg) on [Wikimedia Commons](https://commons.wikimedia.org/wiki/Main_Page)*

It is obvious that there are "easier" and "harder" examples in your dataset. When you split your dataset into the train/test sets, your goal is always that the *test set would be a good representative of the whole data*. Talking about class distribution you can enforce this using stratified split strategy, but what about sample "hardness" it is much more difficult task. Especially if you have a small dataset you could easily get a split where all "hard" samples finish in the training set, so your test set will contain only "easy" samples and thus lead to a good test score. On the other hand if most "hard" examples will be assigned to the test set, you will get a worse result using a model trained with exactly same hyperparameters.

### Hyperparameter optimization

An other very important case is when you are tuning the hyperparameters of your model. In case of [*hyperparameter optimization*](https://en.wikipedia.org/wiki/Hyperparameter_optimization) you iteratively modify some hyperparameter of the model and retrain it with the same training set and check the performance on the validation set. If the model performs better since the last training, you know that the hyperparameter adjustment was likely in the correct direction so you have a clue how to continue with the tuning. 

The problem with this approach is that there is some information leaking from the validation set to the training process. As the tuning process depends on how the model performs on the validation set, you might end up with a hyperparameter set (and model) that is optimized for *that particular validation set* and not for the generic case. The simplest solution to this problem is to divide your dataset to three partitions: [training, validation and test sets](https://en.wikipedia.org/wiki/Training,_validation,_and_test_sets), tune your model with the training and validation set, and get the final performance metrics with the completely unseen test set. Unfortunately this means even less samples that you can use for training as you would split your dataset for example into 60-20-20 sized train-validation-test splits.

## How can cross-validation help?

What is cross-validation? What is SageMaker? What is AWSStep Functions?


