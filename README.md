# Practical Machine Learning - Prediction Assignment Writeup
========================================================

This document describe the analysis for the prediction assignment of the practical machine learning course.

The first part is the declaration of the package which will be used. In addition to caret & randomForest already seen on the course, I used Hmisc to help me on the data analysis phases & foreach & doParallel to decrease the random forrest processing time by parallelising the operation.

Note : To be reproductible, I also set the seed value.

### Background Introduction

These are the files produced during a homework assignment of Coursera's MOOC Practical Machine Learning from Johns Hopkins University.

"Using devices such as Jawbone Up, Nike FuelBand, and Fitbit it is now possible to collect a large amount of data about personal activity relatively inexpensively. These type of devices are part of the quantified self movement ??? a group of enthusiasts who take measurements about themselves regularly to improve their health, to find patterns in their behavior, or because they are tech geeks. One thing that people regularly do is quantify how much of a particular activity they do, but they rarely quantify how well they do it. In this project, your goal will be to use data from accelerometers on the belt, forearm, arm, and dumbell of 6 participants. They were asked to perform barbell lifts correctly and incorrectly in 5 different ways. More information is available from the website here: http://groupware.les.inf.puc-rio.br/har (see the section on the Weight Lifting Exercise Dataset). "

### Data Sources

The training data for this project are available here: 

https://d396qusza40orc.cloudfront.net/predmachlearn/pml-training.csv

The test data are available here: 

https://d396qusza40orc.cloudfront.net/predmachlearn/pml-testing.csv

The data for this project comes from this original source: http://groupware.les.inf.puc-rio.br/har. If you use the document you create for this class for any purpose please cite them as they have been very generous in allowing their data to be used for this kind of assignment. 

### Project Results

The goal of your project is to predict the manner in which they did the exercise. This is the "classe" variable in the training set. You may use any of the other variables to predict with. You should create a report describing how you built your model, how you used cross validation, what you think the expected out of sample error is, and why you made the choices you did. You will also use your prediction model to predict 20 different test cases. 

1. Your submission should consist of a link to a Github repo with your R markdown and compiled HTML file describing your analysis. Please constrain the text of the writeup to < 2000 words and the number of figures to be less than 5. It will make it easier for the graders if you submit a repo with a gh-pages branch so the HTML page can be viewed online (and you always want to make it easy on graders :-).
2. You should also apply your machine learning algorithm to the 20 test cases available in the test data above. Please submit your predictions in appropriate format to the programming assignment for automated grading. See the programming assignment for additional details. 

### Reproduceablity

In order to reproduce the same results, you need a certain set of packages, as well as setting a pseudo random seed equal to the one I used in the code. To install, for instance, the caret package in R, run this command: install.packages("caret")

The following libraries are used in the project, please install if you want to run the same code on your working environment.
```{r}
library(caret)
library(rpart)
library(rpart.plot)
library(RColorBrewer)
library(rattle)
library(randomForest)
```

Finally, load the same seed with the following line of code:
```{r}
set.seed(12345)
```

### Retrieve the data

The training data set can be found on the following URL:

```{r}
trainUrl <- "http://d396qusza40orc.cloudfront.net/predmachlearn/pml-training.csv"
```

The testing data set can be found on the following URL:
```{r}
testUrl <- "http://d396qusza40orc.cloudfront.net/predmachlearn/pml-testing.csv"
```

The first step is to load the csv file data to the memory to analyze type & completion rate of the data

Load data to the memory
```{r}
training <- read.csv(url(trainUrl), na.strings=c("NA","#DIV/0!",""))
testing <- read.csv(url(testUrl), na.strings=c("NA","#DIV/0!",""))
```

### Partioning the training data set into two data sets, 60% for myTraining, 40% for myTesting:

```{r}
inTrain <- createDataPartition(y=training$classe, p=0.6, list=FALSE)
myTraining <- training[inTrain, ]; myTesting <- training[-inTrain, ]
dim(myTraining); dim(myTesting)
```

### Cleaning the data

The following transformations were used for cleaning the data:

Transformation 1: Cleaning NearZeroVariance Variables
-----------------------------------------------------
Run this code to view possible NZV Variables:
```{r}
myDataNZV <- nearZeroVar(myTraining, saveMetrics=TRUE)
```

Run this code to create another subset without NZV variables:

```{r}
myNZVvars <- names(myTraining) %in% c("new_window", "kurtosis_roll_belt", "kurtosis_picth_belt",
"kurtosis_yaw_belt", "skewness_roll_belt", "skewness_roll_belt.1", "skewness_yaw_belt",
"max_yaw_belt", "min_yaw_belt", "amplitude_yaw_belt", "avg_roll_arm", "stddev_roll_arm",
"var_roll_arm", "avg_pitch_arm", "stddev_pitch_arm", "var_pitch_arm", "avg_yaw_arm",
"stddev_yaw_arm", "var_yaw_arm", "kurtosis_roll_arm", "kurtosis_picth_arm",
"kurtosis_yaw_arm", "skewness_roll_arm", "skewness_pitch_arm", "skewness_yaw_arm",
"max_roll_arm", "min_roll_arm", "min_pitch_arm", "amplitude_roll_arm", "amplitude_pitch_arm",
"kurtosis_roll_dumbbell", "kurtosis_picth_dumbbell", "kurtosis_yaw_dumbbell", "skewness_roll_dumbbell",
"skewness_pitch_dumbbell", "skewness_yaw_dumbbell", "max_yaw_dumbbell", "min_yaw_dumbbell",
"amplitude_yaw_dumbbell", "kurtosis_roll_forearm", "kurtosis_picth_forearm", "kurtosis_yaw_forearm",
"skewness_roll_forearm", "skewness_pitch_forearm", "skewness_yaw_forearm", "max_roll_forearm",
"max_yaw_forearm", "min_roll_forearm", "min_yaw_forearm", "amplitude_roll_forearm",
"amplitude_yaw_forearm", "avg_roll_forearm", "stddev_roll_forearm", "var_roll_forearm",
"avg_pitch_forearm", "stddev_pitch_forearm", "var_pitch_forearm", "avg_yaw_forearm",
"stddev_yaw_forearm", "var_yaw_forearm")
myTraining <- myTraining[!myNZVvars]
#To check the new N?? of observations
dim(myTraining)
```

Transformation 2: Remvoing first column of Dataset - ID
------------------------------------------------------
Removing first ID variable so that it does not interfer with Machine Learning Algorithms:

```{r}
myTraining <- myTraining[c(-1)]
```

Transformation 3: Cleaning Variables with too many NAs.
------------------------------------------------------
For Variables that have more than a 60% threshold of NA's, just leave them out:
```{r}

trainingV3 <- myTraining #creating another subset to iterate in loop
for(i in 1:length(myTraining)) { #for every column in the training dataset
        if( sum( is.na( myTraining[, i] ) ) /nrow(myTraining) >= .6 ) { #if n?? NAs > 60% of total observations
  	for(j in 1:length(trainingV3)) {
			if( length( grep(names(myTraining[i]), names(trainingV3)[j]) ) ==1)  { #if the columns are the same:
				trainingV3 <- trainingV3[ , -j] #Remove that column
			}	
		} 
	}
}

# Dimensions of trainingv3
dim(trainingV3)

# Reset myTraining to above data trainingv3
myTraining <- trainingV3
rm(trainingV3)
```

Now let us do the exact same 3 transformations but for myTesting and testing data sets.

```{r}
clean1 <- colnames(myTraining)
clean2 <- colnames(myTraining[, -58]) 
myTesting <- myTesting[clean1]
testing <- testing[clean2]

#To check the new N?? of observations
dim(myTesting)

#To check the new N?? of observations
dim(testing)
```

To ensure proper functioning of Decision Trees and RandomForest Algorithm with the Test data set, we need to coerce the data into the same type.

```{r}
for (i in 1:length(testing)) 
{
  for(j in 1:length(myTraining))
    {
      if( length( grep(names(myTraining[i]), names(testing)[j]) ) ==1)  
        {
          class(testing[j]) <- class(myTraining[i])
        }      
	  }      
}

# To make sure Coertion is really working:
testing <- rbind(myTraining[2, -58] , testing)
testing <- testing[-1,]
```

## Using Machine Learning algorithms for prediction: Decision Tree

```{r}
modFitA1 <- rpart(classe ~ ., data=myTraining, method="class")
```

To view the decision tree with fancy run this command:

```{r}
fancyRpartPlot(modFitA1)
```

Predicting:
```{r}
predictionsA1 <- predict(modFitA1, myTesting, type = "class")
```

To evaluate the model we will use the confusionmatrix method and we will focus on accuracy, sensitivity & specificity metrics :

Test Results:
```{r}
confusionMatrix(predictionsA1, myTesting$class)
```

### Using Machine Learning algorithms for prediction: Random Forests

```{r}
modFitB1 <- randomForest(classe ~. , data=myTraining)

```

Predicting:
```{r}
predictionsB1 <- predict(modFitB1, myTesting, type = "class")
```

Test Results:
```{r}
confusionMatrix(predictionsB1, myTesting$classe)
```

As expected, Random Forests yielded better Results compared to other algorithms. 

For Random Forests is, which yielded a much better prediction:

Predicting:
```{r}
predictionsB2 <- predict(modFitB1, testing, type = "class")
```

### Function to generate files with predictions to submit for assignment part II
----------------------------------------------------------------------------
```{r}
pml_write_files = function(x){
  n = length(x)
  for(i in 1:n){
    filename = paste0("problem_id_",i,".txt")
    write.table(x[i],file=filename,quote=FALSE,row.names=FALSE,col.names=FALSE)
  }
}

pml_write_files(answers)
```

It seems also prediction values are good because it scores 100% (20/20) on the course Project Submission Assignment (the 20 values to predict).
