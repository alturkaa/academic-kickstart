---
title: Nonprofit Mission Statements
summary: Scraping and Parsing Electronically Filed IRS 990 Forms Since 2011
publishdate: 2020-03-02

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
In an earlier [post](https://akramalturk.org/post/bmf-concat/), I shared some Python code that compiles nonprofit data from the Urban Institute's National Center for Charitable Statistics. But those data are relatively limited. To get more detailed data about nonprofits, researchers are increasingly using the tons of information you can get from a nonprofit's IRS filing (e.g., 990, 990EZ, 990PF, etc.). This has been made easier since the IRS started releasing machine-readable filings (of organizations filing their forms electronically). There are now a number of [organizations and researchers](https://registry.opendata.aws/irs990/), including [Open990](https://www.open990.org/org/), where you can get the raw data and, in some cases, tabular datasets. 

I'm sharing some code below that shows you how to 1) scrape the files from (what I can tell is) the [best place](https://docs.opendata.aws/irs-990/readme.html) to get the raw IRS electronic filings (in XML format), and 2) get the data from the XML files and put them into a tabular dataset that can be used for analysis. In this case, I'm only focusing on 990 filings from 2019, getting a relatively small subset of organizations, and getting only the **mission statements** of those nonprofits. The XML files have tons of other information, including executive compensation, grant recipients, an organization's activities, etc.


```python
# import libraries for scraping XML files from AWS site
import pandas as pd
import requests
import os
```


```python
pd.set_option('display.max_colwidth', 100)
```


```python
# import libraries for parsing XML files
from lxml import etree
from collections import Counter
import itertools
import glob
```

#### Step 1: Download Index File from AWS site


```python
# URL of IRS 990 filings index file from 2019
irs_url_2019 = 'https://s3.amazonaws.com/irs-form-990/index_2019.csv'
```


```python
# read in the index file and keep some variables as strings
orgs_2019 = pd.read_csv(irs_url_2019, dtype={'EIN' : str, 'OBJECT_ID' : str, 'RETURN_TYPE' : str})
```


```python
# shape of index file
orgs_2019.shape
```




    (396187, 9)




```python
orgs_2019.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>RETURN_ID</th>
      <th>FILING_TYPE</th>
      <th>EIN</th>
      <th>TAX_PERIOD</th>
      <th>SUB_DATE</th>
      <th>TAXPAYER_NAME</th>
      <th>RETURN_TYPE</th>
      <th>DLN</th>
      <th>OBJECT_ID</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>16285381</td>
      <td>EFILE</td>
      <td>133085892</td>
      <td>201809</td>
      <td>5/10/2019 6:06:12 AM</td>
      <td>LOGOS ENCOUNTER INC</td>
      <td>990</td>
      <td>93493091012069</td>
      <td>201910919349301206</td>
    </tr>
    <tr>
      <th>1</th>
      <td>16279505</td>
      <td>EFILE</td>
      <td>640411847</td>
      <td>201805</td>
      <td>5/8/2019 9:46:22 PM</td>
      <td>MISSISSIPPI CHRISTIAN FOUNDATION</td>
      <td>990</td>
      <td>93493101010839</td>
      <td>201931019349301083</td>
    </tr>
    <tr>
      <th>2</th>
      <td>16279502</td>
      <td>EFILE</td>
      <td>870213529</td>
      <td>201805</td>
      <td>5/8/2019 9:46:20 PM</td>
      <td>INTL SOC DAUGHTERS OF UT PIONEERS</td>
      <td>990</td>
      <td>93493101010539</td>
      <td>201931019349301053</td>
    </tr>
    <tr>
      <th>3</th>
      <td>16279501</td>
      <td>EFILE</td>
      <td>204223437</td>
      <td>201806</td>
      <td>5/8/2019 9:46:19 PM</td>
      <td>SCHOOLHOUSE SUPPLIES INC</td>
      <td>990</td>
      <td>93493101010189</td>
      <td>201931019349301018</td>
    </tr>
    <tr>
      <th>4</th>
      <td>16279248</td>
      <td>EFILE</td>
      <td>475066819</td>
      <td>201806</td>
      <td>5/8/2019 9:13:09 PM</td>
      <td>MINDFUL LIFE PROJECT</td>
      <td>990</td>
      <td>93493099005419</td>
      <td>201910999349300541</td>
    </tr>
  </tbody>
</table>
</div>




```python
# look at how many of each kind of return type there are
orgs_2019['RETURN_TYPE'].value_counts()
```




    990      165201
    990EZ     90003
    990PF     61998
    990O      50643
    990EO     28342
    Name: RETURN_TYPE, dtype: int64




```python
# subset to just 990 filings (i.e., exclude EZ, PF, etc.)
orgs_2019_990 = orgs_2019[orgs_2019['RETURN_TYPE'] == '990']
```


```python
### make sure there aren't duplicate object IDs, which will be used to scrape the actual XML files
# print number of rows in 990 dataframe
print (orgs_2019_990.shape[0])
# print number of unique object IDs
print (orgs_2019_990['OBJECT_ID'].nunique())
```

    165201
    165201
    


```python
# make a list of all object IDs to iterate through later
ids_990 = orgs_2019_990['OBJECT_ID'].tolist()
```


```python
len(ids_990)
```




    165201



#### Create a folder to put all the XML files in


```python
os.mkdir('returns_2019_990')
```

#### Main function to download files from AWS site and put them in the just-created folder


```python
# Note: This just scrapes 100 files. Remove `[:100]` from first line of loop to download all 165K files
# Based on the output from the timeit module, this goes pretty quickly
%%timeit
for i in ids_990[:100]:
    url = 'https://s3.amazonaws.com/irs-form-990/{}_public.xml'.format(i)
    r = requests.get(url)
    with open('returns_2019_990/{}.xml'.format(i), 'wb') as f:
        f.write(r.content)
```

    19.5 s ± 1.14 s per loop (mean ± std. dev. of 7 runs, 1 loop each)
    

#### Step 2: Parse Lots of XML Files


```python
# put all downloaded file names into a list
xml_file_names = glob.glob('returns_2019_990/*.xml')
```


```python
# 100 files
len(xml_file_names)
```




    100




```python
# example file name
xml_file_names[0]
```




    'returns_2019_990\\201900989349301350.xml'




```python
# create new list that takes out ".xml" from each file name
files_edited = []

for file in xml_file_names:
    files_edited.append(file[:-4])
```


```python
# edited file name
files_edited[0]
```




    'returns_2019_990\\201900989349301350'




```python
# namespace used in XML parsing
ns = {'irs' : 'http://www.irs.gov/efile'}
```

#### Main function to parse a bunch of XML files and put text into lists


```python
# This gets the text of a few fields (e.g., EIN, ActivityOrMissionDesc, MissionDesc) in the XML files
# and puts them, first, in a dictionary, and then, in a list. This will then make it easy to convert
# that into a dataframe.
allxmls_dfs = []
allxmls_df_lengths = []

for i in files_edited:
	root = etree.parse('{}.xml'.format(i)).getroot()
	
	# for child in root:
		# print(child.tag, child.attrib)
	
	full_list = []
	
	var_dict = {}
	
	ein = root.find('irs:ReturnHeader', ns).find('irs:Filer', ns).find('irs:EIN', ns)
	var_dict['ein'] = ein.text
	# print (ein.text)	
	act = root.find('irs:ReturnData', ns).find('irs:IRS990', ns).find('irs:ActivityOrMissionDesc', ns)
	if act is not None:
		var_dict['activity'] = act.text
	
	act2 = root.find('irs:ReturnData', ns).find('irs:IRS990', ns).find('irs:ActivityOrMissionDescription', ns)
	if act2 is not None:
		var_dict['activity2'] = act2.text
		
	mission = root.find('irs:ReturnData', ns).find('irs:IRS990', ns).find('irs:MissionDesc', ns)
	if mission is not None:
		var_dict['mission'] = mission.text
	
	mission2 = root.find('irs:ReturnData', ns).find('irs:IRS990', ns).find('irs:MissionDescription', ns)
	if mission2 is not None:
		var_dict['mission2'] = mission2.text
	
	full_list.append(var_dict)
		
	# print ('List length of {} xml is {}'.format(i, len(full_list)))
	
	tmp_df = pd.DataFrame(full_list)
    # create object ID without the file name extras
	tmp_df['OBJECT_ID'] = i[17:]
	# tmp_df = tmp_df.dropna(subset=['ein'])
	# print ('Dataframe shape of {} xml is {}'.format(i, tmp_df.shape))
	allxmls_dfs.append(tmp_df)
	allxmls_df_lengths.append(len(tmp_df))
```


```python
# print out what one dictionary looks like
var_dict
```




    {'ein': '753004503',
     'activity': 'To promote hockey for the youth and young adults in the Pikes Peak Region.',
     'mission': 'To promote hockey for the youth and young adults in the Pikes Peak Region.'}




```python
# print out what above dictionary looks like as one-row dataframe
allxmls_dfs[99]
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>ein</th>
      <th>activity</th>
      <th>mission</th>
      <th>OBJECT_ID</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>753004503</td>
      <td>To promote hockey for the youth and young adults in the Pikes Peak Region.</td>
      <td>To promote hockey for the youth and young adults in the Pikes Peak Region.</td>
      <td>201940939349300424</td>
    </tr>
  </tbody>
</table>
</div>




```python
# Make sure length of what's been parsed is equal to number of files
print ('Total length of all xmls is {}'.format(sum(allxmls_df_lengths)))
```

    Total length of all xmls is 100
    

#### Step 3: Convert List to a Tabular Dataset


```python
irs_2019_990 = pd.concat(allxmls_dfs, ignore_index = True)
```


```python
# Make sure dataset shape is what we expect and that there are no duplicate object IDs
print ('Shape of full dataset is {}'.format(irs_2019_990.shape))
print ('Number of unique OBJECT_ID numbers is {}'.format(irs_2019_990['OBJECT_ID'].nunique()))
```

    Shape of full dataset is (100, 4)
    Number of unique OBJECT_ID numbers is 100
    


```python
# Preview of dataset!
irs_2019_990
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>ein</th>
      <th>activity</th>
      <th>mission</th>
      <th>OBJECT_ID</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>942967138</td>
      <td>Support the educational activities of Grattan Elementary School - students, teachers &amp; administr...</td>
      <td>Support the educational activities of Grattan Elementary School - students, teachers &amp; administr...</td>
      <td>201900989349301350</td>
    </tr>
    <tr>
      <th>1</th>
      <td>261095856</td>
      <td>To seek better treatment options for women with ovarian cancer by providing diagnostic services ...</td>
      <td>The mission is to seek better treatment options for women with ovarian cancer by providing diagn...</td>
      <td>201900989349301400</td>
    </tr>
    <tr>
      <th>2</th>
      <td>990110027</td>
      <td>WE EDUCATE YOUNG WOMEN TO LEAD A LIFE OF ACHIEVEMENT. TO REALIZE OUR VISION WE WILL PROVIDE A VI...</td>
      <td>WE EDUCATE YOUNG WOMEN TO LEAD A LIFE OF ACHIEVEMENT. TO REALIZE OUR VISION WE WILL PROVIDE A VI...</td>
      <td>201900989349301410</td>
    </tr>
    <tr>
      <th>3</th>
      <td>200725426</td>
      <td>SERAPHIC FIRE AIMS TO PRESENT HIGH-QUALITY PERFORMANCES OF UNDER-PERFORMED MUSIC WITH CULTURAL S...</td>
      <td>SERAPHIC FIRE AIMS TO PRESENT HIGH-QUALITY PERFORMANCES OF UNDER-PERFORMED MUSIC WITH CULTURAL S...</td>
      <td>201900989349301510</td>
    </tr>
    <tr>
      <th>4</th>
      <td>521891697</td>
      <td>We advocate and embody independence and equality for all people with disabilities.</td>
      <td>Independence Now, Inc strives to facilitate independent thought and action by people with disabi...</td>
      <td>201900989349301520</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>95</th>
      <td>133712927</td>
      <td>TO SUPPORT THE NEW ROCHELLE LIBRARY.</td>
      <td>THE NEW ROCHELLE PUBLIC LIBRARY FOUNDATION, A NON-PROFIT ORGANIZATION, RAISES RESOURCES AND PROV...</td>
      <td>201940939349300209</td>
    </tr>
    <tr>
      <th>96</th>
      <td>931021970</td>
      <td>THE MISSION OF THE LAGUNA BEACH EDUCATION ENDOWMENT AND CAPITAL FUND IS TO PROMOTE EDUCATIONAL E...</td>
      <td>THE MISSION OF THE LAGUNA BEACH EDUCATION ENDOWMENT AND CAPITAL FUND IS TO PROMOTE EDUCATIONAL E...</td>
      <td>201940939349300214</td>
    </tr>
    <tr>
      <th>97</th>
      <td>205595877</td>
      <td>TO SERVE IN THE AREA OF FIRE SAFETY AND SAVE LIFES AND PROPERTY DURING A TIMES OF DISASTERS AND ...</td>
      <td>TO SERVE IN THE AREA OF FIRE SAFETY AND SAVE LIFES AND PROPERTY DURING A TIMES OF DISASTERS AND ...</td>
      <td>201940939349300224</td>
    </tr>
    <tr>
      <th>98</th>
      <td>561576543</td>
      <td>TO SERVE, EMPOWER AND MINISTER TO CLIENTS IN ORDER TO PREVENT AND END HUNGER AND HOMELESSNESS TH...</td>
      <td>TO SERVE, EMPOWER AND MINISTER TO CLIENTS IN ORDER TO PREVENT AND END HUNGER AND HOMELESSNESS TH...</td>
      <td>201940939349300304</td>
    </tr>
    <tr>
      <th>99</th>
      <td>753004503</td>
      <td>To promote hockey for the youth and young adults in the Pikes Peak Region.</td>
      <td>To promote hockey for the youth and young adults in the Pikes Peak Region.</td>
      <td>201940939349300424</td>
    </tr>
  </tbody>
</table>
<p>100 rows × 4 columns</p>
</div>




```python
# Export to .csv file
irs_2019_990.to_csv('irs_2019_990.csv', index = False)
```
