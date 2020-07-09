---
title: Social Determinants of County-Level COVID-19 Cases and Deaths
summary: Using flexdashboard to share data, code, and analysis
date: '2020-07-09'

reading_time: false  # Show estimated reading time?
share: true  # Show social sharing links?
profile: false  # Show author profile?
comments: false  # Show comments?
draft: false
---
In recent years, there has increasingly been a push to make academic research more transparent and accessible. That has taken many forms, including [pre-registering a study](https://osf.io/registries?view_only=), publishing [preprints](https://osf.io/preprints/socarxiv/) (papers that haven't been through the peer review process), and publicly sharing [data and analysis](https://dataverse.harvard.edu/) (including the underlying code). Since the beginning of the COVID-19 pandemic, this trend—both in academia and outside it—seems to only be accelerating. The amount of data that's being shared is pretty staggering and there are already tens of thousands of [COVID-related preprints](https://github.com/nicholasmfraser/covid19_preprints). 

Another recent trend has been the increasing use of interactive dashboards. I think the most widely used one for COVID-19 right now is this [one](https://www.arcgis.com/apps/opsdashboard/index.html#/bda7594740fd40299423467b48e9ecf6) from Johns Hopkins. But there are a ton of them. Just from a quick search on GitHub, there are already [thousands](https://github.com/search?q=covid+19+dashboard&type=Repositories). 

One way to combine both the utility of sharing data and initial findings (e.g., dataverses, preprints) and visualizing a rapidly changing situation (e.g., dashboards) is to create a standalone site that does both—in a way that can be updated regularly. I've tried to do something like that using [flexdashboard](https://rmarkdown.rstudio.com/flexdashboard/index.html) in R, which is an easier way than [Shiny](https://shiny.rstudio.com/) to get something up and running. 

The nice thing about using flexdashboard is that you can create different pages for your site, and your pages can contain some combination of interactive visualization, tables and figures, and even writeups. In other words, you can basically mirror the structure of an academic paper (e.g., intro, data, methods, results) in a way that is shareable with others and that gets updated as more data are collected or included in your analysis. (This obviously excludes theory or a literature review.)

In my case, I wanted to create a [map](https://socio-covid.netlify.app/#per-capita-cases-and-growth-rates-by-county) that showed per capita COVID-19 cases and a seven day case growth rate, share a [searchable table](https://socio-covid.netlify.app/#social-risk-factors-by-county) of the data, include [regression tables and a coefficient plot](https://socio-covid.netlify.app/#preliminary-analysis), and discuss the [data sources and methods](https://socio-covid.netlify.app/#analysis-details). And, most importantly, all of the R code to collect and analyze the data are in the `index.Rmd` file on this [page](https://github.com/alturkaa/socio-cov), which can be 1) updated regularly with the push of one [button](https://yihui.org/knitr/), and 2) analyzed (and scrutinized) by other researchers. 

The start-up "cost" for doing something like this is pretty minimal. It requires getting a [domain name](https://www.namecheap.com/) and deploying your website, which you can do pretty easily with [Netlify](https://bookdown.org/yihui/rmarkdown/blogdown-deploy.html). The rest is doing your analysis in RMarkdown and learning how to lay out all the different components in [flexdashboard](https://rmarkdown.rstudio.com/flexdashboard/using.html). 