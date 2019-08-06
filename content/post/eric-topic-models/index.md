---
title: Topic Models Test
summary: Test
# date: "2015-06-24T00:00:00Z"
# lastmod: "2015-06-24T00:00:00Z"

reading_time: false  # Show estimated reading time?
share: true  # Show social sharing links?
profile: false  # Show author profile?
comments: false  # Show comments?
draft: true

# Optional header image (relative to `static/img/` folder).
header:
  caption: ""
  image: ""
---

```
library(stm)
library(tidyverse)
library(furrr)
```


```
library(plotly)
library(htmlwidgets)
```

    Warning message:
    "package 'plotly' was built under R version 3.6.1"
    Attaching package: 'plotly'
    
    The following object is masked from 'package:ggplot2':
    
        last_plot
    
    The following object is masked from 'package:stats':
    
        filter
    
    The following object is masked from 'package:graphics':
    
        layout
    
    


```
achieve_orig <- read_csv('achieve_articles_60_94.csv')

ach_inst_dummies <- read_csv('achieve_60_94_inst_dummies.csv')

achieve_pre_83_dummies = read_csv('ach_pre_83_dummies.csv')

achieve_orig_inst <- merge(achieve_orig, ach_inst_dummies, by = 'accno')

# just select last columns, since they're already in achieve_orig
achieve_pre_83_dummies <- achieve_pre_83_dummies %>% 
  select(16:24)

achieve = merge(achieve_orig_inst, achieve_pre_83_dummies, by = 'accno')
```

    Parsed with column specification:
    cols(
      .default = col_double(),
      accno = col_character(),
      authors = col_character(),
      date_published = col_date(format = ""),
      description = col_character(),
      institutions = col_character(),
      issues = col_character(),
      keywords = col_character(),
      keywords_geo = col_character(),
      major_subjects = col_character(),
      publishers = col_character(),
      sources = col_character(),
      sponsors = col_character(),
      subjects = col_character(),
      title = col_character(),
      types = col_logical(),
      date_pub_list = col_character(),
      higher_ed_intl = col_logical(),
      foreign = col_logical(),
      journal = col_logical()
    )
    See spec(...) for full column specifications.
    Warning message:
    "46711 parsing failures.
    row            col   expected     actual                         file
      1 date_published valid date 1964-08-00 'achieve_articles_60_94.csv'
      2 date_published valid date 1964-00-00 'achieve_articles_60_94.csv'
      3 date_published valid date 1964-00-00 'achieve_articles_60_94.csv'
      4 date_published valid date 1963-00-00 'achieve_articles_60_94.csv'
      5 date_published valid date 1961-01-00 'achieve_articles_60_94.csv'
    ... .............. .......... .......... ............................
    See problems(...) for more details.
    "Parsed with column specification:
    cols(
      accno = col_character(),
      state_depts_of_ed = col_double(),
      univ_coll = col_double(),
      main_unions = col_double(),
      other_teach_assoc = col_double(),
      prof_assoc = col_double(),
      districts_schools = col_double(),
      research_inst = col_double(),
      policy_inst = col_double()
    )
    Parsed with column specification:
    cols(
      .default = col_double(),
      higher_ed_intl = col_logical(),
      foreign = col_logical(),
      journal = col_logical(),
      accno = col_character()
    )
    See spec(...) for full column specifications.
    Warning message:
    "5537 parsing failures.
     row     col           expected actual                     file
    2281 journal 1/0/T/F/TRUE/FALSE    1.0 'ach_pre_83_dummies.csv'
    3134 journal 1/0/T/F/TRUE/FALSE    1.0 'ach_pre_83_dummies.csv'
    3214 journal 1/0/T/F/TRUE/FALSE    0.0 'ach_pre_83_dummies.csv'
    3215 journal 1/0/T/F/TRUE/FALSE    0.0 'ach_pre_83_dummies.csv'
    3216 journal 1/0/T/F/TRUE/FALSE    0.0 'ach_pre_83_dummies.csv'
    .... ....... .................. ...... ........................
    See problems(...) for more details.
    "


```
num_k <- c(8, 10, 12, 14, 16, 18, 20)

stop_words <- c('education', 'educational', 'achievement', 'achieve', 'quot', 'author', 'authors', 'review', 'study', 'academic', 'studies', 'research', 'paper', 'document', 'report', 'use', 'can', 'may', 'documents', 'will', 'must')
```


```
load('achieve_multiple_k_diagnostics.RData')
```


```
achieve_process <- textProcessor(achieve$description, metadata = achieve, onlycharacter = TRUE, stem = FALSE, customstopwords = stop_words)

# drop words that appear in fewer than 5 documents and more than 70 percent of documents (lower.thresh default is fewer than 2)
achieve_out <- prepDocuments(achieve_process$documents, achieve_process$vocab, achieve_process$meta, lower.thresh = 4, upper.thresh = floor(nrow(achieve)*.7))
```

A common critique of topic modeling is that researchers have to choose the number of latent topics for the body of texts, an often subjective and difficult to replicate process. I address these concerns in the appendix, but in short, I run models that include different numbers of topics. For each model, I run a regression that estimates the effect of sponsorship by education agencies on the proportion of each article that is about each of the topics in the model. For these models, I control, similarly to the logistic regression above, for publication year and whether an article was funded by a private foundation.


```
plan(multiprocess)

many_models <- tibble(K = c(8, 10, 12, 14, 16, 18, 20)) %>%
  mutate(topic_model = future_map(K, ~stm(documents = achieve_out$documents, vocab = achieve_out$vocab, data = achieve_out$meta, K = ., prevalence = ~ s(pub_year) + sponsor_dummy, max.em.its = 75, seed = 1234, verbose = TRUE)))
```


```
heldout <- make.heldout(documents = achieve_out$documents, vocab = achieve_out$vocab)

k_result <- many_models %>%
  mutate(exclusivity = map(topic_model, exclusivity),
         semantic_coherence = map(topic_model, semanticCoherence, achieve_out$documents),
         eval_heldout = map(topic_model, eval.heldout, heldout$missing),
         residual = map(topic_model, checkResiduals, achieve_out$documents),
         bound =  map_dbl(topic_model, function(x) max(x$convergence$bound)),
         lfact = map_dbl(topic_model, function(x) lfactorial(x$settings$dim$K)),
         lbound = bound + lfact,
         iterations = map_dbl(topic_model, function(x) length(x$convergence$bound)))

k_fig1 <- k_result %>%
  transmute(K,
            `Lower bound` = lbound,
            Residuals = map_dbl(residual, "dispersion"),
            `Semantic coherence` = map_dbl(semantic_coherence, mean),
            `Held-out likelihood` = map_dbl(eval_heldout, "expected.heldout")) %>%
  gather(Metric, Value, -K) %>%
  ggplot(aes(K, Value, color = Metric)) +
  geom_line(size = 1.5, alpha = 0.7, show.legend = FALSE) +
  facet_wrap(~Metric, scales = "free_y") +
  labs(x = "K (number of topics)",
       y = NULL,
       title = "Model diagnostics by number of topics")

ggsave("fig1.png", k_fig1, height = 10, width = 16, units = "in")
```


```
k_result %>%
  transmute(K,
            `Lower bound` = lbound,
            Residuals = map_dbl(residual, "dispersion"),
            `Semantic coherence` = map_dbl(semantic_coherence, mean),
            `Held-out likelihood` = map_dbl(eval_heldout, "expected.heldout")) %>%
  gather(Metric, Value, -K) %>%
  ggplot(aes(K, Value, color = Metric)) +
  geom_line(size = 1.5, alpha = 0.7, show.legend = FALSE) +
  facet_wrap(~Metric, scales = "free_y") +
  labs(x = "K (number of topics)",
       y = NULL,
       title = "Model diagnostics by number of topics")
```


![png](./eric_topic_models_9_0.png)



```
k_result %>%
  select(K, exclusivity, semantic_coherence) %>%
  filter(K %in% c(10, 12, 14)) %>%
  unnest() %>%
  mutate(K = as.factor(K)) %>%
  ggplot(aes(semantic_coherence, exclusivity, color = K)) +
  geom_point(size = 2, alpha = 0.7) +
  labs(x = "Semantic coherence",
       y = "Exclusivity",
       title = "Comparing exclusivity and semantic coherence")
```


![png](./eric_topic_models_10_0.png)



```
map(many_models$topic_model, labelTopics)[[3]]
```


    Topic 1 Top Words:
     	 Highest Prob: children, social, parents, child, class, school, home 
     	 FREX: article, child, childs, influences, emotional, role, home 
     	 Lift: conversation, dep, deviancy, jencks, jensen, malnutrition, nurture 
     	 Score: children, child, article, parents, parent, home, social 
    Topic 2 Top Words:
     	 Highest Prob: students, group, groups, experimental, control, classes, instruction 
     	 FREX: experimental, week, treatment, reinforcement, cai, assigned, programed 
     	 Lift: cai, guides, jigsaw, quizzes, shorthand, stad, tutees 
     	 Score: experimental, guides, group, instruction, treatment, control, cai 
    Topic 3 Top Words:
     	 Highest Prob: rural, family, age, income, population, years, data 
     	 FREX: women, labor, men, employment, rural, income, residence 
     	 Lift: coal, compositional, immigration, mexicans, migrating, owned, owners 
     	 Score: women, income, rural, family, indian, labor, employment 
    Topic 4 Top Words:
     	 Highest Prob: school, schools, students, high, public, district, student 
     	 FREX: desegregation, schools, inner, desegregated, city, busing, private 
     	 Lift: castle, desegregated, ghettos, hoffer, kilgore, bused, busing 
     	 Score: schools, desegregation, school, district, districts, racial, students 
    Topic 5 Top Words:
     	 Highest Prob: grade, test, reading, children, scores, year, tests 
     	 FREX: kindergarten, arithmetic, grade, start, reading, readiness, vocabulary 
     	 Lift: caldwell, delta, piat, ppvt, slosson, binet, mrt 
     	 Score: grade, children, reading, test, scores, kindergarten, arithmetic 
    Topic 6 Top Words:
     	 Highest Prob: students, science, performance, mathematics, course, student, college 
     	 FREX: science, courses, course, checks, naep, chemistry, music 
     	 Lift: booklets, bscs, oec, pssc, xerography, chem, commentsobservations 
     	 Score: science, course, checks, naep, iscs, students, college 
    Topic 7 Top Words:
     	 Highest Prob: system, evaluation, problems, learning, performance, programs, goals 
     	 FREX: accountability, issues, standards, policy, decision, contracting, conference 
     	 Lift: auditing, technologies, transaction, audiences, campbell, cbe, coherent 
     	 Score: accountability, policy, issues, evaluation, discusses, contracting, objectives 
    Topic 8 Top Words:
     	 Highest Prob: data, variables, analysis, used, test, measures, model 
     	 FREX: regression, analyses, variable, variables, multiple, variance, validity 
     	 Lift: squares, longstep, accessioned, equations, jacobson, reanalysis, slopes 
     	 Score: variables, tell, regression, variance, data, test, analysis 
    Topic 9 Top Words:
     	 Highest Prob: self, ability, concept, high, students, low, relationship 
     	 FREX: locus, esteem, boys, anxiety, ses, self, girls 
     	 Lift: internality, ach, locus, arousal, coopersmith, crandall, externality 
     	 Score: self, boys, concept, girls, locus, sex, hall 
    Topic 10 Top Words:
     	 Highest Prob: program, programs, evaluation, project, students, title, language 
     	 FREX: title, bilingual, projects, program, esea, services, career 
     	 Lift: amigos, bedford, bslc, clovis, connell, coordinator, creole 
     	 Score: program, title, staff, migrant, evaluation, services, bilingual 
    Topic 11 Top Words:
     	 Highest Prob: information, data, presented, section, assessment, results, procedures 
     	 FREX: hearing, section, bibliography, chapter, appendix, volume, summary 
     	 Lift: dissertations, entries, alphabetical, alphabetically, collections, debt, indexed 
     	 Score: section, chapter, assessment, information, hearing, contains, summary 
    Topic 12 Top Words:
     	 Highest Prob: teachers, teacher, student, classroom, teaching, behavior, learning 
     	 FREX: teacher, behaviors, classroom, teachers, classrooms, observation, teaching 
     	 Lift: engagement, authorsjd, brophy, btes, objections, pbte, personsinstructors 
     	 Score: teacher, teachers, classroom, teaching, student, behavior, behaviors 

