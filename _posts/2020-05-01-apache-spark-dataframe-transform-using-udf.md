---
layout: post
title: Spark DataFrame transform using a User Defined Function (UDF)
tags: [spark, scala]
subtitle: Transform a Spark DataFrame or Dataset using a UDF.
--- 

This is an extension of my post on [_Transforming Spark Datasets using Scala transformation functions_][dataset-transform].

In the previous post we de-serialized a Spark `Dataset` to a scala `case class` and learnt how to use `Encoders` to run transformations over the `Dataset`. In this post, we'll explore how to transform a `DataFrame` using a User Defined Function - `udf`.


### Expected Results
As with the previous post, this is the input `DataFrame` and the expected output `DataFrame`.
{% highlight text linenos %}
// Input DataFrame
+--------+----------+----------+
|flightNo| departure|   arrival|
+--------+----------+----------+
| GOI-134|1569315038|1569319183|
|  AI-123|1569290498|1569298178|
|  TK-006|1567318178|1567351838|
+--------+----------+----------+

// Output DataFrame
+--------+--------+
|flightNo|duration|
+--------+--------+
| GOI-134|   1 hrs|
|  AI-123|   2 hrs|
|  TK-006|   9 hrs|
+--------+--------+
{% endhighlight %}

### Let's create our DataFrame
Spark defines a `DataFrame` as `type DataFrame = Dataset[Row]`, in essence it's a `Dataset` of a generic `Row`.

{% highlight scala linenos %}
import spark.implicits._
val schedules = Seq(
    ("GOI-134", 1569315038, 1569319183),
    ("AI-123", 1569290498, 1569298178),
    ("TK-006", 1567318178, 1567351838)
).toDF("flightNo", "departure", "arrival")
{% endhighlight %}

### Define the UDF
We have to define our `udf` as a variable so that that too can be passed to functions. For this, we'll need to import `org.apache.spark.sql.functions.udf`. Exactly like the previous post, our function will accept two `Long` parameters i.e. the Departure time and the Arrival time and return a `String` i.e. the duration of the flight.

{% highlight scala linenos %}
import org.apache.spark.sql.functions.udf
val getDurationInHours = udf((arrival: Long, departure: Long) => {
    val duration = (arrival - departure) / 60 / 60
    s"$duration hrs"
})
{% endhighlight %}

### Transform the DataFrame
Now all that's left is to transform the `DataFrame`. We'll do this by calling the `select` function with the `flightNo` column and the `udf` with an alias of "duration".
{% highlight scala linenos %}
import org.apache.spark.sql.functions.col
val flightInfo = schedules
    .select(
        col("flightNo"),
        getDurationInHours(col("arrival"), col("departure")) as "duration")
{% endhighlight %}

### Source Code
Here is the entire source code.

{% highlight scala linenos %}
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions.{col, udf}


object DataFrameTransform {
  def main(args: Array[String]): Unit = {

    val spark = SparkSession.builder().master("local[*]").appName("transform").getOrCreate()

    import spark.implicits._
    val schedules = Seq(
      ("GOI-134", 1569315038, 1569319183),
      ("AI-123", 1569290498, 1569298178),
      ("TK-006", 1567318178, 1567351838)
    ).toDF("flightNo", "departure", "arrival")

    // Print the input DataFrame
    schedules.show()

    // Defining the User Defined Function UDF
    val getDurationInHours = udf((arrival: Long, departure: Long) => {
      val duration = (arrival - departure) / 60 / 60
      s"$duration hrs"
    })

    // Transform DataFrame
    val flightInfo = schedules
      .select(
        col("flightNo"),
        getDurationInHours(col("arrival"), col("departure")) as "duration")

    // Print the output DataFrame
    flightInfo.show()
  }
}
{% endhighlight %}



## References and Further Reading
1. [UDFs — User-Defined Functions - Jacek Laskowski][udf-ref]
2. [User-defined functions Scala - Databricks][udf-scala]



[dataset-transform]: {% post_url 2019-09-25-how-to-transform-a-spark-dataset %}
[udf-ref]: https://jaceklaskowski.gitbooks.io/mastering-spark-sql/spark-sql-udfs.html 
[udf-scala]: https://docs.databricks.com/spark/latest/spark-sql/udf-scala.html 