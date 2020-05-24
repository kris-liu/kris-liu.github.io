---
title: 从零开始写一个分布式事务框架(七)-SpringBootStarter
categories: 架构
tags:
  - 架构
  - Transaction
date: 2020-05-24 20:00:00
---




### SpringBoot-Starter

基于前面开发好的模块，创建SpringBoot-Starter进行整合，通过Spring-Boot的加载机制进行分布式事物组件DT的自动装配，便于使用。



#### pom文件：

在项目的pom文件中定义之前开发过程中需要使用的模块，以及使用的相关依赖。

<!--more-->



#### 自动装配类：

```java
/**
 * @author kris
 */
@Configuration
@ConditionalOnProperty(prefix = "dt", name = "enable", havingValue = "true")
@EnableConfigurationProperties(DistributedTransactionProperties.class)
public class DistributedTransactionConfiguration {

    @Resource
    private DistributedTransactionProperties distributedTransactionProperties;

    @Bean
    public ActivityRepository activityRepository() {
        return new ActivityMybatisRepository();
    }

    @Bean
    public ActionRepository actionRepository() {
        return new ActionMybatisRepository();
    }

    @Bean
    public TransactionManager dtTransactionManager() {
        return new TransactionManagerImpl();
    }

    @Bean
    public ResourceManager dtResourceManager() {
        return new ResourceManagerImpl();
    }

    @Bean
    public TwoPhaseTransactionSynchronization twoPhaseTransactionSynchronization() {
        return new TwoPhaseTransactionSynchronization();
    }
  
		...
		...

}


```

创建一个带`@Configuration`注解的自动装配类 `DistributedTransactionConfiguration`，通过`@ConditionalOnProperty`注解添加启动条件，根据有没有配置符合要求的属性来决定是否启动该自动装配类。然后通过`@Bean`将待初始化的类进行初始化配置。



#### 配置类：

```java
/**
 * @author kris
 */
@Data
@Configuration
@ConfigurationProperties(prefix = "dt")
public class DistributedTransactionProperties {

    /**
     * 分布式事务是否启动
     */
    private boolean enable = false;

    /**
     * 分布式事务名称
     */
    private String name;

    /**
     * 分布式任务超时时间，默认30S
     */
    private long timeoutTime = 30;

    /**
     * ElasticJob属性配置
     */
    private JobProperties job;

}
```

```java
/**
 * Job属性配置
 *
 * @author kris
 */
@Data
public class JobProperties {

    /**
     * Elastic-job zk列表
     */
    private String serverList;

    /**
     * Elastic-job zk节点namespace
     */
    private String namespace = "dtJob";

    /**
     * Elastic-job 任务频率，默认10S执行一次
     */
    private String cron = "0/10 * * * * ?";

    /**
     * Elastic-job 分片数量。默认1，若需要分库分表可以进行自定义
     */
    private int shardingTotalCount = 1;

    /**
     * 任务单个分片单次捞取任务数量
     */
    private int limit = 20;

}
```

通过`@EnableConfigurationProperties(DistributedTransactionProperties.class)`将配置加载入`DistributedTransactionProperties`配置类的属性里。



#### 加载文件：

在`resources/META-INF`目录下的`spring.factories`文件中配置如下：

```java
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
cn.blogxin.dt.client.spring.DistributedTransactionConfiguration
```



至此，整个Starter开发完毕，Deploy到中央仓库或Install到本地仓库后即可使用。



