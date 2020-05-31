---
title: 从零开始写一个分布式事务框架(八)-使用教程
categories: 架构
tags:
  - 架构
  - Transaction
date: 2020-05-24 22:00:00
---



### 从业务服务



#### POM引入

```xml
        <dependency>
            <groupId>cn.blogxin</groupId>
            <artifactId>dt-client-api</artifactId>
            <version>0.0.1-SNAPSHOT</version>
        </dependency>
```

<!--more-->



#### 提供二阶段服务

```java
public interface AccountDubboService {

    @TwoPhaseCommit(name = "transferOutAccount", confirmMethod = "commit", cancelMethod = "unfreeze")
    boolean freeze(AccountDTO accountDTO);

    void commit(DTParam dtParam, AccountDTO accountDTO);

    void unfreeze(DTParam dtParam, AccountDTO accountDTO);

}
```

接口的一阶段方法上添加二阶段提交注解`@TwoPhaseCommit`，设置分支事务名称以及对应的`confirmMethod`和`cancelMethod`方法名称，`confirmMethod`和`cancelMethod`方法第一个参数设置为`DTParam`，包含了分布式事务ID以及事务开始时间等分布式事务上下文信息，后面的参数与一阶段方法参数相同，二阶段方法调用时会将一阶段的参数重新传进来。



### 主业务服务



#### POM引入

```xml
        <dependency>
            <groupId>cn.blogxin</groupId>
            <artifactId>dt</artifactId>
            <version>0.0.1-SNAPSHOT</version>
        </dependency>
```



#### 初始化数据库

```sql
CREATE DATABASE `dt` CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
use dt;

create table activity (
   `id` BIGINT(20) UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '自增主键',
   `xid` VARCHAR(128) NOT NULL DEFAULT '' COMMENT '事务ID',
   `name` VARCHAR(64) NOT NULL DEFAULT '' COMMENT '事务发起者名称',
   `status` TINYINT(4) NOT NULL DEFAULT '0' COMMENT '事务状态',
   `start_time` DATETIME NOT NULL DEFAULT '1971-01-01 00:00:00' COMMENT '事务开始时间',
   `timeout_time` DATETIME NOT NULL DEFAULT '1971-01-01 00:00:00' COMMENT '事务超时时间',
   `execution_time` DATETIME NOT NULL DEFAULT '1971-01-01 00:00:00' COMMENT '执行时间，每次重试后将执行时间向后延迟',
   `retry_count` TINYINT(4) NOT NULL DEFAULT '0' COMMENT '二阶段重试次数',
   `gmt_create` DATETIME NOT NULL DEFAULT '1971-01-01 00:00:00' COMMENT '事务创建时间',
   `gmt_modified` DATETIME NOT NULL DEFAULT '1971-01-01 00:00:00' COMMENT '事务更新时间',
   PRIMARY KEY (`id`),
   UNIQUE KEY `uk_xid` (`xid`),
   KEY `idx_execution_time_status` (`execution_time`, `status`)
)ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='主事务记录表';

create table action (
   `id` BIGINT(20) UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '自增主键',
   `xid` VARCHAR(128) NOT NULL DEFAULT '' COMMENT '主事务ID',
   `name` VARCHAR(64) NOT NULL DEFAULT '' COMMENT '分支事务名称',
   `status` TINYINT(4) NOT NULL DEFAULT '0' COMMENT '分支事务状态',
   `arguments` VARCHAR(2000) NOT NULL DEFAULT '' COMMENT '分支事务一阶段参数',
   `gmt_create` DATETIME NOT NULL DEFAULT '1971-01-01 00:00:00' COMMENT '分支事务创建时间',
   `gmt_modified` DATETIME NOT NULL DEFAULT '1971-01-01 00:00:00' COMMENT '分支事务更新时间',
   PRIMARY KEY (`id`),
   UNIQUE KEY `uk_xid_name` (`xid`, `name`)
)ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='分支事务记录表';
```



#### 添加mybatis配置

```java
@MapperScan({"cn.blogxin.pay.mapper"/*服务本身的mapper路径*/, "cn.blogxin.dt.log.repository.mybatis.mapper"})
```

扫描dt组件的mapper

```
mybatis.mapper-locations=classpath*:dt-mybatis/mapper/*.xml
```

添加dt组件的mapper.xml配置



#### 添加配置

```java
dt.enable=true	//启动dt组件
dt.name=pay_test	//dt服务名称
dt.job.serverList=127.0.0.1:2181	//jobzk地址
dt.job.namespace=pay_test_job	//job的zk路径namespace
```



#### 使用

```java
    @Resource
    private TransactionManager dtTransactionManager;

    @Transactional(rollbackFor = Exception.class)
    public boolean execute(Xxx xxx) {
        dtTransactionManager.start();
      	//执行本地事务
      	//执行分支事务
        return true;
    }
```

引入`TransactionManager`，在`@Transactional`本地事务中，需要执行`dtTransactionManager.start();`开启一个分布式事务。





### 使用DEMO

使用DT分布式事务组件的demo： https://github.com/kris-liu/DT/tree/master/dt-demo



1. 执行初始化SQL并初始化数据：https://github.com/kris-liu/DT/blob/master/dt-demo/sql/init.sql
2. 启动`DemoAccountApplication`，`DemoCouponApplication`两个分支事务提供方。
3. 启动`DemoPayApplication`分布式事务发起方，实现了一个同时使用余额和券两种渠道组合支付的接口demo。
4. 请求测试接口http://127.0.0.1:8082/pay
5. 参数：{"uid":"000001","orderId":"ORDER000002","amount":"200","channels":[{"channelId":"10","amount":"100","assetsId":""},{"channelId":"11","amount":"100","assetId":"COUPON000001"}]}
6. 可以在分布式事务执行过程中的各个环节模拟异常，观察分布式事务会通过补偿达到最终一致。



