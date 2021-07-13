---
title: 窥探Gin框架
date: 2021-06-20 14:25:35
tags:
    - Web
    - golang
categories: Gin
---

几个月前，为了学习Gin框架的基本使用（实际还有许多Golang其他的工具框架），从头搭建了一个简易的服务端+客户端 -- [一个简易的银行系统](https://github.com/Jiaget/simplebank)

作为练手项目，Gin有很多特性与功能都没有研究与使用过， 这次的学习希望可以从深度和广度两个方面，更加全面的一窥Gin框架，这个活跃在社区广受好评的web框架。

# 回顾 HTTP 服务开发的基本流程

- 开启TCP 监听
- 初始化一些 `handler` 处理具体的业务逻辑
- 将业务逻辑和相关的 `Method`, `URL` 绑定，对外暴露一些具体功能服务

# Gin 的几个重要模型：

- Engine 
    - 初始化一个 `gin` 对象实例，该对象实例中包含一些框架的基础功能， `日志`, `中间件设置`, `路由控制`, `handlercontext`

- Router:
    - 定义路由规则与条件，通过HTTP服务将具体路由注册到一个 `context` 实现的Handler 中

- Context:
    - 该模块是框架中重要的一环，它可以让我们在中间件中共享变量，管理整个流程，验证请求的json， 提供一个json的响应。

- Bind:
    - context 中，可以获取到请求的详细信息（ HTTP 的 head 和 body）
    - 但是我们需要根据不同的HTTP协议参数获取相应的格式化数据，来处理底层的业务逻辑，这需要我们使用 `bind` 相关的方法来解析 HTTP数据。