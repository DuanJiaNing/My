使用 Spring 时，XML 和注解是使用得最多的两种配置方式，虽然是两种完全不同的配置方式，但对于 IOC 容器来说，两种方式的不同主要是在 `BeanDefinition` 的解析上。而对于核心的容器启动流程，仍然是一致的。

`AbstractApplicationContext` 的 `refresh` 方法实现了 IOC 容器启动的主要逻辑，启动流程中的关键步骤在源码中也可以对应到独立的方法。接下来以 `AbstractApplicationContext` 的实现类 `ClassPathXmlApplicationContext` 为主 ，并对比其另一个实现类 `AnnotationConfigApplicationContext`， 解读 IOC 容器的启动过程。

AbstractApplicationContext.refresh
```java

	@Override
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				initMessageSource();

				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				onRefresh();

				// Check for listener beans and register them.
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				finishRefresh();
			}

			// ...
		}
	}

```
## ApplicationContext 和 BeanFactory 的关系
`ClassPathXmlApplicationContext` 和 `AnnotationConfigApplicationContext` 的继承树如下所示。两者都继承自 `AbstractApplicationContext`。
![在这里插入图片描述](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9jZG4ubmxhcmsuY29tL3l1cXVlLzAvMjAxOS9qcGVnLzE1MzQ1Ny8xNTY2ODk1NjM4OTM3LTEyYmQ2MzdlLWNhZGUtNDg3My05Y2I3LTdkNjAxYjczOGFiOC5qcGVn?x-oss-process=image/format,png#align=left&display=inline&height=709&name=Spring%20ApplicationContext%20%E7%BB%A7%E6%89%BF%E5%9B%BE.jpg&originHeight=709&originWidth=1000&size=47541&status=done&width=1000)

**ApplicationContext 继承树(**[**高清大图**](https://www.processon.com/view/link/5d6397f7e4b0869fa4228456)**)**

![在这里插入图片描述](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9jZG4ubmxhcmsuY29tL3l1cXVlLzAvMjAxOS9qcGVnLzE1MzQ1Ny8xNTY2ODk1NDc1Mjc4LWYxZDA4M2E0LThmYzQtNDU3Ny1iNjIzLWFiODQwYjJkMGI5OC5qcGVn?x-oss-process=image/format,png#align=left&display=inline&height=684&name=Spring%20BeanFactory%20%E7%BB%A7%E6%89%BF%E5%9B%BE.jpg&originHeight=684&originWidth=1084&size=31064&status=done&width=1084)
**BeanFactory 继承树(**[**高清大图**](https://www.processon.com/view/link/5d639128e4b0ac2b61871cd7)**)**

`ApplicationContext` 是 IOC 容器的承载体，而 `BeanFactory` 是操作这个容器的工具，两者关系紧密，相互协作。`refresh` 方法实现了 `ApplicationContext` 和 `BeanFactory` 相互协作的主要过程，不同之处主要在子类 `AbstractRefreshableApplicationContext` 和 `GenericApplicationContext` 中实现，两者使用的 `BeanFactory` 都为 `DefaultListableBeanFactory`，`DefaultListableBeanFactory` 的定义如下:

`DefaultListableBeanFactory`:
```java
public class DefaultListableBeanFactory extends AbstractAutowireCapableBeanFactory
		implements ConfigurableListableBeanFactory, BeanDefinitionRegistry, Serializable
```

可见 `DefaultListableBeanFactory` 实现了 `ConfigurableListableBeanFactory`，意味着是可配置，可遍历的，至于为什么可以，让我们继续往下寻找找答案。

## BeanDefinition 的获取
`DefaultListableBeanFactory` 中使用 Map 结构保存所有的 `BeanDefinition` 信息：

`DefaultListableBeanFactory`:
```java
private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);
```

#### ClassPathXmlApplicationContext 中的解析
使用 `BeanDefinitionDocumentReader`(可参看 `DefaultBeanDefinitionDocumentReader.processBeanDefinition `方法) 将 xml 中的 bean 解析为 `BeanDefinition`， 然后由 `BeanDefinitionRegistry` 注册到 `BeanFactory` 中。
入口：`AbstractApplicationContext.refreshBeanFactory`(在 refresh 中调用)

#### AnnotationConfigApplicationContext 中的解析
通过 `BeanDefinitionScanner` 扫描 Bean 声明，解析为  `BeanDefinition` 并由 `BeanDefinitionRegistry` 注册到 `BeanFactory` 中。
入口：`AnnotationConfigApplicationContext` 的构造函数。
##### 
##### 为什么 ClassPathXmlApplicationContext 的入口是在 refreshBeanFactory 方法中 ？
`AbstractApplicationContext.refreshBeanFactory` 定义如下: 

`AbstractApplicationContext`:
```java
protected abstract void refreshBeanFactory() throws BeansException, IllegalStateException
```

可见是一个抽象方法，具体实现在子类中。只有 "Refreshable" 的 `BeanFactory` 才会在该方法中实现具体操作，如 `AbstractRefreshableApplicationContext`:

`AbstractRefreshableApplicationContext`:
```java
@Override
	protected final void refreshBeanFactory() throws BeansException {
		if (hasBeanFactory()) {
			destroyBeans();
			closeBeanFactory();
		}
		try {
			DefaultListableBeanFactory beanFactory = createBeanFactory();
			beanFactory.setSerializationId(getId());
			customizeBeanFactory(beanFactory);
			loadBeanDefinitions(beanFactory);
			synchronized (this.beanFactoryMonitor) {
				this.beanFactory = beanFactory;
			}
		}
		catch (IOException ex) {
			throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
		}
	}
```


可见 `AbstractRefreshableApplicationContext.``refreshBeanFactory` 方法会检查 `BeanFactory` 是否已经存在(`hasBeanFactory`)，已经存在就先销毁所有的 Bean(`destoryBeans`)并关闭(`closeBeanFactory`) `BeanFactory`，然后再创建(`createBeanFactory`)新的。
而 `GenericApplicationContext.refreshBeanFactory `中会检查是否为第一次调用，不是就抛出异常，不执行其他逻辑，即 `GenericApplicationContext` 不是 "Refreshable"的。

## 主流程分析
`refresh` 方法在 `AbstractApplicationContext` 中定义，其中的 `obtainFreshBeanFactory` 方法调用了 `getBeanFactory` 方法，该方法用于获取 `BeanFactory`，这里为 `DefaultListableBeanFactory`，接下来无特别说明，大部分的方法和变量都将取自 `AbstractApplicationContext` 和  `DefaultListableBeanFactory`。
![在这里插入图片描述](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9jZG4ubmxhcmsuY29tL3l1cXVlLzAvMjAxOS9qcGVnLzE1MzQ1Ny8xNTY3MDQzMjU0Mzg3LWFlY2IwNDE2LTA3ZDktNDJhMC05NGU2LTFiOGQ2MjdhMjVlNC5qcGVn?x-oss-process=image/format,png#align=left&display=inline&height=2943&name=Spring%20Context%20refresh%20%E4%B8%BB%E6%B5%81%E7%A8%8B%E6%97%B6%E5%BA%8F%E5%9B%BE_%E7%9C%8B%E5%9B%BE%E7%8E%8B.jpg&originHeight=2943&originWidth=1768&size=457822&status=done&width=1768)
[高清大图](https://www.processon.com/view/link/5d5ba3d2e4b04399f5ab926d)

### BeanPostProcessor
`BeanPostProcessor` 接口让开发者在 IOC 容器对 Bean 进行实例化时收到回调(`postProcessAfterInitialization` 和 `postProcessBeforeInitialization` 方法)。spring 框架内部的许多通知(`Aware`)就是通过这个接口实现，如 `ApplicationContextAwareProcessor`, `ServletContextAwareProcessor`，他们的实现会在 `postProcessBeforeInitialization` 方法中进行检查，若实现了特定接口，就会调用 `Aware` 的回调方法，给予通知:

`ServletContextAwareProcessor`:
```java
@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		if (getServletContext() != null && bean instanceof ServletContextAware) {
			((ServletContextAware) bean).setServletContext(getServletContext());
		}
		if (getServletConfig() != null && bean instanceof ServletConfigAware) {
			((ServletConfigAware) bean).setServletConfig(getServletConfig());
		}
		return bean;
	}
```

在 `postProcessBeanFactory` 方法中，子类可以通过 `beanFactory.addBeanPostProcessor` 方法添加自己的 `BeanPostProcessor` 到 `beanFactory` 中，最终将保存到 `BeanFactory` 的 `beanPostProcessors`(实为`CopyOnWriteArrayList`) 中。`prepareBeanFactory` 和 `registerBeanPostProcessors` 方法是集中实例化并添加这些 Bean 的地方。

### BeanFactoryPostProcessor 和 BeanDefinitionRegistryPostProcessor
从"BeanDefinition 的获取"的介绍可以知道 `BeanDefinitionRegistry` 用于将  `BeanDefinition` 注册到 `BeanFactory` 中，`GenericApplicationContext` 和 `DefaultListableBeanFactory` 都实现了该接口，`GenericApplicationContext` 中的实现直接调用了 `beanFactory` 的实现。

`BeanFactoryPostProcessor` 和 `BeanDefinitionRegistryPostProcessor` 与 `BeanPostProcessor` 类似，但从他们的命名就可以看出，所针对的目标不同，分别是 `BeanFactory` 和 `BeanDefinitionRegistry:`
1 `BeanFactoryPostProcessor` 回调让开发者有机会在 `BeanFactory` 已经初始化好的情况下对 `BeanFactory` 的一些属性进行覆盖，或是对 `beanDefinitionMap` 中的 `BeanDefinition` 进行修改。
2 `BeanDefinitionRegistryPostProcessor` 则让开发者可以继续添加 `BeanDefinition` 到 `BeanFactory` 中。

具体逻辑在 `invokeBeanFactoryPostProcessors` 中实现，这里首先会将所有实现了 `BeanFactoryPostProcessors `的 Bean 实例化，然后调用其回调方法(`postProcessBeanDefinitionRegistry` 或 `postProcessBeanFactory` 方法)。

对于这部分 Bean 的实例化和进行回调有一定的优先级规则。`PriorityOrdered` 继承自 `Ordered` 接口，实现了 `PriorityOrdered` 的 `BeanDefinitionRegistryPostProcessor` 将最先被实例化并调用，然后同样的规则来回调实现了 `BeanFactoryPostProcessor` 的 Bean：
`PriorityOrdered` > `Ordered` > 未实现 Ordered 的

在 `registerBeanPostProcessors` 方法中对 `BeanPostProcessor` 的实例化也有这样的优先级规则:
`PriorityOrdered` > `Ordered` > 未实现 Ordered 的 > `MergedBeanDefinitionPostProcessor`

### ApplicationEventMulticaster
在 `initApplicationEventMulticaster` 中会对 `ApplicationEventMulticaster` 进行初始化: 首先会检查是否已经有了`ApplicationEventMulticaster` 的 `BeanDefinition`(在 `beanDefinitionMap` 中检查)，有就让容器进行实例化，没有就使用框架默认的 `ApplicationEventMulticaster` (即 `SimpleApplicationEventMulticaster`)，先实例化，然后注册到容器中(`MessageSource` 在 `initMessageSource` 方法中也是同样的方式进行初始化)。

事件的起始发送处将事件包装为 `ApplicationEvent` ，并通过 `ApplicationEventPublisher` 提交给 `ApplicationEventMulticaster`，`ApplicationEventMulticaster` 会将事件广播给 `ApplicationListener`，处理最终的分发。

`AbstractApplicationEventMulticaster` 中的 `applicationListeners(`实为 `LinkedHashSet<ApplicationListener>)` 变量保存了所有的广播接收者，`registerListeners` 方法会将所有的 `ApplicationListener` 添加到该集合中。

`finishRefresh` 方法中有一个对 `ContextRefreshedEvent` 事件的广播可以作为参考，最终事件会由 `multicastEvent` 方法处理:

`SimpleApplicationEventMulticaster.multicastEvent`
```java
@Override
	public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
		ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
		Executor executor = getTaskExecutor();
		for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
			if (executor != null) {
				executor.execute(() -> invokeListener(listener, event));
			}
			else {
				invokeListener(listener, event);
			}
		}
	}
```

那么在我们自己的 Bean 中如何得到这个 `ApplicationEventPublisher` 呢？
`ApplicationContext` 的定义如下:

`ApplicationContext`:
```java
public interface ApplicationContext extends EnvironmentCapable, ListableBeanFactory, HierarchicalBeanFactory,
		MessageSource, ApplicationEventPublisher, ResourcePatternResolver {
```

可见 `ApplicationContext` 继承了 `ApplicationEventPublisher`，这就说明 `AbstractApplicationContext` 也是一个 `ApplicationEventPublisher`。在我们自己的 Bean 中通过实现 `ApplicationEventPublisherAware`，我们就能通过 `setApplicationEventPublisher` 回调得到 `ApplicationEventPublisher`。

上面我们提到 spring 的许多 `Aware` 是通过 `BeanPostProcessor` 实现的，`ApplicationEventPublisherAware` 也不例外: 

`ApplicationContextAwareProcessor`:
```java
@Override
	@Nullable
	public Object postProcessBeforeInitialization(final Object bean, String beanName) throws BeansException {
		// ...
        if (bean instanceof Aware) {
			if (bean instanceof EnvironmentAware) {
				((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
			}
			if (bean instanceof EmbeddedValueResolverAware) {
				((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(this.embeddedValueResolver);
			}
			if (bean instanceof ResourceLoaderAware) {
				((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
			}
			if (bean instanceof ApplicationEventPublisherAware) {
				((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
			}
			if (bean instanceof MessageSourceAware) {
				((MessageSourceAware) bean).setMessageSource(this.applicationContext);
			}
			if (bean instanceof ApplicationContextAware) {
				((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
			}
		}
        // ...
	}
```

IOC 容器在实例化我们的 Bean 时会调用 `ApplicationContextAwareProcessor`.`postProcessBeforeInitialization` 方法，该方法会检查我们的 Bean，我们的 Bean 如果实现了 `ApplicationEventPublisherAware`，那么就会回调 `setApplicationEventPublisher` 方法将 `applicationContext`(即`ApplicationEventPublisher`) 传给我们，我们就能够发布事件。

### BeanFactory.getBean 
`BeanFactory` 的几个重载了的 `getBean` 方法是 Bean 最终进行实例化的地方，`registerBeanPostProcessors`, `invokeBeanFactoryPostProcessors` 和 `finishBeanFactoryInitialization` 方法都调用了 getBean 方法对一些特定 Bean 进行了实例化。

`finishBeanFactoryInitialization` 中通过调用 `BeanFactory` 的 `preInstantiateSingletons` 对单例 Bean 进行实例化。`BeanFactory` 和 `BeanDefinition` 都具有父子的概念，在子级找不到指定的 Bean 时将一直往上(父级)找，找到就进行实例化

## 总结
spring IOC 容器的启动步骤可总结如下：
1 初始化 ApplicationContext
环境属性的初始化和验证，启动时间记录和相关标记设置，应用事件和监听者的初始化。

2 准备好容器中的 BeanDefinition (eager-initializing beans)
对 `BeanDefinition` 的解析、扫描和注册，`BeanDefinition` 的扫描和注册大致可以分为 XML 和注解两种，两种方式各自使用的组件有所不同，该步骤的时间也可以在最前面。

3 初始化 BeanFactory
准备好 BeanFactory 以供 ApplicationContext 进行使用，对接下来将要使用到的 Bean 进行实例化，资源进行准备，属性进行设置。

4 注册 BeanPostProcessors
BeanPostProcessors 是进行扩展的关键组件，需要在该步骤中进行注册，可分为两种类型: 一种是框架使用者提供的，用于特定业务功能的，另一种是框架开发者提供的，用于扩展框架功能。

5 调用 `BeanDefinitionRegistryPostProcessor`
`BeanDefinitionRegistryPostProcessor` 是一种功能增强，可以在这个步骤添加新的 `BeanDefinition` 到 `BeanFactory` 中。

6 调用 `BeanFactoryPostProcessor`
`BeanFactoryPostProcessor` 是一种功能增强，可以在这个步骤对已经完成初始化的 BeanFactory 进行属性覆盖，或是修改已经注册到 `BeanFactory` 的 `BeanDefinition`。

7 初始化 `MessageSource` 和 `ApplicationEventMulticaster`
`MessageSource` 用于处理国际化资源，`ApplicationEventMulticaster` 是应用事件广播器，用于分发应用事件给监听者。

8 初始化其他 Bean 和进行其他的的上下文初始化
主要用于扩展

9 注册 `ApplicationListene`
将 `ApplicationListene` 注册到 `BeanFactory` 中，以便后续的事件分发

10 实例化剩余的 Bean 单例
步骤 4 到 9 都对一些特殊的 Bean 进行了实例化，这里需要对所有剩余的单例 Bean 进行实例化

11 启动完成
资源回收，分发"刷新完成"事件。

