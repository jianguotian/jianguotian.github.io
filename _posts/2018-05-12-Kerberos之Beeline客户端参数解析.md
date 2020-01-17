---
layout:     post
title:      Kerberos之Beeline客户端参数解析
author:     jianguotian
header-style: text
catalog: true
tags:
    - Kyuubi
---
在Kerberos认证体系下，Beeline客户端连接HiveServer2的参数一般是这样的：beeline -u "jdbc:hive2://<URL>:10000/default;**principal=hive/_HOST@<realm>**;hive.server2.proxy.user=<username>"。在初学Kerberos的时候，关于连接参数中的principal始终是百思不得其解，为什么一定要写成**principal=hive/_HOST@<realm>**这种形式？hive换成其他的可不可以？中间的这个_HOST又是怎么确定的？这个principal到底是用户的principal还是服务的principal？  
这些问题的答案可能对于大部分人来说是显而易见的，然而对于我这只小菜就来说就很容易想不明白。而我对于感兴趣的问题又喜欢一追到底，有点钻牛角尖儿。  
下面我来说一下针对上述问题，自己一些浅显的理解。可能不对，欢迎大家批评指正。
##HiveConnection的openTransport()方法解析
接上文[Kyuubi服务源码解析：FrontendService](https://www.jianshu.com/p/4e733b339f58),**文末**提到了HiveConnection的openTransport()方法，来看一下openTransport()做了什么。
###HiveConnection.java：
- **openTransport()方法**
```
  private void openTransport() throws Exception {
    assumeSubject =
        JdbcConnectionParams.AUTH_KERBEROS_AUTH_TYPE_FROM_SUBJECT.equals(sessConfMap
            .get(JdbcConnectionParams.AUTH_KERBEROS_AUTH_TYPE));
    //一般来说是调用createBinaryTransport()方法
    transport = isHttpTransportMode() ? createHttpTransport() : createBinaryTransport();
    if (!transport.isOpen()) {
      transport.open();
    }
    logZkDiscoveryMessage("Connected to " + connParams.getHost() + ":" + connParams.getPort());
  }
```
- **createBinaryTransport()方法**
```
  /**
   * Create transport per the connection options
   * Supported transport options are:
   *   - SASL based transports over
   *      + Kerberos
   *      + Delegation token
   *      + SSL
   *      + non-SSL
   *   - Raw (non-SASL) socket
   *
   *   Kerberos and Delegation token supports SASL QOP configurations
   * @throws SQLException, TTransportException
   */
  private TTransport createBinaryTransport() throws SQLException, TTransportException {
    try {
      TTransport socketTransport = createUnderlyingTransport();
      // handle secure connection if specified
      if (!JdbcConnectionParams.AUTH_SIMPLE.equals(sessConfMap.get(JdbcConnectionParams.AUTH_TYPE))) {
        // If Kerberos
        Map<String, String> saslProps = new HashMap<String, String>();
        SaslQOP saslQOP = SaslQOP.AUTH;
        if (sessConfMap.containsKey(JdbcConnectionParams.AUTH_QOP)) {
          ***省略部分代码***
        } else {
          // If the client did not specify qop then just negotiate the one supported by server
          saslProps.put(Sasl.QOP, "auth-conf,auth-int,auth");
        }
        saslProps.put(Sasl.SERVER_AUTH, "true");
        //即JDBC连接参数中“principal=hive/_HOST@realms”
        if (sessConfMap.containsKey(JdbcConnectionParams.AUTH_PRINCIPAL)) {
          transport = KerberosSaslHelper.getKerberosTransport(
              //这里的principal获取的是JDBC连接参数中的principal
              sessConfMap.get(JdbcConnectionParams.AUTH_PRINCIPAL), host,
              socketTransport, saslProps, assumeSubject);
        } else {
          // If there's a delegation token available then use token based connection
          String tokenStr = getClientDelegationToken(sessConfMap);
          if (tokenStr != null) {
            transport = KerberosSaslHelper.getTokenTransport(tokenStr,
                host, socketTransport, saslProps);
          } else {
            // we are using PLAIN Sasl connection with user/password
            String userName = getUserName();
            String passwd = getPassword();
            // Overlay the SASL transport on top of the base socket transport (SSL or non-SSL)
            transport = PlainSaslHelper.getPlainTransport(userName, passwd, socketTransport);
          }
        }
      } else {
        // Raw socket connection (non-sasl)
        transport = socketTransport;
      }
    } catch (SaslException e) {
      throw new SQLException("Could not create secure connection to "
          + jdbcUriString + ": " + e.getMessage(), " 08S01", e);
    }
    return transport;
  }
```
该方法内部调用了KerberosSaslHelper.getKerberosTransport()方法。
**KerberosSaslHelper.getKerberosTransport()**中的调用关系如下所示：
![KerberosSaslHelper.getKerberosTransport()方法](https://upload-images.jianshu.io/upload_images/7440793-0663ffbe87f970e4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**createClientTransport()方法定义：**
![createClientTransport方法](https://upload-images.jianshu.io/upload_images/7440793-3252553747c5f4ff.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
&emsp;&emsp;从createClientTransport()方法的注释中，可以看出principalConfig（也就是JDBC连接参数中的“principal=hive/_HOST@realms”）**指的是服务器的principal，就是JDBC客户端请求服务的principal**，当然这个服务的principal是在hive-site.xml中已经配置好的（见如下两张图）。那么这里的**_HOST就应为请求的服务所在机器的hostname**，因为如果principal中的_HOST对应的机器都没有开启要请求的服务，又怎么访问服务呢？**所以Beeline客户端连接参数中principal的_HOST所指的服务器一定要确认开启了要请求的服务**。
![hive-site.xml中metastore服务Kerberos配置](https://upload-images.jianshu.io/upload_images/7440793-915d77a1d2da943d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![hive-site.xml中hiveserver2服务Kerberos配置](https://upload-images.jianshu.io/upload_images/7440793-560041c060b75d9c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
上面两张图中metastore和hiveserver2服务的名字都叫hive，难免有些困惑，这个我后续会调研清楚。  
上面啰嗦了一大堆，可能理解的不对，但初衷还是要讲明白JDBC连接参数中principal的真正含义。这对于在HiveServer2或者Kyuubi业务场景下理解Kerberos认证流程至关重要。
- **HadoopThriftAuthBridge.Client的createClientTransport()方法**
```
    public TTransport createClientTransport(
        String principalConfig, String host,
        String methodStr, String tokenStrForm, final TTransport underlyingTransport,
        final Map<String, String> saslProps) throws IOException {
      final AuthMethod method = AuthMethod.valueOf(AuthMethod.class, methodStr);

      TTransport saslTransport = null;
      switch (method) {
      case DIGEST:
        ***省略部分代码***

      case KERBEROS:
        String serverPrincipal = SecurityUtil.getServerPrincipal(principalConfig, host);
        final String names[] = SaslRpcServer.splitKerberosName(serverPrincipal);
        if (names.length != 3) {
          throw new IOException(
              "Kerberos principal name does NOT have the expected hostname part: "
                  + serverPrincipal);
        }
        try {
          return UserGroupInformation.getCurrentUser().doAs(
              new PrivilegedExceptionAction<TUGIAssumingTransport>() {
                @Override
                public TUGIAssumingTransport run() throws IOException {
                  TTransport saslTransport = new TSaslClientTransport(
                    method.getMechanismName(),
                    null,
                    names[0], names[1],
                    saslProps, null,
                    underlyingTransport);
                  return new TUGIAssumingTransport(saslTransport, UserGroupInformation.getCurrentUser());
                }
              });
        } catch (InterruptedException | SaslException se) {
          throw new IOException("Could not instantiate SASL transport", se);
        }

      default:
        throw new IOException("Unsupported authentication method: " + method);
      }
    }
```