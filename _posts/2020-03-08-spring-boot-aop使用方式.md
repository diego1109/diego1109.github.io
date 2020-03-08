---

layout: post
title: spring boot AOP使用方式
date: 2020-03-08
categories: 学习笔记
tags: SpringBoot
comments: true 

---

## 什么是AOP

`AOP（Aspect-oriented programming，面向切面编程）` 的目的是为了解耦从而提高代码的模块化程度。开发的应用中可以根据功能分为“主业务逻辑”和“通用业务逻辑”，比如：某电商网站当下需要开发用户注册的功能并打印一些日志，注册用户就是主业务，打印日志是通用业务。除了日志还有事物管理、安全管理等等这些都可以划归到通用业务中，该通用业务有个专门的名词 `cross-cutting concerns`，在有的博客中将它翻译成 **横切关注点** ，起初看到这个翻译的时候着实被懵到了，这到底是个什么鬼..........

想象一个场景：网站可不仅仅只在用户注册的时候打日志，商品查询、订单创建.......这些地方都需要的，而且通用业务也不只是打日志这一项，当每一个主业务中都夹杂着各种的通用业务时，开发和维护的难度也就增大了。

再回到打日志这个功能，每个主业务逻辑中负责打日志的代码的逻辑几乎是一样的，开发者将打日志得代码抽成方法被主业务调用就能让业务逻辑变的“干净清晰”，这个方式也可应用到其他的 cross-cutting concerns，而且这的确是个不错的办法，但“调用”没有将主业务和 cross-cutting concerns 彻底的分开，主业务逻辑还是包含了通用业务，尽管它只有一行代码而已。AOP 就做到了彻底分开（解耦），主业务逻辑、通用业务逻辑封装到两个不同的方法中，AOP 通过在主业务方法上做“标记”的方式告诉 Spring Boot：用户注册的时候记得打日志，具体的格式在通用业务方法中。 

## 名词解释

AOP 涉及这里主要介绍 `Aspect` 、`Advice` 、`Joinpoint`、`Pointcut` 这4个(总共有8个)，理解这四个就可以上手做东西了，其他的后续再研究吧。（先做完成主义，再修改优化，最后再做完美主义）。

- **Aspect (切面)**：这是个类，它包含了某个 cross-cutting concerns 的各种场景下的业务逻辑，比如：用户注册的业务需要日志格式“用户名+日期”，创建订单的业务需要的日志格式是“用户+商品名+价钱+日期”。为满足场景需求，我们可以创建切面 `LoggerAspect.class` ，里面包含两个方法，`method1` 负责产生用户注册的日志，`method2` 负责创建订单的日志。

- **Advice (建议)**：这是个方法，里面包含着 cross-cutting concerns 在某个场景需求下要执行业务逻辑，上面的 method1 和 method2 都是 advice。

- **Joinpoint (连接点)**：通过上面两个解释我们可以意识到，当主业务方法执行的时候也执行了 Aspect 中的某个 Advice，joinpoint 规定了当主业务方法(假设方法名是：`registerUser` )执行到什么时刻时 **允许** advice 执行。允许的时刻如下：

  

  > 1. registerUser 执行之前；
  > 2. registerUser 执行之后，不论 registerUser 执行成功还是失败；
  > 3. registerUser 执行之后，而且要执行成功；
  > 4. registerUser 执行出现异常的时候；



- **Pointcut (切点)**：joinPoint 给出了允许 advice 执行的时刻的候选集，切点指的是 advice 从候选集中挑个时间让自己被执行。`Before`、`After`、`AfterReturning`、`AfterThrowing`、`Around`，根据单词的字面意思和上面的允许时刻对比下，就差不错明白了。

## 使用方法

目前有基于 `spring AOP` 和 `AspectJ` 两种 AOP 框架，在一些资料上看到两个框架兼容且 AspectJ 的速度是spring AOP 的8到35倍，所以本文基于后者[(配置方法)](https://blog.csdn.net/gavin_john/article/details/80156963?depth_1-utm_source=distribute.pc_relevant.none-task&utm_source=distribute.pc_relevant.none-task)。同时将基于 AspectJ 的 AOP 应用到开发中也有两种方式：**路径切入** 和 **注解切入** 。

### 通过路径切入

```java
@Aspect
@Component
public class LoggerAspect {
    
  @Pointcut("execution(public * com.forum.controller.*.*(..))")
  public void log() {
  }

  @Before("log()")
  public void doBefore(JoinPoint joinPoint) {
        ........
  }
  
  @After("log()")
  public void doAfter() {
        ........
  }
}

```

1、新建切面类 `LoggerAspect` 上面加俩注解  `@Aspect`、 `@Component`  缺一不可。

2、`@Pointcut` 指定要”切“谁，可以精确到包、类或者方法上。

3、`@Before` 指定切点以及要执行的业务逻辑。

4、`@After` 指定切点以及要执行的业务逻辑。

这种方式的优点是：当 @PointCut 定位到包或者类时，可以执行批量切入。

### 通过注解切入

自定义注解，不做任何操作，全当是个”标识符“。

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Loggable {
}
```

定义切面，`execution(* *(..))` 表示这个advice只在执行的时候调用，否则会执行两次。

```java
@Aspect
@Component
public class LoggerAspect {
    
  @Before("@annotation(com.demo.aop.aspcet.Loggable) && execution(* *(..))")
  public void doBefore(JoinPoint joinPoint) {
        ........
  }
}

```

在主业务代码中使用，该业务代码的功能是创建employee。

```java
@Service
public class EmployeeService {

  @Loggable
  public Employee createEmployee(String empId, String name) {
    return new Employee(empId, name);
  }
}
```

`@Loggable` 注解将切入点精确到了方法上。所以这种方式的优点是灵活、也比较干净。