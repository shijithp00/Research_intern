
Author :Shijith

CLear all Junk from Working Environment and set working directory

``` r
library(rstudioapi)
direc=dirname(getActiveDocumentContext()$path)
print(direc)
setwd(direc)
rm(list=ls())
```

#### (Install and)Load all libraries to the environment

------------------------------------------------------------------------

Checks the local R library(ies) to see if the required package(s) is/are installed or not. If the package(s) is/are not installed, then the package(s) will be installed . Then loads all the libraries requred for the current project, that are not already loded to the current working Environment. 'pkgs' - is the list of packaged that needs to be loaded

``` r
install_load_pkgs <- function(pkgs) { 
  # install packages not already available:
  pkgs_miss <- pkgs[which(!pkgs %in% installed.packages()[, 1])]
  if (length(pkgs_miss) > 0) {
    install.packages(pkgs_miss)
  }
  
  if (length(pkgs_miss) == 0) {
    message("\n ...All Packages required were already installed!...loading.... \n")
  }
  # load packages not already loaded:
  attached <- search()
  attached_pkgs <- attached[grepl("package", attached)]
  need_to_attach <- pkgs[which(!pkgs %in% gsub("package:", "", attached_pkgs))]
  
  if (length(need_to_attach) > 0) {
    for (i in 1:length(need_to_attach)) library(need_to_attach[i], character.only = TRUE, quietly = TRUE)
  }
  
  if (length(need_to_attach) == 0) {
    message("\n ...All Packages required were already Loaded!\n")
  }
}
```

#### Function to build all models

------------------------------------------------------------------------

Here we are building 8 models, as discussed and suggested

'**all\_models**' functions will execute model based on the input, Arguments are :

1.  'do.this' - Which Classification to perform \['glm','nb','c50','knn','rf','svm','ab','xgb'\]

| Algorithm | Description                                              |
|-----------|----------------------------------------------------------|
| 'glm'     | Generalized linear model                                 |
| 'nb'      | Naive Bayes Classifier                                   |
| 'c50'     | C5.0 decision trees Classifier                           |
| 'Knn,     | k-nearest neighbors Classifier                           |
| 'rf'      | Random forest Classifier                                 |
| 'svm'     | Support vector machine Classifier                        |
| 'ab,      | Generalized Boosted Regression Modeling (gbm) Classifier |
| 'xgb'     | eXtreme Gradient Boosting Classifier                     |

1.  'trn' - Training dataset.
2.  'test' - Test/Validation Dataset.
3.  'covertype' - target class which is down/up sampled .
4.  'percent' - Percentage of datapoints from the down/up sampled class selected for building the current Model.
5.  'trh2o, - H2o instance of Train Dataset.
6.  'valh2o' - H2o instance of Test/Validation Dataset.

a list '**res**' is created for each model

    {
    res <- list.append(res,
                            list('coverType'=covertype,
                            'percentage of data'=percent,
                            'algorithm' = do.this,
                            'Train_confusion_matrics'= tr_cm,
                            'Test_confusion_matrics'= tes_cm))
    }

'res' will contain in order the

-   class which is down/up sampled
-   Percentage of datapoints from the down/up sampled class selected for building the current Model
-   The Classification algorithm
-   The Train\_confusion\_matrics
-   The Test\_confusion\_matrics

'res' is then returned to the function call

``` r
all_models <- function(do.this,trn=train,test=validate_data,covertype = covTyp,percent=p,valh2o=validation_h2o,trh2o=train_h2o){
  res <- list()
    switch(do.this,
        glm = {
          print("Multinomial Logistic Regression....................")
            model_glm <- h2o.glm(y=53, x=c(1:52),
                        training_frame = trh2o,
                        family = "multinomial")
            
            predictions <- as.data.frame(h2o.predict(model_glm, trh2o))
            tr_cm <- confusionMatrix(predictions$predict,trn$Cover_Type,mode = "everything")
            predictions <- as.data.frame(h2o.predict(model_glm, valh2o))
            tes_cm <- confusionMatrix(predictions$predict,test$Cover_Type,mode = "everything")
            
            res <- list.append(res,
                        list('coverType'=covertype,
                        'percentage of data'=percent,
                        'algorithm' = do.this,
                        'Train_confusion_matrics'= tr_cm,
                        'Test_confusion_matrics'= tes_cm))
            print("Train :")
            print(res[[1]][[4]][[4]][as.numeric(covertype),c(5,6,7)])
            print("Test :")
            print(res[[1]][[5]][[4]][as.numeric(covertype),c(5,6,7)])       
        },
        nb = {
          print("Naive Bayes Classification..........................")
          
          #reconstructing categorical variables from dummy
          trn <- cbind(trn[,c(1:10)],
                       data.frame(Soil_Type = names(trn[,15:52])[max.col(trn[,15:52])], 
                                  WA = names(trn[,11:14])[max.col(trn[,11:14])]), 
                       'Cover_Type'=trn[,53])
          test <- cbind(test[,c(1:10)],
                       data.frame(Soil_Type = names(test[,15:52])[max.col(test[,15:52])], 
                                  WA = names(test[,11:14])[max.col(test[,11:14])]), 
                       'Cover_Type'=test[,53])
          
          htrn <- as.h2o(trn)
          htest <- as.h2o(test)
          
          model_nb = h2o.naiveBayes(x=c(1:(ncol(trn)-1)), y=ncol(trn), training_frame=htrn)
          
          predictions <- as.data.frame(h2o.predict(model_nb, htrn))
            tr_cm <- confusionMatrix(predictions$predict,trn$Cover_Type,mode = "everything")
            predictions <- as.data.frame(h2o.predict(model_nb, htest))
            tes_cm <- confusionMatrix(predictions$predict,test$Cover_Type,mode = "everything")
          

            res <- list.append(res,
                        list('coverType'=covertype,
                        'percentage of data'=percent,
                        'algorithm' = do.this,
                        'Train_confusion_matrics'= tr_cm,
                        'Test_confusion_matrics'= tes_cm))
            print("Train :")
            print(res[[1]][[4]][[4]][as.numeric(covertype),c(5,6,7)])
            print("Test :")
            print(res[[1]][[5]][[4]][as.numeric(covertype),c(5,6,7)])
        },
        c50 = {
          print("Decision tree C50.........................")
            model_c50 <- C5.0(Cover_Type~.,data = trn)
            predictions <- predict(model_c50,trn[,1:(ncol(trn)-1)])
            tr_cm <- confusionMatrix(predictions,trn$Cover_Type,mode = "everything")
            predictions <- predict(model_c50,test[,1:(ncol(test)-1)])
            tes_cm <- confusionMatrix(predictions,test$Cover_Type,mode = "everything")
            res <- list.append(res,
                        list('coverType'=covertype,
                        'percentage of data'=percent,
                        'algorithm' = do.this,
                        'Train_confusion_matrics'= tr_cm,
                        'Test_confusion_matrics'= tes_cm))
            print("Train :")
            print(res[[1]][[4]][[4]][as.numeric(covertype),c(5,6,7)])
            print("Test :")
            print(res[[1]][[5]][[4]][as.numeric(covertype),c(5,6,7)])
        },
        knn = {
          print("K Nearest Neighbour...........................")
          predictions<-knn(trn[,1:(ncol(trn)-1)],trn[,1:(ncol(trn)-1)],trn$Cover_Type, k = 7)
          tr_cm <- confusionMatrix(predictions,trn$Cover_Type,mode = "everything")
          predictions<-knn(trn[,1:(ncol(trn)-1)],test[,1:(ncol(test)-1)],trn$Cover_Type, k = 7)
          tes_cm <- confusionMatrix(predictions,test$Cover_Type,mode = "everything")
            res <- list.append(res,
                        list('coverType'=covertype,
                        'percentage of data'=percent,
                        'algorithm' = do.this,
                        'Train_confusion_matrics'= tr_cm,
                        'Test_confusion_matrics'= tes_cm))
            print("Train :")
            print(res[[1]][[4]][[4]][as.numeric(covertype),c(5,6,7)])
            print("Test :")
            print(res[[1]][[5]][[4]][as.numeric(covertype),c(5,6,7)])
        },
        rf = {
          print("Random Forest..............................")
            model_rf <- h2o.randomForest(y=53, x=c(1:52),
                                 training_frame = trh2o,
                                 ntrees = 50,
                                 max_depth = 5,
                                 seed = 786)
            
            predictions <- as.data.frame(h2o.predict(model_rf, trh2o))
            tr_cm <- confusionMatrix(predictions$predict,trn$Cover_Type,mode = "everything")
            predictions <- as.data.frame(h2o.predict(model_rf, valh2o))
            tes_cm <- confusionMatrix(predictions$predict,test$Cover_Type,mode = "everything")

            res <- list.append(res,
                        list('coverType'=covertype,
                        'percentage of data'=percent,
                        'algorithm' = do.this,
                        'Train_confusion_matrics'= tr_cm,
                        'Test_confusion_matrics'= tes_cm))
            print("Train :")
            print(res[[1]][[4]][[4]][as.numeric(covertype),c(5,6,7)])
            print("Test :")
            print(res[[1]][[5]][[4]][as.numeric(covertype),c(5,6,7)])
        },
        svm = {
          print("Support vector Machine-Polynomial-degree 2.....................")
            model_svm <- svm(Cover_Type ~ .,data = trn, kernel = 'linear',degree = 2)
            predictions <- predict(model_svm,trn[,1:(ncol(trn)-1)])
            tr_cm <- confusionMatrix(predictions,trn$Cover_Type,mode = "everything")
            predictions <- predict(model_svm,test[,1:(ncol(test)-1)])
            tes_cm <- confusionMatrix(predictions,test$Cover_Type,mode = "everything")
            res <- list.append(res,
                        list('coverType'=covertype,
                        'percentage of data'=percent,
                        'algorithm' = do.this,
                        'Train_confusion_matrics'= tr_cm,
                        'Test_confusion_matrics'= tes_cm))
            print("Train :")
            print(res[[1]][[4]][[4]][as.numeric(covertype),c(5,6,7)])
            print("Test :")
            print(res[[1]][[5]][[4]][as.numeric(covertype),c(5,6,7)])
        },
        ab = {
          print("Multicass adaptive boosting..........................")
          model_adab <- h2o.gbm(y=53, x=c(1:52),
                                training_frame = trh2o, 
                            ntrees = 50, 
                            max_depth = 5,
                            seed = 786)

          predictions <- as.data.frame(h2o.predict(model_adab, trh2o))
            tr_cm <- confusionMatrix(predictions$predict,trn$Cover_Type,mode = "everything")
            predictions <- as.data.frame(h2o.predict(model_adab, valh2o))
            tes_cm <- confusionMatrix(predictions$predict,test$Cover_Type,mode = "everything")

            res <- list.append(res,
                        list('coverType'=covertype,
                        'percentage of data'=percent,
                        'algorithm' = do.this,
                        'Train_confusion_matrics'= tr_cm,
                        'Test_confusion_matrics'= tes_cm))
            print("Train :")
            print(res[[1]][[4]][[4]][as.numeric(covertype),c(5,6,7)])
            print("Test :")
            print(res[[1]][[5]][[4]][as.numeric(covertype),c(5,6,7)])
        },
        xgb = {
          print("Extreme Gradient Boosting.................................")
            labels <- trn$Cover_Type 
            ts_label <- test$Cover_Type
            #convert factor to numeric 
            labels <- as.numeric(labels)-1
            ts_label <- as.numeric(ts_label)-1
            #preparing matrix 
            dtrain <- xgb.DMatrix(data = as.matrix(trn[,1:(ncol(train)-1)]),label = as.matrix(labels)) 
            dtest <- xgb.DMatrix(data = as.matrix(test[,1:(ncol(test)-1)]),label=as.matrix(ts_label))

            model_xgb <- xgboost(data = dtrain,
                                 objective = "multi:softmax", 
                                 early_stopping_rounds = 10,
                                 num_class=length(levels(train$Cover_Type)),
                                 nrounds = 50)

            predictions <- predict(model_xgb,dtrain)+1
            predictions <- as.factor(as.character(predictions))
            levels(predictions) <- c(levels(predictions),setdiff(levels(train$Cover_Type),levels(predictions)))
            tr_cm=confusionMatrix(as.factor(predictions),train$Cover_Type,mode = "everything")
            predictions <- predict(model_xgb,dtest)+1
            predictions <- as.factor(as.character(predictions))
            levels(predictions) <- c(levels(predictions),setdiff(levels(validate_data$Cover_Type),levels(predictions)))
            tes_cm=confusionMatrix(as.factor(predictions),test$Cover_Type,mode = "everything")
            res <- list.append(res,
                        list('coverType'=covertype,
                        'percentage of data'=percent,
                        'algorithm' = do.this,
                        'Train_confusion_matrics'= tr_cm,
                        'Test_confusion_matrics'= tes_cm))
            print("Train :")
            print(res[[1]][[4]][[4]][as.numeric(covertype),c(5,6,7)])
            print("Test :")
            print(res[[1]][[5]][[4]][as.numeric(covertype),c(5,6,7)])
        },
        stop("Invalid case!")
        
    )
  return(res)
}
```

#### Function to Read the original datatsse and preprocess

------------------------------------------------------------------------

-   Read the data using 'read.csv()'.
-   Reaname some of the columns to make the column names compact (using 'plyr::rename()').
-   Renaming Factor Levels of 'Soil\_Type' & 'Wilderness\_Area' for ease of use (using 'plyr::mapvalues()').
-   Remove 'Id' from the dataframe.
-   Make a list of the categorical and Numerical columns
-   Standardizing all numerical data (using 'caret::preProcess(numdata,method = c("center", "scale")'))
-   Dummifying the categorical attributes (using 'dummies::dummy.data.frame')
-   create train-test split - 80%train ,20%- validate (using 'caret::createDataPartition')
-   Retun a dataframe which has combined preprocessed data for train and validation
    -   Test and validation can be seperated using the column 'type'

``` r
readAndPreprocess <- function(csvfilename){
    ##***********READ data *********************************************************************
    population_train_data <- read.csv(csvfilename,header = TRUE,na.strings = c("NA",""))
    ## ****************change names of columns*************************************************
    population_train_data <- plyr::rename(population_train_data,
                                          c("Horizontal_Distance_To_Hydrology" = "HDist_Hydro", 
                                            "Vertical_Distance_To_Hydrology" = "VDist_Hydro",
                                            "Horizontal_Distance_To_Roadways"="HDist_Road",
                                            "Horizontal_Distance_To_Fire_Points"="HDist_FirePt"))
    ## ************************Renaming Factor Levels******************************************
    newLabels<-c('1','2','3','4','5','6','8','9','10','11','12','13','14','16','17','18','19','20','21','22',
                 '23','24','25','26','27','28','29','30','31','32','33','34','35','36','37','38','39','40')
    oldColumns <- c("Soil_Type1","Soil_Type2","Soil_Type3","Soil_Type4","Soil_Type5","Soil_Type6",
                    "Soil_Type8","Soil_Type9","Soil_Type10","Soil_Type11","Soil_Type12","Soil_Type13",
                    "Soil_Type14","Soil_Type16", "Soil_Type17","Soil_Type18","Soil_Type19","Soil_Type20",
                    "Soil_Type21", "Soil_Type22","Soil_Type23","Soil_Type24","Soil_Type25","Soil_Type26",
                    "Soil_Type27","Soil_Type28","Soil_Type29","Soil_Type30","Soil_Type31","Soil_Type32",
                    "Soil_Type33","Soil_Type34","Soil_Type35","Soil_Type36","Soil_Type37","Soil_Type38",
                    "Soil_Type39","Soil_Type40")

    population_train_data$Soil_Type <- plyr::mapvalues(population_train_data$Soil_Type,
                                                       from = oldColumns, to = newLabels)
    newLabels <- c('1','2','3','4')
    oldColumns <- c("Wilderness_Area1","Wilderness_Area2","Wilderness_Area3","Wilderness_Area4")
    population_train_data$Wilderness_Area <- plyr::mapvalues(population_train_data$Wilderness_Area,
                                                             from = oldColumns, to = newLabels)
    ## *********factorising Cover_Type*********************************************************
    population_train_data$Cover_Type <- as.factor(population_train_data$Cover_Type)
    ## *********REmove ID**********************************************************************
    population_train_data$Id <- NULL
    ## ************vector of categorical, numerical and response attribute*********************
    y <- c("Cover_Type")
    x <- colnames(population_train_data)[!(colnames(population_train_data)%in%y)]
    cat_Attr <- c("Wilderness_Area","Soil_Type")
    num_Attr <- colnames(population_train_data)[!(colnames(population_train_data)%in%c(cat_Attr,y))]
    ## *************Standardizing numerical data************************************************
    train_std_obj <- caret::preProcess(population_train_data[,(names(population_train_data) %in% num_Attr)],
                                       method =  c("center", "scale"))
    population_train_data <- predict(train_std_obj,population_train_data)
    ## *************Dummifying data ************************************************************
    population_train_data=dummy.data.frame(population_train_data,cat_Attr)
    ## ************create train-test split - 80%train ,20%- validate****************************
    set.seed(786)
    train_rows <- caret::createDataPartition(population_train_data$Cover_Type, p = 0.8, list = F)
    train_data <- population_train_data[train_rows, ]
    validate_data <- population_train_data[-train_rows, ]
    rm(train_rows)
    #********************Adding a new column to differenciate train_data and Test_data**********
    train_data["type"]= "Train"
    validate_data["type"] = "Test"
    #********************Combining the Train and Validation Data********************************
    combined_data <- rbind(train_data, validate_data)
    combined_data$type <- as.factor(combined_data$type)
    return(combined_data)
}
```

#### Function to first cluster tehn select samples and final build the model

------------------------------------------------------------------------

Different Steps followed in this function are as follows:

-   All data elements which belong to the class selected for sampling are saved to a 'selected\_class' dataframe
-   Rest of the data elements are saved to 'other\_class' dataframe
-   Best 'k' for k-means clustering is found from vegan::cascadeKM
-   find 'k' clusters for selected\_class and label each dataelemnts with the cluster name
-   num will give the total number of data elements to be selected for the given percentage
    -   num is calculated by using the formula below: <a href="https://www.codecogs.com/eqnedit.php?latex=\fn_cm&space;\left(\frac{T*y}{1-y}\right)" target="_blank"><img src="https://latex.codecogs.com/gif.latex?\fn_cm&space;\left(\frac{T*y}{1-y}\right)" title="\left(\frac{T*y}{1-y}\right)" /></a>
-   if (num &gt; total elemnts in the selected\_class), data elemnts has to be selected using 'stratified sampling with replacement' technique, maintaining the proportion of different clusters. (our dataset doesnt need upsampling since each class has about 16% elements) (DescTools::Strata is used for sampling)
-   if(num &gt; total elemnts in the selected class), data has to be downsampled. Here also stratified selection is used maintaining the proportion of different clusters. (caret::createDataPartition is used for sampling)
-   combine the selected elements and 'other\_class' dataframe which will make the final 'train' dataframe.
-   Iterete with each selected (algoritm, class , percentege of data) to build classifiers and get the final output
-   This function return is a list which contains confusion matrics details for validation and train (also contains other detail such as algo, percentahe, class, etc)

``` r
cluster_sample_buildModel <- function(classes,perc_of_data,algo,train_data,validate_data){
  results <- list()
  validation_h2o <- as.h2o(validate_data)
  for(targetclass in classes){
    selected_class <- train_data[train_data$Cover_Type ==targetclass,]
    selected_class$Cover_Type <- NULL
    other_class <- train_data[train_data$Cover_Type !=targetclass,]
    # finding out the optimal clusters (k) and ploting it
    fit <- cascadeKM(selected_class, 1, 10, iter = 100)
    plot(fit, sortg = TRUE, grpmts.plot = TRUE)
    calinski.best <- as.numeric(which.max(fit$results[2,]))
    print(paste0("Calinski criterion opt. no. of clusters for |class-",targetclass,"| is : ", calinski.best))
    #clustering each classes based on optimal clusters (calinski.best)
    kmeans_fit <- kmeans(selected_class, calinski.best)
    #plot the clusters
    plotcluster(selected_class, kmeans_fit$cluster)
    selected_class <- data.frame(selected_class,'Cover_Type'=targetclass,'cluster'=kmeans_fit$cluster)
    print(prop.table(table(kmeans_fit$cluster)))
    # num calculated from the formula discussed in class (T*y/1-y)
    # Here if 100% data of the classs is selected take the entire row from that class 
    # otherwise calculate 'num' using the formula 
    for(p in perc_of_data){
        covTyp <- targetclass
        if(p==100){
          num=nrow(selected_class)
        }else{
          num=round((dim(other_class)[[1]]*(p/100))/(1-(p/100)),0)
      }
      print(paste("nrows otherclass :",nrow(other_class)," p: ",p," num :",num))
      # stratified sampling on cluster percentage is num/nrow(selected_class) eg: for .1%, perc=10/1728
      # then select only the first n-1 columns ie, excluding cluster column
      # for upsampling using stratified random sampling with replacement.
      if(num > table(train_data$Cover_Type)[[targetclass]]){
        nums=c()
        for (clus in seq(1:dim(prop.table(table(kmeans_fit$cluster))))) {
          nums[clus] =  round(prop.table(table(kmeans_fit$cluster))[[clus]]*num,0)
        }
        Stratified_upsamples <- Strata(selected_class,c("cluster"),size=nums, method="srswr")
        rownames(Stratified_upsamples) <- make.names(Stratified_upsamples$id,unique=TRUE)
        Stratified_upsamples <- Stratified_upsamples[,c(2:(ncol(Stratified_upsamples)-3))]
        train <- rbind(other_class,Stratified_upsamples)
        rm(nums,Stratified_upsamples)
      } else{
        train <- rbind(other_class,
                       selected_class[createDataPartition(selected_class$cluster,
                                                          p = num/nrow(selected_class), list = F)
                                      ,][,1:(ncol(selected_class)-1)])
      }
      #shuffle data row wise
      train <- train[sample(nrow(train)),]
      print(paste("nrows train :",nrow(train)))
      train_h2o <- as.h2o(train)
      #build all models for 'p' Percentage 'targetclass' class
      for(algorithm in algo){
        res=all_models(do.this = algorithm,trn=train,test=validate_data,
                       covertype = covTyp,percent=p,
                       valh2o=validation_h2o,trh2o=train_h2o)
        results <- list.append(results,res)
      }
    }
  }
  return(results)
}
```

#### Function to Retrieve data from the list

------------------------------------------------------------------------

This function Will save the following details to different 'csv' sheets as explained below.

1.  '**Final.csv**' will contain.

-   the target class which is down/up sampled.
-   Percentage of the selected class wrt thw whole target.
-   The classifier used for training and validation.
-   *Precision* of the down/up sampled class while predicting on *Train* data.
-   *Recall* of the down/up sampled class while predicting on *Train* data.
-   *Precision* of the down/up sampled class while predicting on *validation* data.
-   *Recall* of the down/up sampled class while predicting on *validation* data.
-   *F1 score* of the down/up sampled class while predicting on *Train* data.
-   *F1 score* of the down/up sampled class while predicting on *Test* data.
-   *S score* of the down/up sampled class (calculation as below): <a href="https://www.codecogs.com/eqnedit.php?latex=\fn_cm&space;S=\frac{F1_{Train}&space;&plus;&space;F1_{Train}}{2(1&plus;(F1_{Train}&space;&plus;&space;F1_{Train}))}&space;,&space;{0<=&space;S&space;<=&space;1}" target="_blank"><img src="https://latex.codecogs.com/gif.latex?\fn_cm&space;S=\frac{F1_{Train}&space;&plus;&space;F1_{Train}}{2(1&plus;(F1_{Train}&space;&plus;&space;F1_{Train}))}&space;,&space;{0<=&space;S&space;<=&space;1}" title="S=\frac{F1_{Train} + F1_{Train}}{2(1+(F1_{Train} + F1_{Train}))} , {0<= S <= 1}" /></a>

-   Overall *F1 score* while predicting on *Train* data
-   Overall *F1 score* while predicting on *Test* data
-   *S score* all the classes (overall, calculation same as above)

1.  '**AUSC\_x\_to\_x\_Perc.csv**' three csv files ((0 to 2%),(2.5 to 10%),(0 to 10%))

-   Area under the S\_Score curve is claculated using sympsons 1/3rd rule <a href="https://www.codecogs.com/eqnedit.php?latex=\fn_jvn&space;AUSC=\frac{0.5}{3}\left[y_0&plus;y_{n&plus;4}*\sum&space;\left(y_1,y_3...y_{n-1}\right)&plus;2*\sum&space;\left(y_2,y_4...y_{n-2}\right)\right]" target="_blank"><img src="https://latex.codecogs.com/gif.latex?\fn_jvn&space;AUSC=\frac{0.5}{3}\left[y_0&plus;y_{n&plus;4}*\sum&space;\left(y_1,y_3...y_{n-1}\right)&plus;2*\sum&space;\left(y_2,y_4...y_{n-2}\right)\right]" title="AUSC=\frac{0.5}{3}\left[y_0+y_{n+4}*\sum \left(y_1,y_3...y_{n-1}\right)+2*\sum \left(y_2,y_4...y_{n-2}\right)\right]" /></a>

<a href="https://www.codecogs.com/eqnedit.php?latex=0<=AUSC<=10" target="_blank"><img src="https://latex.codecogs.com/gif.latex?0<=AUSC<=10" title="0<=AUSC<=10" /></a>

    {
      # How index for score is selected(which elements to be selected )
      # 2*[elements excluding first and last element which are when divided by 1, reminder will be 0] and 
      # 4*[elements excluding first and last element which are when divided by 1, reminder will not be 0] 
      # Code Snippet for testing as below
      j=1
      AUSC_perc <- data.frame(combn(c(0,2.1,10.1),2))
      all_perc <- c(100,.1,seq(.5,10,.5))
      while(j <= length(AUSC_perc)){
          a=all_perc[which(all_perc>AUSC_perc[j][1,]&all_perc<=AUSC_perc[j][2,])]
          cat("a:",match(a,a),"\n")
          cat("no_0.5_a:",a[-c(1,length(a))][which(a[-c(1,length(a))]%%1==0)],"\n")
          cat("index of no_0.5_a:",c(match(a[-c(1,length(a))][which(a[-c(1,length(a))]%%1==0)],a)),"\n")
          cat("0.5_a:",a[-c(1,length(a))][which(a[-c(1,length(a))]%%1!=0)],'\n')
          cat("index of 0.5_a:",c(match(a[-c(1,length(a))][which(a[-c(1,length(a))]%%1!=0)],a)),'\n')
          cat("\n\n")
          j=j+1
      }
    }

``` r
retrieve_data <- function(results_test,flag){
  len_of_results <-length(results_test)
  class_0f_target <- rep(NA,len_of_results)
  Percentage_of_Class <- rep(NaN,len_of_results)
  algorith_used <-rep(NA,len_of_results)
  precision_train<- rep(NaN,len_of_results)
  recall_train <- rep(NaN,len_of_results)
  f1_train <- rep(NaN,len_of_results)
  precision_test<- rep(NaN,len_of_results)
  recall_test <- rep(NaN,len_of_results)
  f1_test <- rep(NaN,len_of_results)
  Train_F1_Overall <- rep(NaN,len_of_results)
  Test_F1_Overall <- rep(NaN,len_of_results)
  S_overall <- rep(NaN,len_of_results)
  S_PerClass <- rep(NaN,len_of_results)
  i <- 1
  while(i <= len_of_results){
    #print(paste("class : ",results_test[[i]][[1]][[1]]," Percentage : ",results_test[[i]][[1]][[2]]," Algorithm : ",results_test[[i]][[1]][[3]]))
    
    all_train_f1 <- results_test[[i]][[1]][[4]][[4]][,7]
    all_train_f1[is.na(all_train_f1)] <- 0
    all_test_f1 <- results_test[[i]][[1]][[5]][[4]][,7]
    all_test_f1[is.na(all_test_f1)]<- 0
    
    class_0f_target[[i]]        <- results_test[[i]][[1]][[1]]
    Percentage_of_Class[[i]]    <- results_test[[i]][[1]][[2]]
    algorith_used[[i]]          <- results_test[[i]][[1]][[3]]
    
    precision_train[[i]]        <- results_test[[i]][[1]][[4]][[4]][as.integer(as.character(class_0f_target[[i]])),5]
    recall_train[[i]]           <- results_test[[i]][[1]][[4]][[4]][as.integer(as.character(class_0f_target[[i]])),6]
    f1_train[[i]]               <- results_test[[i]][[1]][[4]][[4]][as.integer(as.character(class_0f_target[[i]])),7]
    precision_test[[i]]         <- results_test[[i]][[1]][[5]][[4]][as.integer(as.character(class_0f_target[[i]])),5]
    recall_test[[i]]            <- results_test[[i]][[1]][[5]][[4]][as.integer(as.character(class_0f_target[[i]])),6]
    f1_test[[i]]                <- results_test[[i]][[1]][[5]][[4]][as.integer(as.character(class_0f_target[[i]])),7]
    
    Train_F1_Overall[[i]]       <- (sum(all_train_f1 * colSums(results_test[[i]][[1]][[4]][[2]])))/(sum(results_test[[i]][[1]][[4]][[2]]))
    Test_F1_Overall[[i]]        <- sum(all_test_f1 * colSums(results_test[[i]][[1]][[5]][[2]])) /(sum(results_test[[i]][[1]][[5]][[2]]))
    S_overall[[i]]              <- (Train_F1_Overall[[i]] + Test_F1_Overall[[i]])/(2*(1+(abs(Train_F1_Overall[[i]] - Test_F1_Overall[[i]]))))
    
    a <- all_train_f1[as.integer(as.character(class_0f_target[[i]]))]
    b <- all_test_f1[as.integer(as.character(class_0f_target[[i]]))]
    S_PerClass[[i]]             <- (a + b)/(2*(1+(abs(a-b))))
    
    #print(paste("a:",round(a,3)," f1_train[[i]]:"," b:",round(b,3),round(f1_train[[i]],3)," f1_test[[i]]:",round(f1_test[[i]],3) ))
    i=i+1
  }
  result_df <<- data.frame("Class" = class_0f_target,
                          "Percentage" = Percentage_of_Class,
                          "Algorith" = algorith_used,
                          "Train Precision" = precision_train,
                          "Train Recall" = recall_train,
                          "Test Precision" = precision_test,
                          "Test Recall" = recall_test,
                          "Train F1-class" = f1_train,
                          "Test F1-class" = f1_test,
                          "Score-class" = S_PerClass,
                          "Train F1-overall" = Train_F1_Overall,
                          "Test F1-overall" = Test_F1_Overall,
                          "Score-overall" = S_overall
                          )
  write.csv(x = result_df,file = 'Final.csv',row.names = FALSE)
  #------ AUSC--------------------------------------------------------------------
  if(flag==1){  
    classes <- levels(as.factor(result_df$Class))
    algos <- levels(as.factor(result_df$Algorith))
    AUSC_perc <- data.frame(combn(c(0,2.1,10.1),2))
    all_perc <- c(100,.1,seq(.5,10,.5))
    AUSC <- data.frame(matrix(NA, nrow = length(algos)*length(classes), ncol = 4))
    colnames(AUSC) <- c('Class','Algorithm','AUSC_class','AUSC_overall')
    i=1
    j=1
    while(j <= length(AUSC_perc)){
      a=all_perc[which(all_perc>AUSC_perc[j][1,]&all_perc<=AUSC_perc[j][2,])]
      for(targetclass in classes){
        for (algorithms in algos) {
          subset_df <- subset(result_df, Percentage > AUSC_perc[j][1,] & 
                                Percentage <= AUSC_perc[j][2,] & Class == targetclass & 
                                Algorith == algorithms)[,c(1:3,10,13)]
          AUSC[i,1] <- targetclass
          AUSC[i,2] <- algorithms
          AUSC[i,3] <-(0.5/3)*(subset_df$Score.class[1] + 
                                 subset_df$Score.class[length(subset_df$Class)] + 
                                 (2*sum(subset_df$Score.class[c(match(a[-c(1,length(a))]
                                                                      [which(a[-c(1,length(a))]%%1==0)],a))]))+
                                 (4*sum(subset_df$Score.class[c(match(a[-c(1,length(a))]
                                                                      [which(a[-c(1,length(a))]%%1!=0)],a))])))
          AUSC[i,4]<-((0.5/3)*(subset_df$Score.overall[1]+
                                 subset_df$Score.overall[length(subset_df$Class)]+
                                 (2*sum(subset_df$Score.overall[c(match(a[-c(1,length(a))]
                                                                        [which(a[-c(1,length(a))]%%1==0)],a))]))+
                                 (4*sum(subset_df$Score.overall[c(match(a[-c(1,length(a))]
                                                                        [which(a[-c(1,length(a))]%%1!=0)],a))]))))
          i=i+1
        }
      }
      f_name <- print(paste0("AUSC_",round(AUSC_perc[j][1,],0),"_to_",round(AUSC_perc[j][2,],0),"_Percent.csv"))
      write.csv(x = AUSC,file = f_name,row.names = FALSE)
      j=j+1
    }
  }
}
```

#### Function to execute the code with newly selected parameters

------------------------------------------------------------------------

This Function will open a 'shiny widget' from where we can select

-   Classifiers
-   The classes to up/down sample
-   Percentage of data from the 'subsampled' class to be taken

Further execution of the program is based on selected values from the widget.

``` r
select_to_execute <- function(){
  ui <-fluidPage(
    titlePanel("Check the boxes to select"),
    sidebarLayout(
      sidebarPanel(wellPanel(h3("Save"), actionButton("save", "Save and Exit"))),
      mainPanel(fluidRow(
        column(4,checkboxGroupInput(inputId = "check1c", 
                                    label = "Selec Algorithm", c('glm','nb','c50','knn','rf','svm','ab','xgb'))),
        column(4,checkboxGroupInput(inputId = "check2c", 
                                    label = "Selec Class", c("1","2","3","4","5","6","7"))),
        column(4,checkboxGroupInput(inputId = "check3c", 
                                    label = "Selec Percentage", c(100,.1,seq(.5,10,.5))))
      ))
    )
  )
  server <- function(input, output, session) {
    ## Save 
    observeEvent(input$save, {
      finalDF <- list('algo'=input$check1c,'class'=input$check2c,'perc'=input$check3c)
      #saveRDS(finalDF, file=file.path(outdir, sprintf("%s.rds", outfilename)))
      print(finalDF)
      
      stopApp(finalDF)
    })
  }
  ## run app 
  finalDF <- runApp(list(ui=ui, server=server))
  print(finalDF)
}
```

#### Function to Plot the F1 scores Against the % of saubsampled class data

------------------------------------------------------------------------

Interactive plot using ggplot and Plotly

``` r
plot_F1 <- function(){
  plt <- htmltools::tagList()
  plts <- 1
  classes <- c("1","2","3","4","5","6","7")
  #classes <- c("1")
  for(targetclass in classes){
    
    #For TEST
    g=ggplotly(ggplot(subset(result_df, Percentage < 11 & Class == targetclass), 
                            aes(x=Percentage,y=Test.F1.class,colour=Algorith)) + 
                 geom_point() + 
                 geom_line() + 
                 scale_x_continuous("Sample Precentage",breaks = c(.1,seq(.5,10,.5))) +   
                 scale_y_continuous("F1 Scores",breaks = c(seq(0,1,.1))) +  
                 ggtitle(paste0("CLASS: CoverType -", targetclass," :Test")) + 
                 theme(plot.title = element_text(family = "Trebuchet MS", color="#666666", face="bold", size=19, hjust=0.5),
                       axis.title.x = element_text(family = "Trebuchet MS", color="#666666", face="bold", size=15),
                       axis.title.y = element_text(family = "Trebuchet MS", color="#666666", face="bold", size=15))
               )
    plt[[plts]] <- as_widget(g)
    plts <- plts + 1
    
    
    # FOR TRAIN
    g=ggplotly(ggplot(subset(result_df, Percentage < 11 & Class == targetclass), 
                            aes(x=Percentage,y=Train.F1.class,colour=Algorith)) + 
                 geom_point() + 
                 geom_line() + 
                 scale_x_continuous("Sample Precentage",breaks = c(.1,seq(.5,10,.5))) +   
                 scale_y_continuous("F1 Scores",breaks = c(seq(0,1,.1))) +
                 ggtitle(paste0("CLASS: CoverType -", targetclass," :Train")) +
                 theme(plot.title = element_text(family = "Trebuchet MS", color="#666666", face="bold", size=19, hjust=0.5),
                       axis.title.x = element_text(family = "Trebuchet MS", color="#666666", face="bold", size=15),
                       axis.title.y = element_text(family = "Trebuchet MS", color="#666666", face="bold", size=15))
                 
              ) 
    plt[[plts]] <- as_widget(g)
    plts <- plts + 1
    
  } 
  return(plt)
}
```

#### The main part of the program

------------------------------------------------------------------------

Executing this block, will ask to select from two options:

-   **Option1:** Load details from Pre-executed data - All the details (Per class/per algorithm/Per percentage) are saved to a '.Rda' file named 'results.rda'.For this option to work properly, 'results.rda' has to be saved to './Pre-executed/' (folder named Pre-executed inside the present working directory). This will create three csv files.
-   Final.csv , which will contain all the Recall, Precission, F1 and S-Score for per class and Overall class
-   **Option2:** An HTML Widget will open where we can select the classifier the classes and the percentage for which the program has to be executed. The AUSC Has to be calculated Amnually fro excel sheet.
-   ***warning*** At least one value fro the widget has to be selected
-   ***warning*** 'mytrain.csv' should be present in the current working directory

``` r
# installing the pacages required if already not present
pkgs <- c('caret','cluster','e1071','rlist','vegan','DescTools','C50','class','dummies',
          'fpc','plyr','xgboost','dplyr','h2o','ggplot2','plotly','shiny','devtools','rstudioapi')
install_load_pkgs(pkgs)
# prompt user to selct an option (explained above)
to.do <- menu(c('Load details from Pre-executed data', 'New execution'), 
              graphics = FALSE, title = "Please select (Enter number):")
if(to.do == 1){
  if("results_test" %in% ls()){
    retrieve_data(results_test)
    print(plot_F1())
  } else{
    load("./Pre-executed/results.rda")
    retrieve_data(results_test,flag =to.do)
    print(plot_F1())
  }
} else{
  new_selection <-select_to_execute()
  algo <- new_selection$algo
  perc_of_data <- as.numeric(new_selection$perc)
  classes <- new_selection$class
  local_h2o <- h2o.init(nthreads = -1, min_mem_size = "3g")
  data <- readAndPreprocess("mytrain.csv")
  results_after_final_submission <- cluster_sample_buildModel(classes,perc_of_data,algo,
                                                              train_data = data[!(data$type %in% c("Train")),(1:(ncol(data)-1))],
                                                              validate_data = data[!(data$type %in% c("Test")),(1:(ncol(data)-1))])
  #save(results_after_final_submission,file = "results_after_final_submission.rda")
  retrieve_data(results_after_final_submission,flag =to.do)
  h2o.shutdown(prompt = FALSE)
}
```
