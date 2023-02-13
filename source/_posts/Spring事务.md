---
title: Spring事务
tags:
  - Spring
typora-root-url: ../../themes/butterfly/source
date: 2020-11-24 21:19:56
description: 事务简单的说就是ACID，mysql有事务，Spring也有事务，Spring的事务也是依赖于数据库实现的。
cover: /blogImg/spring事务时序图.jpg
categories: Spring
---

# Spring事务的原理



首先，我们先明白 Spring 事务的本质其实就是数据库对事务的支持。没有数据库的事务支持，Spring 是无法提供事务功能的。

那么，我们一般使用 JDBC 操作事务的代码如下：

1. 获取连接 Connection con = DriverManager.getConnection()；
2. 开启事务 con.setAutoCommit(true/false)；
3. 执行 CRUD；
4. 提交事务、回滚事务：con.commit() ，con.rollback()；
5. 关闭连接 conn.close()。

使用 Spring 事务管理后，我们可以省略步骤 2 和步骤 4，让 AOP 帮你去做这些工作，关键类在 TransactionAspectSupport 这个切面里。

![spring事务时序图](/blogImg/spring事务时序图.jpg)

基本原理是：

将对应的方法通过注解元数据，标注在业务方法或者所在的对象上，然后在业务执行期间，通过AOP拦截器反射读取元数据信息，最终将根据读取的业务信息构建事务管理支持。

不同的方法之间的事务传播保证在同一个事务内，是通过统一的数据源来实现的，事务开始时将数据源绑定到ThreadLocal中，后续加入的事务从ThreadLocal获取数据源来保证数据源的统一。

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Transactional {

    @AliasFor("transactionManager")
    String value() default "";
    //事务管理器名称
    @AliasFor("value")
    String transactionManager() default "";
    //事务传播模式
    Propagation propagation() default Propagation.REQUIRED;
    //事务隔离级别
    Isolation isolation() default Isolation.DEFAULT;
    //超时时间
    int timeout() default TransactionDefinition.TIMEOUT_DEFAULT;
    //是否是只读事务
    boolean readOnly() default false;
    //需要回滚的异常类
    Class<? extends Throwable>[] rollbackFor() default {};
    //需要回滚的异常类名称
    String[] rollbackForClassName() default {};
    //排除回滚的异常类
    Class<? extends Throwable>[] noRollbackFor() default {};
    //排除回滚的异常类名称
    String[] noRollbackForClassName() default {};
}
```

这里通过SpringBoot代码来分析实现过程，源码中删除了部分代码，只保留了一些重要部分

```java
// 事务自动配置类 注意类里面使用的@EnableTransactionManagement注解
org.springframework.boot.autoconfigure.transaction.TransactionAutoConfiguration

public class TransactionAutoConfiguration {
    @Configuration
    @ConditionalOnBean(PlatformTransactionManager.class)
    @ConditionalOnMissingBean(AbstractTransactionManagementConfiguration.class)
    public static class EnableTransactionManagementConfiguration {

        //注意这里使用的@EnableTransactionManagement注解
        @Configuration
        @EnableTransactionManagement(proxyTargetClass = false)
        @ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class",
                havingValue = "false", matchIfMissing = false)
        public static class JdkDynamicAutoProxyConfiguration {
        }
        //注意这里使用的@EnableTransactionManagement注解
        @Configuration
        @EnableTransactionManagement(proxyTargetClass = true)
        @ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class",
                havingValue = "true", matchIfMissing = true)
        public static class CglibAutoProxyConfiguration {
        }
    }
}
```

```java
public class TransactionManagementConfigurationSelector extends AdviceModeImportSelector<EnableTransactionManagement> {

	/**
	 * Returns {@link ProxyTransactionManagementConfiguration} or
	 * {@code AspectJ(Jta)TransactionManagementConfiguration} for {@code PROXY}
	 * and {@code ASPECTJ} values of {@link EnableTransactionManagement#mode()},
	 * respectively.
	 */
	@Override
	protected String[] selectImports(AdviceMode adviceMode) {
		switch (adviceMode) {
			case PROXY:
				return new String[] {AutoProxyRegistrar.class.getName(),
						ProxyTransactionManagementConfiguration.class.getName()};
			case ASPECTJ:
				return new String[] {determineTransactionAspectClass()};
			default:
				return null;
		}
	}
}
```

这里只分析proxy模式

```java
@Configuration
public class ProxyTransactionManagementConfiguration extends AbstractTransactionManagementConfiguration {

    @Bean(name = TransactionManagementConfigUtils.TRANSACTION_ADVISOR_BEAN_NAME)
    @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
    public BeanFactoryTransactionAttributeSourceAdvisor transactionAdvisor() {
        BeanFactoryTransactionAttributeSourceAdvisor advisor = new BeanFactoryTransactionAttributeSourceAdvisor();
        advisor.setTransactionAttributeSource(transactionAttributeSource());
        advisor.setAdvice(transactionInterceptor());
        if (this.enableTx != null) {
            advisor.setOrder(this.enableTx.<Integer>getNumber("order"));
        }
        return advisor;
    }

    @Bean
    @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
    public TransactionAttributeSource transactionAttributeSource() {
        return new AnnotationTransactionAttributeSource();
    }

    //这里注入了TransactionInterceptor拦截器bean~~~~
    @Bean
    @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
    public TransactionInterceptor transactionInterceptor() {
        TransactionInterceptor interceptor = new TransactionInterceptor();
        interceptor.setTransactionAttributeSource(transactionAttributeSource());
        if (this.txManager != null) {
            interceptor.setTransactionManager(this.txManager);
        }
        return interceptor;
    }
}
```

TransactionInterceptor拦截器通过元数据获取事务定义信息TransactionDefinition，根据Definition信息获取PlatformTransactionManager（TM），tm接口抽象了事务的实现流程，默认的tm是DataSourceTransactionManager（通过DataSourceTransactionManagerAutoConfiguration初始化的），tm中的getTransaction根据事务的传播方式，开启、加入、挂起事务

```java
@Override
    public final TransactionStatus getTransaction(@Nullable TransactionDefinition definition) throws TransactionException {
        Object transaction = doGetTransaction();

        boolean debugEnabled = logger.isDebugEnabled();

        if (definition == null) {
            // 使用默认的Definition
            definition = new DefaultTransactionDefinition();
        }

        if (isExistingTransaction(transaction)) {
            //已经存在事务，进入单独的方法处理
            return handleExistingTransaction(definition, transaction, debugEnabled);
        }

        // 检查timeout参数
        if (definition.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
            throw new InvalidTimeoutException("Invalid transaction timeout", definition.getTimeout());
        }

        // 当前必须存在事务，否则抛出异常
        if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
            throw new IllegalTransactionStateException(
                    "No existing transaction found for transaction marked with propagation 'mandatory'");
        }
        else if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
                definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
                definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
            //获取当前的一些事务信息，用于当前事务执行完后恢复
            SuspendedResourcesHolder suspendedResources = suspend(null);
            try {
                boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
                //构造一个新事务的TransactionStatus（包含嵌套事务SavePoint的支持）
                DefaultTransactionStatus status = newTransactionStatus(
                        definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
                //开启新的事务
                doBegin(transaction, definition);
                prepareSynchronization(status, definition);
                return status;
            }
            catch (RuntimeException | Error ex) {
                //异常，恢复挂起的事务信息
                resume(null, suspendedResources);
                throw ex;
            }
        }
        else {
            //空的事务
            boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
            return prepareTransactionStatus(definition, null, true, newSynchronization, debugEnabled, null);
        }
    }
```

# @Transactional注解

| 属性名           | 说明                                                         |
| :--------------- | :----------------------------------------------------------- |
| name             | 当在配置文件中有多个 TransactionManager , 可以用该属性指定选择哪个事务管理器。 |
| propagation      | 事务的传播行为，默认值为 REQUIRED。                          |
| isolation        | 事务的隔离度，默认值采用 DEFAULT。                           |
| timeout          | 事务的超时时间，默认值为-1。如果超过该时间限制但事务还没有完成，则自动回滚事务。 |
| read-only        | 指定事务是否为只读事务，默认值为 false；为了忽略那些不需要事务的方法，比如读取数据，可以设置 read-only 为 true。 |
| rollback-for     | 用于指定能够触发事务回滚的异常类型，如果有多个异常类型需要指定，各类型之间可以通过逗号分隔。 |
| no-rollback- for | 抛出 no-rollback-for 指定的异常类型，不回滚事务。            |

## 事务隔离级别

隔离级别是指若干个并发的事务之间的隔离程度。TransactionDefinition 接口中定义了五个表示隔离级别的常量：

- TransactionDefinition.ISOLATION_DEFAULT：**这是默认值**，表示使用底层数据库的默认隔离级别。对大部分数据库而言，通常这值就是TransactionDefinition.ISOLATION_READ_COMMITTED。
- TransactionDefinition.ISOLATION_READ_UNCOMMITTED：该隔离级别表示一个事务可以读取另一个事务修改但还没有提交的数据。该级别不能防止脏读，不可重复读和幻读，因此很少使用该隔离级别。比如PostgreSQL实际上并没有此级别。
- TransactionDefinition.ISOLATION_READ_COMMITTED：该隔离级别表示一个事务只能读取另一个事务已经提交的数据。该级别可以防止脏读，这也是大多数情况下的推荐值。
- TransactionDefinition.ISOLATION_REPEATABLE_READ：该隔离级别表示一个事务在整个过程中可以多次重复执行某个查询，并且每次返回的记录都相同。该级别可以防止脏读和不可重复读。
- TransactionDefinition.ISOLATION_SERIALIZABLE：所有的事务依次逐个执行，这样事务之间就完全不可能产生干扰，也就是说，该级别可以防止脏读、不可重复读以及幻读。但是这将严重影响程序的性能。通常情况下也不会用到该级别。

## 事务传播行为

所谓事务的传播行为是指，如果在开始当前事务之前，一个事务上下文已经存在，此时有若干选项可以指定一个事务性方法的执行行为。在TransactionDefinition定义中包括了如下几个表示传播行为的常量：

- TransactionDefinition.PROPAGATION_REQUIRED：如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务。**这是默认值。**
- TransactionDefinition.PROPAGATION_REQUIRES_NEW：创建一个新的事务，如果当前存在事务，则把当前事务挂起。
- TransactionDefinition.PROPAGATION_SUPPORTS：如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续运行。
- TransactionDefinition.PROPAGATION_NOT_SUPPORTED：以非事务方式运行，如果当前存在事务，则把当前事务挂起。
- TransactionDefinition.PROPAGATION_NEVER：以非事务方式运行，如果当前存在事务，则抛出异常。
- TransactionDefinition.PROPAGATION_MANDATORY：如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常。
- TransactionDefinition.PROPAGATION_NESTED：如果当前存在事务，则创建一个事务作为当前事务的嵌套事务来运行；如果当前没有事务，则该取值等价于TransactionDefinition.PROPAGATION_REQUIRED。

我们都知道，默认的传播行为是 PROPAGATION_REQUIRED。如果外层有事务，则当前事务加入到外层事务，一起提交并一起回滚；如果外层没有事务，新建一个事务执行。也就是说，**默认情况下只有一个事务**。

假设有类A的方法methodB(),有类B的方法methodB().

1. PROPAGATION_REQUIRED

如果B的方法methodB()的事务传播特性是propagation_required，那么如下图

![事务传播栗子](/blogImg/事务传播栗子.png)

A.methodA()调用B的methodB()方法，那么如果A的方法包含事务，则B的方法则不从新开启事务，

- 如果B的methodB()抛出异常，A的methodB()没有捕获，则A和B的事务都会回滚；
- 如果B的methodB()运行期间异常会导致B的methodB()的回滚，A如果捕获了异常，并正常提交事务，则会发生Transaction rolled back because it has been marked as rollback-only的异常。
- 如果A的methodA()运行期间异常，则A和B的Method的事务都会被回滚

1. PROPAGATION_SUPPORTS

如果B的方法methodB()的事务传播特性是propagation_supports

A.methodA()调用B的methodB()方法，那么如果A的方法包含事务，则B运行在此事务环境中，如果A的方法不包含事务，则B运行在非事务环境；

- 如果A没有事务，则A和B的运行出现异常都不会回滚。
- 如果A有事务，A的method方法执行抛出异常，B.methodB和A.methodA都会回滚。
- 如果A有事务，B.method抛出异常，B.methodB和A.methodA都会回滚，如果A捕获了B.method抛出的异常，则会出现异常Transactionrolled back because it has been marked as rollback-only。

1. PROPAGATION_MANDATORY

表示当前方法必须在一个事务中运行，如果没有事务，将抛出异常

B.methodB()事务传播特性定义为:PROPAGATION_MANDATORY

- 如果A的methoda()方法没有事务运行环境，则B的methodB()执行的时候会报如下异常：No existingtransaction found for transaction marked with propagation 'mandatory'
- 如果A的Methoda()方法有事务并且执行过程中抛出异常，则A.methoda（）和B.methodb（）执行的操作被回滚；
- 如果A的methoda()方法有事务，则B.methodB()抛出异常时，A的methoda()和B.methodB()都会被回滚；如果A捕获了B.method抛出的异常，则会出现异常Transaction rolled back because ithas been marked as rollback-only

1. PROPAGATION_NESTED

如有一下方法调用关系，如图：

B的methodB()定义的事务为PROPAGATION_NESTED；

- 如果A的MethodA()不存在事务，则B的methodB()运行在一个新的事务中，B.method()抛出的异常，B.methodB()回滚,但A.methodA()不回滚；如果A.methoda()抛出异常，则A.methodA()和B.methodB()操作不回。
- 如果A的methodA()存在事务，则A的methoda()抛出异常，则A的methoda()和B的Methodb()都会被回滚；
- 如果A的MethodA()存在事务，则B的methodB()抛出异常，B.methodB()回滚，如果A不捕获异常，则A.methodA()和B.methodB()都会回滚，如果A捕获异常，则B.methodB()回滚,A不回滚；

5. PROPAGATION_NEVER

表示事务传播特性定义为PROPAGATION_NEVER的方法不应该运行在一个事务环境中

- 如果B.methodB()的事务传播特性被定义为PROPAGATION_NEVER，则如果A.methodA()方法存在事务，则会出现异常Existingtransaction found for transaction marked with propagation 'never'。

6. PROPAGATION_REQUIRES_NEW

表示事务传播特性定义为PROPAGATION_REQUIRES_NEW的方法需要运行在一个新的事务中。

如有以下调用关系：B.methodB()事务传播特性为PROPAGATION_REQUIRES_NEW.

- 如果A存在事务，A.methodA()抛出异常，A.methodA()的事务被回滚，但B.methodB()事务不受影响；如果B.methodB()抛出异常，A不捕获的话，A.methodA()和B.methodB()的事务都会被回滚。如果A捕获的话，A.methodA()的事务不受影响但B.methodB()的事务回滚。

1. PROPAGATION_NOT_SUPPORTED

表示该方法不应该在一个事务中运行。如果有一个事务正在运行，他将在运行期被挂起，直到这个事务提交或者回滚才恢复执行。

如果B.methodB()方法传播特性被定义为：PROPAGATION_NOT_SUPPORTED。

- 如果A.methodA()存在事务，如果B.methodB()抛出异常，A.methodA()不捕获的话，A.methodA()的事务被回滚，而B.methodB()出现异常前数据库操作不受影响。如果A.methodA()捕获的话，则A.methodA()的事务不受影响，B.methodB()异常抛出前的数据操作不受影响。

## 事务超时

所谓事务超时，就是指一个事务所允许执行的最长时间，如果超过该时间限制但事务还没有完成，则自动回滚事务。在 TransactionDefinition 中以 int 的值来表示超时时间，其单位是秒。

默认设置为底层事务系统的超时值，如果底层数据库事务系统没有设置超时值，那么就是none，没有超时限制。

## 事务回滚规则

Spring使用声明式事务处理，**默认情况下**，如果被注解的数据库操作方法中发生了unchecked异常，所有的数据库操作将rollback；如果发生的异常是checked异常，默认情况下数据库操作还是会提交的。

### checked异常(Exception)：

表示无效，不是程序中可以预测的。比如无效的用户输入，文件不存在，网络或者数据库链接错误。这些都是外在的原因，都不是程序内部可以控制的。 必须在代码中显式地处理。比如try-catch块处理，或者给所在的方法加上throws说明，将异常抛到调用栈的上一层。 继承自java.lang.Exception（java.lang.RuntimeException除外）。

### unchecked异常(RuntimeException)：

表示错误，程序的逻辑错误。是RuntimeException的子类，比如IllegalArgumentException, NullPointerException和IllegalStateException。 不需要在代码中显式地捕获unchecked异常做处理。 继承自java.lang.RuntimeException

# Spring 什么情况下进行事务回滚

首先我们要明白， Spring 事务回滚机制是这样的：当所拦截的方法有指定异常抛出，事务才会自动进行回滚！

因此，如果你默默的吞掉异常，像下面这样：

```java
@Service
public class UserService{
    @Transactional
    public void updateUser(User user) {
        try {
            System.out.println("=====");
            //do something
        } catch {
          //do something
        }
    }
}
```

那切面捕捉不到异常，肯定是不会回滚的。

还有就是，默认配置下，事务只会对 Error 与 RuntimeException 及其子类这些异常做出回滚。一般的 Exception 这些 Checked 异常不会发生回滚。如果一般的 Exception 想回滚，要做出如下配置：

```java
@Transactional(rollbackFor = Exception.class)
```

但是在实际开发中，我们会遇到这么一种情况：就是并没有异常发生，但是由于事务结果未满足具体业务需求，所以我们需要手动回滚事务。于是乎方法也很简单：

- 自己在代码里抛出一个自定义异常（常用）；
- 通过编程用代码回滚（不常用）。

```java
TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
```

# Spring 事务什么时候失效

## 发生自调用

示例代码如下：

```java
@Service
public class UserService{
   public void update(User user) {
        updateUser(user);
    }

    @Transactional
    public void updateUser(User user) {
        System.out.println("孤独烟真帅");
        //do something
    }
}
```

此时是无效的。因此上面的代码等同于：

```java
@Service
public class UserService{
   public void update(User user) {
        this.updateUser(user);
    }

    @Transactional
    public void updateUser(User user) {
        System.out.println("孤独烟真帅");
        //do something
    }
}
```

此时，这个 this 对象不是代理类，而是 UserService 对象本身。

解决方法很简单，让那个 this 变成 UserService 的代理类即可，就不展开说明了。

## 方法修饰符不是 public

OK，我这里不想举源码。大家想一个逻辑就行：

@Transactional 注解的方法都是被外部其他类调用才有效，那么如果方法修饰符是 private 的，这个方法能被外部其他类调到么？

既然调不到，事务生效有意义吗？想通这套逻辑就行了。

**记住** ：@Transactional 注解只能应用到 public 方法上。如果你在 protected、private 或者 package-visible 的方法上使用 @Transactional 注解，它也不会报错， 但是这个被注解的方法将不会加入事务之行。

## 发生了错误的异常

因为默认回滚的是：RuntimeException。如果是其他异常想要回滚，需要在 @Transactional 注解上加 rollbackFor 属性。

又或者是异常被吞了，事务也会失效，这里不再赘述。

## 数据库不支持事务

毕竟 Spring 事务用的是数据库的事务，如果数据库不支持事务，那 Spring 事务肯定是无法生效滴。

另外，随便少一个配置都会导致事务不生效，例如我们在 Spring Boot 中的 Application 类上不加@EnableTransactionManagement 注解也会使事务不生效，难道您能将每种情况下的配置背下来？这种配置的东西，用到的时候临时查询即可。

再比如，你把隔离级别配置成：

```
@Transactional(propagation = Propagation.NOT_SUPPORTED)
```

该隔离级别表示不以事务运行，当前若存在事务则挂起，事务肯定不生效啊！这种属于自己配错的情况。

**Spring 事务控制放在 Service 层，在 Service 方法中一个方法调用 Service 中的另一个方法，默认开启几个事务？**

我们都知道，默认的传播行为是 PROPAGATION_REQUIRED。如果外层有事务，则当前事务加入到外层事务，一起提交并一起回滚；如果外层没有事务，新建一个事务执行。也就是说，**默认情况下只有一个事务**。