---
layout: post
title: Softmax activation function with code (Go)
tags: [go, golang, machine learning, softmax, neural networks]
subtitle: Understanding why and how to use the softmax activation function
katex: yes
--- 

The softmax activation function is commonly used as the output layer in a neural networks.

**Note: These code examples are for educational purposes only and are in no way intended to be used in production.**
## Why
The softmax activation function is often the final layer in multi-class classification neural network. 
This function possesses few important properties:
 - It converts random activations which can be hard to interpret into probabilities between 0 and 1.
 - The output activations sum to 1.
 - It tends to pick one class over the others.


## The Math
Mathematically the softmax function is represented as follows.

\\[
 \huge\sigma(z) = \huge\frac{e^{zi}}{\sum_{j=0}^{K}e^{zj}}
\\]

Which translates to, calculate the exponent for each element of array `z` and divide by the sum of the exponent of each element of `z`.

E.g.
\\[
    \sigma(\begin{bmatrix} 5 \\\ 2 \\\ -1 \\\ 3 \end{bmatrix}) = \begin{bmatrix} 0.842 \\\ 0.042 \\\ 0.002 \\\ 0.114 \end{bmatrix}
\\]

## The code
Let's jump right in with the code.

We'll be extending the concepts explained in my previous post of "Demystfying" with matrix operations,
considering an oversimplified nerual network with a single linear layer. This should give you the intitution 
of what the activations look like with and without using the softmax layer.


_Note: We'll be using the `gonum` library to represet our matrices and perform operations on them._

### Linear Layer
Let's code out the simple linear layer. In keeping with the conventions from the previous post, we'll use `m`, `x` and `c` to represent our weights, inputs and biases.

To keeps things simple, we'll assume we have only 3 classes which need to be predicted, hence our output matrix should have a dimension of `3x1`.
Accordingly well assume `m` to have dimensions `3x3` and `c` to have `3x1`.

E.g. `mx + c`

\\[
    \begin{bmatrix} 5 & 3 & 5 \\\ 3 & 7 & 0 \\\ 1 & 2 & 5 \end{bmatrix} \begin{bmatrix} 1 \\\ 2  \\\ 1 \end{bmatrix}
    + \begin{bmatrix} 1 \\\ 1 \\\ 1 \end{bmatrix}
    =
    \begin{bmatrix} 17 \\\ 18  \\\ 11 \end{bmatrix}
\\]


{% highlight go linenos %}
import "gonum.org/v1/gonum/mat"

func initRandomMatrix(row int, col int) *mat.Dense {
    size := row * col
    arr := make([]float64, size)
    for i := range arr {
        arr[i] = rand.NormFloat64()
    }

    return mat.NewDense(row, col, arr)
}

func linear() *mat.Dense {
    m := initRandomMatrix(3, 3)
    x := initRandomMatrix(3, 1)
    c := initRandomMatrix(3, 1)

    ll := mat.NewDense(3, 1, nil)
    ll.Mul(m, x)
    ll.Add(ll, c)
    
    return ll
}

{% endhighlight %}
`initRandomMatrix` is a helper method to initialize a matrix with random values.

### Softmax Function

{% highlight go linenos %}
import "gonum.org/v1/gonum/mat"

func softmax(matrix *mat.Dense) *mat.Dense {
    var sum float64
    // Calculate the sum
    for _, v := range matrix.RawMatrix().Data {
	    sum += math.Exp(v)
    }

    resultMatrix := mat.NewDense(matrix.RawMatrix().Rows, matrix.RawMatrix().Cols, nil)
    // Calculate softmax value for each element
    resultMatrix.Apply(func(i int, j int, v float64) float64 {
	    return math.Exp(v) / sum
    }, matrix)

    return resultMatrix
}

{% endhighlight %}

As you can see from the code above, the `softmax` function is pretty straight forward.


### Putting it all together
Here `SimpleNN()` is an oversimplified representation of a neural network with a single linear layer and `SoftMaxNN()` is the network with the softmax layer.
{% highlight go linenos %}
func main() {
	rand.Seed(350) // For reproduceable results
	fmt.Println()

	SimpleNN()
	SoftmaxNN()
}

func SimpleNN() {
	nn := linear()
	fmt.Println("Activation after linear layer")
	fmt.Println(mat.Formatted(nn))
}

func SoftmaxNN() {
	nn := linear()
	nnsmax := softmax(nn)

	fmt.Println("Activation after softmax layer")
	fmt.Println(mat.Formatted(nnsmax))
}


{% endhighlight %}
### Output
*Activations after linear layer*
\\[
    \begin{bmatrix} 0.842 \\\ 0.042 \\\ 0.002 \\\ 0.114 \end{bmatrix}
\\]

*Activations after softmax layer*
\\[
    \begin{bmatrix} 0.842 \\\ 0.042 \\\ 0.002 \\\ 0.114 \end{bmatrix}
\\]
## Source Code
All the source code can be found on [Github][github].

## References and Further Reading
1. [Machine Learning MOOC by Andrew Ng][ng-gd-vid]
2. [Introduction to Derivatives by Khan Academy][derivative-khan-academy]
3. [Gradient Descent Image Source][gd-demystified]

[gd-demystified]: https://ml-cheatsheet.readthedocs.io/en/latest/gradient_descent.html
[ng-gd-vid]: https://www.coursera.org/learn/machine-learning
[github]: https://github.com/oliversavio/learn-ml-with-code/tree/main/gradient_descent
[derivative-khan-academy]: https://www.khanacademy.org/math/differential-calculus/dc-diff-intro

