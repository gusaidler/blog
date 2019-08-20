---
layout: post
title: "Simulating an UPSERT between 2 datasets using pySpark"
excerpt: "Sometimes, when working with big datasets in Hadoop, you will need to join data which share the same structure together"
image:
  path: /assets/images/posts/pyspark-upsert/spark_python.png
  thumbnail: /assets/images/posts/pyspark-upsert/spark_python.png
categories:
 - data analysis
tags:
 - hadoop
 - python
 - spark
---

I recently had a situation where an existing dataset was already stored in Hadoop HDFS, and the task was to "append" a new dataset to it.

 This new dataset shared the exact same field structure as the existing one, but it contained new rows of data **as well as data that was already present in the existing one**.  Here's an example to clarify:

<figure style="width: 400px" class="align-center">
  <img src="{{ '/assets/images/posts/pyspark-upsert/sample.png' | absolute_url }}" alt="">
  <figcaption>Dataset sample</figcaption>
</figure>

If you come from the relational database world, you probably know that an UPSERT operation would be a perfect fit for this task.

> **UPSERT(also called MERGE):** INSERTS a record to a table in a database if the record does not exist or, if the record already exists, updates the existing record.

The only available technology for me to handle this at the time was Spark, and by default, Spark doesn't support UPSERTs. Therefore, I had to implement it on my own. **I needed 2 things to make it work: a primary key and a last modified date field**

For reference, I used spark 2.4.3 in Ubuntu 18.04 LTS for this demo{: .notice--warning}

Here's what I did:

First,  let's import the relevant packages.
```python
# SparkSession: main package for DataFrame and SQL
# Window: used to enable window functions
from pyspark.sql import SparkSession, Window

# row_number: window function that will be used to create a row number column
# desc: for descending ordering
from pyspark.sql.functions import row_number, desc

spark = (SparkSession
        .builder
        .appName("Pyspark Upsert Example")
        .getOrCreate()
        )
```
Then,  I'm declaring 2 DataFrames, the first represents the existing data, and the second contains new and updated data
```python
# existing data
df = spark.read.csv(path='sample1.csv'
                    ,sep='\t'
                    ,encoding='UTF-8'
                    ,header=True
                    ,inferSchema=True)
df.show()
+---+-----+---+------+-------------------+
| id| name|age|salary| last_modified_date|
+---+-----+---+------+-------------------+
|  1| John| 30|  1000|2019-08-01 00:00:00|
|  2|Peter| 35|  1500|2019-08-02 00:00:00|
|  3| Gabe| 21|   800|2019-08-03 00:00:00|
|  4|Oscar| 29|  2000|2019-08-04 00:00:00|
|  5| Anna| 20|  1200|2019-08-05 00:00:00|
+---+-----+---+------+-------------------+

# new and updated data
df_new = spark.read.csv(path='sample2.csv'
                        ,sep='\t'
                        ,encoding='UTF-8'
                        ,header=True
                        ,inferSchema=True)
df_new.show()
+---+--------+---+------+-------------------+
| id|    name|age|salary| last_modified_date|
+---+--------+---+------+-------------------+
|  1|    John| 30|  3000|2019-08-12 00:00:00|
|  2|   Peter| 35|  3500|2019-08-12 00:00:00|
|  3|    Gabe| 21|   800|2019-08-12 00:00:00|
|  6|Patricia| 40|  2500|2019-08-12 00:00:00|
+---+--------+---+------+-------------------+
```
Both DataFrames are grouped together with **union (**which is equivalent to UNION ALL in SQL), creating the 3rd and final DataFrame. Inline row comments represent what the final result should look like:
```python
df_upsert = df.union(df_new)
df_upsert.show()
+---+--------+---+------+-------------------+
| id|    name|age|salary| last_modified_date|
+---+--------+---+------+-------------------+
|  1|    John| 30|  1000|2019-08-01 00:00:00| #remove
|  2|   Peter| 35|  1500|2019-08-02 00:00:00| #remove
|  3|    Gabe| 21|   800|2019-08-03 00:00:00| #remove
|  4|   Oscar| 29|  2000|2019-08-04 00:00:00| #keep
|  5|    Anna| 20|  1200|2019-08-05 00:00:00| #keep
|  1|    John| 30|  3000|2019-08-12 00:00:00| #keep
|  2|   Peter| 35|  3500|2019-08-12 00:00:00| #keep
|  3|    Gabe| 21|   800|2019-08-12 00:00:00| #keep
|  6|Patricia| 40|  2500|2019-08-12 00:00:00| #keep
+---+--------+---+------+-------------------+
```
As mentioned earlier, a primary key and a last modified date field are necessary to make this work. In this example, I have 'id' and 'last_modified_date'.
```python
# Set primary keys of dataframe
# It could also be a composite primary key, hence the list (e.g ['id1','id2','id3'])
primary_keys = ['id']

# Partition dataset by primary key and order by the date field
# This step will group duplicated IDs together, putting the latest (date) on top
w = Window.partitionBy(*primary_keys).orderBy(desc('last_modified_date'))
```
Now, a temporary column '_row_number' is created using the window created above. Note how the DataFrame looks like, with the primary key 'id' grouped together and ordered by 'last_modified_date' (descending):
```python
df_upsert = df_upsert.withColumn("_row_number", row_number().over(w))
df_upsert.show()

+---+--------+---+------+-------------------+-----------+
| id|    name|age|salary| last_modified_date|_row_number|
+---+--------+---+------+-------------------+-----------+
|  1|    John| 30|  3000|2019-08-12 00:00:00|          1| #keep
|  1|    John| 30|  1000|2019-08-01 00:00:00|          2| #remove
|  6|Patricia| 40|  2500|2019-08-12 00:00:00|          1| #keep
|  3|    Gabe| 21|   800|2019-08-12 00:00:00|          1| #keep
|  3|    Gabe| 21|   800|2019-08-03 00:00:00|          2| #remove
|  5|    Anna| 20|  1200|2019-08-05 00:00:00|          1| #keep
|  4|   Oscar| 29|  2000|2019-08-04 00:00:00|          1| #keep
|  2|   Peter| 35|  3500|2019-08-12 00:00:00|          1| #keep
|  2|   Peter| 35|  1500|2019-08-02 00:00:00|          2| #remove
+---+--------+---+------+-------------------+-----------+
```
The final step is to filter the DataFrame, keeping only **'_row_number' = 1**, since it represents a newer record. The '_row_number' column is then removed since it's not needed anymore, and we have our final and desired dataset!
```python
df_upsert = df_upsert.where(df_upsert._row_number == 1).drop("_row_number")
df_upsert.orderBy('id').show()

+---+--------+---+------+-------------------+
| id|    name|age|salary| last_modified_date|
+---+--------+---+------+-------------------+
|  1|    John| 30|  3000|2019-08-12 00:00:00|
|  2|   Peter| 35|  3500|2019-08-12 00:00:00|
|  3|    Gabe| 21|   800|2019-08-12 00:00:00|
|  4|   Oscar| 29|  2000|2019-08-04 00:00:00|
|  5|    Anna| 20|  1200|2019-08-05 00:00:00|
|  6|Patricia| 40|  2500|2019-08-12 00:00:00|
+---+--------+---+------+-------------------+
```
And that's how you perform an upsert using PySpark, or at least that's how I managed it. If there is another, better way, please let me know.
