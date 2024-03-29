# Data Science Capstone Project <br/> Milestone Report
Sebastian Garcia  
July 30, 2017  



# Introduction

This report will be applying data science in the area of natural language processing in order to help mobile users to enter text quickly and with less errors. The following lines addressing the data extraction, cleaning and text mining of the so called HC Copora. This report is part of the data science capstone project of [Coursera](https://www.coursera.org) and [Swiftkey](http://swiftkey.com/). The plots, code chunks and remarks will explain the reader the first steps to build the word prediction application.

This document describes the procedures and code that is use to achieve the objective expressed in project's first milestone report:

1. Demonstrate that you have downloaded the data from the source site and loaded into R.
2. Create a basic report of summary statistics about the data sets.
3. Report any interesting findings that you amassed so far.
4. Get feedback on your plans for creating a prediction algorithm and Shiny app.

The final objective of the project is to create a "next word" predictor application that can be used in data input applications to suggest the user a list of alternatives for the next word when writing text. This model can be useful for assisting users when writing text in limited devices like mobile phones.

# Getting and Cleaning the Data
The main objetive of this task is to get the data from the source and then clean and sample it.

## Getting and loading the Data
The data for this project is provided by Coursera, it can be downloaded here [Project Dataset](https://d396qusza40orc.cloudfront.net/dsscapstone/dataset/Coursera-SwiftKey.zip), the following code will download, unzip and save the files related to English language into the */data* folder.


```r
#Set data destination path
destination <- file.path(getwd(), "data")

#Create data folder if doesn't exists
if(!file.exists(destination)){
    dir.create(destination)
}

#Download file if doesn't exists
if(!file.exists(file.path(destination,"Coursera-SwiftKey.zip"))){
    download.file("https://d396qusza40orc.cloudfront.net/dsscapstone/dataset/Coursera-SwiftKey.zip", 
                  url = file.path(destination,"Coursera-SwiftKey.zip"))
}

#Unzip english files
sourceFiles = c('en_US.twitter.txt','en_US.blogs.txt','en_US.news.txt')
if(!all(file.exists(paste(destination,sourceFiles,sep="/")))){
    unzip(file.path(destination,"Coursera-SwiftKey.zip"), files = paste('final/en_US/',sourceFiles,sep=""), exdir=destination, junkpaths = TRUE)
}
```

Now, we load the data into datasets.

```r
maxRows = 100000000

rBlogs <- readLines("./data/en_US.blogs.txt", encoding = "UTF-8", skipNul=TRUE, n=maxRows)
rNews <- readLines("./data/en_US.news.txt", encoding = "UTF-8", skipNul=TRUE, n=maxRows)
rTwitter <- readLines("./data/en_US.twitter.txt", encoding = "UTF-8", skipNul=TRUE, n=maxRows)
```

## Sampling the Data

In order to enable faster data processing, data samples from all three sources were generated, mixed together and saved to a text file.


```r
sampleSize = 10000
# Sample generation
sampleTwitter <- rTwitter[sample(1:length(rTwitter),sampleSize)]
sampleNews <- rNews[sample(1:length(rNews),sampleSize)]
sampleBlogs <- rBlogs[sample(1:length(rBlogs),sampleSize)]

# The samples are mixed in a common dataset and writen to a file
textSample <- c(sampleTwitter,sampleNews,sampleBlogs)

# Save sample
writeLines(textSample, "./MilestoneReport/textSample.txt")
```



## Creating a clean Corpus
By using the [tm package](http://tm.r-forge.r-project.org/index.html) the sample data gets *cleaned*. 
By cleaning I mean that the text data is converted into lower case, further punction, numbers and URLs are removed. Next, stop and profanity words are removed from the text sample. At the end we are getting a clean text corpus which enables an easy subsequent processing.

Next, the final corpus is saved to a file for further use.

The used profanity words can be inspected [in this Github Repository](https://github.com/garciasebastian/DSCapstoneProject/blob/master/MilestoneReport/profanityfilter.txt).


```r
## Build the corpus, and specify the source to be character vectors 
cleanCorpus <- Corpus(VectorSource(textSample))

## Make it work with the new tm package
cleanCorpus <- tm_map(cleanCorpus,
                      content_transformer(function(x) 
                              iconv(x, to="UTF-8", sub="byte")))

## Convert to lower case
cleanCorpus <- tm_map(cleanCorpus, content_transformer(tolower))

## Remove punction, numbers, URLs, stop, profanity 
cleanCorpus <- tm_map(cleanCorpus, content_transformer(removePunctuation))
cleanCorpus <- tm_map(cleanCorpus, content_transformer(removeNumbers))
removeURL <- function(x) gsub("http[[:alnum:]]*", "", x) 
cleanCorpus <- tm_map(cleanCorpus, content_transformer(removeURL))
cleanCorpus <- tm_map(cleanCorpus, stripWhitespace)
cleanCorpus <- tm_map(cleanCorpus, removeWords, stopwords("english"))
profanityWords <- read.table("./MilestoneReport/profanityfilter.txt", header = FALSE, col.names = "word")
cleanCorpus <- tm_map(cleanCorpus, removeWords, profanityWords$word)
cleanCorpus <- tm_map(cleanCorpus, stripWhitespace)

## Saving the final corpus
saveRDS(cleanCorpus, file = "./MilestoneReport/finalCorpus.rds")
```



# Exploring the Data

## Summary Statistics

The following chunk of code summarizes the information for the source files and sample file.

```r
fileSummary <- data.frame(fileName = c("Blogs","News","Twitter","Aggregated Sample"))

## Checking the size and length of the files and calculate the word count
fileSummary$fileSize = c(
    file.info(paste(destination,"en_US.blogs.txt",sep="/"))$size / 1024.0 / 1024.0,
    file.info(paste(destination,"en_US.news.txt",sep="/"))$size / 1024.0 / 1024.0,
    file.info(paste(destination,"en_US.twitter.txt",sep="/"))$size / 1024.0 / 1024.0,
    file.info("./MilestoneReport/textSample.txt")$size / 1024.0 / 1024.0)
fileSummary$fileSize = round(fileSummary$fileSize,3)

fileSummary$lineCount = c(length(rBlogs), length(rNews), length(rTwitter), sampleSize*3)

fileSummary$wordCount = c(
    sum(sapply(gregexpr("\\S+", rBlogs), length)),
    sum(sapply(gregexpr("\\S+", rNews), length)),
    sum(sapply(gregexpr("\\S+", rTwitter), length)),
    sum(sapply(gregexpr("\\S+", textSample), length))
)

colnames(fileSummary) <- c("File Name", "File Size in Megabytes", "Line Count", "Word Count")
```



The following table provides an overview of the imported data. In addition to the size of each data set, the number of lines and words are displayed. 


File Name            File Size in Megabytes   Line Count   Word Count
------------------  -----------------------  -----------  -----------
Blogs                               200.424       899288     37334131
News                                196.278        77259      2643969
Twitter                             159.364      2360148     30373585
Aggregated Sample                     4.832        30000       883749



## Word Cloud

A word cloud provides a first overview of the word frequencies in a easily view. The word cloud displays the data of the aggregated sample file in terms of word frequency.


```r
#Load Corpus
finalCorpus <- readRDS("./MilestoneReport/finalCorpus.rds")
termMatrix <- TermDocumentMatrix(finalCorpus)

words <- data.frame(word = termMatrix$dimnames$Terms, 
                   freq = sapply(c(1:termMatrix$nrow), #termMatrix$nrow
                                  FUN = function(x){sum(termMatrix[x,])}, simplify=TRUE), 
                   stringsAsFactors = FALSE) %>%
            arrange(desc(freq))
        
words <- words[1:1000,]

#Clean memory 
rm(finalCorpus)
gc()

#Create word cloud
wordcloud(words$word,words$freq,
          c(5,.3),50,
          random.order=FALSE,
          colors=brewer.pal(8, "Dark2"))
```

<img src="MilestoneReport_files/figure-html/WordCloud-1.png" style="display: block; margin: auto;" />



# The N-Gram Tokenization

In Natural Language Processing (NLP) an *n*-gram is a contiguous sequence of n items from a given sequence of text or speech.

The following function is used to extract *n*-grams from the cleaned text corpus, where *n* is the number of consecutive words.


```r
#Read corpus and convert to dataframe
finalCorpus <- readRDS("./MilestoneReport/finalCorpus.rds")
finalCorpusDF <- data.frame(text=unlist(finalCorpus),stringsAsFactors = FALSE)
rownames(finalCorpusDF) <- NULL

#Create tokenizer function
ngramTokenizer <- function(theCorpus, ngramCount, topn) {
        ngramFunction <- NGramTokenizer(theCorpus, 
                                Weka_control(min = ngramCount, max = ngramCount, 
                                delimiters = " \\r\\n\\t.,;:\"()?!"))
        ngramFunction <- data.frame(table(ngramFunction))
        ngramFunction <- ngramFunction[order(ngramFunction$Freq, 
                                             decreasing = TRUE),]
        colnames(ngramFunction) <- c("word","wcount")
        ngramFunction
}
```
 
By the usage of the tokenizer function for the *n*-grams a distribution of the following top 10 words and word combinations can be inspected. Unigrams are single words, while bigrams are two word combinations and trigrams are three word combinations.

## Top 20 Unigrams

The following graphs show the Top 20 Unigrams in the sample

<img src="MilestoneReport_files/figure-html/Unigrams-1.png" style="display: block; margin: auto;" />


## Top 20 Bigrams

The following graphs show the Top 20 2-grams in the sample

<img src="MilestoneReport_files/figure-html/Bigrams-1.png" style="display: block; margin: auto;" />

## Top 20 Trigrams

The following graphs show the Top 20 3-grams in the sample

<img src="MilestoneReport_files/figure-html/Trigrams-1.png" style="display: block; margin: auto;" />



# Interesting Findings

+ Loading the dataset costs a lot of time. The processing is time consuming because of the huge file size of the dataset. By avoiding endless runtimes of the code, it was indispensable to create a data sample for text mining and tokenization. Nedless to say, this workaround decreases the accuracy for the subsequent predictions.

+ Removing all stopwords from the corpus is recommended, but, of course, stopwords are a fundamental part of languages. Therefore, consideration should be given to include these stop words in the prediction application again.

# Next Steps For The Prediction Application

As already noted, the next step of the capstone project will be to create a prediction application. 
To create a smooth and fast application it is absolutely necessary to build a fast prediction algorithm. This is also means, I need to find ways for a faster processing of larger datasets. Next to that,  increasing the value of n for n-gram tokenization will improve the prediction accuracy. All in all a shiny application will be created which will be able to predict the next word a user wants to write.

# All Used Code Scripts

All used code snippets to generate this report can be viewed in this [repository](https://github.com/garciasebastian/DSCapstoneProject/tree/master).

# Session Information

For reproducibility, R session information is provided in the following output, variables where cleaned during code execution due to high memory requirements for text processing.

```
## R version 3.3.2 (2016-10-31)
## Platform: x86_64-w64-mingw32/x64 (64-bit)
## Running under: Windows 10 x64 (build 15063)
## 
## locale:
## [1] LC_COLLATE=English_United States.1252 
## [2] LC_CTYPE=English_United States.1252   
## [3] LC_MONETARY=English_United States.1252
## [4] LC_NUMERIC=C                          
## [5] LC_TIME=English_United States.1252    
## 
## attached base packages:
## [1] stats     graphics  grDevices utils     datasets  methods   base     
## 
## other attached packages:
##  [1] dplyr_0.5.0            scales_0.4.1           ggthemes_3.4.0        
##  [4] ggplot2_2.2.1          googleVis_0.6.2        stringi_1.1.5         
##  [7] DT_0.2                 stringr_1.2.0          wordcloud_2.5         
## [10] rJava_0.9-8            RWeka_0.4-34           slam_0.1-40           
## [13] SnowballC_0.5.1        tm_0.7-1               NLP_0.1-10            
## [16] qdap_2.2.5             RColorBrewer_1.1-2     qdapTools_1.3.3       
## [19] qdapRegex_0.7.2        qdapDictionaries_1.0.6 RWekajars_3.9.1-3     
## 
## loaded via a namespace (and not attached):
##  [1] gtools_3.5.0        venneuler_1.1-0     reshape2_1.4.2     
##  [4] reports_0.1.4       colorspace_1.3-2    htmltools_0.3.5    
##  [7] yaml_2.1.14         chron_2.3-50        XML_3.98-1.6       
## [10] DBI_0.6-1           plyr_1.8.4          munsell_0.4.3      
## [13] gtable_0.2.0        htmlwidgets_0.8     codetools_0.2-15   
## [16] evaluate_0.10       labeling_0.3        knitr_1.15.1       
## [19] gender_0.5.1        parallel_3.3.2      xlsxjars_0.6.1     
## [22] highr_0.6           Rcpp_0.12.10        backports_1.0.5    
## [25] gdata_2.17.0        plotrix_3.6-5       xlsx_0.5.7         
## [28] jsonlite_1.4        openNLPdata_1.5.3-2 gridExtra_2.2.1    
## [31] digest_0.6.12       grid_3.3.2          rprojroot_1.2      
## [34] tools_3.3.2         bitops_1.0-6        magrittr_1.5       
## [37] RCurl_1.95-4.8      lazyeval_0.2.0      tibble_1.3.0       
## [40] data.table_1.10.4   assertthat_0.1      rmarkdown_1.4      
## [43] openNLP_0.2-6       R6_2.2.0            igraph_1.0.1
```
