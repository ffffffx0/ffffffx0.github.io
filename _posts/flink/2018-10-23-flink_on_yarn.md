---
title: Flink On Yarn
tagline: ""
category : flink
layout: post
tags : [flink, streamsets, realtime]
---

### 配置环境变量HADOOP_CONF_DIR,将yarn的配置文件解压在此目录

```
export HADOOP_CONF_DIR=/opt/soft/yarn-conf
```
### start yarn-session

```
./yarn-session.sh -n 8 -s 8 -jm 1024 -tm 1024 -nm flink –d
```

### Submit a job to YARN

```
wget -O LICENSE-2.0.txt http://www.apache.org/licenses/LICENSE-2.0.txt
hadoop fs -copyFromLocal LICENSE-2.0.txt /tmp
./bin/flink run ./examples/batch/WordCount.jar --input hdfs:///tmp/LICENSE-2.0.txt  --output  hdfs:///tmp/wordcount-result.txt
```
