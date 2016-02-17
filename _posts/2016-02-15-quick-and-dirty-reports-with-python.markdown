---
layout: post
title:  "Time-series Scatter Plot in Python using Pandas - Part 1"
date:   2016-02-15 23:25:52 +0530
categories: python
---
In this series of post I will attempt to explain how I used Pandas to quickly generate server request reports on a daily basis. 

__Problem To Be Solved:__ Generate a Scatter plot of the number of request to a particular URL along with the 99th, 95th and 90th percentile of request for the duration of a day.

![Scatter Plot Image]({{ site.url }}/images/figure_1.png)

__Breakdown:__

1. Reading multiple server access log files.
2. __Parsing the timestamp fields so that the graph can be scaled appropriately.__
3. __Aggregating the timestamp fields.__
4. __Calculating the Quantile values.__
5. Generating the Scatter Plot.

In this post I will leave out the details related to point number 1 and 4 and will concentrate on the remaining points. 
Since this was a fairly mundane task and had to be done daily for a couple of weeks, I intended to automate as much as possible as well as generate the scatter plot as quickly as possible. 

### Parsing the timestamp fields so that the graph can be scaled appropriately
Let's assume the input file contains only timestamps and the file has been read into a list using the following code. 

{% highlight python %}
lines = [line.rstrip('\n') for line in open(file_name)]
{% endhighlight %}

Since the x-axis of the scatter plot will contain timestamp values, the `lines` list which currently contains string values needs to be converted intos a list of timestamp values. My initial instinct was to use an explicit `for` loop, in an effort to speed up my script I came across python's built-in `map()` function and the concept of `List` comprehension which reduce the `for` loop overhead when the loop body is relatively simple, check [here][Python-optimization] for more details. After a bit of benchmarking I settled on using the `List` comprehension method.

{% highlight python %}
dt_lst = [datetime.strptime(date_str, '%H:%M:%S') for date_str in lines]
{% endhighlight %}

### Aggregating the datetime fields
In this aggregation step the goal was to perform a group-by on the timestamps in order to calculate the number of request per second. To achieve this I used [`Numpy`][Numpy] which is a package for scientific computing and [`Pandas`][Pandas] which is a data analysis library.

_Note: I could have probably achieved the same result using only `Pandas`._

{% highlight python %}
import pandas as pd
import numpy as np
from pandas import DataFrame, Series

# Create a Series named "Request_Time"
sr_dt = Series(dt_lst, name='Request_Time') 
# Create a DataFrame using the Request_Time Series
df = DataFrame(sr_dt)
# Create an array of 1's using Numpy
count = sr_dt.size
ones = np.ones(count, dtype=int)
# Add the ones array to the DataFrame with the header "Counts"
df['Counts'] = ones

{% endhighlight %}
A [`DataFrame`][DataFrame] is a tabular data structure with labeled rows and columns, you may think of it as an Excel Spreadsheet. After creating the `DataFrame` you will end up with a structure as displayed below.

![DataFrame Image]({{ site.url }}/images/DataFrame.png)

The next step is to group-by the _Request_Time_ column and sum up the counts.
{% highlight python %}
grouped = df.groupby('Request_Time').count()
{% endhighlight %}

This gives us an aggregated `DataFrame` as displayed below

![Grouped Image]({{ site.url }}/images/Grouped.png)

### Calculating the Quantile values
Calculating the quantile values on the aggregated `DataFrame` can be done by calling the `quantile()` method which returns a `DataFrame` containing the value and the `dtype`.

{% highlight python %}
ninety_five_quant = grouped.quantile(.95)[0] # [0] since we only need the quantile value
{% endhighlight %}


[Python-optimization]:https://wiki.python.org/moin/PythonSpeed/PerformanceTips#Loops
[Numpy]:http://www.numpy.org/
[Pandas]:http://pandas.pydata.org/
[DataFrame]:http://pandas.pydata.org/pandas-docs/stable/generated/pandas.DataFrame.html