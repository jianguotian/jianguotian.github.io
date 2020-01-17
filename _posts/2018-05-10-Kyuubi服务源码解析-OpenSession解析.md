---
layout:     post
title:      Kyuubi服务源码解析
subtitle:   OpenSession解析
author:     jianguotian
header-style: text
catalog: true
tags:
    - Kyuubi
---
&emsp;&emsp;在[Kyuubi服务源码解析：FrontendService](https://www.jianshu.com/p/4e733b339f58)一文解析HiveConnection的构造函数时，有一行代码是`openSession()`，现在来解析一下。
##Client端OpenSession解析
###HiveConnection.java：
- openSession()方法
```
  private void openSession() throws SQLException {
    TOpenSessionReq openReq = new TOpenSessionReq();
    ***省略部分代码***
    try {
      //Client远程调用Server端的OpenSession方法
      TOpenSessionResp openResp = client.OpenSession(openReq);

      // validate connection
      Utils.verifySuccess(openResp.getStatus());
      if (!supportedProtocols.contains(openResp.getServerProtocolVersion())) {
        throw new TException("Unsupported Hive2 protocol");
      }
      protocol = openResp.getServerProtocolVersion();
      sessHandle = openResp.getSessionHandle();
      ***省略部分代码***
    } catch (TException e) {
      LOG.error("Error opening session", e);
      throw new SQLException("Could not establish connection to "
          + jdbcUriString + ": " + e.getMessage(), " 08S01", e);
    }
    isClosed = false;
  }
```
&emsp;&emsp;`TOpenSessionResp openResp = client.OpenSession(openReq);`
&emsp;&emsp;这句代码中，client的类型是**TCLIService.Iface**，TOpenSessionResp和TCLIService都是由Thrift文件生成的类：
![](https://upload-images.jianshu.io/upload_images/7440793-6ddc2196862108e3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
&emsp;&emsp;Thrift是跨语言的RPC框架，也就是能够远程调用函数。
&emsp;&emsp;`client.OpenSession(openReq)`实际上调用的是Server端的OpenSession()方法，这个Server端也就是Kyuubi(当然也可以是Hiveserver2)。
&emsp;&emsp;接下来我们看一下Server端的OpenSession方法。
---
##Server端OpenSession解析
###FrontendService.scala:
&emsp;&emsp;再看一下FrontendService的类声明：
```scala
private[kyuubi] class FrontendService private(name: String, beService: BackendService)
  extends AbstractService(name) with TCLIService.Iface with Runnable with Logging 
```
&emsp;&emsp;FrontendService**实现了TCLIService.Iface接口**，上文`client.OpenSession(openReq)`这个**client的类型也是TCLIService.Iface**。然后看一下FrontendService的OpenSession方法：
- OpenSession()方法
```scala
  override def OpenSession(req: TOpenSessionReq): TOpenSessionResp = {
    info("Client protocol version: " + req.getClient_protocol)
    val resp = new TOpenSessionResp
    try {
      val sessionHandle = getSessionHandle(req, resp)
      resp.setSessionHandle(sessionHandle.toTSessionHandle)
      // resp.setConfiguration(new Map[String, String]())
      resp.setStatus(OK_STATUS)
      val context = currentServerContext.get
        .asInstanceOf[FrontendService#FeServiceServerContext]
      if (context != null) {
        context.setSessionHandle(sessionHandle)
      }
    } catch {
      case e: Exception =>
        warn("Error opening session: ", e)
        resp.setStatus(KyuubiSQLException.toTStatus(e))
    }
    resp
  }
```
&emsp;&emsp;实际上FrontendService重写了TCLIService.Iface的OpenSession方法，JDBC client调用的也正是Server端这个重写的OpenSession方法。
&emsp;&emsp;关于Thrift远程调用不理解的同学可以参考这篇博文：[thrift远程调用示例](https://blog.csdn.net/micro_hz/article/details/51993246)。