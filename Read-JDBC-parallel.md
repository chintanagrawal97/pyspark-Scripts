## Glue - How To Read A MySQL JDBC Source In Parallel 

### Short Summary 
This article will explain how to read JDBC sources in Parallel using Spark

### Main Text 
Often I have seen customers complaining about slow execution of Spark jobs when they are reading data from JDBC sources. Their Jobs are either very slow or fails with OOM or running out of disk space.

This is caused by one executor doing work. Meaning, 1 connection is opened to the DB and using this 1 connection, 1 executor does the work by reading this massive table. 

In this article, I've used MySQL 8 as a test. As of now, the driver we use in Glue doesnt support MySQL 8. Therefore, a workaround for this is to bring your own JDBC driver compatible with MySQL 8 and up.However, what I am about to explain works for JDBC sources, not just MySQL 8.

So in order to overcome this, we need to parallelize reading from the DB. We can achieve this by using the following parameters:

<strong>partitionColumn </strong> - The incremental column in your table. This MUST be an incremental column in your table.

<strong>lowerBound </strong>lowerBound - The lower boundary of the partitionColum which can be 0 (we start at 0)

<strong>upperBound </strong> - The maximum upper boundary of the partitionColumn which could be set to 10000, for example.

<strong>numPartitions </strong>  - This value is based on the amount of cores you have available in your job, this is the maximum number of partitions that can be used for parallelism in table reading. This also determines the maximum number of concurrent JDBC connections.

Note that lowerBound and upperBound are used only to decide the partition stride, not for filtering the rows in the table. Also, these values besides numPartitions only applies to reading JDBC.

Now lets move to a sample code when reading from a table in my DB.

### MySQL Engine version 8.0.15

The sample table in the DB table looks like this, notice that id is an integer column:
```sql
> select * from animals;
+----+---------+
| id | name    |
+----+---------+
|  1 | dog     |
|  2 | cat     |
|  3 | penguin |
|  4 | lax     |
|  5 | whale   |
|  6 | ostrich |
+----+---------+
```
Now we'll start using Spark to read using the code below:

```python
df1 = sqlContext.read.format("jdbc")\
    .option("url", "jdbc:mysql://end.pointofjdbcmysql.us-east-1.rds.amazonaws.com:3306/glue")\
    .option("dbtable", "animals")\
    .option("driver", "com.mysql.cj.jdbc.Driver")\
    .option("user", "user")\
    .option("password", "password")\
    .option("numPartitions", 10)\
    .option("partitionColumn", "id")\
    .option("lowerBound", 0)\
    .option("upperBound", 10)\
    .load()
```
```python
df1.rdd.getNumPartitions()
10
```

As you can see, the number of partitions read is 10. During our read, I segmented the read to create 10 partitions of the table.

