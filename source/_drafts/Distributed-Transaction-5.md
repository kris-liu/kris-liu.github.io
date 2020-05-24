---
title: 从零开始写一个分布式事务框架(三)-资源管理器
categories: 架构
tags:
  - 架构
  - Transaction
date: 2020-05-16 16:00:00
---



Resource Manager：资源管理器。控制分支事务，负责分支注册、状态汇报，驱动分支事务的提交和回滚。

资源管理器器定义的接口如下：

<!--more-->

```java
/**
 * 资源管理器。
 * 控制分支事务，负责分支注册、状态汇报，驱动分支事务的提交和回滚。
 *
 * @author kris
 */
public interface ResourceManager {

    /**
     * 启动项目前，预先注册所有ActionResource资源
     *
     * @param resource ActionResource
     */
    void registerResource(ActionResource resource);

    /**
     * 注册当前分布式事务的Action
     *
     * @param action Action
     */
    void registerAction(Action action);

    /**
     * 提交当前分布式事务
     *
     * @return 返回true，返回false时需要由TC重试，重试失败需要人工干预
     */
    boolean commitAction();

    /**
     * 提交当前分布式事务
     *
     * @return 返回true，返回false时需要由TC重试，重试失败需要人工干预
     */
    boolean rollbackAction();

}
```



### 注册分支资源

项目启动时，需要先将所有的分支事务资源进行注册，通过`registerResource`方法，将资源`ActionResource`注册进`ResourceManager`中的`RESOURCES`节点保存。

```java
    private static final Map<String, ActionResource> RESOURCES = Maps.newConcurrentMap();

    @Override
    public void registerResource(ActionResource resource) {
        RESOURCES.put(resource.getActionName(), resource);
    }

```

ActionResource的定义如下：

```java
/**
 * @author kris
 */
@Data
public class ActionResource {

    /**
     * action名称
     *
     * @see cn.blogxin.dt.client.util.Utils.getActionName()
     */
    private String actionName;

    /**
     * action的Bean
     */
    private Object actionBean;

    /**
     * 一阶段方法
     */
    private Method tryMethod;

    /**
     * 二阶段提交方法
     */
    private Method confirmMethod;

    /**
     * 二阶段回滚方法
     */
    private Method cancelMethod;

}
```

ActionResource主要保存了分支资源名称，分支资源bean对象以及二阶段提交和回滚的方法，便于在二阶段执行或补偿时使用。

那么如何触发分支资源注册呢，一般情况下我们可以通过Springbean初始化过程中的扩展点，在bean初始化时识别bean是否是一个分支资源，然后调用`registerResource`进行注册。不过在[上一篇](http://blogxin.cn/2020/05/09/Distributed-Transaction-4/)中我们提到，如果不使用dubboxml配置而是直接使用`@org.apache.dubbo.config.annotation.Reference`注解进行bean的引入时，则不会创建`BeanDefinition`，而是直接在`ReferenceAnnotationBeanPostProcessor`中通过`ReferenceBeanBuilder`直接创建一个bean然后放到Springbean工厂里，这种情况下创建的bean不会执行Springbean创建过程中的各个扩展点，比如`BeanPostProcessor`，也就无法通过初始化的相关扩展点，进行分支资源的识别和注册了。所以这里目前没有采用Spring初始化扩展点的方式，而是通过`ApplicationListener`的`ContextRefreshedEvent`事件，在容器初始化或者刷新时，触发分支资源的注册。

```java
/**
 * 注册Dubbo类型的Action
 * dubbo的@Reference注解引入的依赖无法在BeanPostProcessor获取到，需要通过监听器在容器启动前获取并注册
 *
 * @author kris
 */
public class DubboActionRegisterApplicationListener implements ApplicationListener, Ordered {

    @Resource
    private ResourceManager dtResourceManager;

    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        if (event instanceof ContextRefreshedEvent) {
            onContextRefreshedEvent((ContextRefreshedEvent) event);
        }
    }

    private void onContextRefreshedEvent(ContextRefreshedEvent event) {
        ReferenceAnnotationBeanPostProcessor referenceAnnotationBeanPostProcessor = event.getApplicationContext().getBean(ReferenceAnnotationBeanPostProcessor.class);
        Collection<ReferenceBean<?>> referenceBeans = referenceAnnotationBeanPostProcessor.getReferenceBeans();
        if (CollectionUtils.isEmpty(referenceBeans)) {
            return;
        }
        for (ReferenceBean<?> referenceBean : referenceBeans) {
            Class<?> interfaceClass = referenceBean.getInterfaceClass();
            Method[] methods = interfaceClass.getDeclaredMethods();
            for (Method method : methods) {
                TwoPhaseCommit annotation = method.getAnnotation(TwoPhaseCommit.class);
                if (annotation != null) {
                    ActionResource resource = new ActionResource();
                    resource.setActionName(Utils.getActionName(interfaceClass, method, annotation));
                    resource.setActionBean(referenceBean.getObject());
                    resource.setTryMethod(method);
                    resource.setConfirmMethod(Utils.getTwoPhaseMethodByName(annotation.confirmMethod(), methods));
                    resource.setCancelMethod(Utils.getTwoPhaseMethodByName(annotation.cancelMethod(), methods));
                    dtResourceManager.registerResource(resource);
                }
            }
        }
    }

    @Override
    public int getOrder() {
        return 0;
    }

}
```

当触发`ContextRefreshedEvent`事件时，我们先获取`ReferenceAnnotationBeanPostProcessor`，通过其`getReferenceBeans`方法获取所有dubbo引入的远程调用`ReferenceBean`，然后判断Bean的远程方法上是否有`@TwoPhaseCommit`注解，如果有该注解则说明当前Bean包含一个分支资源，创建出对应的`ActionResource`，通过`ResourceManager.registerResource`注册分支资源。



### 注册分支事务

[上一篇](http://blogxin.cn/2020/05/09/Distributed-Transaction-4/)中我们已经提到，通过dubbo的filter，在远程调用发起前，先调用`registerAction`注册分支事务。

```java
    @Override
    public void registerAction(Action action) {
        ActionContext actionContext = DTContext.get(DTContextEnum.ACTION_CONTEXT);
        actionContext.put(action.getName(), action);
        try {
            actionRepository.insert(action);
        } catch (DuplicateKeyException e) {
        }
    }
```

首先将Action注册到`DTContextEnum.ACTION_CONTEXT`这个ThreadLocal上下文中，然后调用`actionRepository.insert`方法写入分支事务，这里的insert方法同样增加了`@Transactional(propagation = Propagation.REQUIRES_NEW)`注解，使用到了`Propagation.REQUIRES_NEW`事务传播属性，该事务传播属性的作用是，如果当前在一个事务中，则挂起当前事务，创建一个新的事务来执行`insert`，这保证无论执行中出现任何异常，分支事务记录一定已经写库完成，后续就可以根据分支事务记录的状态来做补偿等操作了。这里需要忽略掉`DuplicateKeyException`异常，因为dubbo有重试策略，可能导致这里重复执行。



### 提交或回滚分支事务

```java
    @Override
    public boolean commitAction() {
        return doAction(ActionStatus.COMMIT);
    }

    @Override
    public boolean rollbackAction() {
        return doAction(ActionStatus.ROLLBACK);
    }

    private boolean doAction(ActionStatus actionStatus) {
        boolean result = true;
        ActionContext actionContext = DTContext.get(DTContextEnum.ACTION_CONTEXT);
        Map<String, Action> actionMap = actionContext.getActionMap();
        if (MapUtils.isNotEmpty(actionMap)) {
            for (Map.Entry<String, Action> entry : actionMap.entrySet()) {
                ActionResource actionResource = RESOURCES.get(entry.getKey());
                if (!execute(entry.getValue(), actionResource, actionStatus)) {
                    result = false;
                }
            }
        }
        return result;
    }

    private boolean execute(Action action, ActionResource actionResource, ActionStatus actionStatus) {
        try {
            DTParam dtParam = new DTParam();
            dtParam.setXid(DTContext.get(DTContextEnum.XID));
            dtParam.setStartTime(DTContext.get(DTContextEnum.START_TIME));
            Method twoPhaseMethod = Utils.getTwoPhaseMethodByActionStatus(actionResource, actionStatus);
            Object[] args = Utils.getTwoPhaseMethodParam(twoPhaseMethod, action.getArguments(), dtParam);
            twoPhaseMethod.invoke(actionResource.getActionBean(), args);
            actionRepository.updateStatus(action.getXid(), action.getName(), ActionStatus.INIT, actionStatus);
            return true;
        } catch (Exception e) {
            log.error("执行Action二阶段失败，等待重试。xid={}，action={}", DTContext.get(DTContextEnum.XID), action, e);
        }
        return false;
    }
```

`commitAction`和`rollbackAction`方法是在`TransactionManager`的二阶段`commit`或`rollback`内触发。首先获取`DTContextEnum.ACTION_CONTEXT`这个ThreadLocal上下文，拿到当前分布式事务执行过程中的所有分支事务，分别从`RESOURCES`中获取分支事务对应的分支资源`ActionResource`，执行`execute`方法，在`execute`方法中，根据一阶段方法参数和分布式事务信息，拼接出二阶段方法的参数，通过反射，调用分支事务的二阶段方法，执行成功后，通过`actionRepository.updateStatus`更新分支事务状态为提交或回滚。



