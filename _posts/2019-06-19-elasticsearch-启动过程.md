---
layout:     post
title:      elasticsearch启动流程 
subtitle:   elasticsearch启动流程分析
date:       2019-06-19
author:     chencoder
catalog: 	true
tags:
    - elasticsearch
---

本文分析elasticsearch7.2的启动流程。

## 启动过程概览

elasticsearch的启动流程大概分为4个阶段
1. Elasticsearch 类解析 Command，加载各项配置。
2. 通过Bootstrap 类检查运行环境，初始化资源等。
3. 创建节点Node。
4. Bootstrap 启动节点和keepalive线程。

### Elasticsearch 类解析 Command，加载各项配置

Elasticsearch 类的main方法如下:
```
overrideDnsCachePolicyProperties();

System.setSecurityManager(new SecurityManager() {

   @Override
   public void checkPermission(Permission perm) {
       // grant all permissions so that we can later set the security manager to the one that we want
   }

});
LogConfigurator.registerErrorListener();
final Elasticsearch elasticsearch = new Elasticsearch();
int status = main(args, elasticsearch, Terminal.DEFAULT);
if (status != ExitCodes.OK) {
   exit(status);
}
```
1. overrideDnsCachePolicyProperties 覆盖dns缓存的默认配置。
2. System.setSecurityManager 创建 SecurityManager 安全管理器。
3. LogConfigurator.registerErrorListener() 注册侦听器
4. 创建Elasticsearch对象 
![Elasticsearch类结构](https://chenxh.github.io/img/elasticsearch-class.png  "图片title")
5. 执行ElasticSearch的main方法(继承于Command)

1. Elasticsearch() 构造方法中读取命令行中的参数。
2. Command.main 方法, 该方法给当前Runtime类添加一个hook线程，该线程作用是：当Runtime异常关闭时打印异常信息.
3. Command.mainWithoutErrorHandling 方法
4. EnvironmentAwareCommand.execute, 确保 es.path.data, es.path.home, es.path.logs 等参数已设置，否则从 System.properties 中读取.
5. EnvironmentAwareCommand.createEnv，读取config下的配置文件elasticsearch.yml内容，收集plugins，bin，lib，modules等目录下的文件信息
6. Elasticsearch.execute ，读取daemonize， pidFile，quiet 的值，并 确保配置的临时目录(temp)是有效目录。 然后执行Bootstrap初始化。



## Bootstrap初始化阶段

Bootstrap.init 的init函数。
1. 创建Bootstarp对象，创建keepalive线程，addShutdownHook() 在关系的时候关闭此线程。
2. loadSecureSettings， 加载 keystore 安全配置。
3. 根据已有的配置信息，创建一个Environment对象。
4. LogConfigurator log4j日志配置。
5. 检查java版本。
6. 检查pid文件是否存在，不存在则创建
7. checkLucene 检查lucene版本。
8.  设置未捕获异常的处理 Thread.setDefaultUncaughtExceptionHandler。在Thread ApI中提供了UncaughtExceptionHandle，它能检测出某个由于未捕获的异常而终结的情况。

### INSTANCE.setup()函数
9. spawner.spawnNativeControllers(environment), 遍历每个模块，生成本机控制类（native Controller）：读取modules文件夹下所有的文件夹中的模块信息，保存为一个 PluginInfo 对象，为合适的模块生成控制类，通过 Files.isRegularFile(spawnPath) 来判断。 尝试为给定模块生成控制器(native Controller)守护程序。 生成的进程将通过其stdin，stdout和stderr流保持与此JVM的连接，但对此包之外的代码不能使用对这些流的引用。 ?
10. initializeNatives 初始化本地资源.

* 检查用户是否为root用户，是则抛异常; 
* 尝试启用 系统调用过滤器 system call filter; ?
* Natives 设置参数等。Natives类是一个包装类，用于检查调用本机方法所需的类是否在启动时可用。如果它们不可用，则此类将避免调用加载这些类的代码。

11. initializeProbes 初始化进程，系统和jvm的监测类。
12. 添加结束钩子，结束关闭节点和spawner。
13. 使用 JarHell 检查重复的 jar 文件
14. 初始化 SecurityManager
15. 进入创建Node节点。

## 创建Node节点
1. 创建一个 NodeEnvironment 对象保存节点环境信息，如各种数据文件的路径。
2. 读取JVM信息。
3. 创建 PluginsService 对象，创建过程中会读取并加载所有的模块和插件。
4. 创建一个最终的 Environment 对象。
5. 创建线程池 ThreadPool 后面各类对象基本都是通过线程来提供服务，这个线程池可以管理各类线程。
6. 创建 节点客户端 NodeClient。
7. 创建各种服务类对象 ResourceWatcherService、NetworkService、ClusterService、IngestService、ClusterInfoService、UsageService、MonitorService、CircuitBreakerService、MetaStateService、IndicesService、MetaDataIndexUpgradeService、TemplateUpgradeService、TransportService、ResponseCollectorService、SearchTransportService、NodeService、SearchService、PersistentTasksClusterService
8. ModulesBuilder类加入各种模块 ScriptModule、AnalysisModule、SettingsModule、pluginModule、ClusterModule、IndicesModule、SearchModule、GatewayModule、RepositoriesModule、ActionModule、NetworkModule、DiscoveryModule。
9. guice 绑定依赖以及依赖注入。

## Bootstrap 启动
1. 通过 injector 获取各个类的对象，调用 start() 方法启动（实际进入各个类的中 doStart 方法）: LifecycleComponent、IndicesService、IndicesClusterStateService、SnapshotsService、SnapshotShardsService、RoutingService、SearchService、MonitorService、NodeConnectionsService、ResourceWatcherService、GatewayService、Discovery、TransportService。
* IndicesService：索引管理 
* IndicesClusterStateService：跨集群同步 
* SnapshotsService：负责创建快照 
* SnapshotShardsService：此服务在数据和主节点上运行，并控制这些节点上当前快照的分片。 它负责启动和停止分片级别快照 
* RoutingService：侦听集群状态，当它收到ClusterChangedEvent（集群改变事件）将验证集群状态，路由表可能会更新 
* SearchService：搜索服务 
* MonitorService：监控 
* NodeConnectionsService：此组件负责在节点添加到群集状态后连接到节点，并在删除它们时断开连接。 此外，它会定期检查所有连接是否仍处于打开状态，并在需要时还原它们。 请注意，如果节点断开/不响应ping，则此组件不负责从群集中删除节点。 这是由NodesFaultDetection完成的。 主故障检测由链接MasterFaultDetection完成。 
* ResourceWatcherService：通用资源观察器服务 
* GatewayService：网关

2. 集群发现，加入集群
3. 启动 HttpServerTransport, 绑定服务端口。
4. 启动保活线程 keepAliveThread.start 进行心跳检测。

## next
* PluginService
* ThreadPool
* 集群选主流程。
* 读写数据流程
* 。。。。。。







