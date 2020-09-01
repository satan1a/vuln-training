# weblogic-cve_2017_10271

Weblogic XMLDecoder反序化

## 漏洞描述

Oracle Fusion Middleware中的Oracle WebLogic Server组件的**WLS Security子组件**存在安全漏洞。使用精心构造的xml数据可能造成**任意代码执行**，攻击者只需要发送精心构造的**xml恶意数据**，就可以拿到目标服务器的权限。

wls-wsat路径： /root/Oracle/Middleware//user_projects/domains/base_domain/servers/AdminServer/tmp/_WL_internal/wls-wsat/

## 影响版本

Oracle WebLogic Server 10.3.6.0.0版本
Oracle WebLogic Server 12.1.3.0.0版本
Oracle WebLogic Server 12.2.1.1.0版本

## 漏洞利用

1、访问页面

![Snipaste_2020-09-01_22-04-37.png](http://ww1.sinaimg.cn/large/006X0pjBly1gibi3c15nnj31fg095t99.jpg)

2、初步判断

访问：http://127.0.0.1:7001/wls-wsat/CoordinatorPortType11，存在下图则说明可能存在漏洞。

![image-20200831005437218.png](http://ww1.sinaimg.cn/large/006X0pjBly1gi9c1f2ny5j31dh064gm9.jpg)

3、构造post包，执行python验证脚本

![image-20200831012502598.png](http://ww1.sinaimg.cn/large/006X0pjBly1gi9cm9c2hrj30oo01v0st.jpg)

## 漏洞分析

此次漏洞出现在wls-wsat.war中，此组件使用了weblogic自带的webservices处理程序来处理SOAP（一个基于xml格式的web交互协议）请求首先在weblogic.wsee.jaxws.workcontext.WorkContextServerTube类中获取**XML数据**最终传递给**XMLDecoder来解析**，其解析XML的调用链为：

```
1、weblogic.wsee.jaxws.workcontext.WorkContextServerTube.processRequest
// 获取到localHeader1后传递给readHeaderOld方法，其内容为<work:WorkContext>所包裹的数据，然后继续跟进
2、weblogic.wsee.jaxws.workcontext.WorkContextTube.readHeaderOld
// 在此方法中实例化了WorkContextXmlInputAdapter类，并且将获取到的XML格式的序列化数据传递到此类的构造方法中，最后通过XMLDecoder来进行反序列化操作
// public WorkContextXmlInputAdapter(InputStream paramInputStream){this.xmlDecoder=new XMLDecoder(paramInputStream)}
3、weblogic.wsee.workarea.WorkContextXmlInputAdapter
```

## 修复建议

1.临时解决方案
根据攻击者利用POC分析发现所利用的为wls-wsat组件的CoordinatorPortType接口，若Weblogic服务器集群中未应用此组件，建议临时备份后将此组件删除，当形成防护能力后，再进行恢复。

根据实际环境路径，删除WebLogic wls-wsat组件：

rm -f   /home/WebLogic/Oracle/Middleware/wlserver_10.3/server/lib/wls-wsat.war

rm -f   /home/WebLogic/Oracle/Middleware/user_projects/domains/base_domain/servers/AdminServer/tmp/.internal/wls-wsat.war

rm -rf /home/WebLogic/Oracle/Middleware/user_projects/domains/base_domain/servers/AdminServer/tmp/_WL_internal/wls-wsat

重启Weblogic域控制器服务:

DOMAIN_NAME/bin/stopWeblogic.sh           #停止服务

DOMAIN_NAME/bin/startManagedWebLogic.sh    #启动服务

删除以上文件之后，需重启WebLogic。确认http://weblogic_ip/wls-wsat/ 是否为404页面。

2.官方补丁修复
前往Oracle官网下载10月份所提供的安全补丁

http://www.oracle.com/technetwork/security-advisory/cpuoct2017-3236626.html

升级过程可参考：

http://blog.csdn.net/qqlifu/article/details/49423839