# Terminal Code
### After creating your project 2 directory, read in the json file with assessment data
curl -L -o assessment-attempts-20180128-121051-nested.json https://goo.gl/ME6hjp
### Ensure all docker instances have been closed
docker-compose pos
### Spin up the cluster and make sure Kafka is working properly
docker-compose up -d
docker-compose logs -f kafka
### Check that Hadoop is running properly
docker-compose exec cloudera hadoop fs -ls /tmp/
### Create a topic for your exams
docker-compose exec kafka \
  kafka-topics \
    --create \
    --topic exams \
    --partitions 1 \
    --replication-factor 1 \
    --if-not-exists \
    --zookeeper zookeeper:32181
    
### Produce your messages from the json file with kafkacat 
 docker-compose exec mids \
  bash -c "cat /w205/project-2-bkbormes1/assessment-attempts-20180128-121051-nested.json \
    | jq '.[]' -c \
    | kafkacat -P -b kafka:29092 -t exams" && echo 'Produced json messages.'"
### Spin up a pyspark process using the spark container (used python_history file for this)
docker-compose exec spark pyspark
### Read from Kafka
raw_exams = spark \
  .read \
  .format("kafka") \
  .option("kafka.bootstrap.servers", "kafka:29092") \
  .option("subscribe","commits") \
  .option("startingOffsets", "earliest") \
  .option("endingOffsets", "latest") \
  .load() 
### Cache to cut back on warnings
raw_exams.cache()
### Check out what your schema looks like
raw_exams.printSchema()
### Cast your data as strings and check it out and its schema
exams = raw_exams.select(raw_exams.value.cast('string'))
exams.show()
exams.printSchema()
### Write your data to HDFS
exams.write.parquet("/tmp/exams")
### Deal with unicode
import sys
sys.stdout = open(sys.stdout.fileno(), mode='w', encoding='utf8', buffering=1)
### Write to a data frame to extract more fields and look at what you have
import json
extracted_exams = exams.rdd.map(lambda x: json.loads(x.value)).toDF()
extracted_exams.show()
extracted_exams.printSchema()
### Use SparkSQL and create a temporary view
extracted_exams.registerTempTable('exams')
### Query your nested data
disctinct_exams = spark.sql("select count(distinct exam_name) from exams")
distinct_exams.show()
Output:

| count(DISTINCT exam_name)       |
| :-------------: |
|  103 |

<br/>

intro_to_python = spark.sql("select count(exam_name) from exams where exam_name = 'Introduction to Python'")
intro_to_python.show()
Output: <br/>

| count(exam_name)       |
| :-------------: |
|  162 |


### Write your data to HDFS
disctinct_exams.write.parquet("/tmp/disctinct_exams")
intro_to_python.write.parquet("/tmp/intro_to_python")
### In another terminal check out what you have
docker-compose exec cloudera hadoop fs -ls /tmp/
docker-compose exec cloudera hadoop fs -ls /tmp/exams/
### Close your docker instance
docker-compose down

# Describe Queries used and business questions
### Questions answered, assumptions, and analysis
1) How many exams exist in the data?
- disctinct_exams = spark.sql("select count(distinct exam_name) from exams")
- 103 unique exams are listed, most likely for 103 different types of courses assuming that each course has their own exam. This is important to understand because that means there are a lot of courses offered, which means a lot of resources are needed to create these different classes. <br/>

2) How many people took Introduction to Python?
- intro_to_python = spark.sql("select count(exam_name) from exams where exam_name = 'Introduction to Python'")
- 162 people took the exam Introduction to Python. Assuming that each class has only one exam, that means there were 162 students that took the Introduction to Python course. Understanding class size is important when analyzing test scores and understanding how individualized the instruction was for each student. <br/>

3) What's the name of your Kafka topic? How did you come up with that name?
- My Kafka topic is named exams because the data we are analyzing and querying withing the topic is assessment data. If I wanted to make one topic per query, I could have even more specific topic names such as num_distinct_exams to be the topic that queries the number of distinct exams. As I continue using data pipelines, it will be important to name topics with specific names that give detail as to what is going on within the pipeline.


### Describe any issues and how to solve it
Because the json file was heavily nested, it was important to analyze the data using Spark SQL. This allowed me to pull any desired data field regardless of how nested it was. If I wanted to pull the number of correct exam questions, I could do spark.sql("select sequences.counts.correct ... etc. "). This syntax allows me to easily navigate different variables within the nested json file.
