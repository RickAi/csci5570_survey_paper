## Introduction

Big data is a popular term in the last decades, with 3V characteristic: Volume, Velocity and Variety, the traditional stand-alone machine environment may cost months or years to finish a simple big data job. The research and industry have post and develop a complete solution for this, from data collection, storage and processing, the developing and evolution are happening with the development of the real-world requirements. 

<img src="https://raw.githubusercontent.com/RickAi/csci5570_survey_paper/master/images/intro.jpg" style="zoom:80%" />

Especially, in the big data analytics area, with the Google MapReduce purposed, which has had a far-ranging impact on the distributed computing industry, it is built on the simple concept of mapping and reducing big data, the developer and easily overcome the massive data processing difficult. The true value of MapReduce lies with its ability to run these processes in parallel on commodity computers while balancing disk, CPU and I/O. However, the MapReduce have several fatal defects, which makes the system cannot run efficiently on particular jobs. 

The first shortcoming is the intermediate data between different MapReduce job was saved into the HDFS persistent storage system, every job has to reload the data from storage again, which will cause significantly latency on data processing. The HaLoop have purposed a caching and indexing between the reducer and next mapper, which benefit a lot in the iterative processing task. The SparkRDD have purposed an In-Memory solution for data processing, with lineage mechanism, it can built lineage to rebuild the lost data after fault occurs. The second shortcoming of MapReduce is the limitation of its design, every processing task will have to been defined as MapReduce jobs, the relationships between different MapReduce job will be extremely complicated, which will directly increase the latency. The Dryad and Flume have developed a more flexible processing system based on DAG, which can significantly optimize multi-stage tasks processing efficiency. The MapReduce is also unable to process the real-time dataflow, the Spark Streaming and Storm have made improvements and enable fast and reliable processing on the massive of streaming data. For the typically graph problem such as the shortest path, PageRank and Minimum cutting, the graph processing system like GraphLab and Pregel are more suitable. 

In this report, I will conclude the state-of-the-art works of distributed analytics systems, from Hadoop MapReduce, HaLoop, Spark to Naiad and Husky. I will also conclude the reason why those systems were developed, and the main idea behind them. A comparative analysis will be purposed, illustrate the advantages and disadvantages of these works. As the branch of the distributed processing systems, the GraphLab and Strom will also be discussed. At last, I will have a guess about the future development of the distributed data analytics systems.

## Basics

### CAP

The CAP theorem is a tool used to makes system designers aware of the trade-offs while designing networked shared-data systems. CAP has influenced the design of many distributed data systems, it states that networked shraed-data systems can only guarantee support two of the following three properties:

* `Consistency`: Every node in the distributed cluster returns the same,  most recent and successful write, every client having the same view of the data.
* `Availability`: Every non-failing node return a response for all read and write requests in a reasonable amount of time.
* `Partition Tolerant`: The system will continue work in spite of network partitions.

<img src="https://raw.githubusercontent.com/RickAi/csci5570_survey_paper/master/images/CAP.png" style="zoom:50%" />

The CAP theorem categorizes systems into three categories:

* AP: Make sure the system are available and fault tolerance, but no guarantee of the consistency. For example, the user profile in the chat application.
* CP: The availability was sacrificed in case of network partition, application like bank ATM will directly stop the service when error is occur.
* CA: System will be consistent and available in the absence of network partition, in the real world, most of the system will not use this as fault tolerance is the basic of the distributed system.

### Scalability & Availability

`Scalability` is one of the most important design goals for developers of distributed systems. A system is scalable if it remains effective as the number of users and resources increase, the main challenges include controlling resources costs, performance loss and preventing resources from running out.

`Availability` means the distributed should be continuously available, eery request received by non-failing node in the system must result in a response. High availability can be guaranteed with data replication.

### Throughtput & Performance

The key questions between high throughtput and high performance are the granularity and degree of parallelism.

A `fine-grained` system running with independent small bits, the information was exchanged and synchronised often. Which need a smaller number of more expensive processors expensively interconnected, that enables rapid synchronisation between the bits processed in parallel.

A `corarse-grained` system running with large chunks that can be processed independently, the system use a large number of inexpensive processsors, inexpensively interconnected, which can maximizes the number of parts processed per minute.

### Type of Workloads

## The evolution of distributed analytics systems

### Hadoop MapReduce

MapReduce is the heart of `Apache Hadoop`, it is a programming paradigm that enables massive scalability across hundreds or thousands of servers in a Hadoop cluster. The term `MapReduce` actually refers to two separate and distinct tasks that Hadoop programs perform. The first is the map job, which takes a set of data and converts it into another set of data, where individual elements are broken down into key/value pairs.

Although Hadoop MapReduce is a powerful tool of big data, there are various limitations will be discussed:

1. `No Caching`: MapReduce cannot cache the intermediate data in memory for a further requirement, which will diminishes the performance such as iterative tasks (These tasks need each output of the previous task be the input of the next stage). HaLoop and SparkRDD have make improvements on this, as them will accesses data from RAM instead of disk, which dramatically improves the performance of iterative algorithms that access the same dataset repeatedly.
2. `Slow Processing Speed`: When MapReduce process large datasets with different tasks, it will requires a lot of time to perform map and reduce functions, thereby increasing latency. This can be sloved by Dryad and FlumeJava based on the Directed Acyclic Graph (DAG), which use a graph holds the track of operations. DAG will converts logical execution plan to a physical execution plan, which helps in minimize the data shuffling all around and reduce the duration of computations with less data volume, eventuallly increase the efficiency of the process with time.
3. `No Real-time Data Processing`: Hadoop MapReduce is designed for batch processing, which means it take a huge amount of data in input, process it and produce the output. Altough batch processing is very efficient for processing a high volume of data, but the output can be delayed significantly. Which will cause the MapReduce is not suitable for Real-time data processing. Naiad purposed a timely dataflow computational model, which suppport continuous input and output data. It emphasizes on the velocity of the data and it can be processed within a samll period of time.
4. `Spupport for Batch Processing Only`: Hadoop MapReduce only support batch processing, it does not able to process streamed, graph and machine learning data, hence overall performance is slower. Husky have purposed a unified framework, which support different kind of tasks with multiply purposes. Which can achieve high performance and support user-firend API among C++, python and Scala.

### In-Memory Processing

Hadoop MapReduce is not designed for iterative task like K-Means shown below. Every intermediate data will have not saved into persistent storage, which great increase the latency.

<img src="https://raw.githubusercontent.com/RickAi/csci5570_survey_paper/master/images/haloop_kmeans.jpg" style="zoom:50%" />

`HaLoop` is a great extension for Hadoop  as it provides support for iterative application. In order to meet these requirement, several main changes that are made in Hadoop to efficiently support iterative data analysis:

1. Providing a new application programming interface to simplify the iterative expressions.
2. An automatic generation of MapReduce program by the master node using loop control module until the loop condition is met.
3. The new task scheduler supports data locality in these application in order to efficiently perform iterative operations.
4. The task scheduler and task tracker are modified not only to manage execution but also manage cache indices on slave module.

The HaLoop performance results demonstrate that pushing support for iterative programs into the MapReduce engine greatly improves the overall performance of itertive data analysis applications.

`RDD` stands for "Resilient Distributed Dataset", it is the fundamental data structure of Apache Spark. RDD in Apache Spark is an immutable collection of objects which computes on the different node of the cluster. As Hadoop MapReduce makes the iterative computing such as Logistic Regression, K-Means and PageRank slower. Although HaLoop guarantee faster computing with caching extension, the fault tolerance and other questions still exist. RDDs try to solve these problems by enabling fault tolerant distributed In-Memory computations.

Compare with HaLoop, when the worker node in Spark goes down, the system will use Lineage, a track of all the transformations that has to be applied on that RDD including from where it has to read the data, to re-compute the lost partition of RDD from the original one.

### DAG Processing

Hadoop MapReduce restricts all computations to take a single input set and generate a single output set, which will cause extra overhead in solving tasks with multiply stages. 

In `Dryad`, each job will be represented with a DAG, the intermediate vertices were writen to channels, and more operation than map and reduce will be used, such as join and distributed. With dataflow, the developer do not need to worry about the global state of the computing system, just need to write simple vertices that maintain local state and communicate with other vertices throught edges. Compare with DAG, MapReduce is just a simple form of dataflow, with two types vertices: the mapper and the reducer. Compare with MapReduce, Dryad offers more advantages:

1. Big jobs will be more efficient with Dryad
2. Dryad can provides explicit join, comnbines inputs of different types
3. Dryad "Split" proceduces outputs of different types

`FlumeJava` is a higher level interfaces to control data-parallel pipeline of MapReduce jobs. This allowed developers to write code which would be used to build an execution plan for a series of MapReduce jobs. With similar DAG idea, there are some difference between FlumeJava and Dryad:

||Dryad|FlumeJava|
|---|---|---|
|Publish Year|2007|2010|
|Purpose|General purpose distributed execution engine based on DAG Model|A higher level interface to control data-parallel pipeline|
|Implementation|C++|Java|
|Worker Model|Job Manager and Daemons|Pipeline Executor|
|Computation Model|Jobs were specified by arbitrary DAGs with vertexs as a program and each edges as data channel|Primitives: parallelDo, groupByKey, combineValues and flatten|
|Optimization|Graph composition, vertices are grouped into stages, pipelined execution, runtime dynamic graph refinement|Deferred evalution and execution plan|
|Scheduling|Based on network locality|Batch execution and MapReduce scheduling|
|Storage|Use local disks and distributed FS similar to GFS|GFS, local disk cache|
|Fault Tolerance|Task re-execution in context of pipeline|Similar to MapReduce|

### Stream Processing

Hadoop MapReduce was designed to support batch processing, it is not suitable for streaming data processing. Stream processing should enable users to query continuous data stream and detect conditions fast within a small time period fro mthe time of receiving the data, the detection time period may vary from few milliseconds to minutes.

The `Naiad` project is an investigation of data-parallel dataflow computation like Dryad, but with a focus on the low-latency streaming and cyclic computations. It introduces a new computational model called `Timely Dataflow`, which combines low-latency asynchronous message flow with lightweight coordination when required. Naiad's most notable performance property, when compared with other data-parallel dataflow systems, it its ability to quickly coordinate among the workers and establish that stages have completed. Naiad support efficient implemntations of a variety of progremming patterns, including nested iterative algorithms and incremental updates to iterative computations.

Popular stream processing framework includes Spark Streaming, Storm and Flink.

### Machine Learning Processing

Many machine learning problems reply on large amounts of data for training, companies nowadays training algorithms with terabytes or petabytes of data, and create models out of it. Such models consist of weights that will optimize for error in inference for most cases. The number of weights/parameters run into orders billions to trillions. In such big model, learning on a single machine is not possible. It is useful to have a framework that can be used for distributed learning as well as inference.

A system called `Parameter Server` have been purposed for solving LDA algorithm efficiently, this framework have developed as a more general platform  called `ps-lite` for now, the development history is below:

1. In 2010, Alex Smola purposed a parallel-LDA computing framework, which is the first generation parameter server, which use memcached as the parameter storage system. It can successfuly training LDA model in parallel, but still lack of efficiency and flexibility.
2. Jeff Dean from Google purposed the second generation parameter server called DistBelief, which stores massive deep learning model parameters into the global parameter server nodes. It efficiently solve the SGD and L-BFGS algorithm training problem in parallel.
3. Mu Li purposed the third generation parameter server called ps-lite, which is a more general platform support flexible consistency models, elastic scalability, and continuous fault tolerance.

### General Purpose Platform

System like Hadoop and Spark have been widely adopted for big data processing, however, sometimes over-simplified API stop developers from more find-grained control and designing more effifient algorithms, but using sophisticated DSLs may result in development cost and bugprong programming. 

A general research platform called `Husky` is able to help developers implement applications of different characteristics, for example, coarse-grained and fine-grained, iterative and non-iterative, sychronous and asynchronous workloads, and achieves performance close to or better than specialized systems and programs.

## Hadoop MapReduce

`MapReduce` is mainly used for parallel processing of large sets of data stored in Hadoop cluster. In the very begining, it is a hypothesis designed by Google to provide parallism, data distribution and fault-tolerance. MapReduce process data in the form of key-value pairs. A KV pair is the mapping element between two linked data items.

For processing large sets of data MR comes into the picture. The developers are able to write MapReduce applications that could be suitable for their bussiness scenarios. The MR work flow undergoes different phases and the end result will be stored in HDFS with replications. JobTracker plays the vital role in scheduling jobs and will keep track of the entire map and reduce jobs. The detail source code can be found in my Github: [hadoop_mapreduce_process_source_code](https://github.com/RickAi/csci5570_survey_paper/blob/master/hadoop_mapreduce_process_source_code.md)

<img src="https://raw.githubusercontent.com/RickAi/csci5570_survey_paper/master/images/mapreduce.jpg" style="zoom:50%" />

The mapreduce process can be illustrated with core map and reduce process, but some details were hidden, the whole work flow should be: 

Split->Mapper->Partioner->Sort->Combiner->Shuffle->Sort->Reducer->Output

### Mapper Phase

In mapper phase, the input data is going to split into 2 components, key and value. The key is writable and comparable in the processing stage. Value is writable only during the processing stage. In this stage, one block is processed by one mapper at a time, in the mapper, a developer can specify his own bussiness logic as the requirement.

### Partition Phase

The partition module in Hadoop MapReduce also play a very important role to partition the data received from either different mappers or combiners. Partitioner reduce the pressure that builds on reducer and gives more performance. There is a customized partition which can be performed on any relevant data on different basis or conditions.

### Combine Phase

Combiner is also called mini reducer, usually the code is similar to the reducer. When the mapper output is huge amount of data, it will require high network bandwidth. To solve this issuce, the combiner can be placed after mapper to improve performance.

### Shuffle & Sort

After the mapping process, there are shuffle and sorting process on the intermediate data. The data will be stored in the local file system without having any fault tolerance like replications or checkpoint in Hadoop nodes. In detail, Hadoop uses a Round-Robin algorithm to write the intermediate data to the local storage.

### Reducer Phase

When the data was shuffled and sorted, it will be pass as the input to the reducers. In this phase, the developer can define cutomized bussiness code and use writer to writes data from reducer to storage like HDFS. Reducer mainly do operations on data from mapper, and finally will output data named like part-r-0001 etc.

## HaLoop

HaLoop was developed as a extension of Hadoop which along with processing of data providers a alternative way to perform iterative computation on Hadoop MapReduce..

To support extra features, there are some changes based on Hadoop MapReduce:

<img src="https://raw.githubusercontent.com/RickAi/csci5570_survey_paper/master/images/mapreduce.jpg" style="zoom:80%" />


## Dryad

## FlumeJava

## Spark

## Naiad

## PsLite

## Husky

## Related Systems

## Comparison of distributed analytics systems

||Hadoop MapReduce|HaLoop|Dryad|FlumeJava|Spark|Naiad|Ps-Lite|Husky|
|---|---|---|---|---|---|---|---|---|
|Year|||||||||
|Language Support|||||||||
|Type|Batch|Batch|Batch|Batch|Batch|Streaming|ML||
|In-Memory Processing|||||||||
|Data Flow|||||||||
|Data Processing|||||||||
|Scalability|||||||||
|Latency|||||||||
|Throughput|||||||||
|Performance|||||||||
|Fault Tolerance|||||||||
|Messaging|||||||||
|Community Adaption|Wide|Selective|Selective|Wide|Wide|Growing|Selective|Selective|

## Future

## Conclusion

## References