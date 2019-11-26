
WIP

# Overview
The aim of forecastlib is to provide a lightweight, modular framework for dynamic time series modeling, compatible with scikit-learn. forecastlib has two main components that are interlinked.

## TSConfig
The first part, TSConfig, takes as input any number of time series data, and merges them into modeling dataframes. There are two features that make this especially useful:

1. Conveniently add autoregressive (lag) terms for any variable, such as the target or any exogenous variable.
2. Simulate information delays at the variable level. In other words, rows can be shifted so that the prediction for each timestamp uses only what information would have been available at the forecast time.

This simplifies the process of combining datasets from different domains. The data is unified into a single configuration object which is then handled directly by time series models.

## AREX
The second component is AREX (AutoRegression with EXogeneity). AREX is an iterative time series predictor that abstracts away the logic of retraining a model sequentially on time series data. AREX does not impose any modeling constraints -- instead it is a procedure that can handle any model that is compatible with scikit-learn's fit/predict API.

Usually, one retrains a time series model at each time step in order to use the most recent information. The training set at each step can be either rolling (fixed size that discards old data), or expanding (use all data). Often, a time series is predicted using a combination of lags of the time series (AR), concurrent exogenous variables (EX), and lags of the exogenous variables. AREX takes care of these details for you.

On the other hand, the actual model that is applied at each time step is highly important to researchers -- it can involve preprocessing and feature engineering to using various ML algorithms and hyperparameter tuning. Thus this part is flexible and only limited by your creativity. All AREX needs is a model class with fit() and predict() methods, identical to sklearn. In fact, any sklearn model can be passed directly into AREX to get an
out-of-the-box time series modeler.

## Motivation
While the logic for time series modeling isn't particularly complicated, there are potential sources of error when one isn't careful. A simple example is using training data that shouldn't be available in forecasting.

Suppose one has annual climate data from the past century and is trying to predict global temperature 5 years ahead. On January 1, 2005 the prediction target is the entire year of 2010. Then for training we can use the (X, y) pairs from (1999->2004) and earlier, but we cannot use (2000->2005) through (2004->2009). This logic applies to every year's prediction. Otherwise the retrospective forecasts will be unfairly accurate. (Note that in some situations, (2000->2005) will be available. For example, if we make the prediction each December instead of January, we would roughly know the 2005 temperature at the time of prediction. This detail can be specified in AREX.forecast using the t_known parameter.)

It is also absolutely possible to use only one of TSConfig or Arex for your purposes. Either readily accepts or returns their underlying pandas dataframes.

# Examples

Suppose we are modeling flu incidence and our target variable is stored in the dataframe 'cdc'. We wish to use a predictor dataframe 'external'. First register the data:

```python
from forecastlib.datasets import TSConfig
dc = TSConfig()
dc.register_dataset(cdc, name='flu', type='target')
dc.register_dataset(external, name='pred', type='predictor')
```

Add lag terms of the target variable as autoregressive predictors:
```python
dc.add_AR(range(1, 7), dataset='flu')
```

Suppose due to transmission dynamics we also want a lag of a predictor:
```python
dc.add_AR([1], dataset='pred', var_names=['temp'])
```

Call the stack method to combine the datasets. The combined dataframes (as an (X, y) tuple) can be accessed using the data property.
```python
dc.stack()
dc.data
```

We will use a default sklearn random forest as the model:
```python
from sklearn.ensemble import RandomForestRegressor
mod = RandomForestRegressor()
```

The above time series is at a weekly frequency. For nowcasting (predicting target at week t using exogenous data from week t) with
a year-long rolling training window, do:
```python
from forecastlib import Arex
arex = Arex(model=mod, data_config=dc)
pred = arex.nowcast(pred_start='2019-02-19', pred_end='2019-08-20',
                    training='roll', window=52)
```

Suppose we want to predict a week ahead. We would do:
```python
pred2 = arex.forecast(t_plus=1, pred_start='2019-02-19',
                      pred_end='2019-08-20',
                      training='roll', window=52)
```

Note that the timestamps for pred_start and pred_end refer to the time of making the prediction, not the time that is predicted.
