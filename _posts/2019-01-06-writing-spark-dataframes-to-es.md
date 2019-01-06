---
layout: post
title:  Writing a Spark Dataframe to an Elasticsearch Index
date:   2019-01-06 12:00:00 +0530
categories: spark
description: Writing a Spark Dataframe to an Elasticsearch Index
comments: true
---

![Spark Dataframes to ES Index]({{ site.url }}/images/spark_es.png)

In this post we will walk through the process of writing a Spark `DataFrame` to an Elasticsearch index.

Elastic provides Apache Spark Support via [elasticsearch-hadoop][es-spark-support], which has native integration between Elasticsearch and Apache Spark.

__Note: All examples are written in Scala 2.11 with Spark SQL 2.3.x. Prior experience with Apache Spark is a pre-requisite.__


### Breakdown:
- Maven Dependencies.
- Spark-ES Configurations.
- writeToIndex() Code.

### Maven Dependencies
The dependencies mentioned below should be present in your `classpath`. `elasticsearch-spark-20` provides the native Elasticsearch support to Spark and `commons-httpclient` is needed to make RESTful calls to the Elasticsearch APIs. For some strange reason this version of `elasticsearch-spark-20` omitted the http client dependency so it had to be added manually. 
{% highlight xml %}
-------------------
Snippet of the pom.xml
-------------------
<dependency>
    <groupId>org.apache.spark</groupId>
    <artifactId>spark-sql_2.11</artifactId>
    <version>2.3.0</version>
</dependency>
<dependency>
    <groupId>commons-httpclient</groupId>
    <artifactId>commons-httpclient</artifactId>
    <version>3.1</version>
</dependency>
<dependency>
    <groupId>org.elasticsearch</groupId>
    <artifactId>elasticsearch-spark-20_2.11</artifactId>
    <version>6.4.2</version>
</dependency>
{% endhighlight %}

### Spark-ES Configurations
In order for Spark to communicate with the Elasticsearch, we'll need to know where the ES node(s) are located as well as the port to communicate with. These are provided to the `SparkSession` by setting the `spark.es.nodes` and `spark.es.port` configurations.

*Note: The example here used Elasticsearch hosted to AWS and hence needed an additional configuration `spark.es.nodes.wan.only` to be set to `true`.*

Let's see some code to create the `SparkSession`.

{% highlight scala %}
val spark = SparkSession
     .builder()
     .appName("WriteToES")
     .master("local[*]")
     .config("spark.es.nodes","<IP-OF-ES-NODE>")
     .config("spark.es.port","<ES-PORT>")
     .config("spark.es.nodes.wan.only","true") // Needed for ES hosted on AWS
     .getOrCreate()
{% endhighlight %}

### writeToIndex() Code
First we'll define a `case class` to represent our index structure.
{% highlight scala %}
case class AlbumIndex(artist:String, yearOfRelease:Int, albumName: String)
{% endhighlight %}

Next we'll create a `Seq` of `AlbumIndex` objects and convert them to a `DataFrame` using the handy `.toDF()` function, which can be invoked by importing `spark.implicits._`.

{% highlight scala %}
import spark.implicits._

   val indexDocuments = Seq(
        AlbumIndex("Led Zeppelin",1969,"Led Zeppelin"),
        AlbumIndex("Boston",1976,"Boston"),
        AlbumIndex("Fleetwood Mac", 1979,"Tusk")
   ).toDF
{% endhighlight %}

*Note: `spark` here represents the `SparkSession` object.*

Once we have our `DataFrame` ready, all we need to do is import `org.elasticsearch.spark.sql._` and invoke the `.saveToEs()` method on it.

{% highlight scala %}
import org.elasticsearch.spark.sql._

indexDocuments.saveToEs("demoindex/albumindex")
{% endhighlight %}

Here is the entire source code.

{% highlight scala %}
import org.apache.spark.sql.SparkSession
import org.elasticsearch.spark.sql._

object WriteToElasticSearch {

 def main(args: Array[String]): Unit = {
   WriteToElasticSearch.writeToIndex()
 }

 def writeToIndex(): Unit = {

   val spark = SparkSession
     .builder()
     .appName("WriteToES")
     .master("local[*]")
     .config("spark.es.nodes","<IP-OF-ES-NODE>")
     .config("spark.es.port","<ES-PORT>")
     .config("spark.es.nodes.wan.only","true") // Needed for ES hosted on AWS
     .getOrCreate()

   import spark.implicits._

   val indexDocuments = Seq(
   AlbumIndex("Led Zeppelin",1969,"Led Zeppelin"),
   AlbumIndex("Boston",1976,"Boston"),
   AlbumIndex("Fleetwood Mac", 1979,"Tusk")
   ).toDF

   indexDocuments.saveToEs("demoindex/albumindex")
 }
}

case class AlbumIndex(artist:String, yearOfRelease:Int, albumName: String)
{% endhighlight %}


### References
1. [Elasticsearch Spark Support.][es-spark-support]



[es-spark-support]:https://www.elastic.co/guide/en/elasticsearch/hadoop/master/spark.html
