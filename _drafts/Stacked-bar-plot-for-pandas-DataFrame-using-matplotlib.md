---
layout: post
title: Stacked bar plot for pandas DataFrame using matplotlib
date: 2018-02-24
categories: python statistics pandas matplotlib visualization dataviz 
permalink: /stacked-bar-plot-from-pandas-dataframe/
---

This post describes an easy-to-use implementation of stacked bar plot in python straight from a pandas DataFrame (and an .xlsx file).



#### TL;DR

Want to plot a stacked bar plot on your pandas DataFrame? No problem. Just use the function `plot_stacked_bar_for_df(df)` available [in this repository](https://github.com/neuhofmo/stacked_barplot_for_pandas_df). Good luck!



### Easily create stacked bar plots from pandas DataFrames

In my previous post, [Chi-square and post-hoc tests in python](https://neuhofmo.github.io/chi-square-and-post-hoc-in-python/), I described my method of performing chi-square test on tables with multiple categories and multiple groups. After running the chi-square test, followed by additional post-hoc tests and [p-value corrections](http://www.biostathandbook.com/multiplecomparisons.html), we found out that some groups are different in their distribution of categories than others.  

The statistical analysis is crucial for this study, and lets us know if our results have any real-world significance. It seems like they do; we really did find a difference between our cases and control groups. But how can we communicate this message? That's where plotting comes in.



### Why use a stacked bar plot?

We want to compare distribution of discrete categories between different groups. I can think of several ways to do it: 

- We can plot bars representing the values of the different categories for each group. This option is compact and coherent but doesn't necessarily presents the *distribution* of the categories for each group of patients/controls.
- We can plot a pie-chart for each group. A pie-chart emphasizes the distribution but forces us to plot a separate pie for each group, which ends up taking a lot of space (which makes it a little annoying when we are talking about large datasets). 

Stacked bar plots tends to be a good compromise between displaying the distribution while keeping a nice, clean, easy-to-track figure.



### Our data

Just as a reminder, our original dataset looks like this:

| Cell_line    | Normal | Abnormal | Bad  |
| ------------ | ------ | -------- | ---- |
| **Control**  | 5126   | 137      | 5    |
| **Patient1** | 1597   | 20       | 7    |
| **Patient2** | 862    | 12       | 4    |
| **Patient3** | 861    | 22       | 7    |
| **Patient4** | 4546   | 886      | 142  |

Where the data represent the number of appearances of liver cells of each form in microscope images. Note that the total number of cells examined is not identical for all of the groups; this is very common in biology (we can't necessarily generate the same amount of cells for each image, and we can't always pick only some of the cells because then our sample would be biased). It's okay, because we don't compare the total amount of the cells in the different groups, but the *proportion* of each category of cells (normal/abnormal/bad) within the group. That's what the $$\chi^2$$ test does - compares the different proportions (or distribution among categories)[^1].

So if we normalize each of the groups, we would be able to show what we really want - how the distribution changes between the different groups.

Let's first load the data into a dataframe, like we did in the previous post:

```python
import pandas as pd
df = pd.read_excel('groups_sum_demo.xlsx', index_col='Cell_line')
```



In order to normalize the dataframe, we can use the following function:

```python
def get_ratio_table(df):
    """The function receives a df and returns a row-ratio df."""
    df_sum = df.sum(axis=1)  # row sum
    return df.div(df_sum, axis='index')  # normalized dataframe
```

The function simply divided each cell by the sum of the cells in its row (representing the total number of observations for this group).   

   

----

[^1]: Note that chi-square test needs the absolute numbers and not proportions in order to work properly, so we are normalizing the DataFrame only for the plotting part.

### Stacked bar plot with python

I will use the tricky [stacked bar plot](https://matplotlib.org/examples/pylab_examples/bar_stacked.html) feature of [matplotlib.pyplot](https://matplotlib.org/api/pyplot_api.html) to plot the ratio dataframe. In the pyplot API, the difference between a regular bar chart and a stacked bar chart only lies in the `bottom` argument in the bar function, so it seems pretty simple to implement at first. 

However, as you can see from the [example code](https://matplotlib.org/examples/pylab_examples/bar_stacked.html), every layer (or color, or category) would have to be plotted separately, onto the previous layer. If we follow the example I referred to above, we will do something like this:

```python
import matplotlib.pyplot as plt

# normalize the dataframe
portion_df = get_ratio_table(df)

# plot column 1, 'Normal'
plt.bar(portion_df.index, portion_df['Normal'])
# plot column 2, 'Abnormal'
plt.bar(portion_df.index, portion_df['Abnormal'], bottom=portion_df['Normal'])
# plot column 3, 'Bad'
plt.bar(portion_df.index, portion_df['Bad'], bottom=portion_df['Normal'] + portion_df['Abnormal'])
```

For every category (except for the first one, which is at the base of the bar) we have to state a `bottom` parameter which basically mean "vertical start point". It's rather nice when you only have to stack two different categories on top of each other (column 1 and 2); but the more columns you have, you actually have to sum the values of all previous categories in order to calculate the value of the current category.

In our case, we want to plot multiple categories, which makes it a little inconvenient and even inefficient to just manually sum the categories, so it would be easier to generalize it a little:

```python
# first, get a list of all categories (dataframe columns)
categories = list(df.columns)    

# plot first group
plt.bar(df.index, df[categories[0]])
    
# plot all other groups
for i in range(1, len(categories)):
    # starting point of this category:
    bot = sum(df[category] for category in categories[:i])  
    # plot the current category
    plt.bar(df.index, df[categories[i]], bottom=bot)  
```

The loop makes it much easier to plot multiple categories for multiple groups.

We can wrap everything in a nice function:

```python
def plot_stacked_bar_for_df(df, normalize=True, figsize=(10,8), ylabel="Fraction", xlabel="Group", fontsize=14, categories=None, color_list=None, output_filename=None):
    """Receives a dataframe and plots it as a stacked bar plot.
    Optional parameters are:
    normalize: normalize the dataframe according to each row. (default: True)
    categories: a sorted list of the categories to plot. (default: None, inferred from df)
    color_list: a list of colors in matplotlib format (#XXXXXX or (0-1, 0-1, 0-1) RGB values, see matplotlib documentation)
    figsize, ylabel, xlabel, fontsize: see matplotlib documentation.
    """

    if normalize:  # normalize to 1
        df = get_ratio_table(df)
    
    # get a list of all groups (categories)
    if not categories:  # if not already provided by the user
        categories = list(df.columns)
    
    if color_list:  # if the user provided a color list
        assert len(color_list) >= len(categories), "color_list is shorter then the number of categories plotted, cannot plot!"
    else:  
        # generate a list of random colors
        color_list = list(np.random.rand(3, len(categories)))  
    
    # initialize figure
    plt.figure(figsize=figsize)
    
    # plot first group
    plt.bar(df.index, df[categories[0]])
    
    # plot all other groups
    for i in range(1, len(categories)):
        bot = sum(df[category] for category in categories[:i])  # starting point of this group
        plt.bar(df.index, df[categories[i]], bottom=bot)  # plot the current group
    
    # aesthetics
    plt.ylabel(ylabel, fontsize=fontsize)
    plt.xlabel(xlabel, fontsize=fontsize)
    
    sns.despine()  # for aesthetics
    
    # if output_filename provided, save the figure
    if output_filename:
        plt.savefig(output_filename)
```

and now all we need for plotting a stacked bar plot[^2] is to call `plot_stacked_bar_for_df(df)` and that's it.

![stacked_barplot]("C:\Users\User\Documents\data_science_blog\stacked_barplot_fig.png")

![stacked_barplot]({{ "/assets/stacked_barplot_fig.png" | relative_url }})



---

[^2]: Note that my basic assumption here is that you want to plot the stacked bar plot based on the current order of categories, from left to right. If it is not the case, you can provide the function with a list of categories in the order you wish to plot.



### In conclusion

Drawing a stacked bar plot is pretty easy in python, but I wanted to generalize it and automate it a little bit. Usually when plotting, I use templates from other pieces of code that I wrote and modify them. I work a lot with DataFrames, so switching to this template reduces the time I need to play with finding the right columns. For convenience, I also wrapped some of the code into a [module](https://github.com/neuhofmo/stacked_barplot_for_pandas_df/blob/master/stacked_bar_plot_from_df.py) and a [Jupyter notebook](https://github.com/neuhofmo/stacked_barplot_for_pandas_df/blob/master/stacked_bar_plot_example.ipynb) (where it's even more up to date, and you can pick colors).  I hope this post gives you either some intuition to the plotting the stacked bar plot in python or at least an easy-to-use implementation for your data.

​    

------

The code presented here, as well as a more generalized module, is available in the [stacked_barplot_for_pandas_df](https://github.com/neuhofmo/stacked_barplot_for_pandas_df) repository.





## TODO

- [x] Add link to previous post
- [x] Add notebook
- [x] Add module
- [x] Add conclusion
- [ ] Fix image address
- [x] Fix date
- [ ] Update readme for repository
- [x] Add TL;DR
- [x] Add description

