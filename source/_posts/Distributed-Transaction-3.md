---
title: 从零开始写一个分布式事务框架(三)-事务记录
categories: 架构
tags:
  - 架构
  - Transaction
date: 2020-05-05 15:00:00
---




**事务记录**是用来记录每笔分布式事务状态的记录以及事务中各个分支事务状态的记录，需要在实际事务操作发生前，先写入到事务记录表，再进行事务操作，如果遇到异常情况，就可以根据存储的事务记录，通过二阶段接口进行分布式事务的恢复，来保证分布式事务的最终一致性。

这种数据恢复的思路也叫做**WAL(Write-Ahead-Log) 预写式日志** ，该机制用于数据的容错和恢复，在数据写入到数据库之前，先写入到日志，再将日志记录变更到存储器中。

<!--more-->



我们使用数据库来进行事务记录的存储，需要设计主事务记录表以及分支事务记录表。



#### 主事务记录

主事务记录需要记录事务的id，事务的状态，事务发起的基本信息，还需要有能够满足补偿重试逻辑的相关信息，根据这些需要我们设计出如下表结构以及对应的Java数据结构：

##### 表结构

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
```

##### Java数据结构

```java
/**
 * 主事务记录
 *
 * @author kris
 */
@Data
public class Activity implements Serializable {

    private static final long serialVersionUID = -158716849842073007L;

    private long id;

    /**
     * 事务ID
     */
    private String xid;

    /**
     * 事务发起者名称
     */
    private String name;

    /**
     * 事务状态
     *
     * @see ActivityStatus
     */
    private int status;

    /**
     * 事务开始时间
     */
    private Date startTime;
    /**
     * 事务超时时间
     */
    private Date timeoutTime;

    /**
     * 执行时间，每次重试后将执行时间向后延迟
     */
    private Date executionTime;

    /**
     * 二阶段重试次数
     */
    private int retryCount;

    /**
     * 事务创建时间
     */
    private Date gmtCreate;

    /**
     * 事务更新时间
     */
    private Date gmtModified;

}
```

```java
/**
 * @author kris
 */
public enum ActivityStatus {

    /**
     * 主事务初始化
     */
    INIT(0),

    /**
     * 主事务提交
     */
    COMMIT(1),

    /**
     * 主事务提交，分支事务全部执行完成
     */
    COMMIT_FINISH(2),

    /**
     * 主事务回滚
     */
    ROLLBACK(3);

    private int status;

    ActivityStatus(int status) {
        this.status = status;
    }

    public int getStatus() {
        return this.status;
    }

}
```



#### 分支事务记录

分支事务记录需要记录主事务的id，分支名称，分支事务的状态，分支事务基本信息，根据这些需要我们设计出如下表结构以及对应的Java数据结构：

##### 表结构

```sql
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

##### Java数据结构

```java
/**
 * 分支事务记录
 *
 * @author kris
 */
@Data
public class Action implements Serializable {

    private static final long serialVersionUID = -3797643582038854915L;

    private long id;

    /**
     * 主事务ID
     */
    private String xid;

    /**
     * 分支事务名称
     */
    private String name;

    /**
     * 分支事务状态
     *
     * @see ActionStatus
     */
    private int status;

    /**
     * 分支事务一阶段参数
     */
    private String arguments;

    /**
     * 分支事务创建时间
     */
    private Date gmtCreate;

    /**
     * 分支事务更新时间
     */
    private Date gmtModified;

}
```

```java
/**
 * @author kris
 */
public enum ActionStatus {

    /**
     * 分支事务初始化
     */
    INIT(0),

    /**
     * 分支事务提交完成
     */
    COMMIT(1),

    /**
     * 分支事务回滚完成
     */
    ROLLBACK(3);

    private int status;

    ActionStatus(int status) {
        this.status = status;
    }

    public int getStatus() {
        return this.status;
    }

}
```



基于定义的主事务记录和分支事务记录的数据结构，通过`ActivityRepository`和 `ActionRepository` 接口，进行事务记录的读写操作。



