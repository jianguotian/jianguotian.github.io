---
layout:     post
title:      Kyuubi服务源码解析
subtitle:   KyuubiServer
author:     jianguotian
header-style: text
catalog: true
tags:
    - Kyuubi
---
## 前言
Kyuubi服务与HiveServer2服务非常相似，在Kyuubi中很多类的设计和代码逻辑都参照了HiveServer2（Spark SQL Thrift Server也是同样的道理）。  
HiveServer2服务启动的源码解析参见：
[Hive源码剖析之HiveServer2服务启动过程](https://my.oschina.net/u/1778239/blog/738670)  
Spark SQL Thrift Server服务启动的源码解析参见以下两处文章：
[SparkSQL Hive ThriftServer 源码解析：Intro](https://mr-dai.github.io/sparksql_hive_thriftserver_source_2/)
[Spark SQL源码走读（一）：HiveThriftServer2：Intro](https://ieevee.com/tech/2016/06/01/spark-sql-1.html)
（注：以下代码模块中，英文注释皆为作者注释，中文注释是我自己加上的）
## KuubiServer.scala
### KuubiServer类声明及成员
```scala
/**
 * Main entrance of Kyuubi Server
 */
private[kyuubi] class KyuubiServer private(name: String)
  extends CompositeService(name) with Logging {
  //BackendService服务
  private[this] var _beService: BackendService = _
  def beService: BackendService = _beService
  //FrontendService服务
  private[this] var _feService: FrontendService = _
  def feService: FrontendService = _feService
  ***后续代码省略***
}
```
简单介绍一下KyuubiServer中两个很重要的成员**FrontendService**和**BackendService**：FrontendService负责维护与客户端的连接，与客户端进行交互，将客户端的SQL请求转发至FrontendService；BackendService负责执行SQL并将执行结果返回给FrontendService。FrontendService最后将结果返回至客户端。
### main方法
```scala
def main(args: Array[String]): Unit = {
    SparkUtils.initDaemon(logger)
    //加载配置
    val conf = new SparkConf(loadDefaults = true)
    setupCommonConfig(conf)

    try {
      val server = new KyuubiServer()
      //对各种服务进行初始化
      server.init(conf)
      //启动各种服务
      server.start()
      info(server.getName + " started!")
      if (HighAvailabilityUtils.isSupportDynamicServiceDiscovery(conf)) {
        info(s"HA mode: start to add this ${server.getName} instance to Zookeeper...")
        HighAvailabilityUtils.addServerInstanceToZooKeeper(server)
      }
    } catch {
      case e: Exception =>
        error("Error starting Kyuubi Server", e)
        System.exit(-1)
    }
```
### init方法
```scala
  override def init(conf: SparkConf): Unit = synchronized {
    this.conf = conf
    _beService = new BackendService()
    _feService = new FrontendService(_beService)
    //将BackendService和FrontendService服务加入serviceList中
    addService(_beService)
    addService(_feService)
    //调用父类CompositeService的init方法
    super.init(conf)
    SparkUtils.addShutdownHook {
      () => this.stop()
    }
  }
```
**CompositeService的init方法：**
```scala
  override def init(conf: SparkConf): Unit = {
    //依次调用serviceList中各个服务的init方法
    for (service <- serviceList) {
      service.init(conf)
    }
    super.init(conf)
  }
```
### start方法
```scala
  override def start(): Unit = {
    //遍历serviceList中所有的服务并依次启动
    serviceList.zipWithIndex.foreach { case (service, i) =>
      try {
        service.start()
      } catch {
        case e: Throwable =>
          error("Error starting services " + getName, e)
          stop(i)
          throw new ServiceException("Failed to Start " + getName, e)
      }
    }
    super.start()
  }
```
&emsp;&emsp;调用各个服务的init和start方法时，最终都会调用AbstractService的init和start方法（这些服务类要么直接继承AbstractService，要么继承CompositeService。而CompositeService又继承自AbstractService）。

### AbstractService的init和start方法
```scala
  override def init(conf: SparkConf): Unit = {
    ensureCurrentState(State.NOT_INITED)
    this.conf = conf
    changeState(State.INITED)
    info("Service: [" + getName + "] is initialized.")
  }

  override def start(): Unit = {
    startTime = System.currentTimeMillis
    ensureCurrentState(State.INITED)
    changeState(State.STARTED)
    info("Service: [" + getName + "] is started.")
  }
```
### 服务启动日志
对照Kyuubi服务启动日志，可以看到KyuubiServer依次启动的服务有：**KyuubiServer、OperationManager、SessionManager、BackendService和FrontendService**（至于为什么也启动了OperationManager和SessionManager服务，在后续解析BackendService的源码时会提及）。![Kyuubi服务启动日志](https://upload-images.jianshu.io/upload_images/7440793-119651d928659191.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
