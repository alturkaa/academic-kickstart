---
title: Political Polarization by Policy Domain
summary: Using Congressional Roll Call Votes to Measure Polarization by Policy Domain
date: '2020-07-07'
reading_time: false  # Show estimated reading time?
share: true  # Show social sharing links?
profile: false  # Show author profile?
comments: false  # Show comments?
draft: false

# Optional header image (relative to `static/img/` folder).
header:
  caption: ""
  image: ""
---


```
knitr::opts_chunk$set(echo = TRUE)
library(reticulate)
library(tidyverse)
library(plotly)
```


```r
%%R
# reticulate::py_config()

# conda_install(packages = 'xlrd')
```


```r
%%R
# pap_roll = read_csv('../../../../Research/Dissertation/Data/Policy Agendas Project/US-Legislative-roll_call_votes_19.1.csv')
```


```
import pandas as pd
import numpy as np

# import Comparative Agendas Project Roll Call Data
pap_roll = pd.read_csv('../../../../Research/Dissertation/Data/Policy Agendas Project/US-Legislative-roll_call_votes_18.2.csv', low_memory=False)

# import Voteview Roll Call Data
vv_house_roll_call = pd.read_excel('../../../../Research/Dissertation/Data/hdemrep35_113.xlsx')
vv_senate_roll_call = pd.read_excel('../../../../Research/Dissertation/Data/sdemrep35_113.xlsx')

# filter to only House votes from 1965-2015 in CAP
pap_roll_house = pap_roll[pap_roll['filter_House'] == 1]
pap_roll_house_1965_2015 = pap_roll_house[(pap_roll_house['year'] > 1964) & (pap_roll_house['year'] < 2016)]

# filter to only Senate votes from 1965-2015 in CAP
pap_roll_senate = pap_roll[pap_roll['filter_Senate'] == 1]
pap_roll_senate_1965_2015 = pap_roll_senate[(pap_roll_senate['year'] > 1964) & (pap_roll_senate['year'] < 2016)]
```


```

vv_house_roll_call['house_partisan'] = np.abs((vv_house_roll_call['r_yeas'] / (vv_house_roll_call['r_yeas'] + vv_house_roll_call['r_nays'])) - (vv_house_roll_call['d_yeas'] / (vv_house_roll_call['d_yeas'] + vv_house_roll_call['d_nays'])))

vv_senate_roll_call['senate_partisan'] = np.abs((vv_senate_roll_call['r_yeas'] / (vv_senate_roll_call['r_yeas'] + vv_senate_roll_call['r_nays'])) - (vv_senate_roll_call['d_yeas'] / (vv_senate_roll_call['d_yeas'] + vv_senate_roll_call['d_nays'])))

```

$$\left|\frac{R_{y}}{R_{y}+R_{n}}-\frac{D_{y}}{D_{y}+D_{n}}\right|$$


```

vv_house_roll_call_1965_2015 = vv_house_roll_call[(vv_house_roll_call['year'] > 1964) & (vv_house_roll_call['year'] < 2016)]

vv_senate_roll_call_1965_2015 = vv_senate_roll_call[(vv_senate_roll_call['year'] > 1964) & (vv_senate_roll_call['year'] < 2016)]

# print (vv_house_roll_call_1965_2015.shape)
# print (pap_roll_house_1965_2015.shape)

merged_house_roll = pd.merge(vv_house_roll_call_1965_2015, pap_roll_house_1965_2015, on=['year', 'cong', 'rc_count'])

# print (merged_house_roll.shape)
# 
# print (merged_house_roll.columns)
# 
# merged_senate_roll = pd.merge(senate_roll_call_46_2015, pap_roll_senate_46_2015, on=['year', 'cong', 'rc_count'])
# 
house_roll_by_year = merged_house_roll.groupby(['year', 'pap_majortopic']).agg({'house_partisan' : 'mean', 'rc_count' : 'size'}).reset_index().rename(columns={'year': 'Year', 'pap_majortopic' : 'MajorTopic', 'rc_count' : 'house_bills'})
# 
# print (house_roll_by_year.columns)

merged_house_roll.groupby('pap_majortopic')['house_partisan'].mean().reset_index().sort_values(by='house_partisan', ascending=False).head()

# 
# 
# senate_roll_by_year = merged_senate_roll.groupby(['year', 'pap_majortopic']).agg({'senate_partisan' : 'mean', 'rc_count' : 'size'}).reset_index().rename(columns={'year': 'Year', 'pap_majortopic' : 'MajorTopic', 'rc_count' : 'senate_bills'})

# congress_roll_by_year = pd.merge(house_roll_by_year, senate_roll_by_year, on=['Year', 'MajorTopic'], how='outer')
```