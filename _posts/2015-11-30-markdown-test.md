---
layout: post
title: MarkDown
tags:  [MarkDown]
categories: [MarkDown]
author: mingtian
excerpt: "MarkDown"
---

# MarkDown 语法说明

## 一、列表
### 无序列表
 * 1
 * 2
 * 3
 
### 有序列表
 1. 1
 2. 2
 
## 二、 引用
 
 > 这是引用,  
 引用其他的资源
 
## 三、代码框
 
~~~
 registerOrLookupEndpoint(name: String, endpointCreator: => RpcEndpoint):
      RpcEndpointRef = {
      if (isDriver) {
        logInfo("Registering " + name)
        rpcEnv.setupEndpoint(name, endpointCreator)
      } else {
        RpcUtils.makeDriverRef(name, conf, rpcEnv)
      }
~~~

## 四、图片 
![baidu](http://www.baidu.com/img/bdlogo.gif "百度logo")




