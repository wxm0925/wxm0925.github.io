# 为什么可以AutoWired注入HttpServletRequest对象？



## 背景

时间：2024年12月22日13:54:12，周日

这两天在做一个需求的时候，需要在service中使用http调用其他系统的查询接口，调用的时候需要登录态，方案是获取当前request对象，把所有cookie都传过去。于是看看service参数中能不能取到request对象，发现取不到，最后师兄告诉我可以使用AutoWired直接注入HttpServletRequest对象。

问题来了，request应该是和当前请求线程绑定的，给个请求，request中的数据应该都不一样，为什么可以注入呢？带着这个疑问，今天打算从spring源码的角度，去理解这个问题



## 测试代码

```java
package com.wxm.controller;

import org.springframework.beans.factory.InitializingBean;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;

import javax.servlet.http.HttpServletRequest;

/**
 * @author wenxiangmin
 * @ClassName HelloController.java
 * @Description TODO
 * @createTime 2024年04月21日 19:57:00
 */
@Controller
public class HelloController implements InitializingBean {
	@Autowired
	private HttpServletRequest request;
	@Autowired
	private TestComp testComp;
	@GetMapping("hello")
	public String hello () {
		System.out.println(request.toString());
		System.out.println(request.getRequestURI());
		return "index";
	}

	@Override
	public void afterPropertiesSet() throws Exception {
		System.out.println("request=========" + request);
	}
}

```



## debug过程

### 启动阶段



![image-20241222140449507](/images/image-20241222140449507.png)

首先启动tomcat，在DispatcherServlet中会创建容器，然后创建单例bean，比如上面的helloController，newInstance完成后，会调用populateBean方法，这个方法是用来注入依赖的。接着往下看



![image-20241222141542096](/images/image-20241222141542096.png)

断点停在上面，从上图得出以下信息

AutoWiredAnnotationBeanPostProcessor会调用beanFactory#resolveDependency方法（实际上是DefaultListableBeanfactory）来解析依赖，最终返回我们想要的对象

接下来进DefaultListableBeanfactory的resolveDependency方法看看

![image-20241222142719360](/images/image-20241222142719360.png)

①：经过一个内部方法后，进入到findAutowireCandidates方法

②：DefaultListableBeanfactory中有一个resolvableDependencies属性，是一个Map，获取内部的entrySet进行循环，key是一个Class对象，如果与要注入的对象的类型匹配的话，就获取对应的value，然后通过AutowireUtils.resolveAutowiringValue(autowiringValue, requiredType)生成代理对象返回。代码如下



```java
//关键代码
		for (Map.Entry<Class<?>, Object> classObjectEntry : this.resolvableDependencies.entrySet()) {
			Class<?> autowiringType = classObjectEntry.getKey();
			if (autowiringType.isAssignableFrom(requiredType)) {
                //autowiringValue是WebApplicationContextUtils$RequestObjectFactory,实现自ObjectFactory
				Object autowiringValue = classObjectEntry.getValue();
                //生成代理对象
				autowiringValue = AutowireUtils.resolveAutowiringValue(autowiringValue, requiredType);
				if (requiredType.isInstance(autowiringValue)) {
					result.put(ObjectUtils.identityToString(autowiringValue), autowiringValue);
					break;
				}
			}
		}
```



```java
	/**
		
	 * Resolve the given autowiring value against the given required type,
	 * e.g. an {@link ObjectFactory} value to its actual object result.
	 * @param autowiringValue the value to resolve
	 * @param requiredType the type to assign the result to
	 * @return the resolved value
	 */
	public static Object resolveAutowiringValue(Object autowiringValue, Class<?> requiredType) {
		if (autowiringValue instanceof ObjectFactory && !requiredType.isInstance(autowiringValue)) {
			ObjectFactory<?> factory = (ObjectFactory<?>) autowiringValue;
			if (autowiringValue instanceof Serializable && requiredType.isInterface()) {
				autowiringValue = Proxy.newProxyInstance(requiredType.getClassLoader(),
						new Class<?>[] {requiredType}, new ObjectFactoryDelegatingInvocationHandler(factory));
			}
			else {
				return factory.getObject();
			}
		}
		return autowiringValue;
	}
```



AutoWiredUtis$ObjectFactoryDelegatingInvocationHandler

```java
	/**
	 * Reflective {@link InvocationHandler} for lazy access to the current target object.
	 */
	@SuppressWarnings("serial")
	private static class ObjectFactoryDelegatingInvocationHandler implements InvocationHandler, Serializable {

		private final ObjectFactory<?> objectFactory;

		ObjectFactoryDelegatingInvocationHandler(ObjectFactory<?> objectFactory) {
			this.objectFactory = objectFactory;
		}

		@Override
		public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
			switch (method.getName()) {
				case "equals":
					// Only consider equal when proxies are identical.
					return (proxy == args[0]);
				case "hashCode":
					// Use hashCode of proxy.
					return System.identityHashCode(proxy);
				case "toString":
					return this.objectFactory.toString();
			}
			try {
				return method.invoke(this.objectFactory.getObject(), args);
			}
			catch (InvocationTargetException ex) {
				throw ex.getTargetException();
			}
		}
	}
```



```java
@FunctionalInterface
public interface ObjectFactory<T> {

	/**
	 * Return an instance (possibly shared or independent)
	 * of the object managed by this factory.
	 * @return the resulting instance
	 * @throws BeansException in case of creation errors
	 */
	T getObject() throws BeansException;

}
```

至此，把依赖的对象解析成一个代理对象注入到了helloController中了。





还有一个疑问，DefaultListableBeanfactory中有一个resolvableDependencies属性是干嘛的，在什么时候初始化？因为requestObjectFactory是从resolvableDependencies拿到的，搞清楚很有必要

继续debug

![image-20241222151442859](/images/image-20241222151442859.png)

断点打到resolvableDependencies的put方法处，发现很简单，tomcat启动初始化唯一的DispatcherServlet，DispatcherServlet会创建IOC容器，调用AbstractApplicationContext的refresh方法中，会准备BeanFactory

![image-20241222151841564](/images/image-20241222151841564.png)

在这个方法中，会注册一些对象

![image-20241222152356397](/images/image-20241222152356397.png)

还没到我们的RequestObjectFactory对象

refresh往下走

![image-20241222152638182](/images/image-20241222152638182.png)

到了postProcessBeanFactory，之前已经new了DefaultListableBeanfactory对象了，这个方法会做一下后置处理，并且是给子类实现的（当前抽象类是空方法），因为我们是web环境，所以进去后如下图，红框中会注册web环境中相关的scope对象。

![image-20241222153003921](/images/image-20241222153003921.png)

```java
	public static void registerWebApplicationScopes(ConfigurableListableBeanFactory beanFactory,
			@Nullable ServletContext sc) {

		beanFactory.registerScope(WebApplicationContext.SCOPE_REQUEST, new RequestScope());
		beanFactory.registerScope(WebApplicationContext.SCOPE_SESSION, new SessionScope());
		if (sc != null) {
			ServletContextScope appScope = new ServletContextScope(sc);
			beanFactory.registerScope(WebApplicationContext.SCOPE_APPLICATION, appScope);
			// Register as ServletContext attribute, for ContextCleanupListener to detect it.
			sc.setAttribute(ServletContextScope.class.getName(), appScope);
		}
		//就是这个对象
		beanFactory.registerResolvableDependency(ServletRequest.class, new RequestObjectFactory());
		beanFactory.registerResolvableDependency(ServletResponse.class, new ResponseObjectFactory());
		beanFactory.registerResolvableDependency(HttpSession.class, new SessionObjectFactory());
		beanFactory.registerResolvableDependency(WebRequest.class, new WebRequestObjectFactory());
		if (jsfPresent) {
			FacesDependencyRegistrar.registerFacesDependencies(beanFactory);
		}
	}
```



### 请求阶段

我们知道helloController已经注入了一个代理对象，所以在controller使用System.out.println(request.getRequestURI());时，会先执行代理逻辑：调用被代理对象的getObject方法，看下面代码就知道了

```java
/**  切面
	 * Reflective {@link InvocationHandler} for lazy access to the current target object.
	 */
	@SuppressWarnings("serial")
	private static class ObjectFactoryDelegatingInvocationHandler implements InvocationHandler, Serializable {
		//被代理对象
		private final ObjectFactory<?> objectFactory;

		ObjectFactoryDelegatingInvocationHandler(ObjectFactory<?> objectFactory) {
			this.objectFactory = objectFactory;
		}

		@Override
		public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
			switch (method.getName()) {
				case "equals":
					// Only consider equal when proxies are identical.
					return (proxy == args[0]);
				case "hashCode":
					// Use hashCode of proxy.
					return System.identityHashCode(proxy);
				case "toString":
					return this.objectFactory.toString();
			}
			try {
                //执行之前，调用被代理对象的getObject方法，从RequestContextHolder中获取真实request对象，然后执行getRequestURI方法，
				return method.invoke(this.objectFactory.getObject(), args);
			}
			catch (InvocationTargetException ex) {
				throw ex.getTargetException();
			}
		}
	}
```



## 延伸思考

0、RequestContextHolder是用ThreadLocal存储当前线程的request的，当前请求新开了一个线程，还能拿到request吗？

1、将lambda表达式或者匿名内部类参数化，传递到方法中，可以延迟执行代码

2、ObjectFactory在spring中的作用，或者说与aop的作用，bean加载的作用

3、jdk动态代理的参数？ 有类加载器、接口的Class数组、InvocationHandler（方法调用处理器，相当于切面）

4、在spring中，InvocationHandler和切面的关系是什么

