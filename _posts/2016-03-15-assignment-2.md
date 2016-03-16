---
layout: post
title: Assignment 2
---

<!-- Introduce the reader briefly to spark, and, if you like, the way you carry out the assignment: in the terminal room or at home, deviations from the default suggested commands that you needed to get things running conveniently, etc..

Briefly explain what you learned about going through the notebook. Copy the most relevant commands (modified where you thought interesting), and add a brief explanation of what the commands do. (View as report can be a handy feature!)

Do not forget to include what you learn from inspecting the Spark UI after issuing commands from the notebook! (Hint: comment on lazy evaluation and/or the effect of caching RDDs.) -->

# Introduction
This is the blog post for assignment 2a. In this blog post we'll take a brief look at what exactly Spark is, how the first assignment went, some specifics about Spark and how I'm doing these assignments.

## What is Spark
So what exactly is Spark? Spark is a framework used for cluster computing. Frameworks like these are generally used for big data situations, especially those where the data and calculations can't be done in a computer's memory alone, which would likely severely slow down the process as it would have to read memory from disk all the time, or use the swap.

Another popular framework for cluster computing is Hadoop MapReduce

<!-- TODO: Some more about hadoop and the differences -->

## How to run and use Spark

There are different ways to actually run and use Spark, but the simplest way is using Spark Notebook and Docker.

Spark Notebook is based on Scala Notebook, which again is based on iPython Notebook (nowadays known as [Jupyter](http://jupyter.org/)). So what does this mean, a spark notebook?
These notebooks provide interactive programming environments, visualized as a sort of document. The image below shows part of the notebook for the first assignment.

![Spark Notebook](Big Data Spark 101 - Google Chrome_006.png)

In this image alone you can already see three different parts:

1. Normal text
2. Code snippets that can be run (indicated by `In []:`)
3. Output from these snippets (indicated by `Out [*run number*]:`)

Behind this document is the interactive environment, and each snippet is run in this environment, meaning that different snippets have access to the same session.

All of this can be run in [Docker](https://www.docker.com/). Docker provides a way to create containers. In these containers you can easily define what OS should be run, which programs should be installed, and a lot more. The big advantage of this is that these containers can be run on any system on which Docker can be installed, preventing any issues with setup, except for possibly Docker.

## My setup
I'm personally running these exercises on my own laptop with Ubuntu 15.10 (dualboot Windows 10). Reason for this is that I already had most of the stuff up and running under Ubuntu, including Docker. So getting started was only a matter of running two commands:
```
> docker pull andypetrella/spark-notebook:0.6.2-scala-2.11.7-spark-1.6.0-hadoop-2.7.1-with-hive-with-parquet
> docker run -p 9000:9000 -p 4040-4045:4040-4045 andypetrella/spark-notebook:0.6.2-scala-2.11.7-spark-1.6.0-hadoop-2.7.1-with-hive-with-parquet
```

Initially I was a bit annoyed that every time I started the container I had to redownload all required files. But this was just me misunderstanding Docker a bit. Each time you start a container the way I did before it creates a new instance of that image. So at a certain point I had multiple instances on my disk, all shut down. These are visible when you run `docker ps -a`
```
CONTAINER ID        IMAGE                                                                                            COMMAND                CREATED             STATUS              PORTS                                                                NAMES
e75047541f2a        andypetrella/spark-notebook:0.6.2-scala-2.11.7-spark-1.6.0-hadoop-2.7.1-with-hive-with-parquet   "bin/spark-notebook"   7 minutes ago       Up 7 minutes        0.0.0.0:4040-4045->4040-4045/tcp, 0.0.0.0:9000->9000/tcp, 9443/tcp   awesome_lovelace
```
So instead of doing another docker run like above, you can just do `docker run e75047541f2a`, which uses the hash to refer to a previously existing image on my disk. This way any changes I made were also still present.

On top of reusing containers I also created a more permanent place for the relevant files that I'm using in this container. Currently we're using `/data` for data files and `/opt/docker/notebooks/BigData` for notebooks. I replaced these directories with [docker volumes](https://docs.docker.com/engine/userguide/containers/dockervolumes/). By mounting two of my local directories in this container, any change I make in this container is actually a change on my PC. So even when the container is shut down, removed, or anything else, I can still access that data. Mounting these directories was done using the `-v` parameter,
```
> docker run -p 9000:9000 -p 4040-4045:4040-4045 -v ~/rubigdata/data:/data -v ~/rubigdata/BigData:/opt/docker/notebooks/BigData andypetrella/spark-notebook:0.6.2-scala-2.11.7-spark-1.6.0-hadoop-2.7.1-with-hive-with-parquet
```
This mounts `~/rubigdata/data` to `/data` and `~/rubigdata/BigData` to `/opt/docker/notebooks/BigData`

## Going through the first notebook
This section will be quite extensive, as I like to write things out to get a better understanding. There's a TL;DR near the bottom which answers the questions asked in the notebooks.

Spark uses RDDs, Resilient Distributed Datasets. The idea behind this type of dataset is that you can perform different actions in parallel. So how exactly do you create an RDD? Well, that's simple:
```scala
val rdd = sc.parallelize(0 to 999,8)
```
So what exactly does this do? It basically parallelizes access to an array of 0 to 999 through 8 different pools.
So how do we now access that dataset? By performing actions on `rdd`, for example taking four samples without replacement:
```
val sample = rdd.takeSample(false, 4)
```
One important thing that is immediately visible here is that Spark is lazy and only evaluates on actions. So when we defined our initial dataset nothing actually happens, only when we started taking samples this dataset was generated.

<!-- TODO: Why two tasks? -->

Alright, lets get to some more interesting data. We load some Shakespear as an RDD:
```
val lines = sc.textFile("/data/100.txt.utf-8")
```
For the previous RDD, with an array from 0 through 999, it was clear where the partitions were over. In this case `textFile` automatically splits it based on newlines and returns an RDD of strings.

These RDDs have default parameters like `count`, showing how large it is. It gets more interesting once you start applying maps and reduce the results. We start out with a simple one:
```scala
lines.map(s => s.length).reduce((a, b) => a + b)
```
This one counts the total length of all sentences in the document. The initial map converts each sentence to an integer indicating its length. One important thing to keep in mind here is that this map function creates a new RDD and that, on top of that, it is lazy. Only when we actually perform an action, like reducing the results, everything is evaluated. This reduce doesn't take a key, but simply performs the reduction on a dataset, in this case the lengths of the sentences, adding them all together. One thing to note here is the lambda in the reduce. Scala actually has some fancy underscore magic, where we can replace this lambda `(a, b) => a + b` with `_ + _`. These underscores basically tell that 'well, we have two variables called whatever, just add them up', and is an anonymous function. [This](http://www.codecommit.com/blog/scala/quick-explanation-of-scalas-syntax) post goes a bit more in depth on the issue.

Instead of simply counting the number of characters in the document we can also count the words:
```
val wordcount = lines.flatMap(line => line.split(" "))
    .filter(_ != "")
    .map(word => (word,1))
    .reduceByKey(_ + _)
wordcount.take(5)
```
The first map is a bit different from before, because we use `flatMap` instead of `map`. The reason behind this is that `map` is used to basically convert items from one type to another, for example from `string` to `int`. `flatMap` is used when you want to convert single items to multiple items. In this case because we want to change our single string into a set of words. `flatMap` does this by applying the function to all elements and then flattening the result into a new RDD.  
Obviously we're not interested in empty words, so these are filtered out. Any remaining words are then mapped into a tuple with the number one. The first part of the tuple, the word, will act as the key, and the number 1 as the value, indicating how often we've seen this word. The last step is actually reducing these values to what we're looking for.  
`reduceByKey` is a nice function which performs the function you give it on sets with the same key. In this case it's basically summing all the values that we get from the mapper. While you might think that `reduceByKey` behaves the same as `reduce`, it is actually fundamentally different. `reduce` acts as an action on RDDs, so everything that is required for that reduce is evaluated. `reduceByKey` actually generates a new RDD, so even at this point we're still working lazily. Only once we actually run the last line, `.take(5)`, we evaluate the parts we need.

An important thing for spark is that it keeps a lineage for each RDD, this lineage contains all steps that were taken to get at this specific point. This lineage can be seen using the parameter `.toDebugString`. If we run this on `wordcount` we get the following:
```
res15: String =
(2) ShuffledRDD[32] at reduceByKey at <console>:52 []
 +-(2) MapPartitionsRDD[31] at map at <console>:51 []
    |  MapPartitionsRDD[30] at filter at <console>:50 []
    |  MapPartitionsRDD[29] at flatMap at <console>:49 []
    |  MapPartitionsRDD[19] at textFile at <console>:48 []
    |  /data/100.txt.utf-8 HadoopRDD[18] at textFile at <console>:48 []
```
This shows all the steps we've taken, `textFile` to load it, `flatmap`, `filter`, `map` and eventually `reduceByKey`.

If we want to know the count for a specific word we can use a filter to select this:
```scala
wc.filter(_._1 == "Macbeth").collect
```
The lambda in the filter was a bit confusing at first. The first part basically consists of two elements. The first underscore is a wildcard, it simply takes whatever without specifying a name. The second part, `_1`, is actually a function by itself. This function selects the first element of a tuple. So in this case we're simply selecting the count for the word "Macbeth" and collecting all results (as opposed to take, where you can specify how many you want).

It is also possible to store RDDs using `.saveAsTextFile("directory")`. However this can store the data in multiple files in the directory, for example `part-00000` and `part-00001`. The reason for this is that this information was partitioned over multiple instances. If we look at the spark console we see that two tasks were run and each of these stored their data separately:

|Index |	ID|	Attempt|	Status|	Locality| Level|	Executor ID / Host|	Launch Time	Duration|	GC Time|	Input Size / Records|	Output Size / Records|	Errors|
|---|---|---|---|---|---|---|---|---|---|---|---|
|0|	40|	0|	SUCCESS|	PROCESS_LOCAL|	driver / localhost|	2016/03/16 02:13:01|	0.3 s|	18 ms|	2.7 MB (hadoop) / 62608|	4.1 MB / 452273| |
|1|	41|	0|	SUCCESS|	PROCESS_LOCAL|	driver / localhost|	2016/03/16 02:13:01|	0.3 s|	18 ms|	2.7 MB (hadoop) / 62179|	4.1 MB / 451788| |

So what if we want to do some smarter/more efficient word counting:
```scala
val words = lines.flatMap(line => line.split(" "))
    .map(w => w.toLowerCase().replaceAll("(^[^a-z]+|[^a-z]+$)", ""))
    .filter(_ != "")
    .map(w => (w,1))
    .reduceByKey( _ + _ )
```
The only thing that really changed here is the addition of the first map. This step now lowercases every word it encounters and removes special characters. What does this mean? The counts for the words are now a lot more representative than that they were before. Where we first counted 'Macbeth', 'Macbeth,' and 'Macbeth!' as separate instances, they are now all counted under 'Macbeth'.

## TL;DR, just the questions
