# Energy-efficiency
The energy analysis is performed using 12 different building shapes simulated in Ecotect. The buildings differ with respect to the glazing area, the glazing area distribution, and the orientation, amongst other parameters. We simulate various settings as functions of the afore-mentioned characteristics to obtain 768 building shapes. The dataset comprises 768 samples and 8 features, aiming to predict two real valued responses. It can also be used as a multi-class classification problem if the response is rounded to the nearest integer.  
The attributes are:  
The dataset contains eight attributes (or features, denoted by X1...X8) and two responses (or outcomes, denoted by y1 and y2). The aim is to use the eight features to predict each of the two responses.   
Specifically:  
X1	Relative Compactness  
X2	Surface Area  
X3	Wall Area  
X4	Roof Area  
X5	Overall Height  
X6	Orientation  
X7	Glazing Area  
X8	Glazing Area Distribution  
y1	Heating Load  
y2	Cooling Load
# Data Preparation

## Packages :-


library(dplyr)
library(ggplot2)
library(keras)
library(readxl)
library(caret)

source("./functions/train_val_test.R")


## Data Import


# If file does not exist, download it first:-
file_path <- "./data/energy.xlsx"
if (!file.exists(file_path)) { 
  dir.create("./data")
  url <- "http://archive.ics.uci.edu/ml/machine-learning-databases/00242/ENB2012_data.xlsx"
  download.file(url = url, 
                destfile = file_path, 
                mode = "wb")
}

energy_raw <- readxl::read_xlsx(path = file_path)


#Here, nrow(energy_raw) are observations and  ncol(energy_raw) are variables.

#These column names are to be detected:

energy_raw %>% colnames    # eight independent variables and two dependent (target) variables.


energy_raw %>% summary     # All variables are numeric.

## Train / Validation / Test Split:-

# Use 80 % training data and 20 % validation data.

c(train, val, test) %<-% train_val_test_split(df = energy_raw, 
                                              train_ratio = 0.8, 
                                              val_ratio = 0.0, 
                                              test_ratio = 0.2)


#check the target variables for training and validation dataset.


summary(train$Y1)
summary(test$Y1)

summary(train$Y2)
summary(test$Y2)


# Modeling :-

# The data will be transformed to a matrix.


X_train <- train %>% 
  select(-Y1, -Y2) %>% 
  as.matrix()
y_train <- train %>% 
  select(Y1, Y2) %>% 
  as.matrix()

X_test <- test %>% 
  select(-Y1, -Y2) %>% 
  as.matrix()
y_test <- test %>% 
  select(Y1, Y2) %>% 
  as.matrix()

dimnames(X_train) <- NULL
dimnames(X_test) <- NULL


## Data Scaling:-

#The data is scaled. This is highly recommended, because features can have very different ranges. This can speed up the training process #and avoid convergence problems.


X_train_scale <- X_train %>% 
  scale()

# Apply mean and sd from train dataset to normalize test set :-
col_mean_train <- attr(X_train_scale, "scaled:center") 
col_sd_train <- attr(X_train_scale, "scaled:scale")

X_test_scale <- X_test %>% 
  scale(center = col_mean_train, 
        scale = col_sd_train)



## Initialize Model:-

dnn_reg_model <- keras_model_sequential()


## Add Layers :-
# Two hidden layers and one output layer with two units for prediction are added.


dnn_reg_model %>% 
  layer_dense(units = 50, 
              activation = 'relu', 
              input_shape = c(ncol(X_train_scale))) %>% 
  layer_dense(units = 10, activation = 'relu') %>% 
  layer_dense(units = 2, activation = 'relu')


# A look at the model details:-


dnn_reg_model %>% summary()


It is a very small model with only close to 1000 parameters.

## Loss Function, Optimizer, Metric:-

dnn_reg_model %>% compile(optimizer = optimizer_adam(),
                          loss = 'mean_absolute_error')


# Put all in one function, as it is needed to run each time a model would be trained.


create_model <- function() {
  dnn_reg_model <- 
      keras_model_sequential() %>% 
      layer_dense(units = 50, 
              activation = 'relu', 
              input_shape = c(ncol(X_train_scale))) %>% 
      layer_dense(units = 50, activation = 'relu') %>% 
      layer_dense(units = 2, activation = 'relu') %>% 
      compile(optimizer = optimizer_rmsprop(),
              loss = 'mean_absolute_error')
}



## Model Fitting:-

# Fit the model and stop after 80 epochs. A validation ratio of 20 % is used for evaluating the model.


dnn_reg_model <- create_model()
history <- dnn_reg_model %>% 
  keras::fit(x = X_train_scale, 
             y = y_train,
             epochs = 80, 
             validation_split = 0.2,
             verbose = 0, 
             batch_size = 128)
plot(history,
     smooth = F)


# There is not much improvement after 40 epochs. An approach could be implemented, in which training stops if there is #no further improvement.

# A patience parameter can also be used for this. It represents the number of epochs to analyse for possible #improvements.


# Re-create model for this new run :-
dnn_reg_model <- create_model()

early_stop <- callback_early_stopping(monitor = "val_loss", patience = 20)

history <- dnn_reg_model %>% 
  keras::fit(x = X_train_scale, 
             y = y_train,
             epochs = 200, 
             validation_split = 0.2,
             verbose = 0, 
             batch_size = 128,
             callbacks = list(early_stop))
plot(history,
     smooth = F)


# Model Evaluation:-

# Create predictions and plots to show correlation of prediction and actual values:-

## Predictions:-

First, predictions, so that actual values can be compared.


y_test_pred <- predict(object = dnn_reg_model, x = X_test_scale)
y_test_pred %>% head

# Two output columns, which refer to two target variables.

## Check Performance :-


test$Y1_pred <- y_test_pred[, 1]
test$Y2_pred <- y_test_pred[, 2]


# Create correlation plots for Y1 and Y2:-


R2_test <- caret::postResample(pred = test$Y1_pred, obs = test$Y1)
g <- ggplot(test, aes(Y1, Y1_pred))
g <- g + geom_point(alpha = .5)
g <- g + annotate(geom = "text", x = 15, y = 30, label = paste("R**2 = ", round(R2_test[2], 3)))
g <- g + labs(x = "Actual Y1", y = "Predicted Y1", title = "Y1 Correlation Plot")
g <- g + geom_smooth(se=F, method = "lm")
g



R2_test <- caret::postResample(pred = test$Y2_pred, obs = test$Y2)
g <- ggplot(test, aes(Y2, Y2_pred))
g <- g + geom_point(alpha = .5)
g <- g + annotate(geom = "text", x = 15, y = 30, label = paste("R**2 = ", round(R2_test[2], 3)))
g <- g + labs(x = "Actual Y2", y = "Predicted Y2", title = "Y2 Correlation Plot")
g <- g + geom_smooth(se=F, method = "lm")
g


# Hyperparameter Tuning methods can also be implemented to get the desireable results.


# Acknowledgement

the authors of the dataset are:

The dataset was created by Angeliki Xifara (angxifara '@' gmail.com, Civil/Structural Engineer) and was processed by Athanasios Tsanas (tsanasthanasis '@' gmail.com, Oxford Centre for Industrial and Applied Mathematics, University of Oxford, UK).
