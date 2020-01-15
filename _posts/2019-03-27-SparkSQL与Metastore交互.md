---
layout:     post
title:      Spark SQL与Metastore交互
subtitle:   源码逻辑解析
date:       2019-02-27
author:     jianguotian
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Spark SQL
    - Metastore
    - SessionCatalog
    - ExternalCatalog
    - IsolatedClientLoader
    - HiveClientImpl
---

### 示例代码
```scala
val sparkSession = SparkSession.builder()
  .master("local").appName("test")
  .config("spark.sql.warehouse.dir", "/user/hive/warehouse")
  .config("hive.metastore.uris", "thrift://localhost:9083")
  .enableHiveSupport()
  .getOrCreate()
 
//通过SQL形式操作元数据信息
val listTbls = sparkSession.sqlContext.sql("show tables")
listTbls.show()
 
//通过SessionCatalog操作元数据信息
val catalog = sparkSession.sessionState.catalog
 
//列出所有的数据库
catalog.listDatabases()
//获取指定数据库的详情
catalog.getDatabaseMetadata("default")
 
/*
操作数据表相关
*/
val tblSeq = catalog.listTables("default");
for (table <- tblSeq) {
  catalog.getTableMetadata(table)
  println(catalog.getTableMetadata(table))
}
```
### 源码逻辑解析
- SparkSession根据配置信息反射获取SessionState对象时，会得到HiveSessionState对象。SessionState是Spark SQL的核心类
- HiveSessionCatalog继承自SessionCatalog，HiveSessionState中重载了HiveSessionCatalog对象
- SessionCatalog中又封装了ExternalCatalog，ExternalCatalog中对数据库、数据表、分区和注册函数等信息的读取与操作通过HiveClientImpl完成
- 创建HiveClient的操作位于IsolatedClientLoader类中

底层创建Metastore客户端的逻辑如下：

HiveExternalCatalog.client
  ->HiveUtils.withHiveExternalCatalog()
  ->IsolatedClientLoader.createClient()
  ->HiveClientImpl() ➔ newState()
  ->SessionState.start()
  ->Hive.getMSC()  //获取Metastore客户端

其中，SessionState和Hive为Hive中的类。
```scala
sparkSession.sqlContext.sql("show tables")最终会调用HiveClientImpl().listTables():

override def listTables(dbName: String, pattern: String): Seq[String] = withHiveState {
  client.getTablesByPattern(dbName, pattern).asScala
}
```
最后调用的是Hive.getTablesByPattern() ➔ HiveMetaStoreClient.getTables()