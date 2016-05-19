---
layout: post
title: "Spark with Presto"
---

[Presto](https://prestodb.io/) is Facebook's open source SQL query engine.  One of its cool features is that it can combine data from multiple sources such as relational stores, HDFS, Cassandra, even streams like Kafka, and others with a single join query.  This can be useful if you want to run a Spark job on data that sits in multiple places.  Instead of creating separate RDDs for each data source and joining them in memory with Spark, you can create a logical view in Presto that represents your combined dataset.  Then, you can point to the logical view in Presto in your Spark code and pull your data from one place.

Let's say you want to create a DataFrame in Spark using a connection to Presto.  First download Presto's JDBC driver, which you can get [here](https://prestodb.io/docs/current/installation/jdbc.html).  (If you're compiling a JAR then add Presto as a dependency instead).  

In the [Spark guide](http://spark.apache.org/docs/latest/sql-programming-guide.html#jdbc-to-other-databases), it shows you how to load a DataFrame directly from the database's table, or refer you to the JdbcRDD.  *Neither options will work.*  Unfortunately Presto's JDBC driver does not implement `prepareStatement` so you will get an error like this.

{% highlight bash %}
SPARK_CLASSPATH=/path/to/presto-jdbc-0.147.jar ./bin/spark-shell
{% endhighlight %}

{% highlight scala %}
val jdbcDF = sqlContext.read.format("jdbc").options(
     Map("url" -> "jdbc:presto://{presto_ip}:{presto_port}",
     "dbtable" -> "catalog.schema.my_table",
     "user" -> "test",
     "password" -> "test")).load()
{% endhighlight %}

{% highlight bash %}
com.facebook.presto.jdbc.NotImplementedException: Method Connection.prepareStatement is not yet implemented
    at com.facebook.presto.jdbc.PrestoConnection.prepareStatement(PrestoConnection.java:106)
    at org.apache.spark.sql.execution.datasources.jdbc.JDBCRDD$.resolveTable(JDBCRDD.scala:123)
    at org.apache.spark.sql.execution.datasources.jdbc.JDBCRelation.<init>(JDBCRelation.scala:91)
    at org.apache.spark.sql.execution.datasources.jdbc.DefaultSource.createRelation(DefaultSource.scala:60)
    at org.apache.spark.sql.execution.datasources.ResolvedDataSource$.apply(ResolvedDataSource.scala:158)
    at org.apache.spark.sql.DataFrameReader.load(DataFrameReader.scala:119)
    ...
{% endhighlight %}

Are there any alternatives?  We'll have to manually build an RDD and convert it into a DataFrame using the JDBC driver.  The disadvantage to this approach is that the data has to fit in memory on the driver before it can be parallelized across many machines.

Let's try this approach.  Run Spark shell with Presto's JDBC driver.

{% highlight bash %}
./bin/spark-shell --master local[*] --jars presto-jdbc-0.147.jar 
{% endhighlight %}

In Spark shell, we need to import the following.

{% highlight scala %}
import org.apache.spark.{SparkConf, SparkContext}
import org.apache.spark.sql.SQLContext
import org.apache.spark.sql.Row
import org.apache.spark.sql.types._
import com.facebook.presto.jdbc._
import java.util.Properties
import java.sql.SQLException
{% endhighlight %}

Setting up the connection from here is the same for any JDBC driver.

{% highlight scala %}
val info = new Properties()
info.setProperty("user", "test")
info.setProperty("password", "test")

val driver = new PrestoDriver()
val connection = driver.connect("jdbc:presto://{presto_ip}:{presto_port}/catalog/schema", info) 
{% endhighlight %}

It's worth noting that even if you specify a catalog and schema here for the connection, you can still query other catalogs and schema using the fully qualified name of the table (e.g. catalog.schema.my_table), which is a way to hit your logical view.

We're going to grab the schema of the table first.  Then we'll build a buffer to hold our data and then parallelize it across machines to create the RDD.

{% highlight scala %}
val statement = connection.createStatement()
val rs = statement.executeQuery("select * from catalog.schema.my_table")
val rsmd = rs.getMetaData()

// Build schema
val schema = StructType(for (i <- 1 to rsmd.getColumnCount()) yield {
    val colName = rsmd.getColumnName(i)
    val colType = rsmd.getColumnTypeName(i)
    println(s"${colName}, ${colType}")
    colType match {
        case "BIGINT" => StructField(colName, LongType, true)
        case "TIMESTAMP" => StructField(colName, TimestampType, true)
        case "VARCHAR" => StructField(colName, StringType, true)
        case "DOUBLE" => StructField(colName, DoubleType, true)
        case _ => throw new SQLException("Unknown SQL type")
    }
})

// Build RDD
val buf = scala.collection.mutable.ArrayBuffer.empty[Row]
while (rs.next()) {
    val seq = schema.map { col => 
        col.dataType match {
            case LongType => rs.getLong(col.name)
            case TimestampType => rs.getTimestamp(col.name)
            case StringType => rs.getString(col.name)
            case DoubleType => rs.getDouble(col.name)
            case _ => throw new SQLException("Unknown SQL type")
        }
    }
    buf += Row.fromSeq(seq)
}
val rowRDD = sc.parallelize(buf)
{% endhighlight %}

Note that in this code we're only matching against four different types of data.  You can add additional ones from [here](https://prestodb.io/docs/current/language/types.html).

Now we can build the DataFrame using the schema and the RDD, register it as a temporary table, and execute queries.

{% highlight scala %}
val df = sqlContext.createDataFrame(rowRDD, schema)
df.registerTempTable("my_table")
df.sqlContext.sql("select * from my_table").show()
{% endhighlight %}
