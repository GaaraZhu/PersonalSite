---
title: "MapReduce tuning"
date: 2012-12-23T01:33:45+13:00
draft: false
tags: ["MapReduce", "Performance tuning", "Hadoop"]
readingTime: 5
customSummary: MapReduce is not a low latency computing model, minutes at least will go before we got the result. But this does not mean there's no way for us to make it faster. Combined with the nature of Hadoop, here are some solutions from different pespective.
---

## Backgroup
MapReduce is not a low latency computing model, minutes at least will go before we got the result. But this does not mean there's no way for us to make it faster. Combined with the nature of Hadoop, here are some solutions from different pespective.

## Input file size

MapReduce excels at processing a few big files instead of lots small files. This is because too many small files will lead to a memory overflow of Namenode as it holds all metadata of input files, unnecessary threads will also be started for resource initializing, and communications between Namenode and Datanode will become more and more inefficient. Then what can we do when confronted with many small files? CombineFileInputFormat will work in this case, what we need to do is simply override the getReader() method to specify a reader to manage with multiple input splits, then use this format instead of the default input format in your job configuration.

## Local computing

Sometimes, we can do a local "reduce" after the Map process. It can reduce the output in every Map node, and it will be more efficient for the following Reduce process, which will collect output data from all Map nodes and reduce it. Using Combine function can do this . Take the example in my previous blog, the combine function will be like this:

```
public class SearchCriteriaCombiner extends
		Reducer<Text, IntWritable, Text, IntWritable> {
 
	@Override
	protected void reduce(Text searchCrieria, Iterable<IntWritable> counts,
			Context context) throws IOException, InterruptedException {
 
		int totalCount = 0;
 
		for (IntWritable singleCount : counts) {
			totalCount += singleCount.get();
		}
		context.write(searchCrieria, new IntWritable(totalCount));
 
	}
 
}
```
Also, do not forget to set the combiner in your job configuration.

### I/O cost between Mapper and Reducer
One bottleneck of MapReduce is the I/O cost between map nodes and reduce node. First Map process will write its output result into file system(disk, not HDFS) and then will transfer to reduce node for Reduce process. If the output of Map process is a big size, we can do a compression before writing and transferring, and this will comes with a performance improvement as well as disk space saving. Attention that, it will also lead to an extra cost for a decompression when reading data.

## Conclusion
Here lists three main solutions for tuning our MapReduce function, there are also other ways, like tuning task schedule, customize comparator, customize task quantity of Map and Reduce tasks and etc.

Above all, what we need is to analyze the main cost case by case before taking a tuning solution.  
&nbsp;  
&nbsp;
