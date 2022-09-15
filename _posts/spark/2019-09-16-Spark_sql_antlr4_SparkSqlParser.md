---
title: Spark SQL SqlParser
tagline: "spark"
category : spark
layout: post
tags : [Spark Sql]
---



解析sql生成Unresolved Logical Plan


LogicalPlan有三个子类：

UnaryNode 一元节点,即只有一个子节点。如 Limit、Filter 操作
BinaryNode 二元节点,即有左右子节点的二叉节点。如 Join、Union 操作
LeafNode 叶子节点,没有子节点的节点。主要用户命令类操作,如SetCommand.
