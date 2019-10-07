+++
draft = false
date = 2019-10-07T08:59:44-07:00
title = "Spark Streaming on Oracle Cloud"
+++

I've been enjoying the [free trial](https://www.oracle.com/cloud/free/) of Oracle Cloud Infrastructure and seeing what it can do. I've used their products heavily in the past and love the database so figured the cloud environment would be fun. One area that I was looking at was their [Big Data Compute](https://www.oracle.com/big-data/) platform which is basically a Spark / Hadoop cluster that you spin up and attach to your Object Store. They also have their [Streaming](https://www.oracle.com/big-data/streaming/) which is effectively a simple pub/sub queue service.

One of the projects I wanted to investigate was a simple data processing platform that received unstructured data in a queue then use Spark Streaming to process that data and push it out to the Object Store / Data Lake and also an Autonomous Data Warehouse. This would look something like this:
{{<mermaid>}}
graph TD
    Client-->Queue
    Queue-->Spark
    Spark-->DataLake[Data Lake]
    Spark-->DataWarehouse[Data Warehouse]
{{< /mermaid >}}

Or in OCI Terms:
{{<mermaid>}}
graph TD
    Client-->Queue[Analytics Streaming]
    Queue-->Spark[Big Data - Compute Edition]
    Spark-->ObjectStore[Object Store]
    Spark-->DataWarehouse[Autonomous Data Warehouse]
{{< /mermaid >}}

## Streaming
The streaming product is easy enough to spin up but interactions with it were best done through their [SDK](https://docs.cloud.oracle.com/iaas/Content/API/Concepts/sdks.htm) which is available for Java, Python, Ruby, and Go. For my purposes, I'll be using both Python and Java. For working with the queue, I modified their example code and put together a simple Python utility called [stream.py](https://github.com/tdl-jturner/oci-streaming-spark-receiver/blob/master/stream.py). This does require you first install and configure the Python SDK as described in the [install docs](https://docs.cloud.oracle.com/iaas/Content/API/SDKDocs/cliinstall.htm)). Here is the usage of it:
{{< highlight bash >}}
usage: stream.py [-h] [--stream STREAM] [--partitions PARTITIONS]
                 [--compartment COMPARTMENT] [--consume] [--group GROUP]
                 [--instance INSTANCE] [--limit LIMIT] [--produce MESSAGE]
                 [--delete]

Interact with OCI Stream

optional arguments:
  -h, --help            show this help message and exit
  --stream STREAM       name of stream
  --partitions PARTITIONS
                        number of partitions
  --compartment COMPARTMENT
                        UUID of compartment
  --consume             Consume the stream and print messages
  --group GROUP         Name of Consumer Group
  --instance INSTANCE   Name of Consumer Instance
  --limit LIMIT         Limit number of consumed messages
  --produce MESSAGE     Produce a message
  --delete              Delete stream
{{< / highlight >}}
Now we have a very simple command line widget for producing and consuming our stream. If the named stream does not exist, it will create it. I also used this to infinitely generate data with:
{{< highlight bash >}}
while true; do ./stream.py --produce "DateLoop:`date`";sleep 1;done
{{< / highlight >}}
You can consume and view this in another terminal with:
{{< highlight bash >}}
./stream.py --consume
{{< / highlight >}}

## Big Data - Compute Edition
This is a platform offering of OCI and thus took a bit more to spin up but wasn't too bad. Once you fill out the forms and wait for it to provision you end up with a running cluster including a Zeppelin notebook that you can run Spark, PySpark, or MapReduce jobs on. Instead of reiterating all the steps, I'll point you to the [Oracle By Example: Getting Started with Oracle Big Data Cloud Service - Compute Edition](https://www.oracle.com/webfolder/technetwork/tutorials/obe/cloud/bdcsce/gettingstartedwithbdcsce/gettingstartedwithbdcsce.html).

The platform came prebaked with Object store functionality which was great but I could not get it running with the streaming. The only reference I could find was a reference by [Igor Souza on Twitter](https://twitter.com/Igfasouza/status/1165337034669600768) mentioning starting a custom receiver to do this task.

So I figured, I'd take this chance to learn more about [Custom Receivers](https://spark.apache.org/docs/latest/streaming-custom-receivers.html) and build one. It is now available on my github at [oci-streaming-spark-receiver](https://github.com/tdl-jturner/oci-streaming-spark-receiver). As noted in the readme, it should be installed to the BDC node in order to be able to use [InstancePrincipalsAuthentication](https://docs.cloud.oracle.com/iaas/Content/Identity/Tasks/callingservicesfrominstances.htm) which basically lets it authenticate as the instance instead of providing credentials and keys.

## Sample Job
Now with that in place you can do a streaming job like the following which batches the data every 10 seconds and prints it out. You could easily change that to send the data out to your data lake, autonomous data warehouse, etc.
{{< highlight scala >}}
Logger.getRootLogger.setLevel(Level.INFO)
val conf = new SparkConf().setAppName("Streaming Example").setMaster("local[2]");
// Create a Scala Spark Context.
val sc = new SparkContext(conf)
val ssc = new StreamingContext(sc, Seconds(10))
val customReceiverStream = ssc.receiverStream(
  new OCIStreamReceiver(
    OCIStreamReceiver.AuthProvider.INSTANCE_PRINCIPALS,
    "<STREAM_ID>",
    "<STREAM_ENDPOINT>",
    "<GROUP_NAME>",
    "<INSTANCE_NAME>",
    100))
customReceiverStream.print()
ssc.start()
ssc.awaitTermination()
{{< / highlight >}}

## Moving Forward
My trial did expire so I didn't get the whole flow completed. However, the pieces are there and adding in the last few bits wouldn't take much more than modifying the sample job.
