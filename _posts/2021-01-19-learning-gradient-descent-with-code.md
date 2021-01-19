---
layout: post
title: Learning gradient descent with code (Go)
tags: [go, golang, ml]
subtitle: Understanding the gradient descent algorithm with Go code.
katex: yes
--- 

Gradient Descent is one of the most basic and fundamental algorithms in machine learning. In this post I'll attempt to explain how the algorithm works with Go code.


### Motivation
Gradient descent is one of the most fundamental algorithms used to find optimal parameters of machine learning models. In practice, you may rarely need to implement gradient on your own, however understanding how it works will help you better train complex ML models.

**Note: These code examples are for educational purposes only and are in no way intended to be production ready.**

### Getting started
To get started, I've randomly generated points which approximately fall along a straight line. Hence for our model, we'll use the straight line equation to predict points in the future.

![Gradient_Descent]({{ site.url }}/img/gd_1.png)

**Straight Line equation**
{% highlight go %}
  y = mx + c // some countries may use the form y = mx + b

{% endhighlight %}

Think of this as, our model needs to predict the value "y" given an input "x". In order to do this we need to find the best values for "m" and "c".

This is where the gradient descent algorithm comes in, we'll use it to find the best possible values for "m" and "c".

"m" and "c" are the parameters of our model.

### The algorithm
![GD_algo]({{ site.url }}/img/gd_algo.png)

### The model
Let's start by defining our model with a function.

{% highlight go linenos %}
type params struct {
	m float64
	c float64
}

// Equation for a straight line i.e y = mx + c or y = mx + b
func f(x []float64, p *params) []float64 {
	var result []float64
	for _, val := range x {
		y := p.m*val + p.c
		result = append(result, y)
	}
	return result
{% endhighlight %}

### The loss function
We need a way to measure how good or bad values of "m" and "c" are, to do this we need to define a loss function.

We'll use the mean squared error (MSE) loss function, which is the mean of squared difference between predicted and actual values.

\\[
  \frac{1}{n}\displaystyle\sum_{i=0}^{n-1} \big(f(x[i]) - y[i] \big)^2
\\]

{% highlight golang linenos %}
// MSE or Mean Squared Error
func costFunction(actuals []float64, predictions []float64) float64 {
	var diff float64
	for i := range actuals {
		d := predictions[i] - actuals[i]
		diff += math.Pow(d, 2.0)
	}
	loss := diff / float64(len(actuals))
	return loss
}
{% endhighlight %}

### Calculating the gradients
Our object with gradient descent is to minimize the loss (or cost) function.

![Gradient Descent]({{ site.url }}/img/gradient_descent_demystified.png)

_[Image Source][gd-demystified]_

The image above represents the graph of a loss (or cost) function. In order to know how to move to the bottom of the curve from any given point on the curve, we calculate the slope (or derivate or gradient) at that point. 


\\(
    \binom{n}{k} = \frac{n!}{k!(n-k)!}
\\)




## References and Further Reading
1. [Gradient Descent Image Source][gd-demystified]
2. [User-defined functions Scala - Databricks][udf-scala]



[gd-demystified]: https://ml-cheatsheet.readthedocs.io/en/latest/gradient_descent.html
[udf-ref]: https://jaceklaskowski.gitbooks.io/mastering-spark-sql/spark-sql-udfs.html 
[udf-scala]: https://docs.databricks.com/spark/latest/spark-sql/udf-scala.html 
