---
layout:     post
title:      Hadoop输入格式
subtitle:   InputFormat
author:     jianguotian
header-style: text
catalog: true
tags:
    - 分片
    - 记录
    - InputFormat
---

Hadoop中我们经常会见到下面这些概念：文件，块（block），分片（Split）和Record等，它们之间的关系是怎样的？在Hadoop内部，又是经过怎样的流程将它们联系在一起？
### 分片
我们都知道分片和Map Task是一一对应的关系，那么从**文件（块）-->分片-->Map Task**中间经历了什么呢？  
**主要做了两件事情：1）将数据切分成Split；2）将Split解析为<key, value>的键值对。** 这也正是InputFormat的作用：
```java
public abstract class InputFormat<K, V> {
      public abstract 
        List<InputSplit> getSplits(JobContext context
                                   ) throws IOException, InterruptedException;
      
        public abstract 
          RecordReader<K,V> createRecordReader(InputSplit split,
                                               TaskAttemptContext context
                                              ) throws IOException, 
                                                       InterruptedException;
}
```
#### 切分Split  
**如何确定分片的大小?**
```java
public abstract class FileInputFormat<K, V> extends InputFormat<K, V> {
  protected long computeSplitSize(long blockSize, long minSize,
                                  long maxSize) {
    return Math.max(minSize, Math.min(maxSize, blockSize));
  }
}
```
minSize：分片的最小字节数，maxSize：分片的最大字节数
- minSize < blockSize < maxSize：splitSize = blockSize
- blockSize < minSize < maxSize：splitSize = minSize(分片大小 > 块大小)
- minSize < maxSize < blockSize: splitSize = maxSize(分片大小 < 块大小)
**InputSplit**
注意，InputSplit是一个逻辑概念，不包含数据本身，而是指向数据的引用。
```java
public abstract class InputSplit {
  public abstract long getLength() throws IOException, InterruptedException;

  public abstract 
    String[] getLocations() throws IOException, InterruptedException;

  @Evolving
  public SplitLocationInfo[] getLocationInfo() throws IOException {
    return null;
  }
}
```
```java
public class FileSplit extends InputSplit implements Writable {
  //文件
  private Path file;
  //起始位置
  private long start;
  //长度
  private long length;
  //存储位置
  private String[] hosts;
}
```
运行作业的客户端调用getSplits()计算分片，然后将它们发送到AM，AM会使用存储位置信息调度map任务，使map任务尽量放在分片数据附近。 
在yarn.app.mapreduce.am.staging-dir目录下可看到job.split文件：  
![job.split](img\2019-05-21\am_staging.png)
#### Split解析为键值对
InputSplit生成后，map任务会把InputSplit传给InputFormat的createRecordReader()方法，map任务用一个RecordReader来生成记录的键值对，然后再传递给map函数。  
Mapper的run方法（K/V对是从传入的Context获取的）：
```java
public void run(Context context) throws IOException, InterruptedException {
  setup(context);
  while (context.nextKeyValue()) {
    map(context.getCurrentKey(), context.getCurrentValue(), context);
  }cleanup(context);
}
```

一个WordCountMapper示例（输出结果K/V对也是通过Context来完成）：
```java
public class WordCountMapper extends Mapper<LongWritable, Text, Text, IntWritable> {

    @Override
    protected void map(LongWritable k1, Text v1, Context context) throws IOException, InterruptedException {
        // 先将每一行转换为java的String类型
        String line = v1.toString();
        // 将行中的单词以空格作为分隔符分离出来得到一个字符串数组
        String[] words = line.split(" ");
        // 定义输出数据的变量k2和v2，类型分别为Text和IntWritable
        Text k2 = null;
        IntWritable v2 = null;
        // 统计单词并写入到上下文变量context中
        for (String word : words) {
            k2 = new Text(word);
            v2 = new IntWritable(1);
            context.write(k2, v2);
        }
    }
}
```
如何形象地理解文件切分成Split的过程呢？
以TextInputFormat为例，一个文件中包含以下内容：
```
On the top of the Crumpetty Tree
The Quangle Wangle sat,
But his face you could not see,
On account of his Beaver Hat.
......
```
假设该文件前4行记录切分到第一个Split，每条记录对应的<K,V>如下，即为map任务的输入：
```
(0, On the top of the Crumpetty Tree)
(33, The Quangle Wangle sat,)
(57, But his face you could not see,)
(89, On account of his Beaver Hat.)
```
K是偏移量，V为text文本。