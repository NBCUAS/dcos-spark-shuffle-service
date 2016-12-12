# DC/OS Spark Shuffle Service

Enabling Spark Dynamic Allocation in DC/OS

## WARNING: External Volume

Docker volume used in this Marathon config is specific to DC/OS deployed AWS with CloudFormation. If you are using
other cloud provider or bare metal make sure to change this configuration.

* EC2 ephemeral storage is mounted as `/var/lib` in DC/OS CloudFormation

## Deploying into DC/OS

In order to use dynamic allocation in Spark jobs, it's required  to run Spark Mesos shuffle service on each of
mesos slave nodes. Easiest way to do this is through Marathon.

```
> cd ./marathon
> dcos marathon app add < mesos-shuffle-service.json
```

After it's deployed, don't forget to scale it properly to the total number of nodes in DC/OS cluster.

## Running Spark with enabled dynamic allocation

Additional Spark options to enable dynamic allocation and properly work with external shuffle service.

| Config                                | Value |
| ------------------------------------- |-------|
| spark.shuffle.service.enabled         | true  |
| spark.dynamicAllocation.enabled       | true  |
| spark.dynamicAllocation.maxExecutors  | 10    |
| spark.local.dir	                    | /tmp/spark                    |
| spark.mesos.executor.docker.volumes	| /var/lib/tmp/spark:/tmp/spark:rw   |
