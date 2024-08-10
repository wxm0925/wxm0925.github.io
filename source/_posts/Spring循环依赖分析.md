---
title: Spring循环依赖分析
category: Spring
---



## Spring的解释

> ```
> If you use predominantly constructor injection, it is possible to create an unresolvable circular dependency scenario.
> 
> For example: Class A requires an instance of class B through constructor injection, and class B requires an instance of class A through constructor injection. If you configure beans for classes A and B to be injected into each other, the Spring IoC container detects this circular reference at runtime, and throws a BeanCurrentlyInCreationException.
> 
> One possible solution is to edit the source code of some classes to be configured by setters rather than constructors. Alternatively, avoid constructor injection and use setter injection only. In other words, although it is not recommended, you can configure circular dependencies with setter injection.
> 
> Unlike the typical case (with no circular dependencies), a circular dependency between bean A and bean B forces one of the beans to be injected into the other prior to being fully initialized itself (a classic chicken-and-egg scenario).
> ```

如果主要使用构造器注入，就有可能创建一个无法解决的循环依赖场景。

例如：类 A 通过构造器注入需要一个类 B 的实例，而类 B 通过构造器注入又需要一个类 A 的实例。如果你配置了类 A 和类 B 的 Bean 互相注入，Spring IoC 容器会在运行时检测到这个循环引用，并抛出 `BeanCurrentlyInCreationException` 异常。

一种可能的解决方案是将某些类的代码修改为通过 Setter 方法进行配置，而不是使用构造器。或者，可以避免使用构造器注入，改用 Setter 注入。换句话说，尽管不推荐，但你可以通过 Setter 注入来配置循环依赖。

与典型情况（没有循环依赖）不同，Bean A 和 Bean B 之间的循环依赖会迫使其中一个 Bean 在自身尚未完全初始化之前就被注入到另一个 Bean 中（这就像一个经典的“先有鸡还是先有蛋”的问题）。





## 循环依赖的场景

> https://developer.aliyun.com/article/766880

| 依赖情况               | 依赖注入方式                                        | 循环依赖是否被解决 |
| ---------------------- | --------------------------------------------------- | ------------------ |
| AB相互依赖（循环依赖） | 均采用Setter 方法注入                               | 是                 |
| AB相互依赖（循环依赖） | 均采用构造器注入                                    | 否                 |
| AB相互依赖（循环依赖） | A中注入B的方式为Setter 方法，B中注入A的方式为构造器 | 是                 |
| AB相互依赖（循环依赖） | B中注入A的方式为Setter 方法，A中注入B的方式为构造器 | 否                 |



## 三级缓存

所谓的三级缓存就是`beanFactory`对象中的3个实例变量。一级二级三级是怎么命名的？我理解是先从哪个缓存中获取，哪个就是一级，以此类推

- 一级缓存：存放初始化完成的对象，如果被代理了，就是被代理对象

  ```java
  /** Cache of singleton objects: bean name to bean instance. */
  private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
  
  ```

- 二级缓存：不知道怎么描述

  ```java
  /** Cache of early singleton objects: bean name to bean instance. */
  private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);
  ```
- 三级缓存：不知道怎么描述

    ```java
        /** Cache of singleton factories: bean name to ObjectFactory. */
        private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
```



## 流程分析

- 配置一个循环依赖的场景，都使用Setter注入

```xml
<bean id="userService" class="cyclereference.UserService">
    <property name="orderService" ref="orderService"/>
</bean>
<bean id="orderService" class="cyclereference.OrderService">
    <property name="userService" ref="userService"></property>
</bean>
```
- 流程图如下
![循环依赖流程图 (3)](/images/循环依赖流程图.png)

​		首先创建对象A，调用`getSingleton`方法从三级缓存中检查是否存在，`getSingleton`源码如下，

```java
/**
	 * Return the (raw) singleton object registered under the given name.
	 * <p>Checks already instantiated singletons and also allows for an early
	 * reference to a currently created singleton (resolving a circular reference).
	 * @param beanName the name of the bean to look for
	 * @param allowEarlyReference whether early references should be created or not
	 * @return the registered singleton object, or {@code null} if none found
	 */
	@Nullable
	protected Object getSingleton(String beanName, boolean allowEarlyReference) {
		// Quick check for existing instance without full singleton lock
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			singletonObject = this.earlySingletonObjects.get(beanName);
			if (singletonObject == null && allowEarlyReference) {
				synchronized (this.singletonObjects) {
					// Consistent creation of early reference within full singleton lock
					singletonObject = this.singletonObjects.get(beanName);
					if (singletonObject == null) {
						singletonObject = this.earlySingletonObjects.get(beanName);
						if (singletonObject == null) {
							ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
							if (singletonFactory != null) {
								singletonObject = singletonFactory.getObject();
								this.earlySingletonObjects.put(beanName, singletonObject);
								this.singletonFactories.remove(beanName);
							}
						}
					}
				}
			}
		}
		return singletonObject;
	}
```

刚加载肯定是不存在的，代码往下走，接着创建一个bean实例（可以理解成直接newInstance），并调用`addSingletonFacotory`添加到第三级缓存中，`addSingletonFacotory`源码如下，

```java
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isTraceEnabled()) {
				logger.trace("急切地缓存bean： '" + beanName +
						"' 以便解析潜在的循环引用");
			}
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}
```

```java
	/**
	 * Obtain a reference for early access to the specified bean,
	 * typically for the purpose of resolving a circular reference.
	 * @param beanName the name of the bean (for error handling purposes)
	 * @param mbd the merged bean definition for the bean
	 * @param bean the raw bean instance
	 * @return the object to expose as bean reference
	 */
	protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
		Object exposedObject = bean;
		if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			for (SmartInstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().smartInstantiationAware) {
				exposedObject = bp.getEarlyBeanReference(exposedObject, beanName);
			}
		}
		return exposedObject;
	}
```

然后发现需要注入属性b，b需要创建，返回来调用getBean(b)，到此为止，创建对象A的流程暂停，转而去创建依赖对象b，创建b时走的A的老路，发现需要注入a了，这时候循环依赖就体现出来了，好，继续从头开始创建a，再次创建a时，调用`getSingleton(beanName,true)`时，三级缓存`singletonFactories`中已经存在一个key为a的value（注意此时的a是没有初始化的），于是获取到后返回，继续执行getBean(b)剩下的流程（填充其它属性，初始化，添加到一级缓存中），b流程走完了回到getBean(a)，也是填充其它属性，初始化，添加到一级缓存中，至此，两个bean创建完成，循环依赖得以解决。



## 循环依赖的场景分析

正如开头`spring`引用所说，建议使用setter方法注入，可以解决循环依赖，如果使用构造函数注入，是无法解决的，原因就是在创建bean实例阶段（newinstance）就会去查找并创建依赖对象，而当前对象还没有存入第三级缓存中，因此无法解决。



## `AOP`代理

- 如果A和B都需要被代理怎么办，比如开启了声明式事务？

关键点在于3级缓存，三级缓存存的是`() -> getEarlyBeanReference(beanName, mbd, bean)`，源码如上，如果当前`BeanFactory`中存在`InstantiationAwareBeanPostProcessors`，就调用它的返回bean方法，这个`InstantiationAwareBeanPostProcessors`就是`AOP`用来自动创建代理的bean处理器，于`initializationBean`方法中被应用。

当创建B时需要a依赖时（流程图中的第三个流程），会调用返回代理对象，然后把它存入二级缓存，b拿到的a就是一个代理对象。

流程第五步初始化好的a还是原始对象，在放入一级缓存之前，有这样一段逻辑

```java
if (earlySingletonExposure) {
			Object earlySingletonReference = getSingleton(beanName, false);
			if (earlySingletonReference != null) {
				if (exposedObject == bean) {
					exposedObject = earlySingletonReference;
				}
				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
					String[] dependentBeans = getDependentBeans(beanName);
					Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
					for (String dependentBean : dependentBeans) {
						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
							actualDependentBeans.add(dependentBean);
						}
					}
					if (!actualDependentBeans.isEmpty()) {
						throw new BeanCurrentlyInCreationException(beanName,
								"Bean with name '" + beanName + "' has been injected into other beans [" +
								StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
								"] in its raw version as part of a circular reference, but has eventually been " +
								"wrapped. This means that said other beans do not use the final version of the " +
								"bean. This is often the result of over-eager type matching - consider using " +
								"'getBeanNamesForType' with the 'allowEagerInit' flag turned off, for example.");
					}
				}
			}
		}
```

`getSingleton(a,false)`只从一级和二级缓存中拿，二级缓存中是有a的代理对象的，因此替换后，存到一级缓存中。









源码版本 5.3.16.RELEASE