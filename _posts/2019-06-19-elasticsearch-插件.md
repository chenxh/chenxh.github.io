---
layout:     post
title:      elasticsearch 插件
subtitle:   elasticsearch 插件介绍和加载源码分析
date:       2019-06-22
author:     chencoder
catalog: 	true
tags:
    - elasticsearch
---

## elasticsearch plugins 介绍
plugin是一种以自定义方式增强核心Elasticsearch功能的方法。 功能包括添加自定义映射类型，自定义分析器，本机脚本，自定义发现等。

plugin包含JAR文件，但也可能包含脚本和配置文件，并且必须安装在群集中的每个节点上。 安装后，必须重新启动每个节点才能看到插件。


note: 不再支持包含HTML,CSS和JAVASCRIPT的插件。


* 核心插件是elasticsearch项目提供的官方插件,都是开源项目。这些插件会跟着elasticsearch版本升级进行升级,总能匹配到对应版本的elasticsearch,这些插件是有官方团队和社区成员共同开发的。官方插件地址： https://github.com/elastic/elasticsearch/tree/master/plugins


* 第三方插件是有开发者或者第三方组织自主开发便于扩展elasticsearch功能,它们拥有自己的许可协议,在使用它们之前需要清除插件的使用协议,不一定随着elasticsearch版本升级, 所以使用者自行辨别插件和es的兼容性


### 插件安装
1. 命令行安装
```
sudo bin/elasticsearch-plugin install [plugin_name]
```
安装中文elasticsearch自带的中文分词器
```
$ cd /opt/environment/elasticsearch-6.4.0
$ sudo bin/elasticsearch-plugin install analysis-smartcn
$ sudo systemctl restart elasticsearch.service
```
2. URL安装
```
$ cd /opt/environment/elasticsearch-6.4.0
$ sudo bin/elasticsearch-plugin install https://artifacts.elastic.co/downloads/elasticsearch-plugins/analysis-smartcn/analysis-smartcn-6.4.0.zip
$ sudo systemctl restart elasticsearch.service
```
3. 离线安装

```
$ sudo wget -P /opt/packages https://artifacts.elastic.co/downloads/elasticsearch-plugins/analysis-smartcn/analysis-smartcn-6.4.0.zip
$ sudo tar -zxvf /opt/packages/analysis-smartcn-6.4.0.zip -C /opt/apps/elasticsearch-6.4.0/plugins
$ sudo systemctl restart elasticsearch.service
```

## PluginsService 加载插件
PluginsService在构造方法中加载模块和插件
构造参数:
* Settings settings ElasticSearch启动时的配置
* Path configPath config配置文件的目录
* Path modulesDirectory 
* Path pluginsDirectory
* Collection<Class<? extends Plugin>> classpathPlugins 在classpath路径中的插件，是Plugin类的子类集合，有这种情况的出现主要是为了在测试和使用transport clients的情况下加载插件。

### 加载classpathPlugins中的插件

遍历classpathPlugins的值中的Plugin实现类对象，通过反射Plugin的实现类，调用方法getConstructors()得到Plugin的子类的构造函数，得到构造函数后，对构造函数进行一系列的检查：

* 自定义实现的Plugin子类必须有public的构造函数
* 有且只能有一个构造函数
* 接受参数不能大于两个，且第一个为Settings类型，第二个为Path类型。

由此可以知道，想自己实现ElasticSearch的插件，就必须继承Plugin类，定义一个构造函数，根据要实现插件的类型实现不同的接口，比如SearchPlugin，ScriptPlugin，RepositoryPlugin。

### 加载modulesDirectory

接下来遍历modules路径中各个module的plugin-descriptor.properties文件，取出文件中的如下属性：
name,
description,
version,
elasticsearch.version,
java.version,
has.native.controller:该插件是否需要本地控制器
requires.keystore:该插件是否需要ElasticSearch创建秘钥库

构造出各个module的PluginInfo对象，然后遍历出各个module下面的jar包的路径，然后把PluginInfo和jar路径封装到Bundle对象中。

### 加载pluginsDirectory

首先调用checkForFailedPluginRemovals()方法，遍历pluginsDirectory路径中所有的包含.removing-字符的文件，抛出应该remove该插件的异常IllegalStateException。
然后和加载modulesDirectory一样遍历plugins目录，构造Bundle对象。

### 加载所有插件
加载modulesDirectory和pluginsDirectory之后，调用loadBundles方法加载所有Bindle对象的信息。主要作用：

* 检查jar包重复。
* 加载plugin的类。
* 通过反射plugin的实现类，调用方法getConstructors()得到Plugin的子类的构造函数，构造plugin对象。

## PluginsService

PluginsService 在构造Node的时候创建。 在加载完plugin之后，执行 pluginsService.updatedSettings()，将PluginsService类在构造时从modules和plugins路径中加载的plugin对象取出遍历，依次调用各个plugin的additionalSettings()方法。然后put更新Settings对象，达到了更新Settings对象的目的。

然后PluginsService 在Node的创建中多次出现，主要是获取到需要的Plugin对象，用来构建相应的模块。
