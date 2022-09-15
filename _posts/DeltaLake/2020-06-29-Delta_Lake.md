---
title: Delta 实现分析
tagline: "delta lake"
category : delta lake
layout: post
tags : [delta lake]
---
delta lake 

## API DeltaTable

### DeltaTable(DeltaTableOperations)

#### executeDelete

#### executeUpdate

## SQL DeltaSparkSessionExtension

### DeltaDataSource

```
class DeltaDataSource
  extends RelationProvider
  with StreamSourceProvider
  with StreamSinkProvider
  with CreatableRelationProvider
  with DataSourceRegister
  with TableProvider
  with DeltaLogging {
```

1. RelationProvider 批量读取
2. CreatableRelationProvider 批量写入
3. StreamSourceProvider stream source 
4. StreamSinkProvider stream sink 
