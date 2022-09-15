---
title: Building Zeppelin from source
tagline: ""
category : zeppelin
layout: post
tags : [zeppelin]
---


编译，这里使用的zeppelin-0.8.0的zip包,如果直接使用0.8.0的all包注意jdk版本不能低于jdk1.8.0_144 ，反编译javax.ws是基于这个版本编译的

```
cd zeppelin-0.8.0
mvn clean  package  -Pbuild-distr  -DskipTests -Denforcer.skip=true -Dcheckstyle.skip=true -DskipRat=true
```
注意修改npm的源,也可以修改为ali的源

```
npm config set registry "http://registry.npmjs.org/"

```

编译内部基于0.7.0修改版本编译还碰到个问题

```
[INFO] bower jspdf#1.0.272 || 1.1.239          progress Receiving objects:  94% (2493/2652), 11.64 MiB | 1.01 MiB/s
[ERROR] bower filesaver.js#*                     ECMDERR Failed to execute "git ls-remote --tags --heads https://github.com/carlos-algms/FileSaver.js.git", exit code of #128 error: The requested URL returned error: 403 Forbidden while accessing https://github.com/carlos-algms/FileSaver.js.git/info/refs  fatal: HTTP request failed
[ERROR]
[ERROR] Additional error details:
[ERROR] error: The requested URL returned error: 403 Forbidden while accessing https://github.com/carlos-algms/FileSaver.js.git/info/refs
```

这个找了很久 各种dependency-tree 好像都没用 找不到那个依赖了这个库，因为package.json,bower.json都没有明确依赖这个库，
唯一的可能就是依赖的.json文件声明的依赖依赖了这个库,最后用下面的命令一一排除，这个也挺坑，

```
bower install package#version & bower list
```
无意间在这个链接[lorvent-bower](https://lorvent.ticksy.com/ticket/1372972/)发现
```
remove the following dependencies from bower.json, and it will be working fine, also remove the bootstrap_table.vue from routes.js to avoid errors as that page is importing these packages.

"bootstrap-table": "~1.11.0", 

"tableExport.jquery.plugin": "~1.5.2",
```

果然是包含tableExport.jquery.plugin，这个大概是2年前写的，当时这个repositories(https://github.com/carlos-algms/FileSaver.js)大概还在

最后搜索github commit记录,

```
https://github.com/hhurz/tableExport.jquery.plugin/commit/8a314017d803f03dd18dd5999d22121c1cbce2b8
```
从1.6.4开始，依赖的库配置果然从

```
 "filesaver.js": "*",
```
改成了

```
"file-saver": ">=1.2.0",
```
附：[npmjs-tableexport.jquery.plugin](https://www.npmjs.com/package/tableexport.jquery.plugin)

```
Exception in thread "main" java.lang.InstantiationError: org.eclipse.jetty.util.component.Container
        at org.eclipse.jetty.server.Server.<init>(Server.java:66)
        at org.apache.zeppelin.server.ZeppelinServer.setupJettyServer(ZeppelinServer.java:224)
        at org.apache.zeppelin.server.ZeppelinServer.main(ZeppelinServer.java:142)
```
包的问题，引入了多余的包jetty-all-7.6.0.v20120127.jar,这个是hive-jdbc依赖的hive-service引入的，exclusion掉吧

```
# ls lib/jetty-
jetty-6.1.26.jar                     jetty-client-9.2.15.v20160210.jar    jetty-io-9.2.15.v20160210.jar        jetty-server-9.2.15.v20160210.jar    jetty-util-6.1.26.jar                jetty-webapp-9.2.15.v20160210.jar
jetty-all-7.6.0.v20120127.jar        jetty-http-9.2.15.v20160210.jar      jetty-security-9.2.15.v20160210.jar  jetty-servlet-9.2.15.v20160210.jar   jetty-util-9.2.15.v20160210.jar      jetty-xml-9.2.15.v20160210.jar
```
slf4j依赖重复问题

```
<!--
    <dependency>
      <groupId>org.slf4j</groupId>
      <artifactId>slf4j-api</artifactId>
    </dependency>

    <dependency>
      <groupId>org.slf4j</groupId>
      <artifactId>slf4j-log4j12</artifactId>
    </dependency>
-->
    <dependency>
      <groupId>log4j</groupId>
      <artifactId>log4j</artifactId>
    </dependency>
```
