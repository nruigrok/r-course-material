R text analysis: quanteda
================
Kasper Welbers & Wouter van Atteveldt
2018-09

-   [This tutorial](#this-tutorial)
-   [The quanteda package](#the-quanteda-package)
-   [Importing your own data](#importing-your-own-data)

This tutorial
=============

In this tutorial you will learn how to perform text analysis using the quanteda package. In the first tutorial you were already given code to create wordclouds, compare term frequenties in two corpora and create keyword-in-context listings. This time, the goal is to understand this code better, towards becoming self-proficient in using quanteda.

This tutorial consists of two parts. First, we'll walk trough the general steps in a text analysis process and how these steps can be performed in R. Second, you will read your own data into R to perform your own analyses. For the second part, an RMarkdown template is available. Using RMarkdown is convenient because it allows you to clearly document your analyse. You can then either write the entire report in RMarkdown, or knit an .html or .docx file to easily integrate your results into a report.

The quanteda package
====================

The [quanteda package](https://quanteda.io/) is an extensive text analysis suite for R. It covers everything you need to perform a variety of automatic text analysis techniques, and features clear and extensive documentation. Here we'll focus on the main preparatory steps for text analysis, and on learning how to browse the quanteda documentation. The documentation for each function can also be found [here](https://quanteda.io/reference/index.html).

For a more detailed explanation of the steps discussed here, you can read the paper [Text Analysis in R](http://vanatteveldt.com/p/welbers-text-r.pdf) (Welbers, van Atteveldt & Benoit, 2017).

``` r
library(quanteda)
```

Importing text and creating a quanteda corpus
---------------------------------------------

The first step is getting text into R in a proper format. stored in a variety of formats, from plain text and CSV files to HTML and PDF, and with different 'encodings'. There are various packages for reading these file formats, and there is also the convenient [readtext](https://cran.r-project.org/web/packages/readtext/vignettes/readtext_vignette.html) that is specialized for reading texts from a variety of formats.

For this tutorial, we will be importing text from a csv. For convenience, we're using a csv that's available online, but the process is the same for a csv file on your own computer. The data consists of the State of the Union speeches of US presidents, with each document (i.e. row in the csv) being a paragraph. The data will be imported as a data.frame.

``` r
url = "https://raw.githubusercontent.com/ccs-amsterdam/r-course-material/master/miscellaneous/sotu_per_paragraph.csv"
d = read.csv(url)
head(d)   ## view first 6 rows
```

We can now create a quanteda corpus with the `corpus()` function. If you want to learn more about this function, recall that you can use the question mark to look at the documentation.

``` r
?corpus
```

Here you see that for a data.frame, we need to specify which column contains the text field. Also, the text column must be a character vector. In data is imported from a csv, a `character` vector is sometimes interpreted as a `factor` vector. We'll therefore first explicitly convert the column to a `character` vector.

``` r
d$text = as.character(d$text)          ## force 'text' column to be a character vector
corp = corpus(d, text_field = 'text')  ## create the corpus
corp
```

Creating the DTM (or DFM)
-------------------------

Many text analysis techniques only use the frequencies of words in documents. This is also called the bag-of-words assumption, because texts are then treated as bags of individual words. Despite ignoring much relevant information in the order of words and syntax, this approach has proven to be very powerfull and efficient.

The standard format for representing a bag-of-words is as a `document-term matrix` (DTM). This is a matrix in which rows are documents, columns are terms, and cells indicate how often each term occured in each document. We'll first create a small example DTM from a few lines of text. Here we use quanteda's `dfm()` function, which stands for `document-feature matrix` (DFM), which is a more general form of a DTM.

``` r
text <-  c(d1 = "Guns are awesome!",
           d2 = "We need better gun control!",
           d3 = "This is a sentence about cheese.")

dtm <- dfm(text, tolower=F)
dtm
```

Here you see, for instance, that the word `cheese` only occurs in the third document. In this matrix format, we can perform calculations with texts, like analyzing different sentiments of frames regarding guns, or the fact that the third sentence has nothing to do with the first two sentences.

However, directly converting a text to a DTM is a bit crude. Note, for instance, that the words `Gun` and `guns` are given different columns. In this DTM, "Gun" and "guns" are as different as "gun" and "cheese", but for many types of analysis we would be more interested in the fact that both texts are about guns, and not about the specific word that is used. Also, for performance it can be usefull (or necessary) to use fewer columns, and to ignore less interesting words such as `is`.

This can be achieved by using additional `preprocessing` steps. In the next example, we'll again create the DTM, but this time we make all text lowercase, ignore stopwords and punctuation, and perform `stemming`. Simply put, stemming removes some parts at the ends of words to ignore different forms of the same word, such as singular versus plural ("gun" or "gun-s") and different verb forms ("walk","walk-ing","walk-s")

``` r
dtm = dfm(text, tolower=T, remove = stopwords('en'), stem = T, remove_punct=T)
dtm
```

By now you should be able to understand better how the arguments in this function work. The `tolower` argument determines whether texts are (TRUE) or aren't (FALSE) converted to lowercase. `stem` determines whether stemming is (TRUE) or isn't (FALSE) used. The remove argument is a bit more tricky. If you look at the documentation for the dfm function (`?dfm`) you'll see that `remove` can be used to give "a pattern of user-supplied features to ignore". In this case, we actually used another function, `stopwords()`, to get a list of english stopwords. You can see for yourself.

``` r
stopwords('en')
```

This list of words is thus passed to the `remove` argument in the `dfm()` to ignore these words. If you are using texts in another language, make sure to specify the language, such as stopwords('nl') for Dutch or stopwords('de') for German.

There are various alternative preprocessing techniques, including more advanced techniques that are not implemented in quanteda. Whether, when and how to use these techniques is a broad topic that we won't cover today. For more details about preprocessing you can read the [Text Analysis in R](http://vanatteveldt.com/p/welbers-text-r.pdf) paper.

### Filtering the DTM

For this tutorial, we'll use the State of the Union speeches. We already created the corpus above. We can now pass this corpus to the `dfm()` function and set the preprocessing parameters.

``` r
dtm = dfm(corp, tolower=T, stem=T, remove=stopwords('en'), remove_punct=T)
dtm
```

This dtm has 23,469 documents and 20,469 features (i.e. terms), and no longer shows the actual matrix because it simply wouldn't fit.

A final step in our data preparation is to filter out some of less important terms. For didactic reasons, we'll walk through the steps. In the previous tutorial you learned that selection works by comparing a vector to a given value, which then selects all elements for which the comparison is TRUE. To filter on the frequency of terms, we can get the column sums of the matrix (the sum of all values in a column) and see whether the value is higher than a given threshold. Here we use threshold 10.

``` r
## we'll use head() in the following examples to only show the first 6 values (instead of 20,469)
head(colSums(dtm))      ## sums for first six columns/terms
head(colSums(dtm) > 10) ## terms for which value is higher than 20 
```

We can use this comparison in the `dfm_select()` function to select the columns for which the comparison is TRUE (sum is higher than 20).

``` r
dtm = dtm[, colSums(dtm) > 10]
dtm
```

Now we have about 5000 features left.

Analysis
--------

Using the dtm we can now employ various techniques. You've already seen some of them in the first tutorial, but by now you should be able to understand more about the R syntax, and understand how to tinker with different parameters.

### Word frequencies and wordclouds

Get most frequent words in corpus.

``` r
textplot_wordcloud(dtm, max_words = 50)     ## top 50 (most frequent) words
```

![](R_text_3_quanteda_files/figure-markdown_github/unnamed-chunk-11-1.png)

``` r
textplot_wordcloud(dtm, max_words = 50, color = c('blue','red')) ## change colors
```

![](R_text_3_quanteda_files/figure-markdown_github/unnamed-chunk-11-2.png)

``` r
textstat_frequency(dtm, n = 10)             ## view the frequencies 
```

You can also inspect a subcorpus. For example, looking only at Obama speeches. To subset the DTM we can use quanteda's `dtm_subset()`, but we can also use the more general R subsetting techniques (as discussed last week). Here we'll use the latter for illustration.

With `docvars(dtm)` we get a data.frame with the document variables. With `docvars(dtm)$President`, we get the character vector with president names. Thus, with `docvars(dtm)$President == 'Barack Obama'` we look for all documents where the president was Obama. To make this more explicit, we store the logical vector, that shows which documents are 'TRUE', as is\_obama. We then use this to select these rows from the DTM.

``` r
is_obama = docvars(dtm)$President == 'Barack Obama' 
obama_dtm = dtm[is_obama,]
textplot_wordcloud(obama_dtm, max_words = 25)
```

![](R_text_3_quanteda_files/figure-markdown_github/unnamed-chunk-12-1.png)

### Compare corpora

Compare word frequencies between two subcorpora. Here we (again) first use a comparison to get the is\_obama vector. We then use this in the `textstat_keyness()` function to indicate that we want to compare the Obama documents (where is\_obama is TRUE) to all other documents (where is\_obama is FALSE).

``` r
is_obama = docvars(dtm)$President == 'Barack Obama' 
ts = textstat_keyness(dtm, is_obama)
head(ts, 20)    ## view first 20 results
```

We can visualize these results, stored under the name `ts`, by using the textplot\_keyness function

``` r
textplot_keyness(ts)
```

![](R_text_3_quanteda_files/figure-markdown_github/unnamed-chunk-14-1.png)

### Keyword-in-context

As seen in the first tutorial, a keyword-in-context listing shows a given keyword in the context of its use. This is a good help for interpreting words from a wordcloud or keyness plot.

Since a DTM only knows word frequencies, the `kwic()` function requires the corpus object as input.

``` r
k = kwic(corp, 'freedom', window = 7)
head(k, 10)    ## only view first 10 results
```

The `kwic()` function can also be used to focus an analysis on a specific search term. You can use the output of the kwic function to create a new DTM, in which only the words within the shown window are included in the DTM. With the following code, a DTM is created that only contains words that occur within 10 words from `terror*` (terrorism, terrorist, terror, etc.).

``` r
terror = kwic(corp, 'terror*')
terror_corp = corpus(terror)
terror_dtm = dfm(terror_corp, tolower=T, remove=stopwords('en'), stem=T, remove_punct=T)
```

Now you can focus an analysis on whether and how Presidents talk about `terror*`.

``` r
textplot_wordcloud(terror_dtm, max_words = 50)     ## top 50 (most frequent) words
```

![](R_text_3_quanteda_files/figure-markdown_github/unnamed-chunk-17-1.png)

### Dictionary search

You can perform a basic dictionary search. In terms of query options this is less advanced than AmCAT, but quanteda offers more ways to analyse the dictionary results. Also, it supports the use of existing dictionaries, for instance for sentiment analysis (but mostly for english dictionaries).

An convenient way of using dictionaries is to make a DTM with the columns representing dictionary terms.

``` r
dict = dictionary(list(terrorism = 'terror*',
                       economy = c('econom*', 'tax*', 'job*'),
                       military = c('army','navy','military','airforce','soldier'),
                       freedom = c('freedom','liberty')))
dict_dtm = dfm(corp, dictionary = dict)
dict_dtm
```

The "4 features" are the four entries in our dictionary. Now you can perform all the analyses with dictionaries.

``` r
textplot_wordcloud(dict_dtm)
```

![](R_text_3_quanteda_files/figure-markdown_github/unnamed-chunk-19-1.png)

``` r
tk = textstat_keyness(dict_dtm, docvars(dict_dtm)$President == 'Barack Obama')
textplot_keyness(tk)
```

![](R_text_3_quanteda_files/figure-markdown_github/unnamed-chunk-20-1.png)

### Topic modeling

Topic modeling is a method for automatically findings topics in a corpus. These topics are then respresented as clusters of words that are indicative of the topic (also see [Text Analysis in R](http://vanatteveldt.com/p/welbers-text-r.pdf) or [Jacobi et al.](https://www.tandfonline.com/doi/full/10.1080/21670811.2015.1093271) for more details). It has often been used as a tool for exploring huge corpora. For smaller corpora (and the current corpus is in that sense small to moderate size) it is often less usefull.

Topic modeling is not directly supported in quanteda, but quanteda is designed to easily prepare data for use in external packages such as the `topicmodels` package. To illustrate this, we'll use the topicmodels package here to create a topicmodel. If you want to perform these steps, you'll first have to install the topicmodels package (only the first time).

``` r
install.packages('topicmodels')
```

``` r
library(topicmodels)

topmod_dtm <- convert(dtm, to = "topicmodels")

## create a topicmodel with 5 topics and some additional settings
lda <- LDA(topmod_dtm, k = 10, method = 'Gibbs')
terms(lda, 10)
```

### Scrabble word value

Not very usefull.

``` r
nscrabble('nexus')
```

Importing your own data
=======================

For the second assignment of this course, you will have to perform your own text analysis of a corpus of Dutch newspapers articles about a topic of your choosing. You can use the same selection of data as used in the first assignment.

To get the data from AmCAT into R, you will first have to download the data from AmCAT as a CSV file. Instructions for this process can be found on Canvas.

To import the data into R, you can follow the same steps as above. The first step is to use the `read_csv()` function to read the data into R.

``` r
file_location = '~/Downloads/...'
d = read.csv(file_location)
```

If you are not familiar with using paths such as '~/Downloads/data.csv', there are three easy ways to get the path. \* The easiest way is to copy the downloaded file to the folder of your R project. If you are working in an R project, the default path is the one where the project is in. So if the file called "data.csv" is in your project folder, you only have to do `read.csv("data.csv")`. \* A second way is to go to the file on your computer, and if you right-click it, you should be able to request information about the file ('file properties', 'get info', etc.), including the full path that you can copy/paste. \* The fourth way is to use Rstudio's csv importer. Simply use the files browser in the bottom-right panel in Rstudio to find the file on your computer. Then click on the file and choose "Import dataset".

There is one more additional choice to make for preparing the AmCAT data. The "headline" is stored in a separate column. This gives three options for analysis: using only the headlines, using only texts without headlines, or using both. Here we'll make an example data.frame to show all three options.

``` r
d = data.frame(id = c(1,2,3), 
               headline = c('A.','B.','C.'), 
               text = c('aaa.','bbb.','ccc.'))
d$headline = as.character(d$headline) ## make sure headline is character vector
d$text = as.character(d$text)         ## make sure text is character vector
d
```

To use only headlines:

``` r
corp = corpus(d, text_field = 'headline')  ## create the corpus
```

To use only text:

``` r
corp = corpus(d, text_field = 'text')  ## create the corpus
```

To use both, we first paste the headline and text together. We insert '' in between, which means "two hard enters" (identical to pressing enter twice in word).

``` r
d$fulltext = paste(d$headline, d$text, sep='\n\n')
corp = corpus(d, text_field = 'fulltext')  ## create the corpus
```

Now you can create a DTM and perform analyses as shown above. Please look at the description for assignment two, so that you can already work towards your analyses.