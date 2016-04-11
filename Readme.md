# 1 Preparation on the HDP cluster 

**Note:** The description here is for HDP 2.4.0.0

For Hortonworks Data Platform 2.4 the Spark assembly file is not on HDFS. It is helpful to (or ask the HDP admin to) copy the assembly to its default location `hdfs://hdp//hdp/apps/2.4.0.0-169/spark//spark/`

```bash
sudo su - hdfs
hdfs dfs -mkdir "/hdp/apps/2.4.0.0-169/spark/"
hdfs dfs -put "/usr/hdp/2.4.0.0-169/spark/lib/spark-assembly-1.6.0.2.4.0.0-169-hadoop2.7.1.2.4.0.0-169.jar" "/hdp/apps/2.4.0.0-169/spark/spark-hdp-assembly.jar"
```

# 2 Submit a project to Spark from your workstation

## 2.1 Load data and copy it into HDFS

Only a small data set, however sufficient for a sample

```bash
wget https://archive.ics.uci.edu/ml/machine-learning-databases/iris/iris.data
```

Create a project folder in HDFS using WebHDFS

```bash
export WEBHDFS_HOST=http://beebox01:50070

curl -X PUT "$WEBHDFS_HOST/webhdfs/v1/tmp/simple-project?op=MKDIRS"
# {"boolean":true}
```

Upload data to project folder using WebHDFS

```bash
curl -i -X PUT "$WEBHDFS_HOST/webhdfs/v1/tmp/simple-project/iris.data?op=CREATE&overwrite=true"
# HTTP/1.1 307 TEMPORARY_REDIRECT
# Cache-Control: no-cache
# Expires: Sun, 10 Apr 2016 11:35:44 GMT
# Date: Sun, 10 Apr 2016 11:35:44 GMT
# Pragma: no-cache
# Expires: Sun, 10 Apr 2016 11:35:44 GMT
# Date: Sun, 10 Apr 2016 11:35:44 GMT
# Pragma: no-cache
# Location: http://beebox06.localdomain:50075/webhdfs/v1/tmp/simple-project/simple-project_2.10-1.0.jar?op=CREATE&namenoderpcaddress=beebox01.localdomain:8020&createflag=&# createparent=true&overwrite=true
# Content-Type: application/octet-stream
# Content-Length: 0
# Server: Jetty(6.1.26.hwx)

LOCATION="http://beebox06.localdomain:50075/webhdfs/v1/tmp/simple-project/simple-project_2.10-1.0.jar?op=CREATE&namenoderpcaddress=beebox01.localdomain:8020&createflag=&createparent=true&overwrite=true"

curl -i -X PUT -T "iris.data" "$LOCATION"
```

## 2.2 Create Spark project and copy it to HDFS

Simple project to calculate mean of each feature per species

```bash
cd simple-project
sbt package

export APP_FILE=simple-project_2.10-1.0.jar

curl -i -X PUT "$WEBHDFS_HOST/webhdfs/v1/tmp/simple-project/$APP_FILE?op=CREATE&overwrite=true"
# take Location header, see above

LOCATION="http://..."

curl -i -X PUT -T "target/scala-2.10/$APP_FILE" "$LOCATION"

cd ..
```

## 2.3 Populate the control files for the YARN REST API

### 2.3.1 Spark properties

Copy `spark-yarn.properties.template` to `spark-yarn.properties` and edit the first two lines (`spark.app.name`, `spark.yarn.historyServer.address=beebox01.localdomain\:18080`)

Upload `spark-yarn.properties` to the project folder in HDFS

```bash
curl -i -X PUT "$WEBHDFS_HOST/webhdfs/v1/tmp/simple-project/spark-yarn.properties?op=CREATE&overwrite=true"
# take Location header, see above

LOCATION="http://..."
curl -i -X PUT -T "spark-yarn.properties" "$LOCATION"
```

### 2.3.2 The JSON job file for the YARN REST API

For caching purposes Spark needs file sizes and modification times of all project files. The following commands use the json processor `jq` from [https://stedolan.github.io/jq/](https://stedolan.github.io/jq/)

```bash
curl -s "$WEBHDFS_HOST/webhdfs/v1/hdp/apps/2.4.0.0-169/spark/spark-hdp-assembly.jar?op=GETFILESTATUS" \
| jq '.FileStatus | {size: .length, timestamp: .modificationTime}'
# {
#   "size": 191724610,
#   "timestamp": 1460219553714
# }
curl -s "$WEBHDFS_HOST/webhdfs/v1/tmp/simple-project/simple-project_2.10-1.0.jar?op=GETFILESTATUS" \
| jq '.FileStatus | {size: .length, timestamp: .modificationTime}'
# {
#   "size": 10270,
#   "timestamp": 1460288240001
# }
curl -s "$WEBHDFS_HOST/webhdfs/v1/tmp/simple-project/spark-yarn.properties?op=GETFILESTATUS" \
| jq '.FileStatus | {size: .length, timestamp: .modificationTime}'
# {
#   "size": 767,
#   "timestamp": 1460289956356
# }
```

Copy `spark-yarn.json.template` to `spark-yarn.json` and edit all `local-resources`:

- `resource`: adapt namenode address
- `timestamp`, `size`: according to the above values

Next edit the `enviroments` section and modify the keys `SPARK_YARN_CACHE_FILES`, `SPARK_YARN_CACHE_FILES_FILE_SIZES`, `SPARK_YARN_CACHE_FILES_TIME_STAMPS` so that file names, timestamps and sizes are the same as in the `local_resources` section.

Note: The properties file is only for the Application Master and can be ignored here.

Also adapt versions in `CLASSPATH` of section `environment` and in the `command`

## 2.4 Submit a Spark job to YARN

### 2.4.1 Create a YARN application

```bash
export HADOOP_RM=http://beebox04:8088

curl -s -X POST $HADOOP_RM/ws/v1/cluster/apps/new-application | jq .
# {
#   "application-id": "application_1460195242962_0051",
#   "maximum-resource-capability": {
#     "memory": 4000,
#     "vCores": 3
#   }
# }
```

Edit `spark-yarn.json` again and modify the `application-id` to hold the newly create id.

### 2.4.2 Delete the job output folder

```bash
curl -i -X DELETE "$HADOOP_RM/webhdfs/v1//tmp/iris/means?op=DELETE&recursive=true"
```

### 2.4.3 Submit the Spark job

```bash 
curl -s -X POST -H "Content-Type: application/json" $HADOOP_RM/ws/v1/cluster/apps --data-binary spark-yar.json 
# HTTP/1.1 100 Continue
# 
# HTTP/1.1 202 Accepted
# Cache-Control: no-cache
# Expires: Sun, 10 Apr 2016 13:02:47 GMT
# Date: Sun, 10 Apr 2016 13:02:47 GMT
# Pragma: no-cache
# Expires: Sun, 10 Apr 2016 13:02:47 GMT
# Date: Sun, 10 Apr 2016 13:02:47 GMT
# Pragma: no-cache
# Content-Type: application/json
# Location: http://beebox04:8088/ws/v1/cluster/apps/application_1460195242962_0054
# Content-Length: 0
# Server: Jetty(6.1.26.hwx)
```

### 2.4.4 Get job status and result

Take the `Location` header from above:

```bash
curl -s http://beebox04:8088/ws/v1/cluster/apps/application_1460195242962_0054 | jq .
# {
#   "app": {
#     "id": "application_1460195242962_0054",
#     "user": "dr.who",
#     "name": "IrisApp",
#     "queue": "default",
#     "state": "FINISHED",
#     "finalStatus": "SUCCEEDED",
#     "progress": 100,
#     "trackingUI": "History",
#     "trackingUrl": "http://beebox04.localdomain:8088/proxy/application_1460195242962_0054/",
#     "diagnostics": "",
#     "clusterId": 1460195242962,
#     "applicationType": "YARN",
#     "applicationTags": "",
#     "startedTime": 1460293367576,
#     "finishedTime": 1460293413568,
#     "elapsedTime": 45992,
#     "amContainerLogs": "http://beebox03.localdomain:8042/node/containerlogs/container_e29_1460195242962_0054_01_000001/dr.who",
#     "amHostHttpAddress": "beebox03.localdomain:8042",
#     "allocatedMB": -1,
#     "allocatedVCores": -1,
#     "runningContainers": -1,
#     "memorySeconds": 172346,
#     "vcoreSeconds": 112,
#     "queueUsagePercentage": 0,
#     "clusterUsagePercentage": 0,
#     "preemptedResourceMB": 0,
#     "preemptedResourceVCores": 0,
#     "numNonAMContainerPreempted": 0,
#     "numAMContainerPreempted": 0,
#     "logAggregationStatus": "SUCCEEDED"
#   }
# }
```

Get the result (partition name depends on run, find it via WebHDFS and LISTSTATUS)

```bash
curl -s -L $WEBHDFS_HOST/webhdfs/v1/tmp/iris/means/part-r-00000-a1d003bf-246b-47b5-9d61-10dede1c3981?op=OPEN | jq .
# {
#   "species": "Iris-setosa",
#   "avg(sepalLength)": 5.005999999999999,
#   "avg(sepalWidth)": 3.4180000000000006,
#   "avg(petalLength)": 1.464,
#   "avg(petalWidth)": 0.2439999999999999
# }
# {
#   "species": "Iris-versicolor",
#   "avg(sepalLength)": 5.936,
#   "avg(sepalWidth)": 2.77,
#   "avg(petalLength)": 4.26,
#   "avg(petalWidth)": 1.3260000000000003
# }
# {
#   "species": "Iris-virginica",
#   "avg(sepalLength)": 6.587999999999998,
#   "avg(sepalWidth)": 2.9739999999999998,
#   "avg(petalLength)": 5.552,
#   "avg(petalWidth)": 2.026
# }
```