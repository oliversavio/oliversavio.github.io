---
layout: post
title:  "Quick and Dirty Reports in Python using Pandas - Part 1"
date:   2016-02-15 23:25:52 +0530
categories: python
---
In this series of post I will attempt to explain how I used Pandas to quickly generate server request reports on a daily basis. 

__Problem To Be Solved:__ Generate a Scatter plot of the number of request to a particular URL along with the 99th, 95th and 90th percentile of request for the duration of a day.

![Scatter Plot Image]({{ site.url }}/images/figure_1.png)

__Breakdown:__

1. Reading multiple server access logs
2. __Parsing the datetime fields so that the graph can be scaled appropriately__
3. __Aggregating the datetime fields__
4. __Calculating the Quantile values__
5. Generating the Scatter Plot

In this post I will leave out the details related to points number 1 and 4 and will concentrate on the remaining points. Since this was a fairly mundane task and had to be done daily over a period of a couple of weeks, one of my main aims was to automate as much as possible as well as generate the scatter plot as quickly as possible. I may decided to improve on this in future post and possibly generate plots for varying time periods.

### Parsing the datetime fields so that the graph can be scaled appropriately
Lets assume the input file only contains timestamps, the file has been read into a list using the following code. 

{% highlight python %}
lines = [line.rstrip('\n') for line in open(file_name)]
{% endhighlight %}

Since the x-axis contains timestamp values, the `lines` list which is currently contains string values needs to be converted to a list of datetime values. The fastest way of doing this is python is to use the `map` function.
