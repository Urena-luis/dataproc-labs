# About

This lab showcases Spark application acceleration with Spark-RAPIDS on Dataproc on GCE - powered by Nvidia GPUs.<br> 

This lab module demystifies GPU acceleration with Spark-rapids through a minimum viable Spark application on the Kaggle Telco Customer Churn dataset.


<hr>


## 2. Provision a Dataproc on GCE cluster


### 2.1. Declare variables

Paste in Cloud Shell-
```
PROJECT_ID=`gcloud config list --format "value(core.project)" 2>/dev/null`
PROJECT_NBR=`gcloud projects describe $PROJECT_ID | grep projectNumber | cut -d':' -f2 |  tr -d "'" | xargs`

DATAPROC_CLUSTER_NAME=dpgce-cluster-static-gpu-${PROJECT_NBR}
DPGCE_LOG_BUCKET=spark-bucket-dpgce-${PROJECT_NBR}
DATA_BUCKET=data_bucket-${PROJECT_NBR}
CODE_BUCKET=code_bucket-${PROJECT_NBR}
VPC_NM=VPC=vpc-$PROJECT_NBR
SPARK_SUBNET=spark-snet
UMSA_FQN=lab-sa@$PROJECT_ID.iam.gserviceaccount.com
REGION=us-central1
ZONE=us-central1-a
NUM_GPUS=1
NUM_WORKERS=4
```

<hr>

### 2.2. Create a DPGCE cluster with GPUs

Paste in Cloud Shell-

```
gcloud dataproc clusters create $DATAPROC_CLUSTER_NAME  \
    --region $REGION \
    --zone $ZONE \
    --image-version=2.0-ubuntu18 \
    --master-machine-type=n1-standard-4 \
    --num-workers=$NUM_WORKERS \
    --worker-accelerator=type=nvidia-tesla-t4,count=$NUM_GPUS \
    --worker-machine-type=n1-standard-8 \
    --num-worker-local-ssds=1 \
    --initialization-actions=gs://goog-dataproc-initialization-actions-${REGION}/spark-rapids/spark-rapids.sh \
    --optional-components=JUPYTER,ZEPPELIN \
    --metadata gpu-driver-provider="NVIDIA",rapids-runtime="SPARK" \
    --subnet=$SPARK_SUBNET \
    --enable-component-gateway  \
    --service-account $UMSA_FQN   
```

Takes approximately <10 minutes to provision. Largely because of scripts that need to run to install drivers and such.

### 2.3. Quick pictorial walk-through of the cluster

Scroll below to appendix.

<hr>

## 3. Review the lab dataset

The dataset is the famous Kaggle Telco Customer Churn dataset - small data. Review the same.

Paste in Cloud Shell-
```
head -10 ~/dataproc-labs/4-dataproc-with-gpu/01-datasets/telco-customer-churn.csv
```

<hr>

## 4. Generate a larger dataset off of the base lab dataset

The script (generate_data.py) provided to us by Nvidia creates a larger dataset. We will use the same to generate a large dataset. <br>

### 4.1. Review the size of the base dataset

Paste in Cloud Shell-
```
PROJECT_ID=`gcloud config list --format "value(core.project)" 2>/dev/null`
PROJECT_NBR=`gcloud projects describe $PROJECT_ID | grep projectNumber | cut -d':' -f2 |  tr -d "'" | xargs`
DATA_BUCKET=data_bucket-${PROJECT_NBR}

gsutil du -s -h -a gs://$DATA_BUCKET/churn/input/telco-customer-churn.csv | cut -d' ' -f1,2
```
Its 954 KiB.

### 4.2. Review the script
Paste in Cloud Shell-
```
cd ~/dataproc-labs/5-dataproc-gce-with-gpu/00-scripts/data_generator_util
cat generate_data.py
```

### 4.3. Declare variables
Paste in Cloud Shell-
```
PROJECT_ID=`gcloud config list --format "value(core.project)" 2>/dev/null`
PROJECT_NBR=`gcloud projects describe $PROJECT_ID | grep projectNumber | cut -d':' -f2 |  tr -d "'" | xargs`

DATAPROC_CLUSTER_NAME=dpgce-cluster-static-gpu-${PROJECT_NBR}
SPHS_LOG_BUCKET=sphs-bucket-${PROJECT_NBR}
DATA_BUCKET=data_bucket-${PROJECT_NBR}
CODE_BUCKET=code_bucket-${PROJECT_NBR}
VPC_NM=VPC=vpc-$PROJECT_NBR
SPARK_SUBNET=spark-snet
PERSISTENT_HISTORY_SERVER_NM=sphs-${PROJECT_NBR}
UMSA_FQN=lab-sa@$PROJECT_ID.iam.gserviceaccount.com
REGION=us-central1
ZONE=us-central1-a
NUM_GPUS=1
NUM_WORKERS=4

LOG_SECOND=`date +%s`
LAB_LOG_ROOT_DIR="~/dataproc-labs/logs/lab-4/"
mkdir -p $LAB_LOG_ROOT_DIR
LOGFILE="$LAB_LOG_ROOT_DIR/$0.txt.$LOG_SECOND"


# This is used to define the size of the dataset that is generated
# 10000 will generate a dataset of roughly 25GB in size
SCALE=10

# Set this value to the total number of cores that you have across all
# your worker nodes. 
TOTAL_CORES=32
#
# Set this value to 1/4 the number of cores listed above. Generally,
# we have found that 4 cores per executor performs well.
NUM_EXECUTORS=8   # 1/4 the number of cores in the cluster
#
NUM_EXECUTOR_CORES=$((${TOTAL_CORES}/${NUM_EXECUTORS}))
#
# Set this to the total memory across all your worker nodes. e.g. RAM of each worker * number of worker nodes
TOTAL_MEMORY=120   # unit: GB
DRIVER_MEMORY=4    # unit: GB
#
# This takes the total memory and calculates the maximum amount of memory
# per executor
EXECUTOR_MEMORY=$(($((${TOTAL_MEMORY}-$((${DRIVER_MEMORY}*1000/1024))))/${NUM_EXECUTORS}))

# Source base data file to be bulked up
INPUT_FILE="gs://data_bucket-${PROJECT_NBR}/churn/input/telco-customer-churn.csv"
# *****************************************************************
# Output prefix is where the data that is generated will be stored.
# This path is important as it is used for the INPUT_PREFIX for
# the cpu and gpu env files
# *****************************************************************
#
OUTPUT_PREFIX="gs://data_bucket-${PROJECT_NBR}/churn/input/10scale/"
```

### 4.4. Run the script

Paste in Cloud Shell-
```
gcloud dataproc jobs submit pyspark \
--cluster $CLUSTER_NAME \
--id data-generator-$RANDOM \
gs://$CODE_BUCKET/churn/data_generator_util/generate_data.py \
--py-files=gs://$CODE_BUCKET/churn/aux_etl_code_archive.zip \
--properties="spark.executor.cores=${NUM_EXECUTOR_CORES},spark.executor.memory=${EXECUTOR_MEMORY}G,spark.driver.memory=${DRIVER_MEMORY}G" \
--configuration="spark.cores.max=$TOTAL_CORES,spark.task.cpus=1,spark.sql.files.maxPartitionBytes=2G" \
--region $LOCATION \
--project $PROJECT_ID \
-- --input-file=${INPUT_FILE} --output-prefix=${OUTPUT_PREFIX} --dup-times=${SCALE}  2>&1 >> $LOGFILE
```

### 4.5. Review the 10 scale lab dataset generated

Paste in Cloud Shell-
```
gsutil ls $OUTPUT_PREFIX
```

Lets check the size-
```
gsutil du -s -h -a ${OUTPUT_PREFIX} | cut -d' ' -f1,2
```
The author's output is 45.24 MiB

<hr>

## 6. Run an ETL job on CPUs for a baseline performance capture

### 6.1. Declare variables

Paste in Cloud Shell-
```
PROJECT_ID=`gcloud config list --format "value(core.project)" 2>/dev/null`
PROJECT_NBR=`gcloud projects describe $PROJECT_ID | grep projectNumber | cut -d':' -f2 |  tr -d "'" | xargs`

CLUSTER_NAME=dpgce-cluster-static-gpu-${PROJECT_NBR}
DPGCE_LOG_BUCKET=dpgce-cluster-static-gpu-${PROJECT_NBR}-logs
DATA_BUCKET=spark-rapids-lab-data-${PROJECT_NBR}
CODE_BUCKET=spark-rapids-lab-code-${PROJECT_NBR}
VPC_NM=VPC=dpgce-vpc-$PROJECT_NBR
SPARK_SUBNET=spark-snet
PERSISTENT_HISTORY_SERVER_NM=dpgce-sphs-${PROJECT_NBR}
UMSA_FQN=dpgce-lab-sa@$PROJECT_ID.iam.gserviceaccount.com
REGION=us-central1
ZONE=us-central1-a
NUM_GPUS=1
NUM_WORKERS=4

# Log for each execution
LOG_SECOND=`date +%s`
LAB_LOG_ROOT_DIR="~/dataproc-labs/logs/lab-4/"
mkdir -p $LAB_LOG_ROOT_DIR
LOGFILE="$LAB_LOG_ROOT_DIR/$0.txt.$LOG_SECOND"

# Set this value to the total number of cores that you have across all
# your worker nodes. e.g. 8 servers with 40 cores = 320 cores
TOTAL_CORES=32
#
# Set this value to 1/4 the number of cores listed above. Generally,
# we have found that 4 cores per executor performs well.
NUM_EXECUTORS=8  # 1/4 the number of cores in the cluster
#
NUM_EXECUTOR_CORES=$((${TOTAL_CORES}/${NUM_EXECUTORS}))
#
# Set this to the total memory across all your worker nodes. e.g. RAM of each worker * number of worker nodes
TOTAL_MEMORY=120   # unit: GB
DRIVER_MEMORY=4    # unit: GB
#
# This takes the total memory and calculates the maximum amount of memory
# per executor
EXECUTOR_MEMORY=$(($((${TOTAL_MEMORY}-$((${DRIVER_MEMORY}*1000/1024))))/${NUM_EXECUTORS}))

# Input prefix designates where the data to be processed is located
INPUT_PREFIX="gs://spark-rapids-lab-data-$PROJECT_NBR/churn/input/10scale/"
#
# Output prefix is where results from the queries are stored
OUTPUT_PREFIX="gs://spark-rapids-lab-data-$PROJECT_NBR/churn/output/cpu-based-analytics"
```

### 6.2. Run a Spark analytics application on CPUs for a baseline

Paste in Cloud Shell-
```
SPARK_PROPERTIES="spark.executor.cores=${NUM_EXECUTOR_CORES},spark.executor.memory=${EXECUTOR_MEMORY}G,spark.driver.memory=${DRIVER_MEMORY}G,spark.cores.max=$TOTAL_CORES,spark.task.cpus=1,spark.sql.files.maxPartitionBytes=1G,spark.sql.adaptive.enabled=True,spark.sql.autoBroadcastJoinThreshold=-1,spark.rapids.sql.enabled=false "

gcloud dataproc jobs submit pyspark \
gs://$CODE_BUCKET/churn/main_analytics_app.py \
--py-files=gs://$CODE_BUCKET/churn/aux_etl_code_archive.zip \
--cluster $CLUSTER_NAME \
--region $REGION \
--id cpu-etl-baseline-$RANDOM \
--properties=${SPARK_PROPERTIES} \
--project $PROJECT_ID \
-- --coalesce-output=8 --input-prefix=${INPUT_PREFIX} --output-prefix=${OUTPUT_PREFIX}   2>&1 >> $LOGFILE
```

### 6.3. Review the results
Paste in Cloud Shell-
```
gsutil ls -r $OUTPUT_PREFIX
```

### 6.4. Note the execution time

The author's application took ~ 32 minutes to complete across multiple runs.

<hr>

## 7. Run the Nvidia Qualification Tool to see if the Spark application qualifies for GPUs

### 7.1. Find the Public IP address of your Cloud Shell terminal

```
MY_IP_ADDRESS=`curl -s checkip.dyndns.org | sed -e 's/.*Current IP Address: //' -e 's/<.*$//'`
echo $MY_IP_ADDRESS
```

### 7.2. Add an ingress firewall rule to allow yourself SSH access to the cluster
First and foremost, you need to allow yourself ingress to SSH into the cluster. If you use Cloud Shell, the IP address varies with each session. Use the commad below to allow ingress to your IP address.
```
PROJECT_ID=`gcloud config list --format "value(core.project)" 2>/dev/null`
PROJECT_NBR=`gcloud projects describe $PROJECT_ID | grep projectNumber | cut -d':' -f2 |  tr -d "'" | xargs`
VPC_NM=dpgce-vpc-$PROJECT_NBR
REGION=us-central1
ZONE=$REGION-a
CLUSTER_NAME=dpgce-cluster-static-gpu-$PROJECT_NBR
MY_FIREWALL_RULE="allow-me-to-ingress-into-vpc"

gcloud compute firewall-rules delete $MY_FIREWALL_RULE

gcloud compute --project=$PROJECT_ID firewall-rules create $MY_FIREWALL_RULE --direction=INGRESS --priority=1000 --network=$VPC_NM --action=ALLOW --rules=all --source-ranges="$MY_IP_ADDRESS/32"
```

### 7.3. Install RAPIDS user tools in Cloud Shell

paste in Cloud Shell-
```
python -m venv .venv
source .venv/bin/activate

pip install spark-rapids-user-tools
```

Check to see if you can run the Nvidia qualification tool, immediately after-
```
spark_rapids_dataproc qualification --help
```


### 7.4 Run the Qualification tool to find workloads that can benefit from GPU based acceleration

You can run this only after you run a few Spark applications. The tool will review the logs and provide recommendations based on YARN application IDs-
```
# Variables
PROJECT_ID=`gcloud config list --format "value(core.project)" 2>/dev/null`
PROJECT_NBR=`gcloud projects describe $PROJECT_ID | grep projectNumber | cut -d':' -f2 |  tr -d "'" | xargs`
REGION=us-central1
ZONE=$REGION-a
CLUSTER_NAME=dpgce-cluster-static-gpu-$PROJECT_NBR


# Run the tool to check previous Spark applications for qualification-
spark_rapids_dataproc qualification --cluster $CLUSTER_NAME --region $REGION
```

Here are the author's results, that correctly call out the two Spark applications run without GPU acceleration, while omiiting the ones that used GPUs and the speed up is accurate as well-
```
2023-05-11 16:44:50,560 INFO qualification: The original CPU cluster is the same as the submission cluster on which the tool runs. To update the configuration of the CPU cluster, make sure to pass the properties file to the CLI arguments.
2023-05-11 16:44:50,561 INFO qualification: Estimating the GPU cluster based on the submission cluster on which the RAPIDS tool is running [dpgce-cluster-static-gpu-420530778089]. To update the configuration of the GPU cluster, make sure to pass the properties file to the CLI arguments.
2023-05-11 16:44:50,845 INFO qualification: Preparing remote work env
2023-05-11 16:44:52,560 INFO qualification: Upload dependencies to remote cluster
2023-05-11 16:44:54,527 INFO qualification: Executing the tool
2023-05-11 16:44:54,528 INFO qualification: Running the tool as a spark job on dataproc
2023-05-11 16:45:32,875 INFO qualification: Downloading the tool output
2023-05-11 16:45:35,479 INFO qualification: Processing tool output
2023-05-11 16:45:35,503 INFO qualification: Downloading the price catalog from URL https://cloudpricingcalculator.appspot.com/static/data/pricelist.json
2023-05-11 16:45:35,561 INFO qualification: Building cost model based on:
Worker Properties
--------------------  -------------
Workers               4
Worker Machine Type   n1-standard-8
Region                us-central1
Zone                  us-central1-a
GPU device            T4
GPU per worker nodes  2
2023-05-11 16:45:35,570 INFO qualification: Generating GPU Estimated Speedup and Savings as ./wrapper-output/rapids_user_tools_qualification/qual-tool-output/rapids_4_dataproc_qualification_output.csv
Qualification tool output is saved to local disk /home/admin_/wrapper-output/rapids_user_tools_qualification/qual-tool-output/rapids_4_spark_qualification_output
        rapids_4_spark_qualification_output/
                ├── rapids_4_spark_qualification_output.csv
                ├── rapids_4_spark_qualification_output_execs.csv
                ├── rapids_4_spark_qualification_output.log
                └── ui/
                ├── rapids_4_spark_qualification_output_stages.csv
- To learn more about the output details, visit https://nvidia.github.io/spark-rapids/docs/spark-qualification-tool.html#understanding-the-qualification-tool-output
Full savings and speedups CSV report: /home/admin_/wrapper-output/rapids_user_tools_qualification/qual-tool-output/rapids_4_dataproc_qualification_output.csv
+----+--------------------------------+-----------------+----------------------+-----------------+-----------------+---------------+-----------------+
|    | App ID                         | App Name        | Recommendation       |   Estimated GPU |   Estimated GPU |           App |   Estimated GPU |
|    |                                |                 |                      |         Speedup |     Duration(s) |   Duration(s) |      Savings(%) |
|----+--------------------------------+-----------------+----------------------+-----------------+-----------------+---------------+-----------------|
|  0 | application_1683730151313_0009 | churn_utils.etl | Strongly Recommended |            4.18 |          472.65 |       1977.57 |           40.59 |
|  1 | application_1683730151313_0008 | churn_utils.etl | Strongly Recommended |            4.17 |          458.58 |       1914.39 |           40.45 |
+----+--------------------------------+-----------------+----------------------+-----------------+-----------------+---------------+-----------------+
Report Summary:
------------------------------  ------
Total applications                   2
RAPIDS candidates                    2
Overall estimated speedup         4.18
Overall estimated cost savings  40.52%
------------------------------  ------
To launch a GPU-accelerated cluster with RAPIDS Accelerator for Apache Spark, add the following to your cluster creation script:
        --initialization-actions=gs://goog-dataproc-initialization-actions-us-central1/spark-rapids/spark-rapids.sh \ 
        --worker-accelerator type=nvidia-tesla-t4,count=2

```

<hr>

## 8. Run the same ETL job on GPUs 

### 8.1. Declare variables

Paste in Cloud Shell-
```
PROJECT_ID=`gcloud config list --format "value(core.project)" 2>/dev/null`
PROJECT_NBR=`gcloud projects describe $PROJECT_ID | grep projectNumber | cut -d':' -f2 |  tr -d "'" | xargs`

CLUSTER_NAME=dpgce-cluster-static-gpu-${PROJECT_NBR}
DPGCE_LOG_BUCKET=dpgce-cluster-static-gpu-${PROJECT_NBR}-logs
DATA_BUCKET=spark-rapids-lab-data-${PROJECT_NBR}
CODE_BUCKET=spark-rapids-lab-code-${PROJECT_NBR}
VPC_NM=VPC=dpgce-vpc-$PROJECT_NBR
SPARK_SUBNET=spark-snet
PERSISTENT_HISTORY_SERVER_NM=dpgce-sphs-${PROJECT_NBR}
UMSA_FQN=dpgce-lab-sa@$PROJECT_ID.iam.gserviceaccount.com
REGION=us-central1
ZONE=us-central1-a
NUM_GPUS=1
NUM_WORKERS=4

# Log for each execution
LOG_SECOND=`date +%s`
LAB_LOG_ROOT_DIR="~/dataproc-labs/logs/lab-4/"
mkdir -p $LAB_LOG_ROOT_DIR
LOGFILE="$LAB_LOG_ROOT_DIR/$0.txt.$LOG_SECOND"

# Set this value to the total number of cores that you have across all
# your worker nodes. e.g. 8 servers with 40 cores = 320 cores
TOTAL_CORES=32

# Set this value to the number of GPUs that you have within your cluster. If
# each server has 2 GPUs count that as 2
NUM_EXECUTORS=4   # change to fit how many GPUs you have

# This setting needs to be a decimal equivalent to the ratio of cores to
# executors. In our example we have 40 cores and 8 executors. So, this
# would be 1/5, hench the 0.1 value.

RESOURCE_GPU_AMT="0.125"

#
NUM_EXECUTOR_CORES=$((${TOTAL_CORES}/${NUM_EXECUTORS}))
#
# Set this to the total memory across all your worker nodes. e.g. RAM of each worker * number of worker nodes
TOTAL_MEMORY=120   # unit: GB
DRIVER_MEMORY=4    # unit: GB
#
# This takes the total memory and calculates the maximum amount of memory
# per executor
EXECUTOR_MEMORY=$(($((${TOTAL_MEMORY}-$((${DRIVER_MEMORY}*1000/1024))))/${NUM_EXECUTORS}))


# Input prefix designates where the data to be processed is located
INPUT_PREFIX="gs://spark-rapids-lab-data-$PROJECT_NBR/churn/input/10scale/"

# Output prefix is where results from the queries are stored
OUTPUT_PREFIX="gs://spark-rapids-lab-data-$PROJECT_NBR/churn/output/gpu-based-analytics"
```

### 8.2. Run the Spark ETL analytics application on GPUs

Paste in Cloud Shell-
```
SPARK_PROPERTIES="spark.executor.cores=${NUM_EXECUTOR_CORES},spark.executor.memory=${EXECUTOR_MEMORY}G,spark.driver.memory=${DRIVER_MEMORY}G,spark.cores.max=$TOTAL_CORES,spark.task.cpus=1,spark.sql.files.maxPartitionBytes=1G,spark.sql.adaptive.enabled=True,spark.sql.autoBroadcastJoinThreshold=-1,spark.rapids.sql.enabled=True,spark.rapids.sql.decimalType.enabled=True,spark.task.resource.gpu.amount=$RESOURCE_GPU_AMT,spark.plugins=com.nvidia.spark.SQLPlugin,spark.rapids.memory.pinnedPool.size=2G,spark.rapids.sql.concurrentGpuTasks=2,spark.executor.resource.gpu.amount=1,spark.rapids.sql.variableFloatAgg.enabled=True,spark.rapids.sql.explain=NOT_ON_GPU "

gcloud dataproc jobs submit pyspark \
gs://$CODE_BUCKET/churn/main_analytics_app.py \
--py-files=gs://$CODE_BUCKET/churn/aux_etl_code_archive.zip \
--cluster $CLUSTER_NAME \
--region $REGION \
--id gpu-etl-baseline-$RANDOM \
--properties=${SPARK_PROPERTIES} \
--project $PROJECT_ID \
-- --coalesce-output=8 --input-prefix=${INPUT_PREFIX} --output-prefix=${OUTPUT_PREFIX}   2>&1 >> $LOGFILE
```

### 8.3. Review the results

Paste in Cloud Shell-
```
gsutil ls -r $OUTPUT_PREFIX
gsutil du -s -h -a $OUTPUT_PREFIX
```

### 8.4. Note the execution time

The author's application took ~8 minutes to complete across multiple tests.

<hr>

## 9. Tuning GPU based applications - profiling and recommednations from Nvidia

### 9.1. Run the Nvidia profiler on the Spark on GPU applications run already
This unit uses Nvidia's tooling to tune GPU based Spark applications and needs to be run after your initial attempts of runnng GPU based Spark applications.

<br>
Run the below in Cloud Shell-

```
PROJECT_ID=`gcloud config list --format "value(core.project)" 2>/dev/null`
PROJECT_NBR=`gcloud projects describe $PROJECT_ID | grep projectNumber | cut -d':' -f2 |  tr -d "'" | xargs`
REGION=us-central1
CLUSTER_NAME=dpgce-cluster-static-gpu-$PROJECT_NBR

spark_rapids_dataproc profiling --cluster $CLUSTER_NAME --region $REGION
```

Author's sample output (scroll to the right for full details)-
```
2023-05-11 16:44:54,528 INFO qualification: Running the tool as a spark job on dataproc
2023-05-11 16:45:32,875 INFO qualification: Downloading the tool output
2023-05-11 16:45:35,479 INFO qualification: Processing tool output
2023-05-11 16:45:35,503 INFO qualification: Downloading the price catalog from URL https://cloudpricingcalculator.appspot.com/static/data/pricelist.json
2023-05-11 16:45:35,561 INFO qualification: Building cost model based on:
Worker Properties
--------------------  -------------
Region                us-central1
Zone                  us-central1-a
GPU device            T4
GPU per worker nodes  2
```

```
Scroll to the right for explanation-
+--------------------------------+-----------------+------------------------------------------------------------------------------------+----------------------------------------------------------------------------------------------------------------------------------+
| application_1683730151313_0007 | churn_utils.etl | --conf spark.executor.cores=8                                                      | - 'spark.executor.memoryOverhead' must be set if using 'spark.rapids.memory.pinnedPool.size                                      |
|                                |                 | --conf spark.executor.instances=4                                                  | - 'spark.executor.memoryOverhead' was not set.                                                                                   |
|                                |                 | --conf spark.executor.memory=16384m                                                | - 'spark.rapids.shuffle.multiThreaded.reader.threads' was not set.                                                               |
|                                |                 | --conf spark.executor.memoryOverhead=5734m                                         | - 'spark.rapids.shuffle.multiThreaded.writer.threads' was not set.                                                               |
|                                |                 | --conf spark.rapids.memory.pinnedPool.size=4096m                                   | - 'spark.shuffle.manager' was not set.                                                                                           |
|                                |                 | --conf spark.rapids.shuffle.multiThreaded.reader.threads=8                         | - 'spark.sql.shuffle.partitions' was not set.                                                                                    |
|                                |                 | --conf spark.rapids.shuffle.multiThreaded.writer.threads=8                         | - The RAPIDS Shuffle Manager requires the spark.driver.extraClassPath and                                                        |
|                                |                 | --conf spark.shuffle.manager=com.nvidia.spark.rapids.spark313.RapidsShuffleManager | spark.executor.extraClassPath settings to include the path to the Spark RAPIDS                                                   |
|                                |                 | --conf spark.sql.files.maxPartitionBytes=4096m                                     | plugin jar.  If the Spark RAPIDS jar is being bundled with your Spark distribution,                                              |
|                                |                 | --conf spark.sql.shuffle.partitions=200                                            | this step is not needed.                                                                                                         |
|                                |                 | --conf spark.task.resource.gpu.amount=0.125                                        |                                                                                                                                  |
+--------------------------------+-----------------+--------------------------------------------------------------------------
```

### 9.2. Tune the Spark application with the recommedations from the profiler

Lets tune our Spark application parameters based on the recommendation above and run the same Spark application as follows-
```
PROJECT_ID=`gcloud config list --format "value(core.project)" 2>/dev/null`
PROJECT_NBR=`gcloud projects describe $PROJECT_ID | grep projectNumber | cut -d':' -f2 |  tr -d "'" | xargs`

CLUSTER_NAME=dpgce-cluster-static-gpu-${PROJECT_NBR}
DPGCE_LOG_BUCKET=dpgce-cluster-static-gpu-${PROJECT_NBR}-logs
DATA_BUCKET=spark-rapids-lab-data-${PROJECT_NBR}
CODE_BUCKET=spark-rapids-lab-code-${PROJECT_NBR}
VPC_NM=VPC=dpgce-vpc-$PROJECT_NBR
SPARK_SUBNET=spark-snet
PERSISTENT_HISTORY_SERVER_NM=dpgce-sphs-${PROJECT_NBR}
UMSA_FQN=dpgce-lab-sa@$PROJECT_ID.iam.gserviceaccount.com
REGION=us-central1
ZONE=us-central1-a
NUM_GPUS=1
NUM_WORKERS=4

# Log for each execution
LOG_SECOND=`date +%s`
LAB_LOG_ROOT_DIR="~/dataproc-labs/logs/lab-4/"
mkdir -p $LAB_LOG_ROOT_DIR
LOGFILE="$LAB_LOG_ROOT_DIR/$0.txt.$LOG_SECOND"

# Set this value to the total number of cores that you have across all
# your worker nodes. e.g. 8 servers with 40 cores = 320 cores
TOTAL_CORES=32

# Set this value to the number of GPUs that you have within your cluster. If
# each server has 2 GPUs count that as 2
NUM_EXECUTORS=4   # change to fit how many GPUs you have

# This setting needs to be a decimal equivalent to the ratio of cores to
# executors. In our example we have 40 cores and 8 executors. So, this
# would be 1/5, hench the 0.1 value.

RESOURCE_GPU_AMT="0.125"

#
NUM_EXECUTOR_CORES=$((${TOTAL_CORES}/${NUM_EXECUTORS}))
echo "NUM_EXECUTOR_CORES=$NUM_EXECUTOR_CORES"
#
# Set this to the total memory across all your worker nodes. e.g. RAM of each worker * number of worker nodes
TOTAL_MEMORY=120   # unit: GB
DRIVER_MEMORY=4    # unit: GB
#
# This takes the total memory and calculates the maximum amount of memory
# per executor
EXECUTOR_MEMORY=$(($((${TOTAL_MEMORY}-$((${DRIVER_MEMORY}*1000/1024))))/${NUM_EXECUTORS}))
echo "EXECUTOR_MEMORY=$EXECUTOR_MEMORY"

# Input prefix designates where the data to be processed is located
INPUT_PREFIX="gs://spark-rapids-lab-data-$PROJECT_NBR/churn/input/10scale/"

# Output prefix is where results from the queries are stored
OUTPUT_PREFIX="gs://spark-rapids-lab-data-$PROJECT_NBR/churn/output/gpu-based-analytics"

SPARK_PROPERTIES="spark.executor.cores=${NUM_EXECUTOR_CORES},spark.driver.memory=${DRIVER_MEMORY}G,spark.cores.max=$TOTAL_CORES,spark.task.cpus=1,spark.sql.files.maxPartitionBytes=1G,spark.sql.adaptive.enabled=True,spark.sql.autoBroadcastJoinThreshold=-1,spark.rapids.sql.enabled=True,spark.rapids.sql.decimalType.enabled=True,spark.task.resource.gpu.amount=$RESOURCE_GPU_AMT,spark.plugins=com.nvidia.spark.SQLPlugin,spark.rapids.sql.concurrentGpuTasks=2,spark.executor.resource.gpu.amount=1,spark.rapids.sql.variableFloatAgg.enabled=True,spark.rapids.sql.explain=NOT_ON_GPU,spark.executor.instances=4,spark.executor.memory=16384m,spark.rapids.memory.pinnedPool.size=4096m,spark.executor.memoryOverhead=5734m,spark.rapids.shuffle.multiThreaded.reader.threads=8,spark.rapids.shuffle.multiThreaded.writer.threads=8,spark.shuffle.manager=com.nvidia.spark.rapids.spark313.RapidsShuffleManager,spark.sql.files.maxPartitionBytes=4096m,spark.sql.shuffle.partitions=200"

gcloud dataproc jobs submit pyspark \
gs://$CODE_BUCKET/churn/main_analytics_app.py \
--py-files=gs://$CODE_BUCKET/churn/aux_etl_code_archive.zip \
--cluster $CLUSTER_NAME \
--region $REGION \
--id gpu-etl-tuned-$RANDOM \
--properties=${SPARK_PROPERTIES} \
--project $PROJECT_ID \
-- --coalesce-output=8 --input-prefix=${INPUT_PREFIX} --output-prefix=${OUTPUT_PREFIX}   2>&1 >> $LOGFILE
```

### 9.3. Note the execution time

The author's application took ~5 minutes to complete across multiple tests.

## 10.0. Summary

We ran the same Spark ETL application from Nvidia on the same cluster and compared performance across CPUs and GPUs. The Spark applications are in no way perfectly tuned. 
|About|Details|
| :-- | :-- |
| Dataproc | Image version 2.0.63-ubuntu18 | 
| Apache Spark | 3.1.3 | 
| Workload | ETL with PySpark on Dataproc on GCE with Spark 3.1.3 | 
| Data size | 45 MB | 
| Storage system | Google Cloud Storage | 
| Processing complexity | Medium |

|Infrastructure| Specification|
| :-- | :-- |
| Master Node SKU | n1-standard-4  (4 vCPUs, 15 GB RAM)| 
| Worker Node SKU | n1-standard-8 (8 vCPUs, 30 GB RAM) | 
| Worker Node Accelerator | nvidia-tesla-t4 with 1 gpu |
| Worker Node Count | 4 |

The author's results-
|Infrastructure base| Specifics| Average execution time|
| :-- | :-- | :-- |
| CPU-based | Baseline performance | 32 minutes |
| GPU-based | Baseline performance| 8 minutes |
| GPU-based | Tuned with Nvidia profiler recommendations | ~5 minutes |

<hr>
