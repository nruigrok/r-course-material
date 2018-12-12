Reshaping data: wide, long, and tidy
================
Wouter van Atteveldt & Kasper Welbers
2018

-   [Introduction: Long and Wide data](#introduction-long-and-wide-data)
-   [Before we start: Getting and cleaning the data](#before-we-start-getting-and-cleaning-the-data)
-   [Wide to long: Gathering data](#wide-to-long-gathering-data)
-   [A more complicated case: wealth inequality](#a-more-complicated-case-wealth-inequality)
    -   [Gathering columns (wide to long)](#gathering-columns-wide-to-long)
    -   [Separating columns (splitting one column into two)](#separating-columns-splitting-one-column-into-two)
    -   [Spreading a column (long to wide)](#spreading-a-column-long-to-wide)
-   [Recoding data](#recoding-data)
-   [Combining and plotting data](#combining-and-plotting-data)
    -   [Tidyness as a matter of perception](#tidyness-as-a-matter-of-perception)

This tutorial discusses how to *reshape* data, particularly from long to wide format and vice versa. It follows [Chapter 12 of the R4DS book](http://r4ds.had.co.nz/tidy-data.html).

Introduction: Long and Wide data
================================

In a data matrix, normally the rows consist of observations (cases, respondents) and the columns of variables containing information about these cases. As explained in the chapter referenced above, in the `tidyverse` philosophy data is said to be tidy if each observation (case) is exactly one row, and each measurement (variable) is exactly one column. Data is said to be 'untidy' if for example the columns represents measurement years in a longitudinal data set, where each year is really its own observation. Thus, using the names of the tidyverse functions, there is a need to `gather` information from multiple columns into a single column, or inversely to `spread` it from one column accross multiple columns.

Note that what is called 'tidy' here is what is also often called a *long* data format, with most information in separate rows, while a *wide* data format contains most information in separate columns. Another way to view the functions is that `gather` transforms data from wide to long (also called variables to cases), and `spread` converts data from long to wide (cases to variables).

Before we start: Getting and cleaning the data
==============================================

For this tutorial we will use the data from Piketty's Capital in the 21st Century. In particular, we will work with the data on income inquality, with the goal of making a graph showing the evolution of income inequality in different countries.

For this, we load the income inequality data set and remove missing values.

``` r
library(tidyverse)
base = "https://raw.githubusercontent.com/ccs-amsterdam/r-course-material/master/data"
income_raw = read_csv(paste(base, "income_topdecile.csv", sep = "/")) %>% na.omit
income_raw
```

Note that Piketty's data is published as excel files with complex (multi-row) headers, so we uploaded a cleaned version of the data to our github repository. Even though this data is slightly cleaner, you will see that there is plenty to be done to get this data in shape!

Wide to long: Gathering data
============================

As you can see (after getting rid of missing values), the data stores the share of income going to the top decile/percentile of earners per decade per country. This data is 'wide', in the sense that the columns contain observations, while it is normally better (or tidier) to have the observations in different rows. As we will see, that will make it easier to combine or adjust the data later on.

In tidyverse, the function used for transforming data from columns (wide) to rows (long) is `gather`: the idea is that you gather the information from multiple columns into a single column. The syntax for calling gather is as follows: `gather(columns, key=key_column, value=value_column)`. The first argument, columns, is a list of columns which need to be gathered in a single column. You can list the columns one by one, or specify them as a sequence `first:last`. The `key=` argument specifies the name of a new column that will hold the names of the observations, i.e. the old column names. In our case, that would be `country` since the columns refer to countries. The `value=` argument specifies the name of the new column that will hold the values, in our case the top-decile of incomes.

Note that (similar to mutate and other tidyverse functions), the column names don't need to be quoted as long as they don't contain spaces or other characters that are invalid in R names. Rather than naming the columns to include in the gathering, you can also name the variables that should be excluded (in our case only Year), and it will gather all non-excluded columns.

``` r
# both calls below have the same result, you can use the one that makes most sense
income = income_raw %>% gather(U.S.:Europe, key=country, value=income_topdecile)
income = income_raw %>% gather(-Year, key=country, value=income_topdecile)  
income
```

As you can see, every row now specifies the income inequality in a single country in a single year (or actually, decade).

A more complicated case: wealth inequality
==========================================

Let's now look at the wealth inequality data from Piketty. Of course, there is nothing inherently more complex about wealth inequality than income inequality (from the data side, at least), but in this particular case the columns contain the country as well as the measurement level (top-decile, percentile, or promille):

``` r
wealth_raw = read_csv(paste(base, "wealth_inequality.csv", sep = "/"))
wealth_raw
colnames(wealth_raw)
```

As you can see from the column specification or the output, it somehow parsed the UK promille column as character (textual) data rather than as double (numeric) data. On inspection this is caused by there not being any values whatsoever. This will later cause trouble as it will force all columns to become textual. So, we should now either convert the column to numeric, or simply drop it:

``` r
# you can use either option below:
wealth_raw = mutate(wealth_raw, `United Kingdom: top promille`=as.numeric(`United Kingdom: top promille`))
wealth_raw = select(wealth_raw, -`United Kingdom: top promille`)
```

Gathering columns (wide to long)
--------------------------------

We will tidy this data in three steps. First, we gather the columns into a single column with all measurements. Then, we separate the country from the measurement level. Finally, we spread the measurement levels to columns again (since they are measurements on the same observation).

The first step is the same as above: we gather all columns except for the year column into a single column:

``` r
# as always, you can use either %>% notation or specify the data as first argument. so, the below commands are equivalent
wealth = gather(wealth_raw, -Year, key="key", value="value")
wealth = wealth_raw %>% gather(-Year, key="key", value="value")
wealth
```

Separating columns (splitting one column into two)
--------------------------------------------------

The next step is to split the 'key' column into two columns, for country and for measurement. This can be done using the `separate` command, for which you specify the column to split, the new column names, and what `sep`arator to split on:

``` r
wealth = wealth %>% separate(key, into = c("country","measurement"), sep=":")
wealth
```

The `measurement` column is quoted in the output because it stars with a space. We could resolve this by specifying `sep=": "` (i.e. adding the space to the separator). We can also solve this by changing the column after the split with `mutate`. The code below removes the space using the `trimws` (trim white space) function:

``` r
wealth %>% mutate(measurement = trimws(measurement))
```

We can also use `sub` to search and replace (substitute) within a column, in this case changing " top " into "capital\_top\_":

``` r
wealth = wealth %>% mutate(measurement = sub(" top ", "capital_top_", measurement))
wealth
```

Spreading a column (long to wide)
---------------------------------

The wealth data above is now 'too long' to be tidy: the measurement for each country is spread over multiple rows, listing the three different measurement levels (decile, percentile, promille). In effect, we want to undo one level of gathering, by `spread`ing the column over multiple columns.

Ths syntax for the spread call is similar to that for gather: `spread(data, key=key_column, value=value_column)`, but now the key and value columns refer to the existing columns that should be spread into multiple new columns. In effect, for each `key` value a new column will be created, with the corresponding `value` in the cell:

``` r
wealth = wealth %>% spread(key=measurement, value=value)
wealth
```

So now each row contains three measurements (columns, variables) relating to each observation (country x year).

Recoding data
=============

We want to combine the two 'tidy' data sets that we created above. In principle, this should now be really easy as they both use the same join key (Year and country). However, the country names are not identical in both.

You can look at the frequency of values in each column and see the problem:

``` r
table(wealth$country)
table(income$country)
```

Just for fun, below is some code that gives an overview of which set contains which country:

``` r
data.frame(country = union(wealth$country, income$country)) %>%
  mutate(wealth=country %in% wealth$country, income=country %in% income$country)
```

So, the main problem is that the UK and US are named differently in both sets. We could resolve this with a set of `ifelse` statements as done in [a previous tutorial](r-tidy-3_7-visualization.md). However, we can also use the tidyverse `recode` command which was made for this purpose. You call recode by first specifying the column name, and then any `"old"="new"` pairs to recode.

Because I really dislike spaces and periods in identifiers, I recode both to either UK or US:

``` r
wealth = wealth %>% mutate(country = recode(country, "United Kingdom"="UK", "United States"="US"))
income = income %>% mutate(country = recode(country, "U.K."="UK", "U.S."="US"))
table(wealth$country)
table(income$country)
```

Combining and plotting data
===========================

Now, we are finally ready to combine our data sets. One of the advantages of 'tidying' the data is that it becomes easier to combine them, since the 'join keys' are now the same in both data sets: Year and country. So, we can immediately do an inner join:

``` r
inequality = inner_join(income, wealth)
inequality
```

Since this data is also `tidy`, we can immediately plot it, for example to make a plot of income inequality per country. For more information on plotting, please see the [tutorial on visualization](r-tidy-3_7-visualization.md):

``` r
ggplot(inequality) + geom_line(aes(x=Year, y=income_topdecile, colour=country))
```

As you can see, inequality in general dipped after the recession and especially second world war, but is now climing to 'belle epoque' levels again, especially in the US (which actually used to have less inequality).

``` r
ggplot(inequality) + geom_line(aes(x=Year, y=capital_top_decile, colour=country))
```

This broadly shows the same pattern: a big drop in inequality with the destruction of the depression and War, followed by steadily rising inequality in the last decades.

Tidyness as a matter of perception
----------------------------------

As a last exercise, suppose we would like to plot wealth and capital inequality in the same figure as separate lines. You can do this with two separate geom\_line commands, and e.g. use a dashed line for income inequality:

``` r
ggplot(inequality) + geom_line(aes(x=Year, y=capital_top_decile, colour=country)) + 
  geom_line(aes(x=Year, y=income_topdecile, colour=country), linetype="dashed")
```

This works, but it would be nice if we could specify the measurement as colour (or type) and have ggplot automatically make the legend. To to this, the different measurements need to be in rows rather than in columns. In other words, data that is tidy from one perspective can be 'too wide' for another.

Let's gather the data into a single column, and plot the result for the US:

``` r
inequality2 = gather(inequality, income_topdecile:capital_top_promille, key="measurement", value="value")
inequality2 %>% filter(country=="US") %>% ggplot() + geom_line(aes(x=Year, y=value, linetype=measurement))
```

We can also plot only top-decile capital and income in a paneled plot. Note the use of extra options to set legend location and title, vertical label, and main title text and location (horizontal justification):

``` r
inequality2 %>% filter(measurement %in% c("income_topdecile", "capital_top_decile") & country != "Europe") %>% 
  ggplot() + geom_line(aes(x=Year, y=value, linetype=measurement)) + facet_wrap(~ country, nrow = 2) +
  scale_linetype_discrete(name="Variable:", labels=c("Capital", "Income")) +
  theme(legend.position="bottom", plot.title = element_text(hjust = 0.5)) +
  ylab("Inequality") + 
  ggtitle("Capital and income inequality over time")
```

![](img/reshape_inequality-1.png)
