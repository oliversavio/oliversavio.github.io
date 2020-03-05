---
layout: post
title:  Transforming Spark Datasets using Scala transformation functions
tags: [spark, scala]
--- 

There are few times where I've chosen to use Spark as an ETL tool for it's ease of use when it comes to reading/writing parquet, csv or xml files. Reading any of these file formats is as simple as one line of spark code (after you ensure that you have the required dependencies of course)

![Transforming Spark Datasets]({{ site.url }}/img/dataset_transform.png)


### Intent
Most of the reference material available online for transforming Datasets points to calling `createOrReplaceTempView()` and registering the `Dataset/Dataframe` as a table, then performing `SQL` operations on it. 

While this may be fine for most use cases, there are times when it feels more natural to use Spark's Scala transformation functions instead, especially if you already have some pre existing transformation logic written in `Scala` or `ReactiveX` or even `Java 8`.

If this sounds like something you'd like to do, then please read on.

### Getting Started
Let's assume we want to make the following transformation using Datasets.

```
Input Dataset
+--------+----------+----------+
|flightNo| departure|   arrival|
+--------+----------+----------+
| GOI-134|1569315038|1569319183|
|  AI-123|1569290498|1569298178|
|  TK-006|1567318178|1567351838|
+--------+----------+----------+

Output Dataset
+-------+--------+
| flight|duration|
+-------+--------+
|GOI-134|   1 hrs|
| AI-123|   2 hrs|
| TK-006|   9 hrs|
+-------+--------+
```
### Domain objects and the transformation function
We'll assume we have the following domain objects and transformation function that converts a `FlightSchedule` object into a `FlightInfo` object.

{% highlight scala linenos %}
case class FlightSchedule(flightNo: String, departure: Long, arrival: Long)

case class FlightInfo(flight: String, duration: String)


def existingTransformationFunction(flightSchedule: FlightSchedule): FlightInfo = {
  val duration = (flightSchedule.arrival - flightSchedule.departure) / 60 / 60
  FlightInfo(flightSchedule.flightNo, s"$duration hrs")
}

{% endhighlight %}

### Creating the spark session and reading the input Dataset
Creating the input `Dataset` is kept simple for brevity.

{% highlight scala linenos %}
val spark = SparkSession.builder().master("local[*]").appName("transform").getOrCreate()

import spark.implicits._

val schedules: Dataset[FlightSchedule] = Seq(
    FlightSchedule("GOI-134", 1569315038, 1569319183),
    FlightSchedule("AI-123", 1569290498, 1569298178),
    FlightSchedule("TK-006", 1567318178, 1567351838)
).toDS()

{% endhighlight %}

### Defining the Encoder and the Spark transformation
This is where things start to get interesting. In order to transform a `Dataset[FlightSchedule]` to a `Dataset[FlightInfo]`,
Spark needs to know how to _"Encode"_ your `case class`. Omitting this step will give you the following dreadful compile time error.

```
Error:(34, 18) Unable to find encoder for type stored in a Dataset.  Primitive types (Int, String, etc) and Product types (case classes) are supported by importing spark.implicits._  Support for serializing other types will be added in future releases.
    schedules.map(s => existingTransformationFunction(s))
    
```
`Encoders[T]` are used to convert any JVM object or primitive of type `T` to and from Spark SQL's [InternalRow][internal-row] representation.
Since the `Dataset.map()` method requires an encoder to be passed as an implicit parameter, we'll define an `implicit` variable.

{% highlight scala linenos %}
def makeFlightInfo(schedules: Dataset[FlightSchedule]): Dataset[FlightInfo] = {
  implicit val encoder: ExpressionEncoder[FlightInfo] = ExpressionEncoder[FlightInfo]
  schedules.map(s => existingTransformationFunction(s))
}
{% endhighlight %}

### Transforming the Dataset
The only thing left to do now is call the `transform` method on the input `Dataset`. I will include the entire code here along with calls to `show()` so that we can see our results.

{% highlight scala linenos %}
object DatasetTransform {
  def main(args: Array[String]): Unit = {

    val spark = SparkSession.builder().master("local[*]").appName("transform").getOrCreate()

    import spark.implicits._

    val schedules: Dataset[FlightSchedule] = Seq(
      FlightSchedule("GOI-134", 1569315038, 1569319183),
      FlightSchedule("AI-123", 1569290498, 1569298178),
      FlightSchedule("TK-006", 1567318178, 1567351838)
    ).toDS()

    // Print the input Dataset
    schedules.show()

    // Transform Dataset
    val flightInfo: Dataset[FlightInfo] = schedules.transform(makeFlightInfo)

    // Print the output Dataset
    flightInfo.show()
  }


  def makeFlightInfo(schedules: Dataset[FlightSchedule]): Dataset[FlightInfo] = {
    implicit val enc: ExpressionEncoder[FlightInfo] = ExpressionEncoder[FlightInfo]
    schedules.map(s => existingTransformationFunction(s))
  }

  def existingTransformationFunction(flightSchedule: FlightSchedule): FlightInfo = {
    val duration = (flightSchedule.arrival - flightSchedule.departure) / 60 / 60
    FlightInfo(flightSchedule.flightNo, s"$duration hrs")
  }

}
{% endhighlight %}



### References
1. [Spark SQL Encoders][spark-sql-encoder]

[spark-sql-encoder]: https://jaceklaskowski.gitbooks.io/mastering-spark-sql/spark-sql-Encoder.html
[internal-row]:https://jaceklaskowski.gitbooks.io/mastering-spark-sql/spark-sql-InternalRow.html