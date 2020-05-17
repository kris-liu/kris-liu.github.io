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

```java
    private static final Map<String, ActionResource> RESOURCES = Maps.newConcurrentMap();

    @Override
    public void registerResource(ActionResource resource) {
        RESOURCES.put(resource.getActionName(), resource);
    }

```



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





### 注册分支事务

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



