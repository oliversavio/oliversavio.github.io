---
layout: post
title:  Java 8 features I learnt while writing an access log summarizer
tags: [java]
---

Java 8 has been out for quite some time now and I thought, the best way to get up-to speed with the new features, is to dive in head first and learn along the way, while I code a solution to a real world problem.

Hence in this post I will document some of the interesting features I was able to pickup and implement while coding an access log summarizer.  I will not cover the topics in-depth, but will link to useful resources. I hope others like me who have experience with lower versions of Java will find this useful.

The Access Log Summarizer source code is available [here][access-log-summarizer]

### Topics this post will cover in a nutshell
- Using the `java.util.stream` library.
- Using the `try-with-resources` Statement.
- The `AutoCloseable` interface.
- Using `Comparators` the Java 8 way.

### Using the java.util.stream library
The Stream library, along with the introduction of lambda expressions in Java 8, enables us to write code that is more expressive.
It frees us from the clutter of looping syntax and temporary variables and makes the intent of the code a lot more obvious.

Below is an example picked up from the wonderful series of [posts][stream-library] on the `java.util.stream` library written by Brian Goetz.

_Use Case: "Print the names of sellers in transactions with buyers over age 65, sorted by name."_
{% highlight java linenos %}
// Using Looping.
Set<Seller> sellers = new HashSet<>();
for (Txn t : txns) {
    if (t.getBuyer().getAge() >= 65)
        sellers.add(t.getSeller());
}
List<Seller> sorted = new ArrayList<>(sellers);
Collections.sort(sorted, new Comparator<Seller>() {
    public int compare(Seller a, Seller b) {
        return a.getName().compareTo(b.getName());
    }
});
for (Seller s : sorted)
    System.out.println(s.getName());

// Using the Stream Library
txns.stream()
    .filter(t -> t.getBuyer().getAge() >= 65)
    .map(Txn::getSeller)
    .distinct()
    .sorted(comparing(Seller::getName))
    .map(Seller::getName)
    .forEach(System.out::println);
{% endhighlight %}

The Stream API allows you to run your stream in parallel by simply adding `.parallelStream()` at the beginning of the stream pipeline.

You will soon discover the hard way like I did, that this isn't a magic pill that will make your code run faster. There are a couple of things you need to consider, like the amount of work needed to split the source to enable chunks to be executed in parallel, the amount of computation to be performed per chunk, whether the order in which the elements are encountered affect the computation.

I would encourage you to read Part-3 and Part-4 of the [this][stream-library] series, where Brian Goetz explains in great detail how the Stream library works internally and the things to consider while running parallel streams.

### Using the try-with-resources Statement and the AutoCloseable interface
Technically both the `try-with-resources` statement and the `AutoCloseable` interface were introduced with Java 7.

The `try-with-resources` lets you declare one or more resources and ensures the resources are closed once the statement is completed, or an exception is thrown. This eliminates the need to explicitly handle closing resources within a `finally` block.
As per the Java Language Specification documentation, the type of variable declared as a resource must be a subtype of the `AutoCloseable` interface.

It is worth noting, that exceptions thrown by the `try-with-resources` statement while closing resources are suppressed. These exceptions can be retrieved by calling the `Throwable.getSuppressed` method. 

Here is a snippet from the source code which streams lines from a file. Since the `Stream` interface extends the `AutoCloseable` interface, the stream will be closed once the file has been read or in the event an exception is thrown.
{% highlight java linenos %}
try(Stream<String> stream = Files.lines(Paths.get(fileName))) {
	metricMap = parser.parseLog(stream, new ParsingOptions(urlIndx, timeIndx));
} catch (IOException e) {
	logger.error("Problem Streaming File: ", e);
	return;
}
{% endhighlight %}
The `try-with-resources` statement can be used with or without `catch` and `finally` blocks. If present, they are executed after the resources are closed. A `try-with-resources` statement with at least one `catch` or `finally` clause is called an "extended" `try-with-resources` statement.


You can find more details about the the `try-with-resources` statement [here][try-with-resources].

### Using Comparators the Java 8 way
One of the new language features of Java 8 is the `@FunctionalInterface` annotation. I must admit, I haven't been able to wrap my head around this concept completely as yet. But what I can tell you is, that it enables us to write Comparators this way.
{% highlight java linenos %}
Comparator<Metric> comparator = Comparator.comparingLong(Metric::getCount);
Collections.sort(filetredMetrics, comparator.reversed());
{% endhighlight %}
The code above is sorting a collection of `Metric` objects using the `count` value in descending order. 

The `comparingLong()` method specifies that the `count` values are of type `Long`, hence reducing the cost of auto-boxing and boxing incurred when using the the more generic `comparing()` method.

Follow this [link][comparators], written by my good friend Praveer Gupta. Here you will find a very clean and concise explanation of writing Comparators the Java 8 way. 


### Conclusion
I have listed some of the Java 8 features I picked while writing my Access Log Summarizer tool. I have added links to articles where you may explore each topic in-depth. I hope to continually improve and re-factor my code as I get more adept with Java 8. Please feel free to have a look at the source code on GitHub.
 

### References
1. [Try-with-resources Oracle tutorial][try-with-resources]
2. [Java Language Specification][jls].
3. [Posts by Brian Goetz on java.util.stream library][stream-library]
4. [Comparators the Java 8 way by Praveer Gupta][comparators]
5. [Source code to my Access Log Summarizer tool on GitHub][access-log-summarizer]

[access-log-summarizer]:https://github.com/oliversavio/Access-Log-Summarizer
[try-with-resources]:https://docs.oracle.com/javase/tutorial/essential/exceptions/tryResourceClose.html
[jls]:https://docs.oracle.com/javase/specs/jls/se8/html/jls-14.html#jls-14.20.3
[stream-library]:http://www.ibm.com/developerworks/library/j-java-streams-1-brian-goetz/index.html
[comparators]:http://praveer09.github.io/technology/2016/06/21/writing-comparators-the-java8-way/