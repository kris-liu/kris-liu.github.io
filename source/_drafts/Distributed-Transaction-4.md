---
title: 从零开始写一个分布式事务框架(四)-事务管理器
categories: 架构
tags:
  - 架构
  - Transaction
date: 2020-05-05 15:00:00
---



Transaction Manager（事务管理器）：用来控制分布式事务的边界，负责开启一个分布式事务，并最终发起分布式事务提交或回滚的决议。



