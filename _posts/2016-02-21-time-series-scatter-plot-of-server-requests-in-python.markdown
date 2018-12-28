---
layout: post
title:  "Time-Series Scatter Plot of Server Requests using Python"
date:   2016-02-15 23:25:52 +0530
categories: python
description: Time-Series Scatter Plot using Python
comments: true
---
In this post I will attempt to explain how I used [`Pandas`][Pandas] and [`Matplotlib`][Matplotlib] to quickly generate server requests reports on a daily basis. 

__Problem To Be Solved:__ Generate a Scatter plot of the number of requests to a particular URL along with the 99th, 95th and 90th percentile of requests for the duration of a day.

![Scatter Plot Image]({{ site.url }}/images/figure_1.png)

__Breakdown:__

1. Reading multiple server access log files.
2. __Parsing the timestamp fields so that the graph can be scaled appropriately.__
3. __Aggregating the timestamp fields.__
4. __Calculating the Quantile values.__
5. __Generating the Scatter Plot.__

In this post I will leave out the details related to point number 1 and will concentrate on the remaining points. Since this was a fairly mundane task and had to be done daily for a couple of weeks, I intended on automating the process as well as generating the scatter plot as quickly as possible. 

### Parsing the timestamp fields so that the graph can be scaled appropriately
Let's assume the input file contains only timestamps and the file has been read into a list using the following code. 

{% highlight python %}
lines = [line.rstrip('\n') for line in open(file_name)]
{% endhighlight %}

Since the x-axis of the scatter plot will contain timestamp values, the `lines` `List` which currently contains string values, needs to be converted into a `List` of timestamp values. My initial instinct was to use an explicit `for` loop. In an effort to speed up my script I came across python's built-in `map()` function and the concept of `List` comprehension which reduce the `for` loop overhead when the loop body is relatively simple, check [here][Python-optimization] for more details. After a bit of benchmarking I settled on using the `List` comprehension method.

{% highlight python %}
from datetime import timedelta, datetime

dt_lst = [datetime.strptime(date_str, '%H:%M:%S') for date_str in lines]
{% endhighlight %}

### Aggregating the datetime fields
In this aggregation step the goal was to perform a group-by on the timestamps in order to calculate the number of requests per second. To achieve this I used [`Numpy`][Numpy] which is a package for scientific computing and [`Pandas`][Pandas] which is a data analysis library. The [`Series`][Series] and [`DataFrame`][DataFrame] objects from the `Pandas` library and the `ones()` method from the `Numpy` package are used to generate the data structure show in *Figure 1*, the code snippet below contains the details.

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
The [`DataFrame`][DataFrame] object is a tabular data structure with labeled rows and columns, you may think of it as an Excel Spreadsheet. After creating the `DataFrame` `df` you will end up with a structure as displayed below.

![DataFrame Image]({{ site.url }}/images/DataFrame.png)

*Figure 1: Pandas DataFrame*

The next step is to group-by the _Request_Time_ column and sum up the counts, this is achieved by calling the [`groupby`][GroupBy] method on the `DataFrame` object.
{% highlight python %}
grouped = df.groupby('Request_Time').count()
{% endhighlight %}

This gives us an aggregated `DataFrame` as displayed below

![Grouped Image]({{ site.url }}/images/Grouped.png)

### Calculating the Quantile values
Calculating the quantile values on the aggregated `DataFrame` can be done by calling the [`quantile()`][Quantile] method which returns a `DataFrame` containing the value and the `dtype`.

{% highlight python %}
ninety_five_quant = grouped.quantile(.95)[0] # [0] since we only need the quantile value
ninety_ninth_quant = grouped.quantile(.99)[0] # [0] since we only need the quantile value
ninety_eight_quant = grouped.quantile(.98)[0] # [0] since we only need the quantile value
{% endhighlight %}

### Generating the Scatter Plot
The Scatter plot is generated using the [`Matplotlib`][Matplotlib] library. Since we are plotting timestamp values on the x-axis we will use the [`plot_date()`][PlotDate] method.

{% highlight python %}
ax.plot_date(x, y, xdate=True, ydate=False, color='skyblue')
{% endhighlight %}

The x-axis contains the timestamp values, the y-axis contains the request counts, `xdate=True` indicates the x-axis contains date values.

_Note: The `Matplotlib` library contains extensive documentation on all it's API's as well as a vast array of examples, hence I will refrain from going into more details in this post, you may check [this link][PlotDateExample] for more details._

_The snippet below contains code related to rendering the Scatter Plot. I have taken a very simplistic and naive approach since it was good enough for my requirement._ 

{% highlight python %}
import matplotlib.pyplot as plt

total_req =  'Total request: ' + str(count)
nine_five_quant_str =  '95th Quantile: ' + str(ninety_five_quant)
nine_eight_quant_str =  '98th Quantile: ' + str(ninety_eight_quant)
nine_nine_quant_str = '99th Quantile: ' + str(ninety_ninth_quant)

# x and y axis values are extracted from the grouped DataFrame
x = grouped.index
y = grouped.values

print 'Plotting Graph..'
fig = plt.figure()
fig.suptitle('Scatter Plot', fontsize=14, fontweight='bold')
ax = fig.add_subplot(111)
fig.subplots_adjust(top=0.85)

ax.set_xlabel('Request Time')
ax.set_ylabel('Request Count')

text_x_axis_value = 0.9
ax.text(text_x_axis_value, 0.90, total_req, horizontalalignment='center', verticalalignment='center', transform = ax.transAxes)
ax.text(text_x_axis_value, 0.88, nine_five_quant_str, horizontalalignment='center', verticalalignment='center', transform = ax.transAxes)
ax.text(text_x_axis_value, 0.86, nine_eight_quant_str, horizontalalignment='center', verticalalignment='center', transform = ax.transAxes)
ax.text(text_x_axis_value, 0.84, nine_nine_quant_str, horizontalalignment='center', verticalalignment='center', transform = ax.transAxes)

ax.plot_date(x, y, xdate=True, ydate=False, color='skyblue')
plt.show()
{% endhighlight %}

### Summary
In this post we have seen how:

* `List` comprehension can give us better performance over an explicit `for` loop when the loop body is relatively simple. Please check [here][Python-optimization] for more details on the topic.
* The [`Pandas`][Pandas] library can be used to calculate aggregates and quantiles. We used the [`Series`][Series] and [`DataFrame`][DataFrame] objects which are the core data structures of the `Pandas` library to achieve this.
* Generate a Time-Series Scatter Plot using the [`Matplotlib`][Matplotlib] library.

[Python-optimization]:https://wiki.python.org/moin/PythonSpeed/PerformanceTips#Loops
[Numpy]:http://www.numpy.org/
[Pandas]:http://pandas.pydata.org/
[DataFrame]:http://pandas.pydata.org/pandas-docs/stable/generated/pandas.DataFrame.html
[Series]:http://pandas.pydata.org/pandas-docs/stable/generated/pandas.Series.html
[GroupBy]:http://pandas.pydata.org/pandas-docs/stable/generated/pandas.DataFrame.groupby.html#pandas.DataFrame.groupby
[Quantile]:http://pandas.pydata.org/pandas-docs/stable/generated/pandas.DataFrame.quantile.html#pandas.DataFrame.quantile
[Matplotlib]:http://matplotlib.org/
[PlotDate]:http://matplotlib.org/api/pyplot_api.html#matplotlib.pyplot.plot_date
[PlotDateExample]:http://matplotlib.org/examples/pylab_examples/date_demo1.html