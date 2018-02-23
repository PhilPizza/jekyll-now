---
layout: post
title: You're up and running!
---
# Forecasting Statewide Annual Liquor Sales of Iowa (2016)
### - Iowans (Iowites?) bought over \$284 million dollars in liquor in 2015. How much will they buy in 2016?

- Provided all of the transactional data for the year of 2015, and also the transactional data for the first quarter of 2016, can we make a statistically informed estimate of 2016 total annual sales? I sought to predict 2016 total Annual Sales of liquor in Iowa by creating a predictive model trained on first quarter 2015 sales, on a store-by-store basis, and predicting the 2016 annual sales of each store based on available 2016 Q1 sales data. Through careful model selection, feature tuning, and optimization, I predicted that \$289,740,820 in total Annual Sales revenue would be generated for 2016 (a modest 1.9% increase over 2015). 


- Important to note, the increase in the total "true purchasing power" may be mostly offset by the expected inflation rate over 2016, which is estimated by Statista, a web-based statistics portal, to be 1.3%.


- While I feel confident in my model's results, the accuracy can always be improved with additional information. Provided more resources, I might have been able to classify individual stores by type, having noticed that some transactions were over \$100,000, which would seem to indicate that some stores may be wholesale oriented, and not just casual-consumer facing.

# The Process

## - EDA and Data Cleaning


```python
# Pretty big data set, over 2.7 million transactions
iowa_df.shape
```




    (2709552, 24)




```python
# Run a separte notebook with some personal functions that I've been writing to simplify common EDA and conversions
%run TuckerMagic.ipynb
```


```python
import numpy as np
import pandas as pd
import string

from sklearn.model_selection import train_test_split
from sklearn.model_selection import KFold, cross_val_score, cross_val_predict
from sklearn import metrics

from sklearn import datasets, linear_model
from sklearn.linear_model import LinearRegression
from sklearn.linear_model import Ridge
from sklearn.linear_model import RidgeCV

import matplotlib.pyplot as plt
%matplotlib inline
```


```python
# Convert State Bottle Cost, State Bottle Retail, and Sale (Dollars) to numeric types (function from TuckerMagic)
dollar_stripper(iowa_df, 'State Bottle Cost')
dollar_stripper(iowa_df, 'State Bottle Retail')
dollar_stripper(iowa_df, 'Sale (Dollars)')
```


```python
# Conver all the text-based columns to Uppercase to ensure uniformity
iowa_df['City'] = iowa_df['City'].str.upper()
iowa_df['County'] = iowa_df['County'].str.upper()
iowa_df['Category Name'] = iowa_df['Category Name'].str.upper()
iowa_df['Item Description'] = iowa_df['Item Description'].str.upper()
```


```python
# Missing a lot of County Numbers, County, Category, and Category Name
iowa_df.isnull().sum()
```




    Invoice/Item Number          0
    Date                         0
    Store Number                 0
    Store Name                   0
    Address                      0
    City                         0
    Zip Code                     0
    Store Location               0
    County Number            10913
    County                    2150
    Category                   779
    Category Name             6109
    Vendor Number                0
    Vendor Name                  0
    Item Number                  0
    Item Description             0
    Pack                         0
    Bottle Volume (ml)           0
    State Bottle Cost            0
    State Bottle Retail          0
    Bottles Sold                 0
    Sale (Dollars)               0
    Volume Sold (Liters)         0
    Volume Sold (Gallons)        0
    dtype: int64



###  ...Going to omit the balance of (exhaustive) data cleaning details because all the features I cleaned never ended up in my model anyways

## - Feature Engineering


```python
# Looks like our data is for all of 2015, and the first 3 months of 2016
iowa_df['Date'].describe()
```




    count        2709552
    unique           284
    top       12/01/2015
    freq           15588
    Name: Date, dtype: object




```python
# Add a 'Markup per Bottle' Column
iowa_df['Markup per Bottle'] = (iowa_df['State Bottle Retail'] - iowa_df['State Bottle Cost']) / iowa_df['State Bottle Cost']
```


```python
# Add a total 'Profit' from transaction Column
iowa_df['Profit'] = iowa_df['Sale (Dollars)'] - iowa_df['Bottles Sold']*iowa_df['State Bottle Cost']
```


```python
# Let's bring in some population information from the internet
# source: http://www.city-data.com/city/Iowa.html
iowa_pop_path = '../data/iowa_pop.csv'
pop_df = pd.read_csv(iowa_pop_path, sep='\t', names=['City', 'Pop']  )
```


```python
pop_df.head()
```




<div>
<style>
    .dataframe thead tr:only-child th {
        text-align: right;
    }

    .dataframe thead th {
        text-align: left;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>City</th>
      <th>Pop</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Ackley, IA</td>
      <td>1,550</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Adair, IA</td>
      <td>752</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Adel, IA</td>
      <td>4,171</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Afton, IA</td>
      <td>842</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Agency, IA</td>
      <td>636</td>
    </tr>
  </tbody>
</table>
</div>




```python
# Let's strip the ', IA' which only appears on some entries of pop_df, to match our dataset
for i in range(len(pop_df)):
    if pop_df['City'][i][-4:] == ', IA':
        pop_df.set_value(i, 'City', pop_df['City'][i][:-4].upper())
    else:
        pop_df.set_value(i, 'City', pop_df['City'][i].upper())
```


```python
# Check that the stripping worked
pop_df.head()
```




<div>
<style>
    .dataframe thead tr:only-child th {
        text-align: right;
    }

    .dataframe thead th {
        text-align: left;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>City</th>
      <th>Pop</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>ACKLEY</td>
      <td>1,550</td>
    </tr>
    <tr>
      <th>1</th>
      <td>ADAIR</td>
      <td>752</td>
    </tr>
    <tr>
      <th>2</th>
      <td>ADEL</td>
      <td>4,171</td>
    </tr>
    <tr>
      <th>3</th>
      <td>AFTON</td>
      <td>842</td>
    </tr>
    <tr>
      <th>4</th>
      <td>AGENCY</td>
      <td>636</td>
    </tr>
  </tbody>
</table>
</div>




```python
# Get our Pop column numeric by stripping the ',' and casting as int
pop_df['Pop'] = pop_df['Pop'].map(lambda x: x.replace(',', '')).astype(int)
```


```python
# Check that column typing worked
pop_df.dtypes
```




    City    object
    Pop      int64
    dtype: object




```python
# Join our pop_df onto our iowa_df on 'City'
iowa_merged = iowa_df.merge(pop_df, how='left', on='City')
```


```python
# Check that we joined correctly, and didn't lost any data
iowa_merged.shape
```




    (2709552, 27)




```python
# Looks like're missing some population numbers
iowa_merged.isnull().sum()
```




    Invoice/Item Number          0
    Date                         0
    Store Number                 0
    Store Name                   0
    Address                      0
    City                         0
    Zip Code                     0
    Store Location               0
    County Number                0
    County                    1875
    Category                     0
    Category Name              642
    Vendor Number                0
    Vendor Name                  0
    Item Number                  0
    Item Description             0
    Pack                         0
    Bottle Volume (ml)           0
    State Bottle Cost            0
    State Bottle Retail          0
    Bottles Sold                 0
    Sale (Dollars)               0
    Volume Sold (Liters)         0
    Volume Sold (Gallons)        0
    Markup per Bottle            0
    Profit                       0
    Pop                      35165
    dtype: int64




```python
# Looks like we're missing population data for 17 cities
iowa_merged[iowa_merged['Pop'].isnull()]['City'].unique()
```




    array(['LECLAIRE', 'MT VERNON', 'GUTTENBURG', 'OTTUWMA', 'ZWINGLE',
           "ARNOLD'S PARK", 'LEMARS', 'MT PLEASANT', 'DELAWARE', 'JEWELL',
           'ST ANSGAR', 'BALDWIN', 'DEWITT', 'KELLOG', 'ST CHARLES',
           'BEVINGTON', 'ST LUCAS'], dtype=object)




```python
# Hand enter the populations that got missed
# Data credit: Google
complete_pop = {'LECLAIRE':3974, 'OTTUWMA':24487, 'LEMARS':9935, 'DELAWARE':155, 'MT VERNON':4444, 'MT PLEASANT':8392,
               'GUTTENBURG':1840, 'ST ANSGAR':1150, 'BEVINGTON':68, 'ZWINGLE':89, 'DEWITT': 5233, 'ST CHARLES':618, 
               'BALDWIN':106, 'KELLOG':590, 'JEWELL':1176, "ARNOLD'S PARK": 1256, 'ST LUCAS':164}
```


```python
# Make a pass updating populations that got missed in initial pop_df merge
for i in range(len(iowa_merged)):
    if pd.isnull(iowa_merged['Pop'][i]):
        try:
            iowa_merged.set_value(i, 'Pop', complete_pop[iowa_merged['City'][i]])
        except:
            pass
```

## Exploratory Visuals


```python
# For slides, Q1 2015 sales by top 50 stores
fig, ax = plt.subplots(figsize=(8,8))
plt.bar(range(50), Q1_2015_Top50['Sale (Dollars)'])
ax.set_xticklabels(Q1_2015_Top50.index)
plt.title('Q1 2015 Sales of Top 50 Stores', fontsize=16)
ax.set_xlabel('Store Number', fontsize = 12)
ax.set_ylabel('Q1 2015 Sales in USD ($)', fontsize=12)
ax.set
plt.show();
```


![png](starter-code-full-data-set_blog_files/starter-code-full-data-set_blog_28_0.png)



```python
# Top 5 sellers in 2015
sales_2015[['Sale (Dollars)']].head(5)
```




<div>
<style>
    .dataframe thead tr:only-child th {
        text-align: right;
    }

    .dataframe thead th {
        text-align: left;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th></th>
      <th>Sale (Dollars)</th>
    </tr>
    <tr>
      <th>Store Number</th>
      <th>City</th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>2633</th>
      <th>DES MOINES</th>
      <td>9839393.08</td>
    </tr>
    <tr>
      <th>4829</th>
      <th>DES MOINES</th>
      <td>8742779.31</td>
    </tr>
    <tr>
      <th>2512</th>
      <th>IOWA CITY</th>
      <td>4155665.47</td>
    </tr>
    <tr>
      <th>3385</th>
      <th>CEDAR RAPIDS</th>
      <td>3947176.01</td>
    </tr>
    <tr>
      <th>3420</th>
      <th>WINDSOR HEIGHTS</th>
      <td>3422351.55</td>
    </tr>
  </tbody>
</table>
</div>



## Modeling 

### Test/Train Split

- We will hold out 2016 data as our final validation
- We will conduct a Test/Train split on all of our 2015 data

- Going to be predicting End of Year Sales for each store, based on Q1 data only


```python
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)
```

### After some model selection and tuning...


```python
fig, ax = plt.subplots(figsize=(8,8))

plt.title('2016 Annual Store Sale Predictions', fontsize=16)
ax.set_xlabel('Individual Stores', fontsize=12)
ax.set_ylabel('Predicted Annual Store Sale ($)', fontsize=12)

plt.scatter(range(1320),y_2016_predicts_rev)
```




    <matplotlib.collections.PathCollection at 0x1a135093c8>




![png](starter-code-full-data-set_blog_files/starter-code-full-data-set_blog_34_1.png)

