---
title: 从零开始写一个分布式事务框架(三)-事务记录
categories: 架构
tags:
  - 架构
  - Transaction
date: 2020-05-05 15:00:00

---



WAL(Write-Ahead-Log) 

该机制用于数据的容错和恢复

WAL的核心思想是:在数据写入到数据库之前，先写入到日志.再将日志记录变更到存储器中。

事务所引起的所有改动都要记录在日志中，在事务提交完成之前，所有的这些记录必须被写入硬盘；



这个是为了解决数据库意外宕机时，导致数据丢失的问题的。它的机制是当事务提交时，先写入重做日志到磁盘，再修改缓冲池中的页，最后通过Checkpoint刷新到磁盘（事务提交会触发checkpoint）。



> 预写式日志 （WAL） 是一种实现事务日志的标准方法．有关它的详细描述可以在 大多数（如果不是全部的话）有关事务处理的书中找到．简而言之，WAL 的中心思想是对数据文件 的修改（它们是表和索引的载体）必须是只能发生在这些修改已经 记录了日志之后 --也就是说，在日志记录冲刷到永久存储器之后． 如果我们遵循这个过程，那么我们就不需要在每次事务提交的时候都把数据页冲刷到磁盘，因为我们知道在出现崩溃的情况下， 我们可以用日志来恢复数据库：任何尚未附加到数据页的记录都将先从日志记录中重做（这叫向前滚动恢复，也叫做 REDO） 然后那些未提交的事务做的修改将被从数据页中删除 （这叫向后滚动恢复 -UNDO）．





<!--more-->



```java
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



```java
@Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = Exception.class)
```



```java
@Transactional(propagation = Propagation.REQUIRED, rollbackFor = Exception.class)
```