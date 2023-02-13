---
title: Spring Bean加载过程与生命周期
tags:
  - Spring
  - Spring Boot
date: 2020-10-01 16:06:00
description: 这也是实际工作中遇到的一个问题，有一些放在配置里的配置项在MVC升Boot的过程中空指针了！研究了半天原因。。记录一下相关知识
cover: /blogImg/bean的生命周期.png
categories: Spring
typora-root-url: ../../themes/butterfly/source
---

# Bean的加载过程

Spring 作为 Ioc 框架，实现了依赖注入，由一个中心化的 Bean 工厂来负责各个 Bean 的实例化和依赖管理。各个 Bean 可以不需要关心各自的复杂的创建过程，达到了很好的解耦效果。

我们对 Spring 的工作流进行一个粗略的概括，主要为两大环节：

- **解析**，读 xml 配置，扫描类文件，从配置或者注解中获取 Bean 的定义信息，注册一些扩展功能。
- **加载**，通过解析完的定义信息获取 Bean 实例。

![Bean加载总体流程](/blogImg/Bean加载总体流程.png)

我们假设所有的配置和扩展类都已经装载到了 ApplicationContext 中，然后具体的分析一下 Bean 的加载流程。

思考一个问题，抛开 Spring 框架的实现，假设我们手头上已经有一套完整的 Bean Definition Map，然后指定一个 beanName 要进行实例化，需要关心什么？即使我们没有 Spring 框架，也需要了解这两方面的知识：

- **作用域**。单例作用域或者原型作用域，单例的话需要全局实例化一次，原型每次创建都需要重新实例化。
- **依赖关系**。一个 Bean 如果有依赖，我们需要初始化依赖，然后进行关联。如果多个 Bean 之间存在着循环依赖，A 依赖 B，B 依赖 C，C 又依赖 A，需要解这种循环依赖问题。

Spring 进行了抽象和封装，使得作用域和依赖关系的配置对开发者透明，我们只需要知道当初在配置里已经明确指定了它的生命周期和依赖了谁，至于是怎么实现的，依赖如何注入，托付给了 Spring 工厂来管理。

Spring 只暴露了很简单的接口给调用者，比如 `getBean` 

那我们就从 `getBean` 方法作为入口，去理解 Spring 加载的流程是怎样的，以及内部对创建信息、作用域、依赖关系等等的处理细节。

## 总体流程

![Bean加载流程图](/blogImg/Bean加载流程图.png)

上面是跟踪了 getBean 的调用链创建的流程图，为了能够很好地理解 Bean 加载流程，省略一些异常、日志和分支处理和一些特殊条件的判断。

从上面的流程图中，可以看到一个 Bean 加载会经历这么几个阶段（用绿色标记）：

- **获取 BeanName**，对传入的 name 进行解析，转化为可以从 Map 中获取到 BeanDefinition 的 bean name。
- **合并 Bean 定义**，对父类的定义进行合并和覆盖，如果父类还有父类，会进行递归合并，以获取完整的 Bean 定义信息。
- **实例化**，使用构造或者工厂方法创建 Bean 实例。
- **属性填充**，寻找并且注入依赖，依赖的 Bean 还会递归调用 `getBean` 方法获取。
- **初始化**，调用自定义的初始化方法。
- **获取最终的 Bean**，如果是 FactoryBean 需要调用 getObject 方法，如果需要类型转换调用 TypeConverter 进行转化。

整个流程最为复杂的是对循环依赖的解决方案，后续会进行重点分析。

## 细节分析

### 转化 BeanName

而在我们解析完配置后创建的 Map，使用的是 beanName 作为 key。见 DefaultListableBeanFactory：

```java
/** Map of bean definition objects, keyed by bean name */
private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<String, BeanDefinition>(256);
```

`BeanFactory.getBean` 中传入的 name，有可能是这几种情况：

- **bean name**，可以直接获取到定义 BeanDefinition。
- **alias name**，别名，需要转化。
- **factorybean name**, **带 `&` 前缀**，通过它获取 BeanDefinition 的时候需要去除 & 前缀。

为了能够获取到正确的 BeanDefinition，需要先对 name 做一个转换，得到 beanName。

![name转beanName](/blogImg/name转beanName.png)

见 `AbstractBeanFactory.doGetBean`：

```java
protected <T> T doGetBean ... {
    ...  
    // 转化工作 
    final String beanName = transformedBeanName(name);
    ...
}
```

如果是 **alias name**，在解析阶段，alias name 和 bean name 的映射关系被注册到 SimpleAliasRegistry 中。从该注册器中取到 beanName。见 `SimpleAliasRegistry.canonicalName`：

```java
public String canonicalName(String name) {
    ...
    resolvedName = this.aliasMap.get(canonicalName);
    ...
}
```

如果是 **factorybean name**，表示这是个工厂 bean，有携带前缀修饰符 `&` 的，直接把前缀去掉。见 `BeanFactoryUtils.transformedBeanName` :

```java
public static String transformedBeanName(String name) {
    Assert.notNull(name, "'name' must not be null");
    String beanName = name;
    while (beanName.startsWith(BeanFactory.FACTORY_BEAN_PREFIX)) {
        beanName = beanName.substring(BeanFactory.FACTORY_BEAN_PREFIX.length());
    }
    return beanName;
}
```

### 合并 RootBeanDefinition

我们从配置文件读取到的 BeanDefinition 是 **GenericBeanDefinition**。它的记录了一些当前类声明的属性或构造参数，但是对于父类只用了一个 `parentName` 来记录。

```java
public class GenericBeanDefinition extends AbstractBeanDefinition {
    ...
    private String parentName;
    ...
}
```

接下来会发现一个问题，在后续实例化 Bean 的时候，使用的 BeanDefinition 是 **RootBeanDefinition** 类型而非 **GenericBeanDefinition**。这是为什么？

答案很明显，GenericBeanDefinition 在有继承关系的情况下，定义的信息不足：

- 如果不存在继承关系，GenericBeanDefinition 存储的信息是完整的，可以直接转化为 RootBeanDefinition。
- 如果存在继承关系，GenericBeanDefinition 存储的是 **增量信息** 而不是 **全量信息**。

**为了能够正确初始化对象，需要完整的信息才行**。需要递归 **合并父类的定义**：

![合并BeanDefinition](/blogImg/合并BeanDefinition.png)

见 `AbstractBeanFactory.doGetBean` ：

```java
protected <T> T doGetBean ... {
    ...
    // 合并父类定义
    final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName); 
    ...     
    // 使用合并后的定义进行实例化
    bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);  
    ...
}
```

在判断 `parentName` 存在的情况下，说明存在父类定义，启动合并。如果父类还有父类怎么办？递归调用，继续合并。

见`AbstractBeanFactory.getMergedBeanDefinition` 方法：

```java
    protected RootBeanDefinition getMergedBeanDefinition(
            String beanName, BeanDefinition bd, BeanDefinition containingBd)
            throws BeanDefinitionStoreException {
        ...        
        String parentBeanName = transformedBeanName(bd.getParentName());
        ...        
        // 递归调用，继续合并父类定义
        pbd = getMergedBeanDefinition(parentBeanName);        
        ...
        // 使用合并后的完整定义，创建 RootBeanDefinition
        mbd = new RootBeanDefinition(pbd);     
        // 使用当前定义，对 RootBeanDefinition 进行覆盖
        mbd.overrideFrom(bd);
        ...
        return mbd;  
    }
```

每次合并完父类定义后，都会调用 `RootBeanDefinition.overrideFrom` 对父类的定义进行覆盖，获取到当前类能够正确实例化的 **全量信息**。

### 处理循环依赖

循环依赖根据注入的时机分成两种类型：

- **构造器循环依赖**。依赖的对象是通过构造器传入的，发生在实例化 Bean 的时候。
- **设值循环依赖**。依赖的对象是通过 setter 方法传入的，对象已经实例化，发生属性填充和依赖注入的时候。

**如果是构造器循环依赖，本质上是无法解决的**。比如我们准调用 A 的构造器，发现依赖 B，于是去调用 B 的构造器进行实例化，发现又依赖 C，于是调用 C 的构造器去初始化，结果依赖 A，整个形成一个死结，导致 A 无法创建。

**如果是设值循环依赖，Spring 框架只支持单例下的设值循环依赖**。Spring 通过对还在创建过程中的单例，缓存并提前暴露该单例，使得其他实例可以引用该依赖。**非单例的设置循环依赖也无法解决**

#### 构造器循环依赖

this .singletonsCurrentlylnCreation.add(beanName）将当前正要创建的bean 记录在缓存中 Spring 容器将每一个正在创建的bean 标识符放在一个“当前创建bean 池”中， bean 标识：在创建过程中将一直保持在这个池中，因此如果在创建bean 过程中发现自己已经在“当前 创建bean 池” 里时，将抛出BeanCurrentlylnCreationException 异常表示循环依赖；而对于创建 完毕的bean 将从“ 当前创建bean 池”中清除掉。

#### 单例模式的设值循环依赖

**单例模式下，构造函数的循环依赖无法解决，但设值循环依赖是可以解决的**。

这里有一个重要的设计：**提前暴露创建中的单例**。

我们理解一下为什么要这么做。

拿 A、B、C 的的设值依赖做分析，

=>  1. A 创建 -> A 构造完成，开始注入属性，发现依赖 B，启动 B 的实例化

=>  2. B 创建 -> B 构造完成，开始注入属性，发现依赖 C，启动 C 的实例化

=>  3. C 创建 -> C 构造完成，开始注入属性，发现依赖 A

重点来了，在我们的阶段 1中， A 已经构造完成，Bean 对象在堆中也分配好内存了，即使后续往 A 中填充属性（比如填充依赖的 B 对象），也不会修改到 A 的引用地址。

所以，这个时候是否可以 **提前拿 A 实例的引用来先注入到 C** ，去完成 C 的实例化，于是流程变成这样。

=>  3. C 创建 ->  C 构造完成，开始注入依赖，发现依赖 A，发现 A 已经构造完成，直接引用，完成 C 的实例化。

=>  4. C 完成实例化后，B 注入 C 也完成实例化，A 注入 B 也完成实例化。

这就是 Spring 解决单例模式设值循环依赖应用的技巧。流程图为：

![单例模式创建流程](/blogImg/单例模式创建流程.png)

- Spring 为了解决单例的循环依赖问题，使用了 **三级缓存** ，递归调用时发现 Bean 还在创建中即为循环依赖
- 单例模式的 Bean 保存在如下的数据结构中：

```java
/** 一级缓存：用于存放完全初始化好的 bean **/
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(256);

/** 二级缓存：存放原始的 bean 对象（尚未填充属性），用于解决循环依赖 */
private final Map<String, Object> earlySingletonObjects = new HashMap<String, Object>(16);

/** 三级级缓存：存放 bean 工厂对象，用于解决循环依赖 */
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<String, ObjectFactory<?>>(16);

/**
bean 的获取过程：先从一级获取，失败再从二级、三级里面获取

创建中状态：是指对象已经 new 出来了但是所有的属性均为 null 等待被 init
*/
```

检测循环依赖的过程如下：

- A 创建过程中需要 B，于是 **A 将自己放到三级缓里面** ，去实例化 B

- B 实例化的时候发现需要 A，于是 B 先查一级缓存，没有，再查二级缓存，还是没有，再查三级缓存，找到了！

- - **然后把三级缓存里面的这个 A 放到二级缓存里面，并删除三级缓存里面的 A**
	- B 顺利初始化完毕，**将自己放到一级缓存里面**（此时B里面的A依然是创建中状态）

- 然后回来接着创建 A，此时 B 已经创建结束，直接从一级缓存里面拿到 B ，然后完成创建，**并将自己放到一级缓存里面**

- 如此一来便解决了循环依赖的问题

### 创建实例

获取到完整的 RootBeanDefintion 后，就可以拿这份定义信息来实例具体的 Bean。

具体实例创建见 `AbstractAutowireCapableBeanFactory.createBeanInstance` ，返回 Bean 的包装类 BeanWrapper，一共有三种策略：

- **使用工厂方法创建**，`instantiateUsingFactoryMethod` 。
- **使用有参构造函数创建**，`autowireConstructor`。
- **使用无参构造函数创建**，`instantiateBean`。

使用工厂方法创建，会先使用 getBean 获取工厂类，然后通过参数找到匹配的工厂方法，调用实例化方法实现实例化，具体见`ConstructorResolver.instantiateUsingFactoryMethod` ：

```java
public BeanWrapper instantiateUsingFactoryMethod ... (
    ...
    String factoryBeanName = mbd.getFactoryBeanName();
    ...
    factoryBean = this.beanFactory.getBean(factoryBeanName);
    ...
    // 匹配正确的工厂方法
    ...
    beanInstance = this.beanFactory.getInstantiationStrategy().instantiate(...);
    ...
    bw.setBeanInstance(beanInstance);
    return bw;
}
```

使用有参构造函数创建，整个过程比较复杂，涉及到参数和构造器的匹配。为了找到匹配的构造器，Spring 花了大量的工作，见 `ConstructorResolver.autowireConstructor` ：

```java
public BeanWrapper autowireConstructor ... {
    ...
    Constructor<?> constructorToUse = null;
    ...
    // 匹配构造函数的过程
    ...
    beanInstance = this.beanFactory.getInstantiationStrategy().instantiate(...);
    ...
    bw.setBeanInstance(beanInstance);
    return bw;
}           
```

使用无参构造函数创建是最简单的方式，见 `AbstractAutowireCapableBeanFactory.instantiateBean`:

```java
protected BeanWrapper instantiateBean ... {
    ...
    beanInstance = getInstantiationStrategy().instantiate(...);
    ...
    BeanWrapper bw = new BeanWrapperImpl(beanInstance);
    initBeanWrapper(bw);
    return bw;
    ...
}
```

我们发现这三个实例化方式，最后都会走 `getInstantiationStrategy().instantiate(...)`，见实现类 `SimpleInstantiationStrategy.instantiate`：

```java
public Object instantiate ... {
    if (bd.getMethodOverrides().isEmpty()) {
        ...
        return BeanUtils.instantiateClass(constructorToUse);
    }
    else {
        // Must generate CGLIB subclass.
        return instantiateWithMethodInjection(bd, beanName, owner);
    }
}
```

虽然拿到了构造函数，并没有立即实例化。因为用户使用了 replace 和 lookup 的配置方法，用到了动态代理加入对应的逻辑。如果没有的话，直接使用反射来创建实例。

创建实例后，就可以开始注入属性和初始化等操作。

但这里的 Bean 还不是最终的 Bean。返回给调用方使用时，如果是 FactoryBean 的话需要使用 getObject 方法来创建实例。见 `AbstractBeanFactory.getObjectFromBeanInstance` ，会执行到 `doGetObjectFromFactoryBean` ：

```java
private Object doGetObjectFromFactoryBean ... {
    ...
    object = factory.getObject();
    ...
    return object;
}
```

### 注入属性

实例创建完后开始进行属性的注入，如果涉及到外部依赖的实例，会自动检索并关联到该当前实例。

Ioc 思想体现出来了。正是有了这一步操作，Spring 降低了各个类之间的耦合。

属性填充的入口方法在`AbstractAutowireCapableBeanFactory.populateBean`。

```java
protected void populateBean ... {
    PropertyValues pvs = mbd.getPropertyValues();
    
    ...
    // InstantiationAwareBeanPostProcessor 前处理
    for (BeanPostProcessor bp : getBeanPostProcessors()) {
        if (bp instanceof InstantiationAwareBeanPostProcessor) {
            InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
            if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
                continueWithPropertyPopulation = false;
                break;
            }
        }
    }
    ...
    
    // 根据名称注入
    if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
        autowireByName(beanName, mbd, bw, newPvs);
    }

    // 根据类型注入
    if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
        autowireByType(beanName, mbd, bw, newPvs);
    }

    ... 
    // InstantiationAwareBeanPostProcessor 后处理
    for (BeanPostProcessor bp : getBeanPostProcessors()) {
        if (bp instanceof InstantiationAwareBeanPostProcessor) {
            InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
            pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
            if (pvs == null) {
                return;
            }
        }
    }
    
    ...
    // 应用属性值
    applyPropertyValues(beanName, mbd, bw, pvs);
}
```

可以看到主要的处理环节有：

- 应用 InstantiationAwareBeanPostProcessor 处理器，在属性注入前后进行处理。**假设我们使用了 @Autowire 注解，这里会调用到 AutowiredAnnotationBeanPostProcessor 来对依赖的实例进行检索和注入的**，它是 InstantiationAwareBeanPostProcessor 的子类。
- 根据名称或者类型进行自动注入，存储结果到 PropertyValues 中。
- 应用 PropertyValues，填充到 BeanWrapper。这里在检索依赖实例的引用的时候，会递归调用 `BeanFactory.getBean` 来获得。

### 初始化

#### 触发 Aware

如果我们的 Bean 需要容器的一些资源该怎么办？比如需要获取到 BeanFactory、ApplicationContext 等等。

Spring 提供了 Aware 系列接口来解决这个问题。比如有这样的 Aware：

- BeanFactoryAware，用来获取 BeanFactory。
- ApplicationContextAware，用来获取 ApplicationContext。
- ResourceLoaderAware，用来获取 ResourceLoaderAware。
- ServletContextAware，用来获取 ServletContext。

Spring 在初始化阶段，如果判断 Bean 实现了这几个接口之一，就会往 Bean 中注入它关心的资源。

见 `AbstractAutowireCapableBeanFactory.invokeAwareMethos` :

```java
private void invokeAwareMethods(final String beanName, final Object bean) {
    if (bean instanceof Aware) {
        if (bean instanceof BeanNameAware) {
            ((BeanNameAware) bean).setBeanName(beanName);
        }
        if (bean instanceof BeanClassLoaderAware) {
            ((BeanClassLoaderAware) bean).setBeanClassLoader(getBeanClassLoader());
        }
        if (bean instanceof BeanFactoryAware) {
            ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
        }
    }
}
```

#### 触发 BeanPostProcessor

在 Bean 的初始化前或者初始化后，我们如果需要进行一些增强操作怎么办？

这些增强操作比如打日志、做校验、属性修改、耗时检测等等。Spring 框架提供了 BeanPostProcessor 来达成这个目标。比如我们使用注解 @Autowire 来声明依赖，就是使用  `AutowiredAnnotationBeanPostProcessor` 来实现依赖的查询和注入的。接口定义如下：

```tsx
public interface BeanPostProcessor {

    // 初始化前调用
    Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;

    // 初始化后调用
    Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;

}
```

**实现该接口的 Bean 都会被 Spring 注册到 beanPostProcessors 中，**见 `AbstractBeanFactory` :

```java
/** BeanPostProcessors to apply in createBean */
private final List<BeanPostProcessor> beanPostProcessors = new ArrayList<BeanPostProcessor>();
```

只要 Bean 实现了 BeanPostProcessor 接口，加载的时候会被 Spring 自动识别这些 Bean，自动注册，非常方便。

然后在 Bean 实例化前后，Spring 会去调用我们已经注册的 beanPostProcessors 把处理器都执行一遍。

```java
public abstract class AbstractAutowireCapableBeanFactory ... {
        
    ...
    
    @Override
    public Object applyBeanPostProcessorsBeforeInitialization ... {

        Object result = existingBean;
        for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
            result = beanProcessor.postProcessBeforeInitialization(result, beanName);
            if (result == null) {
                return result;
            }
        }
        return result;
    }

    @Override
    public Object applyBeanPostProcessorsAfterInitialization ... {

        Object result = existingBean;
        for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
            result = beanProcessor.postProcessAfterInitialization(result, beanName);
            if (result == null) {
                return result;
            }
        }
        return result;
    }
    
    ...
}
```

这里使用了责任链模式，Bean 会在处理器链中进行传递和处理。当我们调用 `BeanFactory.getBean` 的后，执行到 Bean 的初始化方法 `AbstractAutowireCapableBeanFactory.initializeBean` 会启动这些处理器。

```java
protected Object initializeBean ... {   
    ...
    wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    ...
    // 触发自定义 init 方法
    invokeInitMethods(beanName, wrappedBean, mbd);
    ...
    wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    ...
}
```

#### 触发自定义 init

自定义初始化有两种方式可以选择：

- 实现 InitializingBean。提供了一个很好的机会，在属性设置完成后再加入自己的初始化逻辑。
- 定义 init 方法。自定义的初始化逻辑。

见 `AbstractAutowireCapableBeanFactory.invokeInitMethods` ：

```java
    protected void invokeInitMethods ... {

        boolean isInitializingBean = (bean instanceof InitializingBean);
        if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
            ...
            
            ((InitializingBean) bean).afterPropertiesSet();
            ...
        }

        if (mbd != null) {
            String initMethodName = mbd.getInitMethodName();
            if (initMethodName != null && !(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
                    !mbd.isExternallyManagedInitMethod(initMethodName)) {
                invokeCustomInitMethod(beanName, bean, mbd);
            }
        }
    }
```

### 类型转换

Bean 已经加载完毕，属性也填充好了，初始化也完成了。

在返回给调用者之前，还留有一个机会对 Bean 实例进行类型的转换。见 `AbstractBeanFactory.doGetBean` ：

```kotlin
protected <T> T doGetBean ... {
    ...
    if (requiredType != null && bean != null && !requiredType.isInstance(bean)) {
        ...
        return getTypeConverter().convertIfNecessary(bean, requiredType);
        ...
    }
    return (T) bean;
}
```

# Bean的生命周期

## Bean的生命周期概要

![bean的生命周期](/blogImg/bean的生命周期.png)

Bean 的生命周期概括起来就是 **4 个阶段**：

1. 实例化：第 1 步，实例化一个 bean 对象；
2. 属性赋值：第 2 步，为 bean 设置相关属性和依赖；
3. 初始化：第 3~7 步，步骤较多，其中第 5、6 步为初始化操作，第 3、4 步为在初始化前执行，第 7 步在初始化后执行，该阶段结束，才能被用户使用；
4. 销毁：第 8~10步，第8步不是真正意义上的销毁（还没使用呢），而是先在使用前注册了销毁的相关调用接口，为了后面第9、10步真正销毁 bean 时再执行相应的方法。

下面我们结合代码来直观的看下，在 doCreateBean() 方法中能看到依次执行了这 4 个阶段：

```java
// AbstractAutowireCapableBeanFactory.java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
    throws BeanCreationException {

    // 1. 实例化
    BeanWrapper instanceWrapper = null;
    if (instanceWrapper == null) {
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    
    Object exposedObject = bean;
    try {
        // 2. 属性赋值
        populateBean(beanName, mbd, instanceWrapper);
        // 3. 初始化
        exposedObject = initializeBean(beanName, exposedObject, mbd);
    }

    // 4. 销毁-注册回调接口
    try {
        registerDisposableBeanIfNecessary(beanName, bean, mbd);
    }

    return exposedObject;
}
```

由于初始化包含了第 3~7步，较复杂，所以我们进到 initializeBean() 方法里具体看下其过程（注释的序号对应图中序号）：

```java
// AbstractAutowireCapableBeanFactory.java
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
    // 3. 检查 Aware 相关接口并设置相关依赖
    if (System.getSecurityManager() != null) {
        AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
            invokeAwareMethods(beanName, bean);
            return null;
        }, getAccessControlContext());
    }
    else {
        invokeAwareMethods(beanName, bean);
    }

    // 4. BeanPostProcessor 前置处理
    Object wrappedBean = bean;
    if (mbd == null || !mbd.isSynthetic()) {
        wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    }

    // 5. 若实现 InitializingBean 接口，调用 afterPropertiesSet() 方法
    // 6. 若配置自定义的 init-method方法，则执行
    try {
        invokeInitMethods(beanName, wrappedBean, mbd);
    }
    catch (Throwable ex) {
        throw new BeanCreationException(
            (mbd != null ? mbd.getResourceDescription() : null),
            beanName, "Invocation of init method failed", ex);
    }
    // 7. BeanPostProceesor 后置处理
    if (mbd == null || !mbd.isSynthetic()) {
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }

    return wrappedBean;
}
```

在 invokInitMethods() 方法中会检查 InitializingBean 接口和 init-method 方法，销毁的过程也与其类似：

```java
// DisposableBeanAdapter.java
public void destroy() {
    // 9. 若实现 DisposableBean 接口，则执行 destory()方法
    if (this.invokeDisposableBean) {
        try {
            if (System.getSecurityManager() != null) {
                AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {
                    ((DisposableBean) this.bean).destroy();
                    return null;
                }, this.acc);
            }
            else {
                ((DisposableBean) this.bean).destroy();
            }
        }
    }
    
	// 10. 若配置自定义的 detory-method 方法，则执行
    if (this.destroyMethod != null) {
        invokeCustomDestroyMethod(this.destroyMethod);
    }
    else if (this.destroyMethodName != null) {
        Method methodToInvoke = determineDestroyMethod(this.destroyMethodName);
        if (methodToInvoke != null) {
            invokeCustomDestroyMethod(ClassUtils.getInterfaceMethodIfPossible(methodToInvoke));
        }
    }
}
```

从 Spring 的源码我们可以直观的看到其执行过程，而我们记忆其过程便可以从这 4 个阶段出发，实例化、属性赋值、初始化、销毁。其中细节较多的便是初始化，涉及了 Aware、BeanPostProcessor、InitializingBean、init-method 的概念。这些都是 Spring 提供的扩展点。

## 扩展点的作用

### Aware 接口

若 Spring 检测到 bean 实现了 Aware 接口，则会为其注入相应的依赖。所以**通过让bean 实现 Aware 接口，则能在 bean 中获得相应的 Spring 容器资源**。

Spring 中提供的 Aware 接口有：

1. BeanNameAware：注入当前 bean 对应 beanName；
2. BeanClassLoaderAware：注入加载当前 bean 的 ClassLoader；
3. BeanFactoryAware：注入 当前BeanFactory容器 的引用。

其代码实现如下：

```java
// AbstractAutowireCapableBeanFactory.java
private void invokeAwareMethods(final String beanName, final Object bean) {
    if (bean instanceof Aware) {
        if (bean instanceof BeanNameAware) {
            ((BeanNameAware) bean).setBeanName(beanName);
        }
        if (bean instanceof BeanClassLoaderAware) {
            ((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
            
        }
        if (bean instanceof BeanFactoryAware) {
            ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
        }
    }
}
```

以上是针对 BeanFactory 类型的容器，而对于 ApplicationContext 类型的容器，也提供了 Aware 接口，只不过这些 Aware 接口的注入实现，是通过 BeanPostProcessor 的方式注入的，但其作用仍是注入依赖。

1. EnvironmentAware：注入 Enviroment，一般用于获取配置属性；
2. EmbeddedValueResolverAware：注入 EmbeddedValueResolver（Spring EL解析器），一般用于参数解析；
3. ApplicationContextAware（ResourceLoader、ApplicationEventPublisherAware、MessageSourceAware）：注入 ApplicationContext 容器本身。

其代码实现如下：

```java
// ApplicationContextAwareProcessor.java
private void invokeAwareInterfaces(Object bean) {
    if (bean instanceof EnvironmentAware) {
        ((EnvironmentAware)bean).setEnvironment(this.applicationContext.getEnvironment());
    }

    if (bean instanceof EmbeddedValueResolverAware) {
        ((EmbeddedValueResolverAware)bean).setEmbeddedValueResolver(this.embeddedValueResolver);
    }

    if (bean instanceof ResourceLoaderAware) {
        ((ResourceLoaderAware)bean).setResourceLoader(this.applicationContext);
    }

    if (bean instanceof ApplicationEventPublisherAware) {
        ((ApplicationEventPublisherAware)bean).setApplicationEventPublisher(this.applicationContext);
    }

    if (bean instanceof MessageSourceAware) {
        ((MessageSourceAware)bean).setMessageSource(this.applicationContext);
    }

    if (bean instanceof ApplicationContextAware) {
        ((ApplicationContextAware)bean).setApplicationContext(this.applicationContext);
    }
}
```

### BeanPostProcessor

BeanPostProcessor 是 Spring 为**修改 bean**提供的强大扩展点，其可作用于容器中所有 bean，其定义如下：

```java
public interface BeanPostProcessor {

	// 初始化前置处理
	default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}

	// 初始化后置处理
	default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}
}
```

常用场景有：

1. 对于标记接口的实现类，进行自定义处理。例如上面所说的ApplicationContextAwareProcessor，为其注入相应依赖；再举个例子，自定义对实现解密接口的类，将对其属性进行解密处理；
2. 为当前对象提供代理实现。例如 Spring AOP 功能，生成对象的代理类，然后返回。

```java
// AbstractAutoProxyCreator.java
public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
    TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
    if (targetSource != null) {
        if (StringUtils.hasLength(beanName)) {
            this.targetSourcedBeans.add(beanName);
        }
        Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
        Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
        this.proxyTypes.put(cacheKey, proxy.getClass());
        // 返回代理类
        return proxy;
    }
    return null;
}
```

### InitializingBean 和 init-method

InitializingBean 和 init-method 是 Spring 为 **bean 初始化**提供的扩展点。

InitializingBean接口 的定义如下：

```java
public interface InitializingBean {
	void afterPropertiesSet() throws Exception;
}
```

在 afterPropertiesSet() 方法写初始化逻辑。

指定 init-method 方法，指定初始化方法：

```java
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="demo" class="com.chaycao.Demo" init-method="init()"/>
</beans>
```

DisposableBean 和 destory-method 与上述类似，就不描述了。