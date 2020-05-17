---
title: 从零开始写一个分布式事务框架(四)-事务管理器
categories: 架构
tags:
  - 架构
  - Transaction
date: 2020-05-09 15:00:00
---




Transaction Manager：事务管理器。控制分布式事务的边界，负责开启一个分布式事务，并最终发起分布式事务提交或回滚的决议。

事务管理器定义的接口如下：

<!--more-->

```java
/**
 * 分布式事务管理器。
 * 控制分布式事务的边界，负责开启一个分布式事务，并最终发起分布式事务提交或回滚的决议。
 *
 * @author kris
 */
public interface TransactionManager {

    /**
     * 开始分布式事务
     *
     * @param xidSuffix 事务id后缀。有分库分表需求时使用，用来保证事务记录与业务记录在同一个分库分表中
     */
    void start(String xidSuffix);

    /**
     * 提交分布式事务
     *
     * @return 返回true，返回false时需要由TC重试，重试失败需要人工干预
     */
    boolean commit();

    /**
     * 回滚分布式事务
     *
     * @return 返回true，返回false时需要由TC重试，重试失败需要人工干预
     */
    boolean rollback();

}
```



### 开启分布式事务

首先我们看下`start`方法：

```java
    @Override
    public void start(String xidSuffix) {
        if (!TransactionSynchronizationManager.isActualTransactionActive()) {
            throw new DTException("分布式事务需要在一个本地事务环境中开启");
        }
        try {
            String xid = idGenerator.getId(xidSuffix);
            Activity activity = initActivity(xid);
            Utils.initDTContext(activity);
            activityRepository.insert(activity);
            activityRepository.updateStatus(activity.getXid(), ActivityStatus.INIT, ActivityStatus.COMMIT);
        } finally {
            TransactionSynchronizationManager.registerSynchronization(twoPhaseTransactionSynchronization);
        }
    }

    private Activity initActivity(String xid) {
        LocalDateTime now = LocalDateTime.now();
        LocalDateTime timeoutTime = now.plusSeconds(distributedTransactionProperties.getTimeoutTime());
        LocalDateTime executionTime = timeoutTime.plusSeconds(Constant.TIMEOUT_RETRY_EXECUTION_TIME);
        Date nowDate = Date.from(now.atZone(ZoneId.systemDefault()).toInstant());
        Activity activity = new Activity();
        activity.setXid(xid);
        activity.setName(distributedTransactionProperties.getName());
        activity.setStatus(ActivityStatus.INIT.getStatus());
        activity.setStartTime(nowDate);
        activity.setTimeoutTime(Date.from(timeoutTime.atZone(ZoneId.systemDefault()).toInstant()));
        activity.setExecutionTime(Date.from(executionTime.atZone(ZoneId.systemDefault()).toInstant()));
        activity.setRetryCount(NumberUtils.INTEGER_ZERO);
        activity.setGmtCreate(nowDate);
        activity.setGmtModified(nowDate);
        return activity;
    }
```



首先，分布式事务需要在一个本地事务的环境中执行，才能将主事务记录的更新和业务本地事务的更新放在同一个本地事务环境中一起提交或回滚，因为我们是根据本地事务是否提交完成来做为整笔分布式事务是否完成的标志，所以先校验当前是否已经在本地事务环境中；然后生成事务ID，并生成Activity主事务记录，Activity记录中包含了事务ID，事务名称，事务的超时时间，补偿字段等信息；然后使用`ActivityRepository.insert`插入Activity记录，这里需要注意，我们在insert接口上增加了`@Transactional(propagation = Propagation.REQUIRES_NEW)`注解，使用到了`Propagation.REQUIRES_NEW`事务传播属性，该事务传播属性的作用是，如果当前在一个事务中，则挂起当前事务，创建一个新的事务来执行`insert`，这保证无论执行中出现任何异常，分布式事务记录一定已经写库完成，后续就可以根据事务记录的状态来做补偿等操作了；然后执行`ActivityRepository.update`将主事务状态从`INIT`更新为`COMMIT`状态，`update`使用了`Propagation.REQUIRED`事务传播属性，是为了将该更新操作和本地业务事务放到一个本地事务中；最后使用`TransactionSynchronizationManager`注册一个事务同步器，事务同步器提供了本地事务执行过程中的一系列扩展点，可以在本地事务提交前后进行拦截，这里我们使用到了`beforeCommit`和`afterCompletion`扩展点，用来进行校验操作和分布式事务的二阶段提交操作。

```java
/**
 * 本地事务同步器。
 * 用于在本地事务提交或回滚后执行分布式事务整体的提交或回滚
 *
 * @author kris
 */
public class TwoPhaseTransactionSynchronization implements TransactionSynchronization {

    @Resource
    private TransactionManager dtTransactionManager;

    @Override
    public void beforeCommit(boolean readOnly) {
        if (!DTContext.inTransaction()) {
            return;
        }
        Activity activity = DTContext.get(DTContextEnum.ACTIVITY);
        if (activity.getTimeoutTime().before(new Date())) {
            throw new DTException("分布式事务提交超时");
        }
    }

    @Override
    public void afterCompletion(int status) {
        if (!DTContext.inTransaction()) {
            return;
        }
        try {
            if (status == TransactionSynchronization.STATUS_COMMITTED) {
                dtTransactionManager.commit();
            } else if (status == TransactionSynchronization.STATUS_ROLLED_BACK) {
                dtTransactionManager.rollback();
            }
        } finally {
            DTContext.clear();
        }
    }

}

```

`beforeCommit`中，我们对分布式事务的超时时间进行校验，超时则禁止本地事务提交，进而触发分布式事务的回滚；`afterCompletion`中我们根据本地事务提交的状态是提交还是回滚，就可以触发`TransactionManager`的二阶段`commit`或`rollback`方法了。



### 分支事务注册

服务之间的调用一般通过RPC或HTTP框架进行通信，远程调用发起的过程中我们一般可以通过切面SpringAOP的方式，拦截远程调用，注册分支事务。这里我们使用Dubbo作为服务之间的通信框架，Dubbo在使用xml的配置方式时，通过命名空间提供的解析器，会将分支事务的服务接口创建一个`BeanDefinition`，这样的bean就会执行Spring的扩展点，比如通过`BeanPostProcessor`相关扩展点比如`AbstractAutoProxyCreator`，对bean进行AOP增强，然而如果不使用xml而是直接使用`@org.apache.dubbo.config.annotation.Reference`注解进行bean的引入时，则不会创建`BeanDefinition`，而是直接在`ReferenceAnnotationBeanPostProcessor`中直接创建一个bean然后放到Springbean工厂里。所以我们没有采用SpringAOP的方式，而是使用dubbo的Filter扩展点。进行分支事务的注册。

```java
/**
 * @author kris
 */
@Slf4j
@Activate(group = CommonConstants.CONSUMER, order = Integer.MIN_VALUE)
public class ActionFilter implements Filter {

    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        if (!DTContext.inTransaction()) {
            return invoker.invoke(invocation);
        }
        try {
            String serviceName = invocation.getServiceName();
            Class<?> aClass = Class.forName(serviceName, true, Thread.currentThread().getContextClassLoader());
            Method method = aClass.getDeclaredMethod(invocation.getMethodName(), invocation.getParameterTypes());
            TwoPhaseCommit annotation = method.getAnnotation(TwoPhaseCommit.class);
            if (annotation != null) {
                Date now = new Date();
                Action action = new Action();
                action.setXid(DTContext.get(DTContextEnum.XID));
                action.setName(Utils.getActionName(aClass, method, annotation));
                action.setStatus(ActionStatus.INIT.getStatus());
                action.setArguments(JSONArray.toJSONString(invocation.getArguments()));
                action.setGmtCreate(now);
                action.setGmtModified(now);
                ExtensionFactory objectFactory = ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension();
                ResourceManager resourceManager = objectFactory.getExtension(ResourceManager.class, "dtResourceManager");
                resourceManager.registerAction(action);
            }
        } catch (Exception e) {
            log.error("ActionFilter error", e);
            throw new DTException("插入分支事务失败");
        }
        return invoker.invoke(invocation);
    }

}
```

```java
    public static String getActionName(Class<?> interfaceClass, Method method, TwoPhaseCommit annotation) {
        if (annotation == null || StringUtils.isBlank(annotation.name())) {
            return interfaceClass.getSimpleName() + "#" + method.getName();
        }
        return annotation.name();
    }
```

META-INF/dubbo/com.alibaba.dubbo.rpc.Filter文件：

```xml
dtActionFilter=cn.blogxin.dt.client.dubbo.ActionFilter
```

首先判断当前是否在分布式事务环境中；然后判断当前调用的方法是否有`@TwoPhaseCommit`注解，如果有该注解则说明调用的方法是一个分支事务的一阶段方法，然后创建一个Action分支事务记录，Action记录中包含主事务的id，分支名称，分支事务的状态，参数等分支事务基本信息；然后执行`ResourceManager.registerAction`注册一个分支事务。



### 提交或回滚分布式事务

```java
    @Override
    public boolean commit() {
        try {
            boolean commitAction = dtResourceManager.commitAction();
            if (commitAction) {
                activityRepository.updateStatus(DTContext.get(DTContextEnum.XID), ActivityStatus.COMMIT, ActivityStatus.COMMIT_FINISH);
                log.info("执行分布式事务二阶段提交成功。xid={}", (String) DTContext.get(DTContextEnum.XID));
                return true;
            }
        } catch (Exception e) {
            log.error("执行分布式事务二阶段提交异常，等待重试。xid={}", DTContext.get(DTContextEnum.XID), e);
            return false;
        }
        log.warn("执行分布式事务二阶段提交失败，等待重试。xid={}", (String) DTContext.get(DTContextEnum.XID));
        return false;
    }
```

`commit`方法中，我们先执行`ResourceManager.commitAction`方法将之前通过`ResourceManager.registerAction`注册的所有分支事务提交，然后将主事务记录的状态从`COMMIT`更新为`COMMIT_FINISH`。

```java
    @Override
    public boolean rollback() {
        try {
            boolean rollbackAction = dtResourceManager.rollbackAction();
            if (rollbackAction) {
                activityRepository.updateStatus(DTContext.get(DTContextEnum.XID), ActivityStatus.INIT, ActivityStatus.ROLLBACK);
                log.info("执行分布式事务二阶段回滚成功。xid={}", (String) DTContext.get(DTContextEnum.XID));
                return true;
            }
        } catch (Exception e) {
            log.error("执行分布式事务二阶段回滚异常，等待重试。xid={}", DTContext.get(DTContextEnum.XID), e);
            return false;
        }
        log.warn("执行分布式事务二阶段回滚失败，等待重试。xid={}", (String) DTContext.get(DTContextEnum.XID));
        return false;
    }
```

`rollback`方法中，我们先执行`ResourceManager.rollbackAction`方法将所有分支事务回滚，然后将主事务记录的状态从`COMMIT`更新为`COMMIT_FINISH`。



至此，整个`TransactionManager`事务管理器的相关逻辑就都完成了，接下来我们看下本文中多次提到的`ResourceManager`资源管理器。

