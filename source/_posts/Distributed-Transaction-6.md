---
title: 从零开始写一个分布式事务框架(六)-事务协调器
categories: 架构
tags:
  - 架构
  - Transaction
date: 2020-05-24 15:00:00
---




Transaction Coordinator：事务协调器。维护分布式事务的状态，负责对分布式事务进行补偿提交或回滚。

事务协调器定义的接口如下：

<!--more-->

```java
/**
 * 事务协调器。
 * 维护分布式事务的状态，负责对分布式事务进行补偿提交或回滚。
 *
 * @author kris
 */
public interface TransactionCoordinator {

    /**
     * 对二阶段提交失败的事务进行批量补偿
     *
     * @param param CoordinatorParam
     */
    void batchReTry(CoordinatorParam param);

}
```

除协调器外还需要使用分布式任务进行补偿任务的调度。



### 事务协调器

```java
/**
 * @author kris
 */
@Slf4j
public class TransactionCoordinatorImpl implements TransactionCoordinator {

    @Resource
    private ActivityRepository activityRepository;

    @Resource
    private ActionRepository actionRepository;

    @Resource
    private TransactionManager dtTransactionManager;

    @Override
    public void batchReTry(CoordinatorParam param) {
        List<Activity> activities = activityRepository.queryUnfinished(param.getShardingKey(), new Date(), param.getLimit());
        for (Activity activity : activities) {
            if (!execute(activity)) {
                activityRepository.updateRetry(activity.getXid(), activity.getStatus(), activity.getRetryCount() + 1, getRetryTime(activity.getExecutionTime()));
            }
        }
    }

    /**
     * 执行单个Activity补偿操作
     *
     * @param activity
     * @return
     */
    private boolean execute(Activity activity) {
        boolean result = false;
        ActivityStatus activityStatus = Utils.getFinalStatus(activity);
        if (activityStatus == null) {
            return false;
        }
        try {
            Utils.initDTContext(activity);
            ActionContext actionContext = DTContext.get(DTContextEnum.ACTION_CONTEXT);
            List<Action> actions = actionRepository.query(activity.getXid());
            for (Action action : actions) {
                if (ActionStatus.INIT.getStatus() == action.getStatus()) {
                    actionContext.put(action.getName(), action);
                }
            }
            if (ActivityStatus.COMMIT_FINISH == activityStatus) {
                result = dtTransactionManager.commit();
            } else if (ActivityStatus.ROLLBACK == activityStatus) {
                result = dtTransactionManager.rollback();
            }
        } finally {
            DTContext.clear();
        }
        return result;
    }

    private Date getRetryTime(Date executionTime) {
        Instant instant = executionTime.toInstant();
        ZoneId zoneId = ZoneId.systemDefault();
        LocalDateTime tmp = instant.atZone(zoneId).toLocalDateTime();
        LocalDateTime retryTime = tmp.plusMinutes(5);
        return Date.from(retryTime.atZone(ZoneId.systemDefault()).toInstant());
    }

}
```

```java
    public static ActivityStatus getFinalStatus(Activity activity) {
        if (ActivityStatus.INIT.getStatus() == activity.getStatus() && activity.getTimeoutTime().before(new Date())) {
            return ActivityStatus.ROLLBACK;
        } else if (ActivityStatus.COMMIT.getStatus() == activity.getStatus()) {
            return ActivityStatus.COMMIT_FINISH;
        }
        return null;
    }
```

```java
/**
 * 协调器参数
 *
 * @author kris
 */
@Data
public class CoordinatorParam {

    /**
     * 任务分片key，分库分表场景使用
     */
    private String shardingKey;

    /**
     * 任务单个分片单次捞取任务数量
     */
    private int limit = 20;

}
```

`batchReTry`批量补偿方法，首先通过`activityRepository.queryUnfinished`查询所有已到达执行时间，但状态仍未达到终态的主事务记录，然后根据当前状态以及超时时间，判断当前事务应该做提交补偿还是应该做回滚补偿，如果状态为`INIT`且事务已超时，则需要进行回滚补偿，若状态已经是`COMMIT`则需要进行提交补偿，否则先跳过不做补偿，等待状态确定下来后再进行补偿，然后查询所有未达到终态的分支事务记录，通过`TransactionManager`重新进行提交或回滚的补偿操作，若补偿失败，则将执行时间往后推迟一段时间且增加重试次数记录，若多次重试仍未达到终态，则需要报警进行人工干预，一般出现该状况是因为分支事务服务的二阶段实现有坑，分支事务需要保证只要一阶段提交成功，二阶段一定可以执行成功或回滚成功，回滚时也需要支持空回滚。



### 分布式补偿任务

```java
/**
 * 分布式事务协调器任务，负责任务分片和调度
 *
 * @author kris
 */
public class CoordinatorJob implements SimpleJob {

    @Resource
    private TransactionCoordinator transactionCoordinator;

    @Resource
    private DistributedTransactionProperties distributedTransactionProperties;

    @Override
    public void execute(ShardingContext shardingContext) {
        CoordinatorParam param = new CoordinatorParam();
        param.setShardingKey(String.valueOf(shardingContext.getShardingItem()));
        param.setLimit(distributedTransactionProperties.getJob().getLimit());
        transactionCoordinator.batchReTry(param);
    }

}
```

```java
    @Bean
    public CoordinatorJob coordinatorJob() {
        return new CoordinatorJob();
    }

    @Bean(initMethod = "init")
    public CoordinatorRegistryCenter dtRegCenter() {
        JobProperties job = distributedTransactionProperties.getJob();
        return new ZookeeperRegistryCenter(new ZookeeperConfiguration(job.getServerList(), Constant.DT_COORDINATOR_JOB_BASE_NAMESPACE + job.getNamespace()));
    }

    @Bean
    public LiteJobConfiguration dtLiteJobConfiguration(CoordinatorJob coordinatorJob) {
        JobProperties job = distributedTransactionProperties.getJob();
        JobCoreConfiguration jobCoreConfiguration = JobCoreConfiguration.newBuilder(coordinatorJob.getClass().getName(), job.getCron(), job.getShardingTotalCount())
                .description("DT分布式事务协调器任务").build();
        SimpleJobConfiguration simpleJobConfiguration = new SimpleJobConfiguration(jobCoreConfiguration, coordinatorJob.getClass().getCanonicalName());
        return LiteJobConfiguration.newBuilder(simpleJobConfiguration).overwrite(true).build();
    }

    @Bean(initMethod = "init")
    public JobScheduler dtScheduler(CoordinatorRegistryCenter dtRegCenter, LiteJobConfiguration dtLiteJobConfiguration, CoordinatorJob coordinatorJob) {
        return new SpringJobScheduler(coordinatorJob, dtRegCenter, dtLiteJobConfiguration);
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

分布式补偿任务，我们选用Elastic-Job作为分布式定时任务组件，通过cron表达式的规则，定时调度起补偿任务，不断的进行扫描，将中间状态的待补偿的事务记录捞取出来重新补偿，通过这样不断补偿重试机制，保证达到最终一致性。



