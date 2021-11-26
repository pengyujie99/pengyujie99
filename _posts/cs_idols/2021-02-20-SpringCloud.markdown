---
layout:     post
title:      "微服务了解"
subtitle:   "仅限了解还未开始学习"
date:       2021-02-20 12:00:00
author:     "Pengyujie"
header-img: "img/tag-bg.jpg"
tags:
    - SpringBoot
    - SpringCloud
    - 微服务 
---
>SpringCloud、微服务，通过观看视频之后的个人理解。

框架：spring （简化开发而生），springboot（在spring基础上更为简化），
springcloud的就是业务模块化。  

微服务架构：
通常而言，微服务架构就是根据业务拆分成一个个服务，彻底解耦。每一个服务提供单个业务功能，能够自行的单独启动或者销毁，有自己独立的数据库。  

微服务：
就是上面拆分之后的一个个小小的模块、业务。  

pringCloud：
是基于springboot提供的一套微服务解决方案。是各个微服务架构落地技术的集合体，也叫微服务全家桶。
是来管理协调微服务。为各个服务之间提供：配置管理、服务发现、断路由、路由、微代理、分布式会话等等集成服务。  

springboot:
是用来构建微服务，是用来发开一个个微服务。

> 微服务是一种生态，实现模块化开发，它并不是一个技术，而是由多个技术掺杂起来的。

~~~
微服务需要解决的四个问题

​	1、服务很多，客户端如何访问？

​	2、服务很多，服务之间如何通信？

​	3、服务很多，如何管理？

​	4、服务挂了怎么办？

1.API
2.HTTP RPC
3.注册和发现
4.熔断机制


三套解决方案：
	
	1.Spring Cloud NetFlix	一站式解决方案
	
		api网关，zuul组件
		通信：Feign-- HTTPClinet --HTTP通信方式 同步 阻塞
		服务注册发现：Eureka
		熔断机制：Hystrix
	
	
	
    
    2.Apache Dubbo Zookeeper	半自动 还需要整合其他的
    API:没有 需要找第三方组件或者自己实现
    通信：Dubbo --RPC通信
    服务注册发现：Zookeeper
    熔断机制：没有 可以借助Hystrix
    
    
    
    
    
    3.Spring Cloud Alibaba	最新的一站式解决方案
	







~~~

未完待续

