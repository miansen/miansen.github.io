---
layout: post
title: ApplicationContext 发布事件报错：Caused by: java.lang.IllegalStateException: ApplicationEventMulticaster not initialized - call 'refresh' before multicasting events via the context: Root WebApplicationContext: startup date [Sat Dec 14 15:02:30 CST 2019]; root of context hierarchy
date: 2019-12-16
categories: SpringMVC
tags: ApplicationContext 事件
author: 龙德
---

* content
{:toc}

**报错信息如下：**

```java
Caused by: java.lang.IllegalStateException: ApplicationEventMulticaster not initialized - call 'refresh' before multicasting events via the context: Root WebApplicationContext: startup date [Sat Dec 14 15:02:30 CST 2019]; root of context hierarchy
	at org.springframework.context.support.AbstractApplicationContext.getApplicationEventMulticaster(AbstractApplicationContext.java:344)
	at org.springframework.context.support.AbstractApplicationContext.publishEvent(AbstractApplicationContext.java:331)
	at wang.miansen.roothub.common.dao.jdbc.spring.DataSourceConfiguration.afterPropertiesSet(DataSourceConfiguration.java:163)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.invokeInitMethods(AbstractAutowireCapableBeanFactory.java:1633)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.initializeBean(AbstractAutowireCapableBeanFactory.java:1570)
	... 132 more
```

**抛异常的地方：**

![](https://i.loli.net/2019/12/16/UjzJ7BnChtMVxfc.png)

可以看到 applicationEventMulticaster 为 null 就抛出了异常。

**代码如下**：

```java
package wang.miansen.roothub.common.dao.jdbc.spring;

public class DataSourceConfiguration implements FactoryBean<DataSource>, ApplicationContextAware, InitializingBean {

	private static final Logger logger = LoggerFactory.getLogger(DataSourceConfiguration.class);

	/**
	 * 数据库基本配置
	 */
	private DataSourceProperties dataSourceProperties;

	/**
	 * 数据源初始化器
	 */
	private DataSourceInitializer dataSourceInitializer;

	/**
	 * 数据源
	 */
	private DataSource dataSource;

	/**
	 * 上下文容器，主要的作用是防止重复初始化数据源。
	 */
	private ApplicationContext applicationContext;

	/**
	 * 创建数据源
	 * @param type
	 * @return
	 */
	@SuppressWarnings("unchecked")
	protected <T> T createDataSource(Class<? extends DataSource> type) {
		return (T) dataSourceProperties.initializeDataSourceBuilder().type(type).build();

	}

    /**
     * 最终注入的数据源
     */
	@Override
	public DataSource getObject() throws Exception {
		return null;
	}

	@Override
	public Class<DataSource> getObjectType() {
		return DataSource.class;
	}

	@Override
	public boolean isSingleton() {
		return true;
	}

	
	public void setDataSourceProperties(DataSourceProperties dataSourceProperties) {
		this.dataSourceProperties = dataSourceProperties;
	}

	public DataSourceInitializer getDataSourceInitializer() {
		if (this.dataSourceInitializer == null) {
			this.dataSourceInitializer = new DataSourceInitializer(this.dataSource, this.dataSourceProperties);
		}
		return this.dataSourceInitializer;
	}

	@Override
	public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
		this.applicationContext = applicationContext;
	}

	@Override
	public void afterPropertiesSet() throws Exception {
		// 只存在父容器时才初始化数据源，防止重复初始化。
		if (this.applicationContext.getParent() == null) {
			DataSourceInitializer initializer = getDataSourceInitializer();
			this.applicationContext.publishEvent(new DataSourceSchemaCreatedEvent(initializer));
		}
	}

}

```

DataSourceConfiguration 类实现了 ApplicationContextAware 和 InitializingBean 接口，本来想在 afterPropertiesSet 方法里发布初始化数据源的事件，结果就报了上面那个错误。

**分析一下原因：**

在 afterPropertiesSet() 方法中发布事件时，ApplicationContext 的 applicationEventMulticaster 属性为 null，导致抛出异常

跟一下 ApplicationContext 初始化过程看看为什么 applicationEventMulticaster 会是 null。

Spring 中 AbstractApplicationContext 抽象类的 refresh() 方法是用来刷新 Spring 的应用上下文的，所以在 refresh() 方法处打个断点。

afterPropertiesSet() 方法也打上一个断点，然后启动项目。

进入了 refresh() 方法

![image](https://i.loli.net/2019/12/14/DMau9tqQLIK3vhH.jpg)

其它的方法先不看，重点是 registerBeanPostProcessors(beanFactory) 方法。

这个方法是用来注册 BeanPostProcessor 的，需要在所有的 application bean 初始化之前调用，拦截 Bean 的创建。

![](https://i.loli.net/2019/12/14/b8HTS7CWNfojXde.jpg)

进去看看

![](https://i.loli.net/2019/12/14/O8I1eLNyTGkFnRK.jpg)

调用了一个静态方法，继续往下走。

![](https://i.loli.net/2019/12/14/kxeG1Hn9F6drs8B.jpg)

这里是找出 BeanPostProcessor 接口的所有实现类。就不仔细看了，直接看这里：

![](https://i.loli.net/2019/12/14/CP3b9Y2hJ5umdfX.jpg)

可以看到这里调用了 beanFactory.getBean() 方法，触发了 bean 实例的创建。从而触发 setApplicationContext() 和 afterPropertiesSet() 方法。

但是初始化事件广播器的方法 initApplicationEventMulticaster() 是在 registerBeanPostProcessors() 方法之后。

![](https://i.loli.net/2019/12/14/FwKtPj8J3d9AOfy.jpg)

所以导致在 afterPropertiesSet() 方法中发布事件时，ApplicationContext 的 applicationEventMulticaster 属性为 null，导致抛出异常。

![](https://i.loli.net/2019/12/14/jdHKTzeSvrGXxJy.jpg)

![](https://i.loli.net/2019/12/14/8aAeMBSDfvuF2Yp.jpg)

![](https://i.loli.net/2019/12/14/ONT715H4Jn6PjB3.jpg)

**解决方法：**

既然执行 afterPropertiesSet() 方法时，ApplicationContext 的 applicationEventMulticaster 属性还没准备好，那么可以等 initApplicationEventMulticaster() 方法执行完之后才能发布事件了。

继续往下看，注册监听器的方法是 registerListeners();

![](https://i.loli.net/2019/12/14/hw52siMmz4uap3y.jpg)

那么可以换一种思路，当监听器注册完毕之后在发布事件，因为注册监听器的方法是在初始化事件广播器之后，可以保证在发布事件的时候事件广播器是可用的。

可以这么做，新建一个类，实现 BeanPostProcessor 和 ApplicationContextAware 接口，在 postProcessAfterInitialization() 方法里判断 bean 的类型是否为 DataSourceInitializerListener，如果是就发布事件，具体代码如下：

```java
public class DataSourceInitializedPublisher implements BeanPostProcessor, ApplicationContextAware {

	/**
	 * 上下文容器，可以使用该对象来发布事件。
	 */
	private ApplicationContext applicationContext;

	/**
	 * 数据源
	 */
	private DataSource dataSource;

	/**
	 * 数据库基本配置
	 */
	private DataSourceProperties dataSourceProperties;

	/**
	 * 数据源初始化器
	 */
	private DataSourceInitializer dataSourceInitializer;
	
	@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}

	@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		// 当事件监听器初始化好之后发布初始化数据源的事件。
		if (bean instanceof DataSourceInitializerListener) {
			// 只存在父容器时才发布，防止重复初始化。
			if (this.applicationContext.getParent() == null) {
				DataSourceInitializer initializer = getDataSourceInitializer();
				this.applicationContext.publishEvent(new DataSourceSchemaCreatedEvent(initializer));
			}
		}
		return bean;
	}

	@Override
	public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
		this.applicationContext = applicationContext;
	}

	/**
	 * 实例化 DataSourceInitializer
	 * @return DataSourceInitializer
	 */
	public DataSourceInitializer getDataSourceInitializer() {
		if (this.dataSourceInitializer == null) {
			this.dataSourceInitializer = new DataSourceInitializer(this.dataSource, this.dataSourceProperties);
		}
		return this.dataSourceInitializer;
	}

	public void setDataSource(DataSource dataSource) {
		this.dataSource = dataSource;
	}

	public void setDataSourceProperties(DataSourceProperties dataSourceProperties) {
		this.dataSourceProperties = dataSourceProperties;
	}

}
```

**xml 配置：**

```xml
<!-- 数据源初始化事件监听器 -->
<bean id="dataSourceInitializerListener" class="wang.miansen.roothub.common.dao.jdbc.spring.DataSourceInitializerListener" />
	
<!-- 发布初始化数据源事件 -->
<bean id="dataSourceInitializedPublisher" class="wang.miansen.roothub.common.dao.jdbc.spring.DataSourceInitializedPublisher" >
		<property name="dataSourceProperties" ref="dataSourceProperties" />
		<property name="dataSource" ref="dataSource" />
</bean>
```