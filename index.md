---
title: "HAR: Predict Activity"
author: "Ash Chakraborty"
date: "December 26, 2015"
output: 
  html_document: 
    theme: cosmo
    toc: yes
---  

```{r global, ECHO=TRUE, message=FALSE, warning=FALSE, error=FALSE}
library(caret)
train.url <- "https://d396qusza40orc.cloudfront.net/predmachlearn/pml-training.csv"
test.url <- "https://d396qusza40orc.cloudfront.net/predmachlearn/pml-testing.csv"
```  

# OVERVIEW  

This HAR analysis aims to predict the type of activity performed by subjects wearing a wearable computing device. The type of activities fall into 5 classes:   

* A: exactly according to the specification,  
* B: throwing the elbows to the front,  
* C: lifting the dumbbell only halfway,  
* D: lowering the dumbbell only halfway,   
* E: throwing the hips to the front.  

The dataset provided by [this project](http://groupware.les.inf.puc-rio.br/har) will be used to train a few classification models to attempt to determine the likelihood of the activity. The results of each model will be evaluated against a separate cross validation (testing) dataset that has not been included in training the models.  

# DATA PREVIEW  

We begin by reading in the raw data provided from the project. The dataset has 159 potential predictors for the outcome variable "classe".  The "test" set here is the target set where the outcome is unknown and we want to use our best solution to predict on this set. This set will need to eliminate the same variables as the raw dataset.  


```{r read_raw, cache=TRUE, message=FALSE, warning=FALSE, error=FALSE}
raw <- read.csv(train.url, header = T, na.strings = c(""," ","#DIV/0!","NA"))
test <- read.csv(test.url, header = T, na.strings = c(""," ","#DIV/0!","NA"))
dim(raw)
```  

We now explore these potential predictors to evaluate their suitability for inclusion in the classification exercise. We see that the first 5 columns consist of subject ids, names, and activity timestamps. We eliminate these because they aren't likely contributors to the models. Further, since we want to train our models on as complete a dataset as possible, we conduct an exercise to only keep features where at least 75% of the values are _known_.  


```{r eda1, message=FALSE, warning=FALSE, error=FALSE}
raw <- raw[,-c(1:5)]
test <- test[,-c(1:5)]
raw <- raw[, -which(colSums(is.na(raw))/nrow(raw) > 0.75)]
test <- test[, -which(colSums(is.na(test))/nrow(test) > 0.75)]

dim(raw)
```   

Quite a few variables were eliminated because of extremely sparse data.  

## Data Partitioning  

We'll split the raw dataset into the train and test (cross validation, "cv") dataset. This'll ensure that the same features are used in training as well as during validation, while the model is trained **only** on the training set. 

```{r test_train, message=FALSE, warning=FALSE, error=FALSE}
set.seed(31415)
trainInds <- createDataPartition(raw$classe, p=.7, list = F)
train <- raw[trainInds, ] 
cv <- raw[-trainInds, ]
```  

In order to get an idea of the dumbell orientation features with the outcome, let's look at a scatterplot of the Total Measurements with the Classe outcome.  

```{r viz1, echo=FALSE, message=FALSE, warning=FALSE, error=FALSE, fig.width=7, fig.height=9}
library(GGally)
ggpairs(data=train, columns=c(grep(".*(total).*",colnames(train),value = F),ncol(train)), title = "Measurement Totals and Classe", upper = list(continuous='density', combo='box'), lower = list(continuous='points', combo='dot'))
```  

We see that each of the total measurements have small correlations to the Classe outcome. For e.g. while the _totalaccelbelt_ measurement seems to have a lot of data indicating classe A, it's not so well correlated with the other classes.  


# PRE-PROCESSING, BUILDING AND TUNING MODELS  

We now apply various classification algorithms to our training dataset. Each training model will be run in a parallelized fashion with tuning parameters that enable multiple cross fold validation. This will help determine the accuracy of the final model.  

Pre-Processing Note: I pre-process the training set by centering and scaling before training each model. This is to account for the differences in scales between some features in teh dataset. Moreover, features, especially discrete types, that exhibit near zero variability will not inform the analysis. It's therefore prudent to remove these features from the analysis, in addition to centering and scaling the training sets.  


## Model 1: Recursive Partitioning  
We'll start by using a classification tree to try and find the decision points in the dataset.The aim here is to try and find the decision points that best split the data until we arrive at homogenous activities. We use repeated cross validation as our accuracy metric to arrive at average predictions on multiple runs.  


```{r cartTrain, cache=TRUE, message=FALSE, warning=FALSE, error=FALSE}
# enable parallelization 
library(doMC)
registerDoMC(cores=8)
system.time(
        rpart.fit <- train(classe~., data=train, preProcess=c('center','scale','nzv'), 
                           method="rpart",
                           trControl = trainControl(
                                   method='repeatedcv',
                                   number = 10,
                                   repeats = 3,
                                   allowParallel = TRUE
                           ))
)
rpart.fit
```  
We see Recursive Partitioning has a very poor classification accuracy of `r max(rpart.fit$results[,2])*100`% on the training set. Let's now attempt a more time and process intensive random traverse approach.  

## Model 2: Random Forest  
We'll next run a random forest classifier to try and improve the accuracy of the partitioning algorithm. Perhaps randomizing the walk down various trees in teh dataset will offer better insights into the partitioning.  


```{r rfTrain, cache=TRUE, message=FALSE, warning=FALSE, error=FALSE}
library(doMC)
registerDoMC(cores = 8)
system.time(
        rf.fit <- train(classe~., data=train, preProcess=c('center','scale','nzv'),
                        method="rf", 
                        trControl = trainControl(
                                method = "repeatedcv",
                                number = 5,
                                repeats = 5,
                               allowParallel = TRUE
                        )
        )
)
rf.fit
```   
The random forest's accuracy is a big improvement over the partitioning algorithm run earlier. The accuracy behaved thus on upon repeated cross validation:  
```{r rfViz, echo=FALSE, message=FALSE, warning=FALSE, error=FALSE}
plot(rf.fit)
```  

From this, and taking the average across all runs, we get an *expected accuracy* of `r mean(rf.fit$results[,2])*100`% from the random forest algorithm. This gives us an *Expected Out of Sample error rate*: `r  1-mean(rf.fit$results[,2])`%.


## Model 3: Support Vector Machine  
Now, since we have a good chunk of data, with > 10-20 features, a Support Vector Machine classifier might help in determining a good classification boundary between the various outcome classes. The support vector machine improves on logistic regression by using the the Gaussian Kernel; it constructs a set of hyperplanes in a high dimensional space, which can be used for classification. Intuitively, a good separation is achieved by the hyperplane that has the largest distance to the nearest training-data point of any class. The "tuneLength" parameter expands the range of _regularization_ values that the SVM may be run against (0.1 to 1000).  


```{r svmTrain, cache=TRUE, message=FALSE, warning=FALSE, error=FALSE}
library(doMC)
registerDoMC(cores=8)
system.time(
        svm.fit <- train(classe~., data=train,
                         preProcess=c('center','scale','nzv'),
                         method='svmRadial',
                         tuneLength=5,
                         trControl = trainControl(
                                method = "repeatedcv",
                                number = 10,
                                repeats = 5,
                             allowParallel = TRUE
                                )
                         )
        
)
svm.fit
```  

Finally, we see how the SVM behaves with repeated cross fold validation, when compared to its regularization parameter, or the Cost:  

```{r svmViz, echo=FALSE, message=FALSE, warning=FALSE, error=FALSE}
plot(svm.fit)
```  

We see that an *expected accuracy* of `r max(svm.fit$results[,3])*100`% from the SVM is reasonable, at the most optimal regularization parameter. This gives us an Out of Sample error rate that we can expect: `r  1-max(svm.fit$results[,3])`%. 


# PREDICTING and VALIDATING RESULTS  

## Confusion Matrices - Out of Sample Performance  

We'll now run each of the solutions against the untouched test (cross validation) set to generate confusion matrices and generate true out of sample errors for each algorithm.  

```{r confusionMatrix, warning=FALSE, error=FALSE, message=FALSE}
pred.rpart <- predict(rpart.fit, newdata=cv)
pred.rf <- predict(rf.fit, newdata=cv)
pred.svm <- predict(svm.fit, newdata=cv)

 confusionMatrix(pred.rpart, cv$classe) # CART, recursive partitioning
 confusionMatrix(pred.rf, cv$classe) # Random Forest
 confusionMatrix(pred.svm, cv$classe) # Support Vector Machine

```  

We see that the Random Forest algorithm with a `r confusionMatrix(pred.rf, cv$classe)$overall[1]*100`% accuracy on the test set causes the *least* amount of Out of Sample error (`r 1-confusionMatrix(pred.rf, cv$classe)$overall[1]`%), when compared to SVM and Recursive Partitioning. This is within our previously anticipated Out of Sample error. *We therefore select the Random Forest solution (model #2) as the final model.*  

```{r echo=FALSE}
 rf.fit$finalModel
```  


End.  
---









