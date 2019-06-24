---
layout:     post
title:      elasticsearch 请求处理流程 
subtitle:   详细分析 elasticsearch的http请求的处理流程
date:       2019-06-23
author:     chencoder
catalog: 	true
tags:
    - elasticsearch
---

elasticsearch 的增,删,改,查等操作都可以通过http接口完成。
下面来看以下elasticsearch的内置的HttpServer的启动过程，并且跟踪请求处理的流程。
note:本文基于elasticsearch 7.2版本. 

## HttpServer的启动过程

### TransportService 构造

TransportService 的构造是在Node构造函数中完成的。
Node.node()中代码如下:
```
            final RestController restController = actionModule.getRestController(); //操作对应的处理控制器
            final NetworkModule networkModule = new NetworkModule(settings, false, pluginsService.filterPlugins(NetworkPlugin.class),
                threadPool, bigArrays, pageCacheRecycler, circuitBreakerService, namedWriteableRegistry, xContentRegistry,
                networkService, restController); 
            ........

            final HttpServerTransport httpServerTransport = newHttpTransport(networkModule); //创建HttpServerTransport
```

* RestController 是请求操作的处理控制器，实现Dispatcher接口。
* NetworkModule 网络模块
* Transport transport 通过NetworkModule获取的Transport的具体实现。

### RestController
RestController 从ActionModule中获取。Rest请求的地址和处理函数通过
Node.node()
```
 logger.debug("initializing HTTP handlers ...");
            actionModule.initRestHandlers(() -> clusterService.state().nodes());
            logger.info("initialized");
```
注册。

例如get操作:
ActionModule.initRestHandlers
```
registerHandler.accept(new RestGetAction(settings, restController));
```

RestGetAction.RestGetAction()
```
 		controller.registerHandler(GET, "/{index}/_doc/{id}", this);
        controller.registerHandler(HEAD, "/{index}/_doc/{id}", this);
        // Deprecated typed endpoints.
        controller.registerHandler(GET, "/{index}/{type}/{id}", this);
        controller.registerHandler(HEAD, "/{index}/{type}/{id}", this);
```
指定根据Id获取doc的处理类是RestGetAction。

### NetworkModule

* 构造函数中通过遍历NetworkPlugin来注册HttpTransport, Transport和TransportInterceptor。
* networkModule.getTransportSupplier().get() 获取Transport。

```
 if (TRANSPORT_TYPE_SETTING.exists(settings)) {
            name = TRANSPORT_TYPE_SETTING.get(settings);
        } else {
            name = TRANSPORT_DEFAULT_TYPE_SETTING.get(settings);
        }
        final Supplier<Transport> factory = transportFactories.get(name);
        if (factory == null) {
            throw new IllegalStateException("Unsupported transport.type [" + name + "]");
        }
```
先从settings中查找TRANSPORT_TYPE_KEY的设置。settings 是elasticsearch启动流程中提到的，PluginService在扫描完成所有的module目录和plugin目录之后，收集所有的插件的additionalSettings的内容。
然后根据settings中的配置，获取到的Transport实现。

例如：Netty4Plugin，使用Netty实现的NetworkPlugin。

NetworkPlugin.additionalSettings
```
public Settings additionalSettings() {
        return Settings.builder()
        		//配置TRANSPORT_DEFAULT_TYPE_SETTING和TRANSPORT_TYPE_SETTING
                .put(NetworkModule.HTTP_DEFAULT_TYPE_SETTING.getKey(), NETTY_HTTP_TRANSPORT_NAME)
                .put(NetworkModule.TRANSPORT_DEFAULT_TYPE_SETTING.getKey(), NETTY_TRANSPORT_NAME)
                .build();
    }

```

NetworkPlugin.getTransports
```
  public Map<String, Supplier<Transport>> getTransports(Settings settings, ThreadPool threadPool, PageCacheRecycler pageCacheRecycler,
                                                          CircuitBreakerService circuitBreakerService,
                                                          NamedWriteableRegistry namedWriteableRegistry, NetworkService networkService) {
        return Collections.singletonMap(NETTY_TRANSPORT_NAME, () -> new Netty4Transport(settings, Version.CURRENT, threadPool,
            networkService, pageCacheRecycler, namedWriteableRegistry, circuitBreakerService));
    }

       public Map<String, Supplier<HttpServerTransport>> getHttpTransports(Settings settings, ThreadPool threadPool, BigArrays bigArrays,
                                                                        PageCacheRecycler pageCacheRecycler,
                                                                        CircuitBreakerService circuitBreakerService,
                                                                        NamedXContentRegistry xContentRegistry,
                                                                        NetworkService networkService,
                                                                        HttpServerTransport.Dispatcher dispatcher) {
        return Collections.singletonMap(NETTY_HTTP_TRANSPORT_NAME,
            () -> new Netty4HttpServerTransport(settings, networkService, bigArrays, threadPool, xContentRegistry, dispatcher));
    }
```



### HttpServerTransport
通过NetworkModule获取

```
networkModule.getHttpServerTransportSupplier().get();
```
上面的Netty4Plugin 实现了getHttpTransports接口，在NetworkModule构造函数中通过此接口把 Netty4HttpServerTransport 注册了HttpTransport工厂中。

所以在有Netty4Plugin 存在于module目录中时，使用的HttpServer是Netty4HttpServerTransport， 并且数据处理函数都注册在RestController中。


### HttpServerTransport 启动

HttpServerTransport 的启动是Node.start()中完成。
有多种不同HttpServerTransport实现类。
Netty4HttpServerTransport 使用netty实现HttpServer， 请求通过Netty4HttpRequestHandler来处理。Netty的使用可以参看Netty文档。

Netty4HttpRequestHandler.channelRead0()
```
if (request.decoderResult().isFailure()) {
                Throwable cause = request.decoderResult().cause();
                if (cause instanceof Error) {
                    ExceptionsHelper.maybeDieOnAnotherThread(cause);
                    serverTransport.incomingRequestError(httpRequest, channel, new Exception(cause));
                } else {
                    serverTransport.incomingRequestError(httpRequest, channel, (Exception) cause);
                }
            } else {
                serverTransport.incomingRequest(httpRequest, channel);
            }
```
然后 AbstractHttpServerTransport.incomingRequest调用dispatchRequest方法处理。

```
    void dispatchRequest(final RestRequest restRequest, final RestChannel channel, final Throwable badRequestCause) {
        final ThreadContext threadContext = threadPool.getThreadContext();
        try (ThreadContext.StoredContext ignore = threadContext.stashContext()) {
            if (badRequestCause != null) {
                dispatcher.dispatchBadRequest(restRequest, channel, threadContext, badRequestCause);
            } else {
                dispatcher.dispatchRequest(restRequest, channel, threadContext);
            }
        }
    }

```

dispatcher的实现类就是 RestController。通过匹配路劲，http请求方法等找到对应处理类，比如RestGetAction。

匹配Handler 类有点复杂。后续再研究。
