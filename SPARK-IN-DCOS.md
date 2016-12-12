# Running Spark in DC/OS with dynamic allocation (including Apache Zeppelin)

Dynamic allocation in Spark enables elastic scaling of executor nodes depending on a workload. This is extremely useful
when amount of data that has to be processed is not known upfront (Spark Streaming) or has significant variation
from run to run (hourly batch jobs).

Another relevant example is Spark interpreter in Apache Zeppelin. Spark Interpreter can be up and running for days and weeks, but used only
couple of times a day to run exploratory Spark SQL queries. Without dynamic allocation it will unnecessary waste of resources.

## External Shuffle Service

During large shuffles Spark spits out bunch of temporary shuffle blocks to local directory configured by `spark.local.dir` property.
When executor stops, external shuffle service keeps track of these files locations and makes sure that they are
not removed while the driver is still alive, so that they can be retrieved later by other executors.

By default Spark writes this temporary blocks to `/tmp`, which works fine when Spark is running on bare metal. However when it's running
with dockerized executors (default way of running in DC/OS) default location is inside the running Docker container and all the data
goes away when executor stops.

Solution is to map underlying host volume into Spark Executors and Spark Mesos Shuffle Service, and update `spark.local.dir` configuration.

```
"volumes": [
  {
    "containerPath": "/tmp/spark",
    "hostPath": "/var/lib/tmp/spark",
    "mode": "RW"
  }
]
```

In this example I'm using `/var/lib` because by default AWS CloudFormation for DC/OS mounts ephemeral store to this directory.

### Deploying External Shuffle Service
 
Easiest way to deploy Spark Mesos Shuffle service in DC/OS environment is to use Marathon and scale number of instances to match the number
of nodes in DC/OS cluster. Unique host constraint makes sure that shuffle service in spread across all of the nodes.

See [mesos-shuffle-service](https://github.com/NBCUAS/dcos-spark-shuffle-service/blob/master/marathon/mesos-shuffle-service.json)
for full Marathon configuration file.

```
> cd ./marathon
> dcos marathon app add < mesos-shuffle-service.json
```

## Enabling Dynamic Allocation In Spark

It's required to enable [dynamic allocation](http://spark.apache.org/docs/latest/configuration.html#dynamic-allocation), map external
volumes into dockerized executors, and update "scratch" space location

| Config                                | Value |
| ------------------------------------- |-------|
| spark.shuffle.service.enabled         | true  |
| spark.dynamicAllocation.enabled       | true  |
| spark.dynamicAllocation.maxExecutors  | 10    |
| spark.local.dir	                    | /tmp/spark                   |
| spark.mesos.executor.docker.volumes	| /var/lib/tmp/spark:/tmp/spark:rw   |

## Enabling Dynamic Allocation in Apache Zeppelin

To enable dynamic allocation in Apache Zeppelin it's required to pass the same configuration parameters to Spark Interpreter and restart it.
