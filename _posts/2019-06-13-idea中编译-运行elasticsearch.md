---
layout:     post
title:      在idea中编译、运行elasticsearch 
subtitle:   在idea中编译、运行elasticsearch
date:       2019-06-13
author:     chencoder
catalog: 	true
tags:
    - elasticsearch
---
在idea中编译和运行elasticsearch7.2

## 环境
* jdk： jdk12
* 操作系统： windows10
* elasticsearch： 7.2

## 下载elasticsearch

克隆代码:

```
git clone https://github.com/elastic/elasticsearch.git
```
切换版本

```
git checkout 7.2
```

## 配置idea环境
设置环境变量
JAVA_HOME=F:\files\Java\jdk-12.0.1

在源码目录运行以下命令，生成idea工程，时间有点长。
```
./gradlew idea
```

## 导入idea工程
1. 打开idea，import 工程， 选择源码目录，
2. 选择gradle， 设置gradle JVM 为jdk-12
3. 等待编译，构建工程结束。

## 在idea中启动elasticsearch

1. 复制一套可用配置到server目录下src/main/resources 下
2. 添加run/debug，配置如下
* main class : org.elasticsearch.bootstrap.Elasticsearch
* VM options: -Xms1g -Xmx1g -Des.path.conf={源码目录}\server\src\main\resources\config -Dlog4j2.disable.jmx=true -Des.path.home={源码目录}\server\src\main\resources\config
* working directory: {源码目录}\
3. 下载elasticsearch的发行包，我下载的7.1版本，复制modules/transport-netty4 文件到
	{源码目录}\server\src\main\resources\config\modules\transport-netty4
4. 编译modules:transport-netty4 ,覆盖复制
	transport-netty4-client-7.2.0-SNAPSHOT.jar,
	plugin-descriptor.properties,
	plugin-security.policy	
到 {源码目录}\server\src\main\resources\config\modules\transport-netty4

5. 修改jdk 的java 的 conf/sercurity/java.policy 文件，添加以下配置
```
    permission java.lang.RuntimePermission "createClassLoader"; 
    permission java.lang.RuntimePermission "getClassLoader"; 
    permission java.lang.RuntimePermission "accessDeclaredMembers";
    permission java.lang.reflect.ReflectPermission "suppressAccessChecks";
    permission java.util.PropertyPermission "*", "read";
    permission java.util.PropertyPermission "*", "write";
```
6. 启动 org.elasticsearch.bootstrap.Bootstrap 的main 函数
