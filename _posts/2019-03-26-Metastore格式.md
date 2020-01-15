---
layout:     post
title:      Metastore格式
subtitle:   格式和分隔符
date:       2019-02-26
author:     jianguotian
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - 原创
    - Metastore
    - INPUTFORMAT/OUTPUTFORMAT
    - SerDe
---


Hive目前支持的数据格式包括Text File、SequenceFile、RCFile、Avro、ORC 和Parquet，同时可以自定义INPUTFORMAT和OUTPUTFORMAT来支持其他格式。
https://cwiki.apache.org/confluence/display/Hive/FileFormats
具体到某种数据格式，Hive是如何进行支持的呢？主要涉及INPUTFORMAT/OUTPUTFORMAT、SerDe分片（Split）和记录（Record）等概念。
具体来说，分片被划分为若干个记录；每条记录就是一个键值对，对应一行。InputFormat将输入文件生成分片，并将分片分割成记录。SerDe是SerializerDeserializer的缩写。以Hive读取HDFS文件为例，会有以下处理：
1）调用InputFormat，将文件生成分片并分割成记录；
2）调用SerDe的Deserializer，将一行(Row)，切分为各个字段。
Hive向HDFS写入文件时，顺序相反。大体流程如下所示：
HDFS files --> InputFileFormat --> Deserializer --> Row object --> Serializer --> OutputFileFormat --> HDFS files
![Hive读写记录](https://upload-images.jianshu.io/upload_images/7440793-bd3cb5c8c71f65e6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### INPUTFORMAT/OUTPUTFORMAT
通常，Hive中的建表语句如下：
```sql
CREATE TABLE my_table(a string, b string, ...)
COMMENT 'This is the page view table'
PARTITIONED BY(dt STRING, country STRING)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY '\001'
WITH SERDEPROPERTIES (
   "separatorChar" = "\t",
   "quoteChar"     = "'",
   "escapeChar"    = "\\"
)  
STORED AS TEXTFILE;
```
![Hive数据表](https://upload-images.jianshu.io/upload_images/7440793-107796043c0fc80e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
INPUTFORMAT/OUTPUTFORMAT对应创建表时的STORED AS语句，如果自定义编写的INPUTFORMAT/OUTPUTFORMAT，在创建表时可以写成STORED AS INPUTFORMAT '<packagepath.classname>' OUTPUTFORMAT '<packagepath.classname>' 的形式。
Hive中支持的format参照 https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DDL#LanguageManualDDL-StorageFormats
### SerDes 
官方wiki https://cwiki.apache.org/confluence/display/Hive/SerDe
TEXTFILE格式使用默认的SerDe，LazySimpleSerDe。
Hive中支持的SerDe参照 https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DDL#LanguageManualDDL-RowFormats&SerDe

&emsp;总结：Metastore对文件格式的支持基于以上概念和原理，如果有自定义格式的需求（以JSON为例），无论基于现有的开源方案，还是自定义编写，都是可以的。