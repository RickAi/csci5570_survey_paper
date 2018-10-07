## Example

Write a MapReduce program to find the top-N pairs of similar users from a repository called Movielens. Provided is the dataset of Movielens 20M, which can be found in http://grouplens.org/datasets/movielens/20m/. The original range of the rating scores is in [0.5,5.0]. For simplicity, we ignore the missing values in the ratings. You need to calculate the top-N pairs of similar users based on the ratings. In order to calculate the similarity, we redefine the rating score in [0.5,2.5] as unlike, and the rating score in (2.5,5.0] as like. Then, for a pair of users of (𝑖, 𝑗), you can calculate the similarity between them via the Jaccard Distance.

```java
public static void main(String[] args) {
    try {
        Configuration conf = new Configuration();
        String[] otherArgs = new GenericOptionsParser(conf, args).getRemainingArgs();

        Job userPairJob = Job.getInstance();
        userPairJob.setJarByClass(SimilarUserPairMain.class);

		// set the mapper and reducer class
        userPairJob.setMapperClass(UserPairMapper.class);
        userPairJob.setReducerClass(UserPairReducer.class);

		// set the format of the input and output
        userPairJob.setMapOutputKeyClass(UserPairWritable.class);
        userPairJob.setMapOutputValueClass(CommonPrefWritable.class);
        userPairJob.setOutputKeyClass(UserPairWritable.class);
        userPairJob.setOutputValueClass(DoubleWritable.class);

		// set the HDFS url for input and output
        FileInputFormat.addInputPath(userPairJob, new Path(otherArgs[0])); // input
        FileOutputFormat.setOutputPath(userPairJob, new Path(otherArgs[1])); // output

		// start the job in BSP mode
        if (userPairJob.waitForCompletion(true)) {
            System.out.println("userPairJob job success.");
        } else {
            System.out.println("userPairJob job failed.");
        }
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

This is a simple outline of the MapReduce program, the detailed code can be found in my GitHub: [RickAi/MapReduce_UserSimilarity](https://github.com/RickAi/MapReduce_UserSimilarity)

Usually, the MapReduce program will have multiply Job executed in BSP mode, and the data input for the first Mapper should been carefullly preprocessed.

## Job.waitForCompletion()

org/apache/hadoop/mapreduce/Job.java

```java
public boolean waitForCompletion(boolean verbose
                                   ) throws IOException, InterruptedException, ClassNotFoundException {
    if (state == JobState.DEFINE) {
    	// submit and start the job
        submit();
    }
    if (verbose) {
    	// when the verbose is on, the extra log will be print during the job execution
        monitorAndPrintJob();
    } else {
        ...
    }
    return isSuccessful();
}
```

### Job.submit()

```java
public void submit() 
    throws IOException, InterruptedException, ClassNotFoundException {
    ...
    // connect and initialize the cluser
    connect();
    final JobSubmitter submitter = 
        getJobSubmitter(cluster.getFileSystem(), cluster.getClient());
    // instead looking at the workflow of UserGroupInformation ugi, we concentrate on the job submit
    status = ugi.doAs(new PrivilegedExceptionAction<JobStatus>() {
        public JobStatus run() throws IOException, InterruptedException, 
        ClassNotFoundException {
        	// submit the job via JobSubmitter
            return submitter.submitJobInternal(Job.this, cluster);
        }
    });
    state = JobState.RUNNING;
    LOG.info("The url to track the job: " + getTrackingURL());
}
```

org/apache/hadoop/mapreduce/JobSubmitter.java

```java
JobStatus submitJobInternal(Job job, Cluster cluster) 
    throws ClassNotFoundException, InterruptedException, IOException {
    Path jobStagingArea = JobSubmissionFiles.getStagingDir(cluster, conf);
    ...
    Path submitJobDir = new Path(jobStagingArea, jobId.toString());
    ...
    try {
        if (conf.getBoolean(
            // Now, actually submit the job (using the submit name)
            ...
            status = submitClient.submitJob(
                jobId, submitJobDir.toString(), job.getCredentials());
            ...
        } finally {
            ...
        }
    }
}
```

org/apache/hadoop/mapred/LocalJobRunner.java

```java
public org.apache.hadoop.mapreduce.JobStatus submitJob(
    org.apache.hadoop.mapreduce.JobID jobid, String jobSubmitDir,
    Credentials credentials) throws IOException {
    // the major job is insdie the construction of Job class
    Job job = new Job(JobID.downgrade(jobid), jobSubmitDir);
    job.job.setCredentials(credentials);
    return job.status;
}
```

org/apache/hadoop/mapred/LocalJobRunner.class

```java
public Job(JobID jobid, String jobSubmitDir) throws IOException {
    // initialize the system conf file
    this.systemJobDir = new Path(jobSubmitDir);
    this.systemJobFile = new Path(this.systemJobDir, "job.xml");
    this.id = jobid;
    JobConf conf = new JobConf(this.systemJobFile);
    this.localFs = FileSystem.getLocal(conf);

    ...
    // start the job
    this.start();
}
```

From now on, the job have been successfully initialized,  the Job class is actually is sub-class of the Thread, for the real implementation of the core operation such as map, combine and reduce, we need look at the Job's `run()` method, as Job is a sub-class of the class Thread.

## Mapper.map()

org/apache/hadoop/mapred/LocalJobRunner.java

```java
public void run() {
    ...
    // return a MapTaskRunnable list, initialized with taskSplitMetaInfos
    List<RunnableWithThrowable> mapRunnables = getMapTaskRunnables(
        taskSplitMetaInfos, jobId, mapOutputFiles);

    initCounters(mapRunnables.size(), numReduceTasks);
    // create a thread pool to run map tasks
    ExecutorService mapService = createMapExecutor();
    runTasks(mapRunnables, mapService, "map");
    ...
}
```

```java
private void runTasks(List<RunnableWithThrowable> runnables,
                      ExecutorService service, String taskType) throws Exception {
    // Start populating the executor with work units.
    // They may begin running immediately (in other threads).
    for (Runnable r : runnables) {
        service.submit(r);
    }
    ...
}
```

At here, we can notice the MapTaskRunnable will be put into the thread pool, when the system resources is ready, the MapTask will be executed by one of the thread in the pool. Then let's look at the MapTaskRunnable.run() for the execution details.

### MapTaskRunnable.run()

```java
public void run() {
    try {
        ...
        // initialize a new MapTask
        MapTask map = new MapTask(systemJobFile.toString(), mapId, taskId, info.getSplitIndex(), 1);
        ...
        try {
            ...
            // real execution
            map.run(localConf, Job.this);
        } finally {
            map_tasks.getAndDecrement();
        }

        LOG.info("Finishing task: " + mapId);
    } catch (Throwable e) {
        this.storedException = e;
    }
}
```

org/apache/hadoop/mapred/MapTask.class

```java
public void run(final JobConf job, final TaskUmbilicalProtocol umbilical)
    throws IOException, ClassNotFoundException, InterruptedException {
    ...
    if (useNewApi) {
        runNewMapper(job, splitMetaInfo, umbilical, reporter);
    } else {
        ...
    }
    done(umbilical, reporter);
}

private <INKEY, INVALUE, OUTKEY, OUTVALUE> void runNewMapper(JobConf job, TaskSplitIndex splitIndex, TaskUmbilicalProtocol umbilical, TaskReporter reporter) throws IOException, ClassNotFoundException, InterruptedException {
    TaskAttemptContext taskContext = new TaskAttemptContextImpl(job, this.getTaskID(), reporter);
    // initialzed Mapper class with reflection
    Mapper<INKEY, INVALUE, OUTKEY, OUTVALUE> mapper = (Mapper)ReflectionUtils.newInstance(taskContext.getMapperClass(), job);
    ...

    try {
        input.initialize(split, mapperContext);
        // run the mapper
        mapper.run(mapperContext);
        ...
    } finally {
        this.closeQuietly((org.apache.hadoop.mapreduce.RecordReader)input);
        this.closeQuietly((RecordWriter)output, mapperContext);
    }

}
```

org/apache/hadoop/mapreduce/Mapper.java

```java
public void run(Mapper<KEYIN, VALUEIN, KEYOUT, VALUEOUT>.Context context) throws IOException, InterruptedException {
    this.setup(context);

    try {
    	// start the map process loop until no next value need to be process
        while(context.nextKeyValue()) {
            this.map(context.getCurrentKey(), context.getCurrentValue(), context);
        }
    } finally {
        this.cleanup(context);
    }

}
```

Here comes the core operation of the map process, it will first call `setUp` and then enter the map() process loop, and finally call te cleanUp for the termination of the Mapper process.

## Combiner.reduce()

```java
private <INKEY, INVALUE, OUTKEY, OUTVALUE> void runNewMapper(JobConf job, TaskSplitIndex splitIndex, TaskUmbilicalProtocol umbilical, TaskReporter reporter) throws IOException, ClassNotFoundException, InterruptedException {
	...
	// when the combiner's number is not 0, tray to init a sorting collector
    if (job.getNumReduceTasks() == 0) {
        output = new MapTask.NewDirectOutputCollector(taskContext, job, umbilical, reporter);
    } else {
        output = new MapTask.NewOutputCollector(taskContext, job, umbilical, reporter);
    }
    ...
}
```

This operation will finllay call into MapOutputBuffer, which will initialize a SpillThread

org/apache/hadoop/mapred/MapTask.java

```java
public void init(Context context) throws IOException, ClassNotFoundException {
	...
    this.spillThread.setDaemon(true);
    this.spillThread.setName("SpillThread");
    this.spillLock.lock();

    try {
        this.spillThread.start();

        while(!this.spillThreadRunning) {
            this.spillDone.await();
        }
    } catch (InterruptedException var10) {
        throw new IOException("Spill thread failed to initialize", var10);
    } finally {
        this.spillLock.unlock();
    }
    ...
}
```

Notice teh SpillThread is started, then we look at the `run()` method

```java
protected class SpillThread extends Thread {
    protected SpillThread() {
    }

    public void run() {
        MapOutputBuffer.this.spillLock.lock();
        MapOutputBuffer.this.spillThreadRunning = true;

        try {
            while(true) {
                MapOutputBuffer.this.spillDone.signal();

                while(!MapOutputBuffer.this.spillInProgress) {
                	// wait for map process to be done
                    MapOutputBuffer.this.spillReady.await();
                }

                try {
                    MapOutputBuffer.this.spillLock.unlock();
                    // start to use combiner to reduce the value
                    MapOutputBuffer.this.sortAndSpill();
                }
                ...
            }
        }
    }
}
```

Inside the `MapOutputBuffer.this.sortAndSpill()`, combiner will be initialized and start executing, after that, the data from mapter will be merged and write into the local data storage. 

To prove that, we just look at `MapOutputBuffer.this.spillReady.await()`, it should be waiting for mapper progress to be done:

```java
private void startSpill() {
	...
    this.spillReady.signal();
}
```

The `startSpill` method was called right after the `context.write()` in the mapper.

org/apache/hadoop/mapred/MapTask.java

```java
private void sortAndSpill() throws IOException, ClassNotFoundException, InterruptedException {
    ...
    if (spstart != spindex) {
        this.combineCollector.setWriter(writer);
        RawKeyValueIterator kvIter = new MapTask.MapOutputBuffer.MRResultIterator(spstart, spindex);
        this.combinerRunner.combine(kvIter, this.combineCollector);
    }
    ...
}
```

With above analysis, we can know every time when mapper called `context.write`, the `sortAndSpill` will be called. However, only `spstart != spindex` met, the combiner will be called.

Anyway, We can know the combine progress is after the mapper, and when combiner finished, the data will be stored into the local disk for fault torance, after that, theose data will be sent to the reducer for final merge.

## Reducer.reduce()

After the map process have been finished, the combiner and reducer's work continues at `LocalJobRunner.run()`.

org/apache/hadoop/mapred/LocalJobRunner.java

```java
public void run() {
    // submit the map task into thread pool
    ...

    if (numReduceTasks > 0) {
        List<RunnableWithThrowable> reduceRunnables = getReduceTaskRunnables(
            jobId, mapOutputFiles);
        ExecutorService reduceService = createReduceExecutor();
        // run the task with thread pool
        runTasks(reduceRunnables, reduceService, "reduce");
    }
}
```

As we see, Reducer's process is pretty like the mapper.

org/apache/hadoop/mapred/ReduceTask.java

```java
public void run(JobConf job, final TaskUmbilicalProtocol umbilical)
    throws IOException, InterruptedException, ClassNotFoundException {
    ...
    ShuffleConsumerPlugin.Context shuffleContext = ...
    shuffleConsumerPlugin.init(shuffleContext);

    // shuffle the data
    rIter = shuffleConsumerPlugin.run();

    // free up the data structures
    mapOutputFilesOnDisk.clear();

    sortPhase.complete();                         // sort is complete
    setPhase(TaskStatus.Phase.REDUCE); 
    statusUpdate(umbilical);
    Class keyClass = job.getMapOutputKeyClass();
    Class valueClass = job.getMapOutputValueClass();
    RawComparator comparator = job.getOutputValueGroupingComparator();

    if (useNewApi) {
        runNewReducer(job, umbilical, reporter, rIter, comparator, 
                      keyClass, valueClass);
    } else {
        runOldReducer(job, umbilical, reporter, rIter, comparator, 
                      keyClass, valueClass);
    }

    shuffleConsumerPlugin.close();
    done(umbilical, reporter);
}
```

Follow the same pattern as Mapper, the reducer will be called in a while loop, until all the data processed by the reducer.

org/apache/hadoop/mapred/Task.java

```java
public void done(TaskUmbilicalProtocol umbilical, Task.TaskReporter reporter) throws IOException, InterruptedException {
    LOG.info("Task:" + this.taskId + " is done." + " And is in the process of committing");
    this.updateCounters();
    ...
}
```

The mapreduce job will be finished after `LocalJobRunner.run()` is done.

