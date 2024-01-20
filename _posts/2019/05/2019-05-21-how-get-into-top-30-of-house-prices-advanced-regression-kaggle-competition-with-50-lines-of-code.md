---
title: "How get into top 30% of House Prices: Advanced Regression Kaggle competition with 50 lines of code"
date: "2019-05-21"
categories: ["data-science"]
---

This will be a quick guide on a very easy way to do machine learning competitions. This approach is very general and easily applicable to other competitions. I just want to make it clear that using this approach you will not win any competitons, but my goal is present a really nice and almost automated process which can get you above average scores in competitions.  
  
It consists of the following steps:  
  

1. Data cleaning
2. Feature engineering
3. Training the model and final predictions

  
You can also check out the according Jupyter notebook [here](https://github.com/dorianb96/House-Prices-Advanced-Regression-Techniques/blob/master/HousePrices.ipynb).  
  

### 1. Data cleaning

This is a simple process where we create the dataframes, read the data labels and other preparation simple tasks. When applying this framework to other problems you will have to make most of the changes in this part of code because labels or row indices might have different names.  

```python
# read the data  
train_data = pd.read_csv('train.csv')  
test_data = pd.read_csv('test.csv')  
# get the training and testing data  
x_train_raw = train_data.drop(['Id','SalePrice'],axis=1)  
x_test_raw = test_data.drop('Id',axis=1)  
# remember the lenghts so we can split the data later  
train_length = len(x_train_raw)  
test_length = len(x_test_raw)  
# concatenate train + test data into one df  
all_data = pd.concat([x_train_raw, x_test_raw])  
# remember the test data indices only for the final submission  
x_test_ids = test_data['Id']  
# get the training data - house prices  
# normal prices have a skew so adjust them to log space  
y_train = np.log(train_data['SalePrice'])
```

### 2. Feature engineering

Here we will apply the simplest preprocessing steps in order to prepare the data into a format for machine learning models:  
  

- missing values-encode a special meaning (“Missing” or 999999)
- categorical -one-hot-encoding to get a nice vector format
- numerical -removing the mean and scaling to unit variance

  
The following code does that:  

```python
# a nice trick to find out numeric vs categorical features  
numerical_features = x_train_raw.columns[x_train_raw.dtypes != 'object']  
categorical_features = x_train_raw.columns[x_train_raw.dtypes == 'object']  
# encode missing numbers as a special large number  
all_data[numerical_features] = all_data[numerical_features].fillna(99999999)  
# encode missing data as a special category -missing  
all_data[categorical_features] = all_data[categorical_features].fillna("Missing")  
# transform numeric variables  
ss = StandardScaler()  
all_data[numerical_features] = ss.fit_transform(all_data[numerical_features])  
# transform categorical variables  
all_data = pd.get_dummies(data=all_data, columns=categorical_features)  
Then we just have to split the data back into training and testing dat  
# re-split again  
x_train = all_data.head(train_length)  
x_test = all_data.tail(test_length)
```

### 3. Training the model and getting predictions

As now we have the usual machine learning ingredients:  

```python
x_train and y_train  
x_test
```

The next step is training the model with training data and submitting the predictions on test data.  
By far the most popular (and for a good reason) machine learning model is XGBoost. There is a specific library you can install, but for this problem the Scikit-learn version suffices.  

```python
est = GradientBoostingRegressor(n_estimators=2000, learning_rate=0.05, max_depth=3, max_features='sqrt',min_samples_leaf=15, min_samples_split=10,  
loss='huber',random_state = 13)  
est.fit(x_train, y_train)
```

The model I chose above was after a very time intensive grid search with cross validation process on the training data and a few submission to the leaderboard (check out the original code in bonus section).  
Finally we just have to package the predictions into a submission file  

```python
y_test = np.exp(est.predict(x_test))  
result = pd.DataFrame({'Id': x_test_ids.tolist(), 'SalePrice': y_test.ravel()})  
result.to_csv('submission.csv',index=False)
```

**And that’s it !**  
With this model you should be ableto get a 0.12104 score which would place you into top 30% (~685/2300 place at the time of writing: 26.04.2017).  
  

### Bonus

If you’d like to explore other possible improvments to get a better score (with my personal model I got a score of ~0.116 which is good for 274th place at the moment).  
So the possible improvment options are:  
  

- grid search (with cross validation) on the Gradient Boosting regressor
- keras and deep learning models
- model stacking (very advanced and powerful technique)

  
You can find code examples for each of those uncommented at the bottom of the Jupyter notebook I linked [before](https://github.com/dorianb96/House-Prices-Advanced-Regression-Techniques/blob/master/HousePrices.ipynb).
