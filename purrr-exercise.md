---
title: "Data Wrangling Wrapup"
author: Shreeram Senthivasan
output: github_document
---



# Intro

For this assignment, we will be using the `purrr` package to more efficiently wrangle data. More specifically we were asked to choose two of six tasks to complete... but I was only really interested in one of the six tasks. So, I chose to break form and do a deeper dive into a single task. While I recognize this might just seem like laziness, I hope by the end of this report you will agree it was still done in the spirit of the assignment.

# Task 2: Writing Functions

## Background

In my research I will be dealing with messy animal behaviour and performance metrics that could depend on any number of other factors. While the goal is of course to design efficient experiments that address targeted questions, it can be a little overwhelming to make targetted hypotheses in the first place. Luckily, I've got access to some large datasets that include some of the variables I am interested in. Accordingly, I've gotten into using AIC scores and weights to fit many different models to these datasets to help plan my future experiments.

"But now hold on a minute," you might be thinking. "That sounds a whole lot like data dredging!" And you would be right, my hypothetical strawman audience, you. But the important thing is that I only plan to use this data dredging to generate hypotheses, not to make formal inferences. Once I've got a sense of explanatory variables that might be interesting, I can design targetted experiments to test my smaller set of hypotheses.

But now that I've got that disclaimer out of the way, I imagine some of you might be wondering, "What's data dredging, and what's so bad about it anyway?" Unfortunately getting into that might be a bit beyond the scope of this assignment, so I'll just point you towards [this](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC1124898/) great primer and epidemeology case study by George Smith and Shah Ebrahim.

Anyway, I thought I would make a couple functions that make this sort of analysis easier to do since I find myself doing it to every new dataset I get a hold of.

## The Functions

The first thing I like to do is build a named list of formulas to analyse. However, I don't see an obvious way of automating the construction since I basically never want to look at every possible combination of explanatory variables. So at least for now, I will start at the next step which is building model objects. But even before that, let's build some utility functions that will be useful later.

### Utility Functions

So it turns out a lot of the functions I wrote have an eclectic mix of dependencies, so rather than checking them every time and writing a relevant error message, let's functionize it!


```r
# Attempt to load a given package and return an error message if not installed
check_dep <- function(package_name){
  tryCatch(
    suppressPackageStartupMessages(library(package_name, character.only = T)),
    error = function(e)
      paste0("This function requires the `", package_name, "` package to be installed. Please install it with: `install.packages('", package_name, "')` and try again!") %>%
      stop()
    )
}
```

This function takes the package name as an character argument and attempts to load it. The function also returns a more friendly error in case the package is not available. Unforunately, I couldn't find a way to intercept the error thrown by `library` for a missing package without using a `tryCatch` statement, which is pretty clunky. Notice that we don't need any error checking beyond this since this function isn't intended to be used by the end user, so we have full control of what gets passed in.

I also found that I end up with a lot of named lists that I would like to print. While the `print` function handles named lists pretty well, I would like to change the surrounding padding of empty lines and prefer the markdown-style header tags for separating entries. So let's build a function for that too!


```r
# Print a named list using markdown style header tags
print_list <- function(list_to_print){
  map2(list_to_print, names(list_to_print), function(x,y){
    cat("#### ")
    cat(y)
    print(x)
    cat("\n")
    }
  )
  return()
}
```

Another thing I was surprised to find myself doing a lot was double checking that the user input a valid list of model objects into many of my functions. Accordingly, I built a function to check this and return a friendly message. Unfortunately, I can't simply check the class of the input since each type of model has its own class name. Accordingly, I once again took to the clunky solution of `tryCatch` functions.


```r
# Run a chunk of code and stop with a friendly error if the provided list if the input is not valid
check_models <- function(code){
  tryCatch(
    result <- code,
    error = function(e)
      stop("Something went wrong! Are you sure you provided a list of valid model objects?")
  )

  return(result)
}
```

### Build Models

Now let's get right to it! Let's build a list of models given a list of formulas.


```r
# Convert a list of formulas and a dataset to a list of model objects
build_model_objects <- function(formulas, data, model = lm, ...){
  check_dep('purrr') # for the `map` family of functions

  # Check that both the arguments without defaults were provided
  if(missing(formulas) | missing(data))
    stop("Please provide a list of formulas and a dataset")

  # Check that every element of the formula list is a valid formula
  if(!all(map_chr(formulas, class) == "formula"))
    stop("You didn't enter a valid list of formulas.")

  # Check for other problems with tryCatch
  result <- tryCatch(
    map(formulas, model, ..., data = data),# Create and return the list of model objects
    error = function(e){ # Return an error if the model cannot be constructed
      print(e) # Print the specific error
      stop("One or more of the formulas, dataset, and model are incompatible. Please double check that:\n
            - there are no typos in the formula
            - the correct dataset was provided
            - the model function returns a model object
            - the package with the model was loaded
            - all the mandatory arguments are provided for the model")
    }
  )

  return(result)
}
```

Importantly, the function takes the list of formulas and the dataset with no defaults to build the models. By default, it will fit a linear model with `lm`, but the function should work with most functions that return a model object, such as generalized linear models with `glm` and mixed effect models with `lme4::lmer`. Since different models take different arguments, I use the elipses to generalize the model creation.

The function makes considerable use of the `map` family of functions, so I use the `check_dep` function from before to ensure `purrr` is installed and loaded. I also do some other basic error checking on the arguments passed into the function. Unfortunately I don't know how to check if the formulas are specifically valid for the given dataset without once again resorting to `tryCatch`.

### Summarize Models

Great! Now that we've built our models, one of the first things I like to do is get a quick tabular run down of *p*-values, parameter estimates, and confidence intervals for the parameters. Let's streamline this as well.


```r
# Print tidied tables of model summary stats of a list of models
summarize_models <- function(models, ...){
  check_dep("purrr")
  check_dep("car") # I like the `Anova` function for the choice of Sum of Square types
  check_dep("broom") # for tidy outputs in tables
  check_dep("knitr") # for pretty tables
  check_dep("dplyr") # for mutate_if

  check_models({
    # Print out a nice table of anova outputs
    cat("### Anova tables: \n")
    models %>%
      map(Anova, ...) %>%
      map(tidy) %>%
      map(mutate_if, # Use mutate_if to convert numerics to scientific notation as needed
        is.numeric,
        function(x) as.character(signif(x, 3))) %>%
      map(kable) %>%
      print_list

    # Print out a nice table of confidence intervals
    cat("### Confidence Intervals for Coefficients: \n")
    models %>%
      map(confint) %>%
      map(as_tibble) %>%
      map(mutate_if, # as before
        is.numeric,
        function(x) as.character(signif(x, 3))) %>%
      map(kable) %>%
      print_list
  })

  # Don't return anything at the end
  invisible()
}
```

Note the use of `check_dep` again. It was definitely worth functionizing! I also make use of `print_list` for the first time here to clean up the the lists of tables that are produced. The actual generation of the tables are also wrapped in a call of `check_model` in case the provided list of models is not valid. Finally, I use `invisible` at the end of the function since I do not want anything to be returned by this function.

### Predictions

Next, how about a tidied tibble of predicted variables? We can bind the rows together with an added column naming the model it comes from to easily plot the predictions with `ggplot2`.


```r
# Build a single tibble of predictions from a list of models
tidy_predictions <- function(models){
  check_dep("purrr")
  check_dep("broom") # for `augment` function
  check_dep("dplyr") # for bind_rows

  check_models(
    models %>%
      map(augment) %>%
      map2(names(.), ~ mutate(.x, model_name = .y)) %>%
      bind_rows()
  )
}
```

Notice that I have to pass both the list of augmented tibbles and the names of this list to the `map2` function in order to create a column of model names. This helps me distinguish when one model ends and the next begins after I bind the list of tibbles together in the next step. For a more concrete example of why this is more useful than a list of tibbles, see my worked example below.

### AIC

Now we're at the meat of the problem: AIC scores. Afterall, what I'm hoping to get out of this whole exercise is a sense of which explanatory variables are worth exploring in more detail.


```r
# Return a tidy tibble of AIC information given a list of model objects
summarize_aic <- function(models){
  check_dep("purrr")
  check_dep("tibble")

  check_models(
    tibble(
      model = names(models),
      aic = map_dbl(models, AIC),
      delta_aic = aic - min(aic),
      likelihood = exp(-0.5 * delta_aic),
      aic_weight = likelihood / sum(likelihood)
      )
  )
}
```

This is a pretty straight forward function that calculates some AIC summary statistics and returns them organized into a tibble. See the [Fitting Models](https://www.zoology.ubc.ca/~schluter/R/fit-model/) section of Dolph Schluter's amazing [R Tips](https://www.zoology.ubc.ca/~schluter/R/) website for the formulas used to calculate delta AIC, relative likelihoods, and AIC weights.

## A Toy Example

Alright, well that sure was a whole lot of coding without anything pretty to look at. Let's use the `gapminder` dataset to see what these functions will look like in practise.

Let's start by making an example list of formulas that attempt to explain life expectancy based on population, GDP per capita, and the year.


```r
example_formulas <- list(
  pop = lifeExp ~ pop,
  gdp = lifeExp ~ gdpPercap,
  year = lifeExp ~ year)
```

We can now build a list of model objects and print some summary stats about these models:


```r
models <- build_model_objects(example_formulas, gapminder)
```


```r
summarize_models(models)
```

### Anova tables: 
#### pop

|term      |sumsq  |df   |statistic |p.value |
|:---------|:------|:----|:---------|:-------|
|pop       |1200   |1    |7.21      |0.00731 |
|Residuals |283000 |1700 |NA        |NA      |

#### gdp

|term      |sumsq  |df   |statistic |p.value   |
|:---------|:------|:----|:---------|:---------|
|gdpPercap |96800  |1    |880       |3.57e-156 |
|Residuals |187000 |1700 |NA        |NA        |

#### year

|term      |sumsq  |df   |statistic |p.value  |
|:---------|:------|:----|:---------|:--------|
|year      |53900  |1    |399       |7.55e-80 |
|Residuals |230000 |1700 |NA        |NA       |

### Confidence Intervals for Coefficients: 
#### pop

|2.5 %    |97.5 %   |
|:--------|:--------|
|58.6     |59.9     |
|2.13e-09 |1.37e-08 |

#### gdp

|2.5 %    |97.5 %   |
|:--------|:--------|
|53.3     |54.6     |
|0.000714 |0.000815 |

#### year

|2.5 % |97.5 % |
|:-----|:------|
|-649  |-522   |
|0.294 |0.358  |

While it's pretty clear from the ANOVA tables that GDP per capita is the best explanatory variable for life expectancy, let's use AIC scores to formally check which model has the most support.


```r
summarize_aic(models) %>%
  kable
```



|model |      aic| delta_aic| likelihood| aic_weight|
|:-----|--------:|---------:|----------:|----------:|
|pop   | 13553.08|  702.6753|          0|          0|
|gdp   | 12850.41|    0.0000|          1|          1|
|year  | 13201.73|  351.3222|          0|          0|

Perfect! Now let's try something a little more interesting.

## Another Example

Now that we know that GDP is a better predictor of life expectancy than population or the year, we can dig a little deeper. For example, perhaps the cost of healthcare has gone down over the years. If so, we would expect an interesting interaction between GDP and the year in determining life expectancy. Let's see what the data says!


```r
gdp_formulas <- list(
  no_year = lifeExp ~ gdpPercap,
  year_no_interaction = lifeExp ~ gdpPercap + year,
  year_interaction = lifeExp ~ gdpPercap * year
  )

gdp_models <- build_model_objects(gdp_formulas, gapminder)
```


```r
summarize_models(gdp_models)
```

### Anova tables: 
#### no_year

|term      |sumsq  |df   |statistic |p.value   |
|:---------|:------|:----|:---------|:---------|
|gdpPercap |96800  |1    |880       |3.57e-156 |
|Residuals |187000 |1700 |NA        |NA        |

#### year_no_interaction

|term      |sumsq  |df   |statistic |p.value   |
|:---------|:------|:----|:---------|:---------|
|gdpPercap |70400  |1    |749       |5.77e-137 |
|year      |27500  |1    |293       |1.18e-60  |
|Residuals |160000 |1700 |NA        |NA        |

#### year_interaction

|term           |sumsq  |df   |statistic |p.value   |
|:--------------|:------|:----|:---------|:---------|
|gdpPercap      |70400  |1    |755       |8.54e-138 |
|year           |27500  |1    |295       |4.69e-61  |
|gdpPercap:year |1280   |1    |13.7      |0.000222  |
|Residuals      |159000 |1700 |NA        |NA        |

### Confidence Intervals for Coefficients: 
#### no_year

|2.5 %    |97.5 %   |
|:--------|:--------|
|53.3     |54.6     |
|0.000714 |0.000815 |

#### year_no_interaction

|2.5 %    |97.5 %   |
|:--------|:--------|
|-473     |-364     |
|0.000622 |0.000718 |
|0.212    |0.266    |

#### year_interaction

|2.5 %    |97.5 %   |
|:--------|:--------|
|-417     |-289     |
|-0.0137  |-0.00376 |
|0.174    |0.238    |
|2.23e-06 |7.27e-06 |

Great! But which model is best? Let's use AIC to check!


```r
summarize_aic(gdp_models) %>%
  kable
```



|model               |      aic| delta_aic| likelihood| aic_weight|
|:-------------------|--------:|---------:|----------:|----------:|
|no_year             | 12850.41| 280.13621|  0.0000000|  0.0000000|
|year_no_interaction | 12581.94|  11.66798|  0.0029264|  0.0029178|
|year_interaction    | 12570.27|   0.00000|  1.0000000|  0.9970822|

Would you look at that! It looks like there is very strong support for the inclusion of the interaction between GDP per capita and year in modelling life expectancy. Why don't we explore this visually as well?


```r
gdp_models %>%
  tidy_predictions %>%
  ggplot(aes(gdpPercap, lifeExp)) +
    geom_point(alpha = 0.1) +
    geom_smooth(aes(y = .fitted, colour = model_name), se = F, method = "lm") +
    scale_x_log10() + # Since gdpPercap has many small values and a few that are much larger
    theme_classic() +
    scale_colour_hue(labels = c("GDP per capita alone", "Year included, no interaction", "Year included, with interaction")) +
    labs(x = "GDP per Capita", y = "Life Expectancy", colour = "Model")
```

![plot of chunk example2_plotting](figure/example2_plotting-1.png)

Unfortunately, it seems that the three fits looks fairly similar visually. Nonetheless, hopefully you can see the utility of getting a single dataset of predicted variables for all the models.
