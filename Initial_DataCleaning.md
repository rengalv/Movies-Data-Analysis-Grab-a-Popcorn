---
title: "Grab some popcorn: Analysis of TMDB Database"
author: "Renganathan Lalgudi Venkatesan,
          Suresh Karthik Balasundaram"
date: "30 March 2018"
output: html_document
theme: cayman
---

## Introduction

The movie industry has been one of the forerunners in the entertainment sector. Every year hundreds of movies get released but not all of them are successful. The aim of the project is to analyze the TMDB movie dataset which has details about the movies, their production cost and revenue generated along with rating inforamtion. We want to come up with insights by analyzing the dataset. 


### Problem Statement

As mentioned on the [Kaggle](https://www.kaggle.com/tmdb/tmdb-movie-metadata) website, the major question we are trying to answer here is what can we say about the success of a movie before it is released? Are there certain companies (Pixar?) that have found a consistent formula? Given that major films costing over $100 million to produce can still flop, this question is more important than ever to the industry. 


### Analysis and Business Impact proposed

We are trying to analyze the dataset to find answers to the questions posed in the problem statement. We will start with a univariate analysis and then move ahead with a mutivariate analysis to understand the impact of certain factors in determining the success of the movie. The success of a movie is measured in terms of the following metrics:

* **Return on Investment**: Which will be a measure of the Revenue generated
* **Ratings by Public**: Which will also take into account the number of votes posted


## Packages Required

To start with the data analysis, we have used the following R packages:

* **readr :** To read the dataset which is in the csv format 
* **jsonlite :** To extract the columns that are in json format in the dataset 
* **tidyr :** To perform data cleaning operations
* **dplyr :** To manipulate the dataset
* **DT :** To display the cleaned dataset in a tabular format
* **knitr :** To display a table of column names along with datatypes 

```{r message = FALSE, warning = FALSE}
library(readr)
library(jsonlite)
library(tidyr)
library(dplyr)
library(DT)
library(knitr)
```

## Data Preparation {.tabset .tabset-fade}

### Data Source

#### Original Source
The dataset is obtained from [Kaggle](https://www.kaggle.com/tmdb/tmdb-movie-metadata/data). 


#### Hosted in Github
We have downloaded the dataset from this source and hosted in our custom [GitHub profile](https://github.com/rengalv/Movies-Data-Analysis-Grab-a-Popcorn) for creating a robust source of data. This will make sure that we can even have mutiple versions of the data along with corresponding analysis making it easier for code sharing.

### Data Importing

#### Fetching the Data from GitHub
We perform the data importing from the github profile where we have hosted the data. The url for the data is set to the variable `url` and the data is read into the object `df` 

```{r message = FALSE, warning = FALSE}
#URL to read the data from
url <- "https://raw.githubusercontent.com/rengalv/Movies-Data-Analysis-Grab-a-Popcorn/master/tmdb_5000_movies.csv"

#Reading the csv file from the URL
movies <- read_csv(url,col_names = TRUE,na = "NA")

#Preview of the data dimensions and column names
dim(movies)

#Examining the column names in the dataset
colnames(movies)

```


#### Looking at the structure of the Dataset

When we examine the Structure of the dataset, we find that the columns can be in any of the following datatypes:

* Interger
* Numeric
* Character
* Date

We also find that even though some columns have a class as `chr`, they are actually in JSON format which needs to be converted to columns with one of the base r datatypes.

### Data Cleaning

#### Removing Duplicates
The first thing we wanted to do was to remove the duplicate values from the dataset. We did this by checking if there were two rows in the dataset that had the same movue name.

```{r }
movies <- movies[!duplicated(movies$title), ]

```

The de-duplicated dataset has the following dimensions:
```{r }
dim(movies)

```


#### Working with the JSON Format

We notice from the dataset is that it has columns with data in the JSON format. So, we need to bring those columns to the base datatypes in r so that we can perform analysis.

Following are the columns found to be in JSON format:

* Genres: id, name
* Keywords: id, name
* Production Companies: name, id
* Production Countries: iso_3166_1, name
* Spoken Languages: iso_639_1, name

We worked on converting each of these columns into separate dataframes.

Since the implementation was replicable for each of the columns in the JSON format, we wrote a function to implement the same. Finally we have 5 new data frames which can then be merged with our base `movies` dataset.

##### **Function to convert the JSON column to a dataframe**
```{r}
json_to_df <- function(df, column){
  column_1 <- df[apply(df[,column],1,nchar)>2,]  
   
  list_1 <- lapply(column_1[[column]], fromJSON)
  values <- data.frame(unlist(lapply(list_1, function(x) paste(x$name,collapse = ","))))
  
  final_df <- cbind(column_1$id, column_1$title, values)
  names(final_df)  <- c("id", "title", column)
  return(final_df)
  
}
```

##### **Calling the json_to_df() to generate the dataframes for all the JSON Columns**

```{r }
genres_df <- json_to_df(movies, "genres")
keywords_df <- json_to_df(movies, "keywords")
prod_cntry_df <- json_to_df(movies, "production_countries")
prod_cmpny_df <- json_to_df(movies, "production_companies")
spoken_lang_df <- json_to_df(movies, "spoken_languages")

```


#### Merging the dataset
Now that we have created them as separate dataframes, we want to combine all these dataframes with the `movies` dataframe to get the final dataset with which we will be using for the analysis going forward

For that, we first remove the JSON columns present in the `movies` dataset and then combine the new columns we have created for all the JSON columns

```{r warning = FALSE}
#Subset the movies dataframe by removing the JSON columns
movies_1 <- subset(movies, select =  -c(genres,keywords,production_companies, production_countries,spoken_languages))

#Join the columns from all the generated dataframes from previous step
movies_new <- movies_1 %>%
  full_join(genres_df, by = c("id", "title")) %>%
  full_join(keywords_df, by = c("id", "title")) %>%
  full_join(prod_cntry_df, by = c("id", "title")) %>%
  full_join(prod_cmpny_df, by = c("id", "title")) %>%
  full_join(spoken_lang_df, by = c("id", "title"))

#Have a look at the final dataset
glimpse(movies_new)
size <- dim(movies_new)
```
We find that there are `r size[1]` observations and `r size[2]` columns.

#### **Missing values**

We wanted to check there were how many rows in the data set with complete values for all  the columns. 

```{r }
complete_data <- sum(complete.cases(movies_new))
```

We find that there are `r complete_data` rows with no missing data in the dataset. We did not remove any of the missing values for now. We are planning to look at each column separately and see if we can perform any imputations (if required) while performing the analysis.

### Data Preview
The table below is the preview of the final dataset. We have printed the first 100 rows of the dataset. 

Each row corresponds to a movie and each column is a feature corresponding to the movie.

```{r}
datatable(head(movies_new,100))
```

### Summary of Data

The final dataset after performing data cleaning has the following columns. The class of each of the column is also presented below.

```{r warning = FALSE}
col <- data.frame(sapply(movies_new, class))
Row_names <- rownames(col)
class <- col[,1] 

Data_types <- cbind(Column = Row_names, Class = as.character(class))
Data_types <- Data_types[2:nrow(Data_types),]

kable(Data_types)

```

## Proposed EDA





