---
layout: post
title: Building an analytical data lake with Apache Spark and Apache Hudi - Part 1
tags: [spark, big-data , scala]
subtitle: Using Apache Spark and Apache Hudi to build and manage data lakes on DFS and Cloud storage.
--- 

Most modern data lakes are build using some sort of distributed file system (DFS) like HDFS or cloud based storage like AWS S3.
One of the underlying principles followed is the "write-once-read-many" access model for files. This is great when working with large volumes of data, think hundreds of gigabytes to terabytes.

However, when building an analytical data lake it is not uncommon to have data that gets updated. Depending on you use case these updates could be as frequent as hourly to probably daily or weekly updates. You may also need to run analytics over the most up-to-date data, historical data containing all the updates or even just the latest increments.

Very often this leads to having separate systems for stream and batch processing. The former handling incremental data while the latter deals with historical data.

![Stream and Batch Processing pipelines]({{ "/img/data_processing_pipelines.png" | absolute_url }})


A common workflow to maintain incremental updates when working with data stored on HDFS is the Ingest-Reconcile-Compact-Purge strategy described [here][hive-four-step-strategy].

![Insert-Append-Compact-Overwrite Data Flow]({{ "/img/Four-step-strategy.png" | absolute_url }})

Here's where a framework like Apache Hudi comes in to play, it manages this workflow for us under the hood which leads to our core application code being a lot cleaner. Hudi supports queries over the most up-to-date view of the data as well as incremental changes at a point in time.

Part one will introduce the core Hudi concepts and working with Copy on Write tables.

## Outline
- [Perquisites and framework versions](#perquisites-and-framework-versions)
- [Core Hudi concepts](#core-hudi-concepts)
- [Initial Setup and Dependencies](#initial-setup-and-dependencies)
- [Working with CoW tables](#working-with-cow-tables)
- [Selecting Records](#monitoring-your-docker-container) 
- [Deleting Records](#monitoring-your-docker-container) 

## [Perquisites and framework versions](#perquisites-and-framework-versions)
Prior knowledge of writing spark jobs in scala and reading and writing parquet files would make following this post a breeze.

#### Framework versions
 - __JDK:__ openjdk version 1.8.0_242
 - __Scala:__ version 2.12.8
 - __Spark:__ version 2.4.4
 - __Hudi Spark bundle:__ version 0.5.2-incubating 
 
 _Note: As of writing this, AWS EMR comes bundled with Hudi v0.5.0-incubating which has a bug which causes `upsert` operations to freeze or take a long time to complete. You can read more about the issue [here][hudi-issue]. The issue has been fixed in the current version of Hudi. If you plan on running on your code on AWS EMR you may want to consider overriding the bundled version with the latest version._
 
## [Core Hudi concepts](#core-hudi-concepts)
Let's start off with some of the core concepts that need to be understood.

### Types of Tables
Hudi supports two types of tables
 1. __Copy on Write (CoW):__ 
When writing to a CoW tables, the __*Insert-Compact-Overwrite*__ cycle is run. Data in CoW tables will always be up-to-date with the latest records after every write operation. This mode is preferred for use cases that need to read the most up-to-date data as quickly as possible. Data is exclusively stored in the columnar file format (parquet) in CoW tables. Since every write operation involves compaction and overwrite, this mode produces the smallest files. 

 2. __Merge on Read (MoR):__ MoR tables are focused on fast write operations. Writing to these tables creates delta files which are later compacted to produce the up-to-date data on reading. The compaction operation may be done synchronously or asynchronously. Data is stored in a combination of columnar file format (parquet) as well as row based file format (avro).

 Here are some of the tradeoffs between the two table formats as mentioned in the Hudi documentation.

|Trade-off           |  CopyOnWrite                      |	MergeOnRead                              |
|--------------------|-----------------------------------|-------------------------------------------|
|Data Latency        | 	Higher                           |	Lower                                    |
|Update cost (I/O) 	 |  Higher (rewrite entire parquet)  |	Lower (append to delta log)              |    
|Parquet File Size 	 |  Smaller (high update(I/0) cost)  |	Larger (low update cost)                 |
|Write Amplification |  Higher                           |	Lower (depending on compaction strategy) |


### Types of Queries
Hudi supports two main types of queries, __"Snapshot Queries"__ and __"Incremental Queries"__. In addition to the two main types of queries, MoR tables support __"Read Optimized Queries"__.
 1. __Snapshot Queries:__ Snapshot queries return the latest view of the data in case of CoW tables and a near real time view with MoR tables. In the case of MoR tables, a snapshot query will merge the base files and the delta files on the fly, hence you can expect some latency. With CoW, since write are responsible for merging, reads are quick since you only need to read the base files.

 2. __Incremental Queries:__ Incremental queries allow you to view data after a particular commit time by specifying a "begin" time or at a point in time by specifying a "begin" and an "end" time.

 3. __Read Optimized Queries:__ For MoR tables, read optimized queries return a view which only contains the data in the base files without merging delta files.


### Important properties when writing a Dataframe in Hudi format
 - `hoodie.datasource.write.table.type` - This defines the table type, the default value is `COPY_ON_WRITE`. For a MoR table, set this value to `MERGE_ON_READ`. 

 - `hoodie.table.name` - This is a mandatory filed, every table you write should have a unique name.

 - `hoodie.datasource.write.recordkey.field` - Think of this as the primary key of your table. The value for this property is the name of the column in your Dataframe which is the primary key.

 - `hoodie.datasource.write.precombine.field` - When upserting data, if there exists two records with the same primary key, the value in this column will decided which records get upserted. Selecting a column like timestamp will ensure the record with the latest timestamp is picked.

 - `hoodie.datasource.write.operation` - This defines the type of write operation, the possible values are `upsert`, `insert`, `bulk_insert` and `delete`, `upsert` is the default.


## [Initial Setup and Dependencies](#initial-setup-and-dependencies)
  
### Declaring the dependencies 
In order to use Hudi with your Spark jobs you'll need the `spark-sql`, `hudi-spark-bundle` and `spark-avro` dependencies. Additionally you'll need to configure spark to use the `KryoSerializer`.

Here a snippet of the `pom.xml`
{% highlight xml linenos %}
<properties>
    <maven.compiler.source>1.8</maven.compiler.source>
    <maven.compiler.target>1.8</maven.compiler.target>
    <encoding>UTF-8</encoding>
    <scala.version>2.12.8</scala.version>
    <scala.compat.version>2.12</scala.compat.version>
    <spec2.version>4.2.0</spec2.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.scala-lang</groupId>
        <artifactId>scala-library</artifactId>
        <version>${scala.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-sql_${scala.compat.version}</artifactId>
        <version>2.4.4</version>
    </dependency>
    <dependency>
        <groupId>org.apache.hudi</groupId>
        <artifactId>hudi-spark-bundle_${scala.compat.version}</artifactId>
        <version>0.5.2-incubating</version>
    </dependency>
    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-avro_${scala.compat.version}</artifactId>
        <version>2.4.4</version>
    </dependency>
</dependencies>
{% endhighlight %}


### Setting up the schema 
We'll use the following `Album` case class to represent the schema of our table.

{% highlight scala linenos %}
case class Album(albumId: Long, title: String, tracks: Array[String], updateDate: Long)   
{% endhighlight %}


### Creating some test data
Let's create some data we'll use for the upsert operation. 

Things to notice:
- `INITIAL_ALBUM_DATA` has two records with the key `801`
- `UPSERT_ALBUM_DATA` contains one updated record and two new records.

{% highlight scala linenos %}
def dateToLong(dateString: String): Long = LocalDate.parse(dateString, formatter).toEpochDay

private val INITIAL_ALBUM_DATA = Seq(
    Album(800, "6 String Theory", Array("Lay it down", "Am I Wrong", "68"), dateToLong("2019-12-01")),
    Album(801, "Hail to the Thief", Array("2+2=5", "Backdrifts"), dateToLong("2019-12-01")),
    Album(801, "Hail to the Thief", Array("2+2=5", "Backdrifts", "Go to sleep"), dateToLong("2019-12-03"))
  )

  private val UPSERT_ALBUM_DATA = Seq(
    Album(800, "6 String Theory - Special", Array("Jumpin' the blues", "Bluesnote", "Birth of blues"), dateToLong("2020-01-03")),
    Album(802, "Best Of Jazz Blues", Array("Jumpin' the blues", "Bluesnote", "Birth of blues"), dateToLong("2020-01-04")),
    Album(803, "Birth of Cool", Array("Move", "Jeru", "Moon Dreams"), dateToLong("2020-02-03"))
  )
{% endhighlight %}


### Initializing the Spark context
Finally we'll initialize the Spark context. One important point to note here is the use of the `KryoSerializer`.
{% highlight scala linenos %}
val spark: SparkSession = SparkSession.builder()
    .appName("hudi-datalake")
    .master("local[*]")
    .config("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
    .config("spark.sql.hive.convertMetastoreParquet", "false") // Uses Hive SerDe, this is mandatory for MoR tables
    .getOrCreate()
{% endhighlight %}

## [Working with CoW tables](#working-with-cow-tables)
In this section we'll go over writing, reading and deleting records when working with CoW tables.

### Base path & Upsert method
Define a `basePath` where the table will be written and an `upsert` method. The method will write the `Dataframe` in the `org.apache.hudi` format. Notice all the Hudi properties that were discussed above have been set.

{% highlight scala linenos %}
val basePath = "/tmp/store"

private def upsert(albumDf: DataFrame, tableName: String, key: String, combineKey: String) = {
    albumDf.write
      .format("hudi")
      .option(DataSourceWriteOptions.TABLE_TYPE_OPT_KEY, DataSourceWriteOptions.COW_TABLE_TYPE_OPT_VAL)
      .option(DataSourceWriteOptions.RECORDKEY_FIELD_OPT_KEY, key)
      .option(DataSourceWriteOptions.PRECOMBINE_FIELD_OPT_KEY, combineKey)
      .option(HoodieWriteConfig.TABLE_NAME, tableName)
      .option(DataSourceWriteOptions.OPERATION_OPT_KEY, DataSourceWriteOptions.UPSERT_OPERATION_OPT_VAL)
      .mode(SaveMode.Append)
      .save(s"$basePath/$tableName/")
  }
{% endhighlight %}

### Initial Upsert
Insert `INITIAL_ALBUM_DATA`, we should have 2 records created and for `801`,the records with date `2019-12-03`.
{% highlight scala linenos %}
val tableName = "Album"
upsert(INITIAL_ALBUM_DATA.toDF(), tableName, "albumId", "updateDate")
spark.read.format("hudi").load(s"$basePath/$tableName/*").show()
{% endhighlight %}

Reading a `CoW` table is as simple as a regular `spark.read` with `format("hudi")`.
{% highlight text linenos %}
// Output
+-------------------+--------------------+------------------+----------------------+--------------------+-------+-----------------+--------------------+----------+
|_hoodie_commit_time|_hoodie_commit_seqno|_hoodie_record_key|_hoodie_partition_path|   _hoodie_file_name|albumId|            title|              tracks|updateDate|
+-------------------+--------------------+------------------+----------------------+--------------------+-------+-----------------+--------------------+----------+
|     20200412182343|  20200412182343_0_1|               801|               default|65841d0a-0083-447...|    801|Hail to the Thief|[2+2=5, Backdrift...|     18233|
|     20200412182343|  20200412182343_0_2|               800|               default|65841d0a-0083-447...|    800|  6 String Theory|[Lay it down, Am ...|     18231|
+-------------------+--------------------+------------------+----------------------+--------------------+-------+-----------------+--------------------+----------+

{% endhighlight %}

Another way to be sure is to look at the `Workload profile` log output.
{% highlight text linenos %}
Workload profile :WorkloadProfile {globalStat=WorkloadStat {numInserts=2, numUpdates=0}, partitionStat={default=WorkloadStat {numInserts=2, numUpdates=0}}}
{% endhighlight %}

### Updating Records
{% highlight scala linenos %}
upsert(UPSERT_ALBUM_DATA.toDF(), tableName, "albumId", "updateDate")
{% endhighlight %}

Let's look at the `Workload profile` any verify it's as expected
{% highlight text linenos %}
Workload profile :WorkloadProfile {globalStat=WorkloadStat {numInserts=2, numUpdates=1}, partitionStat={default=WorkloadStat {numInserts=2, numUpdates=1}}}
{% endhighlight %}

Let's look at the output
{% highlight text linenos %}
spark.read.format("hudi").load(s"$basePath/$tableName/*").show()

//Output
+-------------------+--------------------+------------------+----------------------+--------------------+-------+--------------------+--------------------+----------+
|_hoodie_commit_time|_hoodie_commit_seqno|_hoodie_record_key|_hoodie_partition_path|   _hoodie_file_name|albumId|               title|              tracks|updateDate|
+-------------------+--------------------+------------------+----------------------+--------------------+-------+--------------------+--------------------+----------+
|     20200412183510|  20200412183510_0_1|               801|               default|65841d0a-0083-447...|    801|   Hail to the Thief|[2+2=5, Backdrift...|     18233|
|     20200412184040|  20200412184040_0_1|               800|               default|65841d0a-0083-447...|    800|6 String Theory -...|[Jumpin' the blue...|     18264|
|     20200412184040|  20200412184040_0_2|               802|               default|65841d0a-0083-447...|    802|  Best Of Jazz Blues|[Jumpin' the blue...|     18265|
|     20200412184040|  20200412184040_0_3|               803|               default|65841d0a-0083-447...|    803|       Birth of Cool|[Move, Jeru, Moon...|     18295|
+-------------------+--------------------+------------------+----------------------+--------------------+-------+--------------------+--------------------+----------+

{% endhighlight %}

### Querying Records
The way we've been looking at our data above is known as a "Snapshot query", this is the default. Another query that's supported is an "Incremental query".

#### Incremental Queries
To perform an incremental query, we'll need to set the `hoodie.datasource.query.type` property to `incremental` when reading as well as specify the `hoodie.datasource.read.begin.instanttime` property. This will read all records after the specified instant time, for our example here, looking at the `_hoodie_commit_time` colum we have two distinct values, we'll specify an instant time of `20200412183510`.

{% highlight scala linenos %}
spark.read
    .format("hudi")
    .option(DataSourceReadOptions.QUERY_TYPE_OPT_KEY, DataSourceReadOptions.QUERY_TYPE_INCREMENTAL_OPT_VAL)
    .option(DataSourceReadOptions.BEGIN_INSTANTTIME_OPT_KEY, "20200412183510")
    .load(s"$basePath/$tableName")
    .show()
{% endhighlight %}

This will return all records after commit time `20200412183510` which should be 3.

{% highlight text linenos %}
+-------------------+--------------------+------------------+----------------------+--------------------+-------+--------------------+--------------------+----------+
|_hoodie_commit_time|_hoodie_commit_seqno|_hoodie_record_key|_hoodie_partition_path|   _hoodie_file_name|albumId|               title|              tracks|updateDate|
+-------------------+--------------------+------------------+----------------------+--------------------+-------+--------------------+--------------------+----------+
|     20200412184040|  20200412184040_0_1|               800|               default|65841d0a-0083-447...|    800|6 String Theory -...|[Jumpin' the blue...|     18264|
|     20200412184040|  20200412184040_0_2|               802|               default|65841d0a-0083-447...|    802|  Best Of Jazz Blues|[Jumpin' the blue...|     18265|
|     20200412184040|  20200412184040_0_3|               803|               default|65841d0a-0083-447...|    803|       Birth of Cool|[Move, Jeru, Moon...|     18295|
+-------------------+--------------------+------------------+----------------------+--------------------+-------+--------------------+--------------------+----------+

{% endhighlight %}

### Deleting Records
The last operation we'll look at is delete. Delete is similar to upsert, we'll need a `Dataframe` of the records that need to be deleted. The entire row isn't necessary, we only need `keys`, as you'll see in the sample code below.
{% highlight scala linenos %}
val deleteKeys = Seq(
    Album(803, "", null, 0l),
    Album(802, "", null, 0l)
)

import spark.implicits._

val df = deleteKeys.toDF()

df.write.format("hudi")
    .option(DataSourceWriteOptions.TABLE_TYPE_OPT_KEY, DataSourceWriteOptions.COW_TABLE_TYPE_OPT_VAL)
    .option(DataSourceWriteOptions.RECORDKEY_FIELD_OPT_KEY, "albumId")
    .option(HoodieWriteConfig.TABLE_NAME, tableName)
    // Set the option "hoodie.datasource.write.operation" to "delete"
    .option(DataSourceWriteOptions.OPERATION_OPT_KEY, DataSourceWriteOptions.DELETE_OPERATION_OPT_VAL)
    .mode(SaveMode.Append) // Only Append Mode is supported for Delete.
    .save(s"$basePath/$tableName/")

spark.read.format("hudi").load(s"$basePath/$tableName/*").show()
{% endhighlight %}


## References
1. [Hudi upsert hangs #1328][hudi-issue]
2. [Hudi Documentation][hudi-documentation]
3. [Hoodie: An Open Source Incremental Processing Framework From Uber][hudi-youtube]
4. [Hive four step strategy][hive-four-step-strategy]



[Part1]:{% post_url 2020-02-29-running-java-apps-with-docker %}
[hudi-issue]: https://github.com/apache/incubator-hudi/issues/1328
[hudi-documentation]: https://hudi.apache.org/docs/quick-start-guide.html
[hudi-youtube]: https://www.youtube.com/watch?v=7Wudjc-v7CA
[hive-four-step-strategy]: https://community.cloudera.com/t5/Community-Articles/FOUR-STEP-STRATEGY-FOR-INCREMENTAL-UPDATES-IN-APACHE-HIVE-ON/ta-p/246015 