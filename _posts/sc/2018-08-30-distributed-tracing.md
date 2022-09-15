---
title: 使用Spring Cloud Sleuth、Zipkin、Kafka、Mysql实现分布式追踪
tagline: ""
category : Java
layout: post
tags : [Java, streamsets, realtime]
---


### tracing-zipkin-server

1. zipkin-server 实现

```
package com.tpc.zipkinserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.sleuth.zipkin.stream.EnableZipkinStreamServer;

@SpringBootApplication
@EnableZipkinStreamServer
public class ZipkinServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(ZipkinServerApplication.class, args);
	}
}
```
2. 想看表结构，所以后端存储这里用的mysql

```
zipkin.storage.type: mysql
```

3. kafka配置

```
spring.cloud.stream.kafka.binder.brokers: 1172.28.3.159:9092,172.28.3.158:9092
spring.cloud.stream.kafka.binder.zkNodes: 172.28.3.169:2181/kafkaroot
```
完整的配置文件

```
spring.application.name: tracing-zipkin-server

spring.cloud.stream.kafka.binder.brokers: 1172.28.3.159:9092,172.28.3.158:9092
spring.cloud.stream.kafka.binder.zkNodes: 172.28.3.169:2181/kafkaroot

spring.sleuth.enabled: false
zipkin.storage.type: mysql
spring.datasource.schema[0]: classpath:/mysql.sql
spring.datasource.url: jdbc:mysql://172.28.3.159:3306/zipkin?autoReconnect=true&useUnicode=true&characterEncoding=UTF-8&zeroDateTimeBehavior=convertToNull&useSSL=false
spring.datasource.username: canal
spring.datasource.password: canal
spring.datasource.driver-class-name: com.mysql.jdbc.Driver
spring.datasource.initialize: true
```

服务提供者provider与消费者consumer需要引入配置

```
spring.sleuth.sampler.percentage: 1.0
spring.cloud.stream.kafka.binder.brokers: 1172.28.3.159:9092,172.28.3.158:9092
spring.cloud.stream.kafka.binder.zkNodes: 172.28.3.169:2181/kafkaroot
```
[github-distributed-tracing](https://github.com/2pc/SpringCloudNotes/tree/master/distributed-tracing)

