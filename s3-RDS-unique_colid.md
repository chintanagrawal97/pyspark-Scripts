#Glue Code to add a unique column ID to data frame taken from S3 before writing to RDS

```python
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job
from pyspark.sql.functions import monotonically_increasing_id
from awsglue.dynamicframe import DynamicFrame

## @params: [JOB_NAME]

args = getResolvedOptions(sys.argv, ['JOB_NAME'])
 
sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

#adding column
df1 = spark.read.option("header","true").csv("s3://<path-to-file>").withColumn("id", monotonically_increasing_id()) 
 
#converting data frame to dybamic frame
dyF = DynamicFrame.fromDF(df1, glueContext, "dyF")
 
#write dynamic frame to RDS
outRds = glueContext.write_dynamic_frame.from_jdbc_conf(frame = dyF, catalog_connection = "<jdbc-connection-name>", connection_options = {"database" : "<db-name>", "dbtable" : "<name-of-table-to-create>"}, transformation_ctx = "outRds")

job.commit()
```