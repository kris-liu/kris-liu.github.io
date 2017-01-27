---
title: Spring AspectJ事务
date: 2015-08-20 00:00:00
categories: Spring
tags: Spring
---

当事务管理使用的是代理形式，仅有在公有方法上标记的@Transactional是有效的，所有的私有的、受保护的或者包可见性的方法即使标记了@Transactional也不会有实质性的事务管理行为产生，并且系统不会给出任何错误或者提示信息。如果有必要在非公有方法上标记事务，那么不应当使用代理模式的事务管理，可以考虑使用AspectJ。

注意：Spring提倡将@Transactional标记在类（或者类的方法上），不提倡对接口（或者接口方法进行标记）。在接口或者接口方法上进行@Transactional标记是可行的，但是仅有系统运行在基于接口的代理前提下事务管理才会发生。实际上Java标记不会通过接口继承，这意味着如果你使用基于类的代理（prox-target-calss=”ture”）或者使用weaving-based aspact（mode=”aspectj”）,这样在接口上的标记将不会被代理或者织入架构识别，并且对象不会被事务代理包装，最终无法实现事务管理。

注意：在代理模式中（是Spring事务管理默认使用的），仅有外部方法调用过程才会被代理截获，这意味着自身调用，即一个方法调用了本对象的另外一个方法不会导致一个实质的事务管理代理过程产生，即使是被调用的方法标记了@Transactional。

AspectJ模式与代理模式不同，能够使应用自身调用依然被事务管理包围。AspectJ不使用代理，而是在字节码基础上将事务处理逻辑添加到对象中。（这样代码量明显增加）。`<tx:annotation-driven/>`的配置属性描述如下：

1. transaction-manager，默认transactionManager，事务管理器的名字，仅当事务管理器不是transactionManager时，需要显示指定。
2. mode，默认proxy，proxy模式仅当外部调用产生时才产生代理。可替代的选项是aspectj，通过更改类的字节码来提供事务功能的织入。aspectJ织入需要提供spring-aspects.jar包支持。
3. proxy-target-class，默认为false。只适用于proxy方式，用于控制代理创建的方式。当值为true时，基于class类型的代理将被创建，否则，基于标准的JDK接口的代理将被创建。

当使用mode=“aspectj”的时候，需要配合aspectj的maven插件，需要使用aspectj的编译器在编译时在字节码基础上将事务处理逻辑添加到对象中：
编译时织入：
优点：

1. 由于将切面直接编译进了字节码，所以运行时不再需要动态创建代理对象，节约了内存呢和 CPU 消耗；
2. 通过AspectJ，方法被织入了切面后，方法上的 Annotation 还是有效的，因为对象类型没有变。而动态代理可能会以代理类替代原类型，也就失去了 Annotation。
缺点：
3. 编写 aspect 文件有一定的难度；
4. 编译过程稍显复杂（借助工具可简化：Eclipse AJDT, Maven AspectJ 插件等）。

aspectj插件配置如下：

```
            <plugin>
                <groupId>org.codehaus.mojo</groupId>
                <artifactId>aspectj-maven-plugin</artifactId>
                <version>1.4</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>compile</goal>
                            <goal>test-compile</goal>
                        </goals>
                    </execution>
                </executions>
                <configuration>
                    <source>1.6</source>
                    <target>1.6</target>
                    <encoding>UTF-8</encoding>
                    <aspectLibraries>
                        <aspectLibrary>
                            <groupId>org.springframework</groupId>
                            <artifactId>spring-aspects</artifactId>
                        </aspectLibrary>
                    </aspectLibraries>
                </configuration>
            </plugin>

```

aspectj的编译器编译结果(class反编译的结果)：

```
    @Transactional
    public BonusBookingResult doBooking(UserBonusModel bonusModel, Map<String, String> promotionMap) {
        BonusBookingResult var10;
        try {
            BonusBookingResult var8;
            try {
                AnnotationTransactionAspect.aspectOf().ajc$before$org_springframework_transaction_aspectj_AbstractTransactionAspect$1$2a73e96c(this, ajc$tjp_0);

		... ...
                doXxx业务逻辑
		... ...
                var8 = new BonusBookingResult(bestBonus, defaultBonus);
            } catch (Throwable var11) {
                AnnotationTransactionAspect.aspectOf().ajc$afterThrowing$org_springframework_transaction_aspectj_AbstractTransactionAspect$2$2a73e96c(this, var11);
                throw var11;
            }

            AnnotationTransactionAspect.aspectOf().ajc$afterReturning$org_springframework_transaction_aspectj_AbstractTransactionAspect$3$2a73e96c(this);
            var10 = var8;
        } catch (Throwable var12) {
            AnnotationTransactionAspect.aspectOf().ajc$after$org_springframework_transaction_aspectj_AbstractTransactionAspect$4$2a73e96c(this);
            throw var12;
        }

        AnnotationTransactionAspect.aspectOf().ajc$after$org_springframework_transaction_aspectj_AbstractTransactionAspect$4$2a73e96c(this);
        return var10;
    }
```

## 参考

<http://opoo.org/aspectj-compile-time-weaving/>

<http://blog.sina.com.cn/s/blog_616e189f0100zpd3.html>

<http://www.iflym.com/index.php/code/the-theory-about-spring-and-aspectj-pre.html>
