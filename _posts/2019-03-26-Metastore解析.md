---
layout:     post
title:      Metastore解析
subtitle:   Thrift服务端解析
date:       2019-02-26
author:     jianguotian
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - 原创
    - Metastore
    - Thrift服务
    - DataNucleus
    - JDO
---

![Metastore Internal](https://upload-images.jianshu.io/upload_images/7440793-42f2711a226843a7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### HiveMetastore
HiveMetastore是Metastore的Thrift程序，Thrift文件为hive_metastore.thrift，Thrift服务接口文件为ThriftHiveMetastore.java。
首先，要理解Thrift服务模型和概念，对应服务模型，再理解下面的Metastore Thrift服务逻辑。
![Thrift通信协议栈](https://upload-images.jianshu.io/upload_images/7440793-5cfe0ea5fcee6f82.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
Thrift协议栈简介：
- **底层IO模块**：负责实际的数据传输，包括Socket，文件，或者压缩数据流等。
- **TTransport**：负责以字节流方式发送和接收Message，是底层IO模块在Thrift框架中的实现，每一个底层IO模块都会有一个对应TTransport来负责Thrift的字节流(Byte Stream)数据在该IO模块上的传输。例如TSocket对应Socket传输，TFileTransport对应文件传输。
- **TProtocol**：主要负责结构化数据组装成Message，或者从Message结构中读出结构化数据。TProtocol将一个有类型的数据转化为字节流以交给TTransport进行传输，或者从TTransport中读取一定长度的字节数据转化为特定类型的数据。如int32会被TBinaryProtocol Encode为一个四字节的字节数据，或者TBinaryProtocol从TTransport中取出四个字节的数据Decode为int32。
- **TServer**：负责接收Client的请求，并将请求转发到Processor进行处理。TServer主要任务就是高效的接受Client的请求，特别是在高并发请求的情况下快速完成请求。
- **Processor(或者TProcessor)**：负责对Client的请求做出相应，包括RPC请求转发，调用参数解析和用户逻辑调用，返回值写回等处理步骤。Processor是服务器端从Thrift框架转入用户逻辑的关键流程。Processor同时也负责向Message结构中写入数据或者读出数据。 
注意下面代码中将handler封装进processor，我的理解handler是metastore服务逻辑核Thrift服务模型的一个桥梁。
```java
  public static void startMetaStore(int port, HadoopThriftAuthBridge bridge,
      HiveConf conf, Lock startLock, Condition startCondition,
      AtomicBoolean startedServing) throws Throwable {
    try {
      isMetaStoreRemote = true;
      // Server will create new threads up to max as necessary. After an idle
      // period, it will destroy threads to keep the number of threads in the
      // pool to min.
      ......
      TProcessor processor;
      TTransportFactory transFactory;
      final TProtocolFactory protocolFactory;
      final TProtocolFactory inputProtoFactory;
      if (useCompactProtocol) {
        ......
      } else {
        protocolFactory = new TBinaryProtocol.Factory();
        inputProtoFactory = new TBinaryProtocol.Factory(true, true, maxMessageSize, maxMessageSize);
      }
      HMSHandler baseHandler = new HiveMetaStore.HMSHandler("new db based metaserver", conf,
          false);
      IHMSHandler handler = newRetryingHMSHandler(baseHandler, conf);
      TServerSocket serverSocket  = null;

      if (useSasl) {
        // we are in secure mode.
        ......
        saslServer = bridge.createServer(
            conf.getVar(HiveConf.ConfVars.METASTORE_KERBEROS_KEYTAB_FILE),
            conf.getVar(HiveConf.ConfVars.METASTORE_KERBEROS_PRINCIPAL));
        // start delegation token manager
        saslServer.startDelegationTokenSecretManager(conf, baseHandler, ServerMode.METASTORE);
        transFactory = saslServer.createTransportFactory(
                MetaStoreUtils.getMetaStoreSaslProperties(conf));
        //将服务逻辑通过handler封装进processor
        processor = saslServer.wrapProcessor(
          new ThriftHiveMetastore.Processor<IHMSHandler>(handler));
        serverSocket = HiveAuthUtils.getServerSocket(null, port);

        LOG.info("Starting DB backed MetaStore Server in Secure Mode");
      } else {
        // we are in unsecure mode
        ......
      if (tcpKeepAlive) {
        serverSocket = new TServerSocketKeepAlive(serverSocket);
      }

      TThreadPoolServer.Args args = new TThreadPoolServer.Args(serverSocket)
          .processor(processor)
          .transportFactory(transFactory)
          .protocolFactory(protocolFactory)
          .inputProtocolFactory(inputProtoFactory)
          .minWorkerThreads(minWorkerThreads)
          .maxWorkerThreads(maxWorkerThreads);

      TServer tServer = new TThreadPoolServer(args);
      TServerEventHandler tServerEventHandler = new TServerEventHandler() {
         ......
      };

      tServer.setServerEventHandler(tServerEventHandler);
      ......
      tServer.serve();
    } catch (Throwable x) {
      ......
    }
```
### HMSHandler
HMSHandler实现了IMSHandler的接口，是HiveMetastore的内部类，它定义了对元数据操作和获取信息的各种方法，如下图所示。
![HMSHandler方法](https://upload-images.jianshu.io/upload_images/7440793-3f560286466f9ff0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
通过动态代理的方式调用HMSHandler的各种方法。动态代理类为RetryingHMSHandler。
```java
  public static IHMSHandler getProxy(HiveConf hiveConf, IHMSHandler baseHandler, boolean local)
      throws MetaException {

    RetryingHMSHandler handler = new RetryingHMSHandler(hiveConf, baseHandler, local);

    return (IHMSHandler) Proxy.newProxyInstance(
      RetryingHMSHandler.class.getClassLoader(),
      new Class[] { IHMSHandler.class }, handler);
  }
```
### ObjectStore
该类是应用程序逻辑与包含对象的database store之间的接口，定义了操作元数据表的各种方法，如下图所示（上文中HMSHandler也定义了操作元数据表的各种方法，实际上最终会调用ObjectStore中的方法）。
![ObjectStore](https://upload-images.jianshu.io/upload_images/7440793-27ff244c3e0e642c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
ObjectStore是RawStore的实现类，通过动态代理的方式调用ObjectStore的各种方法。动态代理类为RawStoreProxy。
```java
  public static RawStore getProxy(HiveConf hiveConf, Configuration conf, String rawStoreClassName,
      int id) throws MetaException {

    Class<? extends RawStore> baseClass = (Class<? extends RawStore>) MetaStoreUtils.getClass(
        rawStoreClassName);

    RawStoreProxy handler = new RawStoreProxy(hiveConf, conf, baseClass, id);
    // Look for interfaces on both the class and all base classes.
    return (RawStore) Proxy.newProxyInstance(RawStoreProxy.class.getClassLoader(),
        getAllInterfaces(baseClass), handler);
  }
```
### DataNucleus
DataNucleus实现了JDO规范，JDO(Java Data Object )是Java对象持久化的规范。了解Hibernate或者MyBatis的同学，对于对象持久化或者ORM（Object Relational Mapping，对象关系映射）应该不会陌生。DataNucleus做的正是这样的事情。
Metastore中实现DataNucleus比较直观的两部分是M*类和package.jdo，package.jdo定义了对象和元数据表的映射，M*类则表示映射元数据表的类。详细内容如下图。
#### M* model类
![M*](https://upload-images.jianshu.io/upload_images/7440793-fc21f48bfffe2b83.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![Mdatabase类示意](https://upload-images.jianshu.io/upload_images/7440793-036c2c550229f893.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
#### package.jdo
如下图所示，DBS是MySQL元数据中的一张表，MDatabase则表示对应的model类，MDatabase同样定义了相应的属性和方法。
![package.jdo](https://upload-images.jianshu.io/upload_images/7440793-be8140d5789cf379.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### MySQL元数据示例
![MySQL中元数据表](https://upload-images.jianshu.io/upload_images/7440793-9bc120e64d25fd87.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![DBS元数据表](https://upload-images.jianshu.io/upload_images/7440793-6ad93cd27d5bc06e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
