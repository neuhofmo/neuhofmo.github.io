---
layout: post
title: Chi-square (and post-hoc) tests in Python
date: 2018-02-14 14:00:00
categories: python statistics pandas 
permalink: /chi-square-and-post-hoc-in-python/
---

## $\chi^2$ and post-hoc tests in Python - the easy way



### TL;DR

If you want to run $$\chi^2$$ contingency test, with post-hoc and multiple comparisons correction, on a pandas DataFrame, just use the function `chisq_and_posthoc_corrected(df)` [available in this repository](https://github.com/neuhofmo/chisq_test_wrapper/blob/master/chisq_test_wrapper.py). Good luck!



### Chi-square and post-hoc tests in Python 

[Chi-squared test](https://en.wikipedia.org/wiki/Chi-squared_test), or in its more formal notation,  $$\chi^2$$ test, is widely used in research when there's a need to compare the number of observations between different experimental conditions. It is a pretty standard test, and pretty easy to perform using python, but when dealing with multiple groups and multiple conditions (i.e. more than 2 columns and 2 rows) these contingency tables (yet another name for chi-square test) become a little harder to implement.

In this post I will explain the ideas behind the multiple chi-square test and offer implemented functions that only require a [pandas](https://pandas.pydata.org/) DataFrame to streamline chi-square test for multiple groups, post-hoc chi-square tests, multiple testing correction and a nice stacked bar-plot representation of the results.

This post is inspired by [Shir](https://twitter.com/ShirToubiana), a PhD student at the Technion, and is based on the data she has collected. Since her results are still unpublished, I had to change the data a little bit and avoid details about the real purpose of her experiment, which doesn't matter much for this demonstration (but is really cool nonetheless). 



### Our data

Let's say we want to study the difference in the shape of certain cell types (let's say liver cells) between patients suffering from a specific condition and people that do not suffer from that condition. For this purpose, we took tissue samples from four patients and from a few healthy persons (which we collectively call "Control")*. We then took a look at the shape of their liver cells, and divided them to 3 different shapes: "normal", "abnormal" (these cells don't look so good, probably non-functional) and "bad" (these cells are really deformed). 

We counted how many time we saw each of the above conditions in our samples, and got the following results:

|  Cell_line   | Normal | Abnormal | Bad  |
| :----------: | :----: | :------: | :--: |
| **Control**  |  5126  |   137    |  5   |
| **Patient1** |  1597  |    20    |  7   |
| **Patient2** |  862   |    12    |  4   |
| **Patient3** |  861   |    22    |  7   |
| **Patient4** |  4546  |   886    | 142  |

From this table we can see that the "abnormal" and "bad" cells were observed even in healthy people. It makes sense - after all, it's still biology, and cells vary within individuals. Thus, we are actually more interested to know whether these abnormalities are more *frequent* in our patients. 

More formally, we wanted to see if there's any difference in the distribution of the three cell shapes between the healthy and sick. 

If you are familiar with the $$\chi^2$$ test**, it looks like it would be very useful in telling us if the distributions are different.

----

*Since the patients differ from each other, we couldn't group them into one "Patients" or "Cases" group, but had to compare them separately. 

**For this post, I assume you have some basic knowledge of what chi-square tests do and how they work. If you don't, it may be a good idea to check out [this post](http://www.statisticshowto.com/probability-and-statistics/chi-square/) or [this post](https://onlinecourses.science.psu.edu/statprogram/node/158), but don't be afraid to start with [Khan Academy's introduction video](https://www.khanacademy.org/math/statistics-probability/inference-categorical-data-chi-square-tests/chi-square-goodness-of-fit-tests/v/pearson-s-chi-square-test-goodness-of-fit) to capture the intuition behind this test.



### $\chi^2$ test in Python 

What we want to do in this case is to compare the different groups and get a p-value that tells us whether these groups are actually different than each other. 

There are a few different implementations to chi-square test in python, but [scipy.stats.chi2_contingency](http://lagrange.univ-lyon1.fr/docs/scipy/0.17.1/generated/scipy.stats.chi2_contingency.html) is the easiest to use. You may find some code examples using this module [here](https://www.programcreek.com/python/example/100318/scipy.stats.chi2_contingency) and [here](https://codereview.stackexchange.com/questions/96761/chi-square-independence-test-for-two-pandas-df-columns), but the usage is pretty straightforward. 

First, let's load our table into a pandas dataframe:

```python
import pandas as pd
df = pd.read_excel('groups_sum_demo.xlsx', index_col='Cell_line')
```

And then use the contingency table function to get a p-value:

```python
from scipy.stats import chi2_contingency
chi2, p, dof, ex = chi2_contingency(df, correction=True)
print(f"Chi2 result of the contingency table: {chi2}, p-value: {p}")
```

Which return the $$\chi^2$$ statistic, the p-value, Degrees of Freedom and the expected distribution of the categories. We are mostly interested in the p-value*.

In this case, the p-value = 1.27e-232, which seems pretty significant. But in which direction? Are the patients different from the control? Are the patients different from each other? 



*If what I wrote here still doesn't make enough sense, [this post](http://hamelg.blogspot.co.il/2015/11/python-for-data-analysis-part-25-chi.html) explains chi-square in python pretty nicely, including code snippets and some background.  



### Chi Square post-hoc tests

In order to check the differences between each pair of groups, we would have to individually compare every pair separately (a valid post-hoc test for such comparisons). Acquiring a list of all possible pairs is pretty simple using the built-in [itertools.combinations](https://docs.python.org/3/library/itertools.html):

```python
# gathering all combinations for post-hoc chi2
all_combinations = list(combinations(df.index, 2))
print("Significance results:")
for comb in all_combinations:
    # subset df into a dataframe containing only the pair "comb"
    new_df = df[(df.index == comb[0]) | (df.index == comb[1])]
    # running chi2 test
    chi2, p, dof, ex = chi2_contingency(new_df, correction=False)
    print(f"Chi2 result for pair {comb}: {chi2}, p-value: {p}")
```



### Avoiding multiple comparisons bias

In the previous step, we have calculated the p-value for each pair. However, since we have performed the test several times, we have to correct our results for [multiple comparisons](https://en.wikipedia.org/wiki/Multiple_comparisons_problem). One such way to do it to correct our p-values using [FDR correction](https://en.wikipedia.org/wiki/False_discovery_rate). Such correction is already available using [statsmodels.sandbox.stats.multicomp.multipletests](http://www.statsmodels.org/dev/generated/statsmodels.sandbox.stats.multicomp.multipletests.html) which offers many other corrections (actually, the [statsmodels](http://www.statsmodels.org/stable/index.html) and [scipy](https://www.scipy.org/) modules cover almost 100% of my statistics needs).

`multipletests` receives a list of p-values  (as well as other optional arguments), and returns the respective list of reject decision (reject/do not reject) as well as a list of the corrected p-values (it also returns additional values which are not relevant to our case).

So instead of printing the p-values for each test like we did before, we can collect them in a list:

```python
from itertools import combinations

# gathering all combinations for post-hoc chi2
all_combinations = list(combinations(df.index, 2))
p_vals = []
for comb in all_combinations:
    # subset df into a dataframe containing only the pair "comb"
    new_df = df[(df.index == comb[0]) | (df.index == comb[1])]
    # running chi2 test
    chi2, p, dof, ex = chi2_contingency(new_df, correction=True)
    p_vals.append(p)
```

And then correct the list:

```python
from statsmodels.sandbox.stats.multicomp import multipletests

reject_list, corrected_p_vals = multipletests(p_vals, method='fdr_bh')[:2]
```

In order to demonstrate that the p-values were indeed corrected, we can print them in order:

```python
print("original p-value\tcorrected p-value\treject?")
for p_val, corr_p_val, reject in zip(p_vals, corrected_p_vals, reject_list):
    print(p_val, "\t", corr_p_val, "\t", reject)
```

And get the following results:

original p-value|corrected p-value|reject?
----------------|-----------------|-------
0.000100825375712 | 0.000168042292854 | True
0.00323083142242 | 0.0046154734606 | True
8.43128799303e-05 | 0.000168042292854 | True
2.48244478974e-153 | 2.48244478974e-152 | True
0.955635429992 | 0.955635429992 | False
0.0342345357251 | 0.0427931696564 | True
2.68447965589e-62 | 1.34223982794e-61 | True
0.158923855452 | 0.176582061613 | False
2.55242848911e-34 | 8.5080949637e-34 | True
6.12967436168e-29 | 1.53241859042e-28 | True

Now we can easily tell which of our pairs differ significantly (assuming $$\alpha = 0.05$$; if you would like to use a different threshold, for example 0.01, you should use `multipletests(p_vals, method='fdr_bh', alpha=0.01)`). 



In order to make it even more intuitive, following the standard in biological publications I would like to add mark the significance with asterisks (according to the p-value). 

So for convenience, let's define:

```python
def get_asterisks_for_pval(p_val):
    """Receives the p-value and returns asterisks string."""
    if p_val > 0.05:
        p_text = "ns"  # above threshold => not significant
    elif p_val < 1e-4:  
        p_text = '****'
    elif p_val < 1e-3:
        p_text = '***'
    elif p_val < 1e-2:
        p_text = '**'
    else:
        p_text = '*'
    
    return p_text
```

Which returns a string of asterisks for each p-value we provide. 

Then, we can actually combine our initial test, post-hoc tests and multiple comparisons correction into one function, which receives a pandas DataFrame object and prints out the results for the entire table, and then for each and every pair:

```python
def chisq_and_posthoc_corrected(df):
    """Receives a dataframe and performs chi2 test and then post hoc.
    Prints the p-values and corrected p-values (after FDR correction)"""
    # start by running chi2 test on the matrix
    chi2, p, dof, ex = chi2_contingency(df, correction=True)
    print(f"Chi2 result of the contingency table: {chi2}, p-value: {p}")
    
    # post-hoc
    all_combinations = list(combinations(df.index, 2))  # gathering all combinations for post-hoc chi2
    p_vals = []
    print("Significance results:")
    for comb in all_combinations:
        new_df = df[(df.index == comb[0]) | (df.index == comb[1])]
        chi2, p, dof, ex = chi2_contingency(new_df, correction=True)
        p_vals.append(p)
        # print(f"For {comb}: {p}")  # uncorrected

    # checking significance
    # correction for multiple testing
    reject_list, corrected_p_vals = multipletests(p_vals, method='fdr_bh')[:2]
    for p_val, corr_p_val, reject, comb in zip(p_vals, corrected_p_vals, reject_list, all_combinations):
        print(f"{comb}: p_value: {p_val:5f}; corrected: {corr_p_val:5f} ({get_asterisks_for_pval(p_val)}) reject: {reject}")
```



Then, by simply running `chisq_and_posthoc_corrected(df)`, we would get the following output:

> ```
> Chi2 result of the contingency table: 1095.406615116616, p-value: 3.761331610902334e-231
> Significance results:
> ('Control', 'Patient1'): p_value: 0.000101; corrected: 0.000168 (***) reject: True
> ('Control', 'Patient2'): p_value: 0.003231; corrected: 0.004615 (**) reject: True
> ('Control', 'Patient3'): p_value: 0.000084; corrected: 0.000168 (****) reject: True
> ('Control', 'Patient4'): p_value: 0.000000; corrected: 0.000000 (****) reject: True
> ('Patient1', 'Patient2'): p_value: 0.955635; corrected: 0.955635 (ns) reject: False
> ('Patient1', 'Patient3'): p_value: 0.034235; corrected: 0.042793 (*) reject: True
> ('Patient1', 'Patient4'): p_value: 0.000000; corrected: 0.000000 (****) reject: True
> ('Patient2', 'Patient3'): p_value: 0.158924; corrected: 0.176582 (ns) reject: False
> ('Patient2', 'Patient4'): p_value: 0.000000; corrected: 0.000000 (****) reject: True
> ('Patient3', 'Patient4'): p_value: 0.000000; corrected: 0.000000 (****) reject: True
> ```



We can see that while all patients differ from the control group significantly, some of them also differ from each other. 



###In conclusion

Python has very intuitive packages for dealing with chi-square tests and multiple corrections. Here I provide my suggestion for performing a valid chi-square test on large dataframes, using an example dataset based on a real experiment. I also wrapped some of the code into a [module](https://github.com/neuhofmo/chisq_test_wrapper/blob/master/chisq_test_wrapper.py) and a [Jupyter notebook](https://github.com/neuhofmo/chisq_test_wrapper/blob/master/chi_square_test_and_post_hoc.ipynb). 

I hope this post gives you either some intuition to the chi-square test in python or at least an easy-to-use implementation of the chi-square test for your data.



----

The code presented here, as well as a more generalized module, is available in the [chisq_test_wrapper repository](https://github.com/neuhofmo/chisq_test_wrapper).