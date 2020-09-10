---
title: "Practical Machine Learning Exercise Course Project"
author: "Edmond Ho-Yin Lau"
output:
  html_document:
    keep_md: yes
---

Load necessary packages:

```r
library(caret)
```

```
## Loading required package: lattice
```

```
## Loading required package: ggplot2
```

Downloading and reading in the data:


```r
if (!file.exists("pml-training.csv")){
      download.file(url = "https://d396qusza40orc.cloudfront.net/predmachlearn/pml-training.csv", 
                    method = "wget", 
                    destfile = "pml-training.csv")
      }
if (!file.exists("pml-testing.csv")){
      download.file(url = "https://d396qusza40orc.cloudfront.net/predmachlearn/pml-testing.csv", 
                    method = "wget",
                    destfile = "pml-testing.csv")
      }
training <- read.csv("pml-training.csv", na.strings = c("#DIV/0!", "NA"))
test <- read.csv("pml-testing.csv", na.strings = c("#DIV/0!", "NA"))
```

## Data processing
There are some variables in the data which will be dropped:

* X is the row number
* num_window is an increasing count
* The name of the participant will be dropped. It may be helpful in prediction here but the intention is to build a more general model.
* timestamps should theoretically play no role regarding the execution of exercises. Time differences between the observations have been checked and are quite constant except for periodical outliers.


```r
# Drop variables as described above
training <- training[, -(1:7)]
```

Additionally, there may be variables that contain virtually no variation. Those are not helpful in prediction and will be dropped as well. First, columns (features) that contain only NAs will be dropped.


```r
training <- training[, colSums(is.na(training)) != nrow(training)]
nzv <- nearZeroVar(training) # 29
training <- training[, -nzv]; rm(nzv)
```

Furthermore, variables that have virtually only missing values will be dropped, too. Many variables have over 19000 missing values. The number of rows is 19622.


```r
# Which features have many NAs?
NAsummary <- apply(X = training, 2, function(x) sum(is.na(x)))
plot(NAsummary, ylab = "Number of missing values", xlab = "Variables")
```

![](Prediction-Exercise_files/figure-html/unnamed-chunk-5-1.png)<!-- -->

```r
# Drop all that have more than 5000 NAs, those are the ones with practically
# only missing values
training <- training[, -which(NAsummary > 5000)]
```

There are 52 predictors left. The variable "classe" is to be predicted. It represents the correct way of doing an exercise (class A) and several typical mistakes while doing a certain exercise (the other four classes).


```r
str(training$classe)
```

```
##  chr [1:19622] "A" "A" "A" "A" "A" "A" "A" "A" "A" "A" "A" "A" "A" "A" "A" ...
```

#### Partition data
There are 19622 cases in the training data. The data will be split into a training set (70%) and test set(30%). The testing set contains just 20 cases and won't be used in this paper.


```r
inTest <- createDataPartition(y = training$classe, p=0.3, list=FALSE)
test <- training[inTest,]
training <- training[-inTest,]
# Check
dim(training)
```

```
## [1] 13733    53
```

```r
dim(test)
```

```
## [1] 5889   53
```

## Model building
For prediction the Random Forest algorithm will be used. It uses bootstrapped samples to estimate decision trees. At each split, also the variables that are used as predictors are bootstrapped. All trained trees then "vote" for the class that should be predicted. This algorithm is relatively accurate but prone to overfitting. I chose 20 trees to be trained by the randomForest() function because the estimation takes very long otherwise.


```r
# All variables as predictors
modFit <- train(classe ~ ., data = training, method="rf", ntree=20)
modFit
```

```
## Random Forest 
## 
## 13733 samples
##    52 predictor
##     5 classes: 'A', 'B', 'C', 'D', 'E' 
## 
## No pre-processing
## Resampling: Bootstrapped (25 reps) 
## Summary of sample sizes: 13733, 13733, 13733, 13733, 13733, 13733, ... 
## Resampling results across tuning parameters:
## 
##   mtry  Accuracy   Kappa    
##    2    0.9808471  0.9757612
##   27    0.9851645  0.9812272
##   52    0.9740459  0.9671583
## 
## Accuracy was used to select the optimal model using the largest value.
## The final value used for the model was mtry = 27.
```

Additionally, more models will be trained that contain less predictors. Instead of the original predictors n principal components will be used. In the paper that accompanies the data set the authors used only 17 of 96 derived features (p. 3, [Qualitative Activity Recognition of Weight Lifting Exercises](http://groupware.les.inf.puc-rio.br/public/papers/2013.Velloso.QAR-WLE.pdf)). 

Thus, a model with 17 PCs and a model that uses PCs that contain 90% of the variance will be estimated.


```r
modFit17 <- train(classe ~ ., data = training, method="rf", ntree = 20, 
                  preProcess = "pca", pcaComp = 17)
modFit17
```

```
## Random Forest 
## 
## 13733 samples
##    52 predictor
##     5 classes: 'A', 'B', 'C', 'D', 'E' 
## 
## Pre-processing: principal component signal extraction (52), centered
##  (52), scaled (52) 
## Resampling: Bootstrapped (25 reps) 
## Summary of sample sizes: 13733, 13733, 13733, 13733, 13733, 13733, ... 
## Resampling results across tuning parameters:
## 
##   mtry  Accuracy   Kappa    
##    2    0.9364367  0.9195484
##   27    0.9249735  0.9050702
##   52    0.9256946  0.9059688
## 
## Accuracy was used to select the optimal model using the largest value.
## The final value used for the model was mtry = 2.
```

```r
modFit90p <- train(classe ~ ., data = training, method="rf", ntree = 20,
                  preProcess = "pca", thresh = 0.9)
modFit90p
```

```
## Random Forest 
## 
## 13733 samples
##    52 predictor
##     5 classes: 'A', 'B', 'C', 'D', 'E' 
## 
## Pre-processing: principal component signal extraction (52), centered
##  (52), scaled (52) 
## Resampling: Bootstrapped (25 reps) 
## Summary of sample sizes: 13733, 13733, 13733, 13733, 13733, 13733, ... 
## Resampling results across tuning parameters:
## 
##   mtry  Accuracy   Kappa    
##    2    0.9372023  0.9204927
##   27    0.9257340  0.9059874
##   52    0.9260787  0.9064154
## 
## Accuracy was used to select the optimal model using the largest value.
## The final value used for the model was mtry = 2.
```

To sum up, the models achieved the following accuracies:

* All variables: 98.0847139 percent
* 17 PCs: 93.6436732 percent
* 90% variance PCs: 93.7202274 percent

The model with all 52 chosen features has shown the best performance and will be tested using the test set.


```r
pred <- predict(modFit, test)
test$predRight <- pred==test$classe
modtable <- table(pred, test$classe)
# Accuracy
acctest <- sum(diag(modtable)) / nrow(test)
```

The model achieves an accuracy of 0.9882832. This is also to be expected in further out of sample applications since the test set was not used before to judge or evaluate the model. Below is a table that compares predictions and actual classes in the test set. All values on the main diagonal represent correct predictions.


```r
modtable
```

```
##     
## pred    A    B    C    D    E
##    A 1668   10    0    1    0
##    B    2 1120   12    0    2
##    C    2    9 1007   18    2
##    D    0    1    8  946    0
##    E    2    0    0    0 1079
```
