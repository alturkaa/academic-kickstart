---
title: Nonprofits in the U.S. since 1989
summary: Using Python to Get Nonprofit Data from NCCS
# date: "2015-06-24T00:00:00Z"
# lastmod: "2015-06-24T00:00:00Z"

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
The Python code below lets you compile a large dataset of all nonprofits in the U.S. since 1989. The data are based on IRS filings and have information such as founding date, yearly revenue and expenditures, location, etc. To compile the dataset, you first need to download each of the yearly data files from the [National Center For Charitable Statistics](https://nccs-data.urban.org/index.php).


```python
# Import modules
import numpy as np
import pandas as pd
```


```python
# create a list of all the years you'd like. This is 1989, 1995-2015.
years = []
for i in range(1995, 2016):
    years.append(i)
years.append(1989)
```


```python
# create an empty list to put all your dataframes in
pieces = []
```


```python
# This is a dictionary that, for each year, gives you the month of the data extraction.
# This is based on the file names from NCCS.
yr_mos = {1989 : '12', 1995 : '08', 1996 : '06',
             1997 : '10', 1998 : '09', 1999 : '12',
			 2000 : '05', 2001 : '07', 2002 : '07',
             2003 : '11', 2004 : '12', 2005 : '11',
			 2006 : '11', 2007 : '09', 2008 : '12',
			 2009 : '10', 2010 : '11', 2011 : '12', 
			 2012 : '12', 2013 : '12', 2014 : '12',
			 2015 : '12'}
```

##### This is the main function that combines all the data files from the NCCS website. It imports each one, makes sure some columns are read in as strings (not integers) and adds year and month columns. 
##### This initially took too long or froze, so I needed to run it using our research computing clusters.


```python
for year in years:
	eo = pd.read_csv(r'bmf/bmf.bm{}.csv'.format(year), dtype={'EIN' : str, 'NTEECC' : str, 'NTEE1' : str, 'MSA_NECH' : str, 'FIPS' : str}, low_memory = False)
	eo['year'] = year
	for k in yr_mos:
		if k == year:
			eo['month'] = yr_mos[k]
	eo['city_lower'] = eo.CITY.str.lower()
	eo['state_lower'] = eo.STATE.str.lower()
    # print out each file's shape
	print (eo.shape)
	# print ({year : eo.columns.tolist()})
	pieces.append(eo)

full_data = pd.concat(pieces, ignore_index=True)
```

##### Print the shape of the full dataset and number of orgs by year.


```python
print (full_data.shape)
print (full_data['year'].value_counts())
```

##### Export to .csv file


```python
full_data.to_csv(r'bmf_89_15.csv', index=False)
```

The Python code below lets you compile a large dataset of all nonprofits in the U.S. since 1989. The data are based on IRS filings and have information such as founding date, yearly revenue and expenditures, location, etc. To compile the dataset, you first need to download each of the yearly data files from the [National Center For Charitable Statistics](https://nccs-data.urban.org/index.php).


```python
# Import modules
import numpy as np
import pandas as pd
```


```python
# create a list of all the years you'd like. This is 1989, 1995-2015.
years = []
for i in range(1995, 2016):
    years.append(i)
years.append(1989)
```


```python
# create an empty list to put all your dataframes in
pieces = []
```


```python
# This is a dictionary that, for each year, gives you the month of the data extraction.
# This is based on the file names from NCCS.
yr_mos = {1989 : '12', 1995 : '08', 1996 : '06', 
          1997 : '10', 1998 : '09', 1999 : '12',
          2000 : '05', 2001 : '07', 2002 : '07',
          2003 : '11', 2004 : '12', 2005 : '11',
          2006 : '11', 2007 : '09', 2008 : '12',
          2009 : '10', 2010 : '11', 2011 : '12', 
          2012 : '12', 2013 : '12', 2014 : '12',
          2015 : '12'}
```

##### This is the main function that combines all the data files from the NCCS website. It imports each one, makes sure some columns are read in as strings (not integers) and adds year and month columns. 
##### This initially took too long or froze, so I needed to run it using our research computing clusters.


```python
for year in years:
    eo = pd.read_csv(r'bmf/bmf.bm{}.csv'.format(year), dtype={'EIN' : str, 'NTEECC' : str, \
                                                              'NTEE1' : str, 'MSA_NECH' : str, \
                                                              'FIPS' : str}, low_memory = False)
    eo['year'] = year
    for k in yr_mos:
        if k == year:
            eo['month'] = yr_mos[k]
    eo['city_lower'] = eo.CITY.str.lower()
    eo['state_lower'] = eo.STATE.str.lower()
    # print out each file's shape
    print (eo.shape)
    # print ({year : eo.columns.tolist()})
    pieces.append(eo)

full_data = pd.concat(pieces, ignore_index=True)
```

##### Print the shape of the full dataset and number of orgs by year.


```python
print (full_data.shape)
print (full_data['year'].value_counts())
```

##### Export to .csv file


```python
full_data.to_csv(r'bmf_89_15.csv', index=False)
```
