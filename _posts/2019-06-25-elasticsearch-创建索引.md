---
layout:     post
title:      elasticsearch 索引创建 
subtitle:   elasticsearch 创建索引的流程分析,以bulk接口为例
date:       2019-06-25
author:     chencoder
catalog: 	true
tags:
    - elasticsearch
---

## 前言
上回分析了elasticsearch的http请求的处理流程。最终的处理都通过一系列的Action类实现，这些类都继承了BaseRestHandler。

今天以bulk接口为例，分析以下索引的创建过程。




