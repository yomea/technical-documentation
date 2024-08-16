<p>
&nbsp;&nbsp;&nbsp;&nbsp;前面的章节我们分析了容器refresh方法中前5个方法，下次我们来分析下一个方法registerBeanPostProcessors。
</p>


```
public void org.springframework.context.support.AbstractApplicationContext.refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			//预备刷新，子类实现，可以设置环境变量，校验必需属性等操作
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			//构建嵌入BeanFactory（被装饰），并刷新，加载解析xml配置文件
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			//做一些准备工作，比如设置表达式引擎，可忽略的接口，需要额外依赖的bean，注册没有
			//BeanDefinition的单例bean，注册一些内部使用后置处理器
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				//子类实现，比如我们可以增加一些BeanFactoryProcessor等等
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
				//调用BeanFactory后置处理，主要用于添加新的BeanDefinition
				//比如扫描@Configuration注解，mybatis的@Mapper
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				//注册BeanPostProcessor，用于在bean创建阶段的扩展
				registerBeanPostProcessors(beanFactory);

			。。。。。。省略部分代码
		}
	}
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;一切需要被spring容器托管的bean都已经加载，那么我们可以开始bean的创建了，但是在创建之前还是要做一些用于扩展的准备，比如准备bean创建的后置处理器，给用于一个参与bean创建的机会
</p>


```
public static void org.springframework.context.support.PostProcessorRegistrationDelegate.registerBeanPostProcessors(
			ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {
        
        //获取BeanPostProcessor类型beanName，当不允许对工厂bean提早初始化,内部具体逻辑请参照第三节
		String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);

		// Register BeanPostProcessorChecker that logs an info message when
		// a bean is created during BeanPostProcessor instantiation, i.e. when
		// a bean is not eligible for getting processed by all BeanPostProcessors.
		//统计所有beanProcessor个数
		int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
		//添加BeanPostProcessorChecker后置处理器，用于打印日志，当beanFactory.getBeanPostProcessorCount()的个数小于beanProcessorTargetCount时
		//并且不是RootBeanDefinition.ROLE_INFRASTRUCTURE角色，将打印日志，具体内容请自行查看其postProcessAfterInitialization方法
		beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));

		// Separate between BeanPostProcessors that implement PriorityOrdered,
		// Ordered, and the rest.
		//用于存放具有优先级的后置处理器
		List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<BeanPostProcessor>();
		//用于存放内部后置处理器，就是实现了MergedBeanDefinitionPostProcessor接口的后置处理
		List<BeanPostProcessor> internalPostProcessors = new ArrayList<BeanPostProcessor>();
		//用于存放具有order排序的后置处理
		List<String> orderedPostProcessorNames = new ArrayList<String>();
		//用于存放普通的后置处理器
		List<String> nonOrderedPostProcessorNames = new ArrayList<String>();
		for (String ppName : postProcessorNames) {
		    //检查是否实现了PriorityOrdered
			if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
				BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
				priorityOrderedPostProcessors.add(pp);
				//如果也实现了MergedBeanDefinitionPostProcessor接口，那么加入到内部后置处理器集合中
				if (pp instanceof MergedBeanDefinitionPostProcessor) {
					internalPostProcessors.add(pp);
				}
			}
			//如果是Ordered类型，加入到具有Oder顺序的集合中
			else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
				orderedPostProcessorNames.add(ppName);
			}
			else {
			    //存放普通的后置处理器
				nonOrderedPostProcessorNames.add(ppName);
			}
		}

		// First, register the BeanPostProcessors that implement PriorityOrdered.
		//首选对实现了PriorityOrdered接口的后置处理器进行排序
		sortPostProcessors(beanFactory, priorityOrderedPostProcessors);
		//将后置处理器注册到BeanFactory的beanPostProcessors集合中
		registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

		// Next, register the BeanPostProcessors that implement Ordered.
		//注册实现了Ordered接口的后置处理器
		List<BeanPostProcessor> orderedPostProcessors = new ArrayList<BeanPostProcessor>();
		for (String ppName : orderedPostProcessorNames) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			orderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		//排序并注册
		sortPostProcessors(beanFactory, orderedPostProcessors);
		registerBeanPostProcessors(beanFactory, orderedPostProcessors);

		// Now, register all regular BeanPostProcessors.
		//注册普通的后置处理器
		List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<BeanPostProcessor>();
		for (String ppName : nonOrderedPostProcessorNames) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			nonOrderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);

		// Finally, re-register all internal BeanPostProcessors.
		//重新注册实现了MergedBeanDefinitionPostProcessor接口的后置处理器
		sortPostProcessors(beanFactory, internalPostProcessors);
		registerBeanPostProcessors(beanFactory, internalPostProcessors);
        //用于添加我们定义的ApplicationListener监听器
		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
	}
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;这样bean的后置处理器就处理完毕，我们把refresh方法剩下的逻辑快速了解一遍吧！
</p>


```
public void org.springframework.context.support.AbstractApplicationContext.refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			//初始化属性资源，校验必需的属性，早期发布事件
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			//构建BeanFactory，并设置一些初始化属性，比如是否允许循环依赖，是否允许BeanDefinition的覆盖
			//加载xml配置BeanDefinition
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			//设置spring的表达式引擎，默认的属性编辑器，ApplicationContextAware通知器，设置忽略依赖，注册额外的解析依赖
			//运行时加载处理器，注册诸如环境对象，系统属性，系统变量
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				//添加ServletContextAwareProcessor，注册ServletConfigAware，ServletContextAware忽略依赖
				//注册web范围，比如request（RequestScope），session（SessionScope），globalSession（SessionScope）
				//注册额外依赖，比如request，response，session，jsf
				//注册contextParameters，web的上下文参数，以Map的形式存在单例bean集合中
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
				//获取实现了BeanFactoryPostProcessor接口，BeanDefinitionRegistryPostProcessor接口的BeanFactory后置处理器，并调用
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				//注册实现BeanPostProcessor接口bean后置处理器
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				//初始化国际化资源
				initMessageSource();

				// Initialize event multicaster for this context.
				//初始化事件广播器
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				//由子类实现，在目前我们使用的容器继承结构中，AbstractRefreshableWebApplicationContext类中进行了主题的初始化
				//主题，在web应用中使用，配合spring的标签来使用，现在多数使用模板引擎，或前后端分离，很少会使用一些jstl类型的标签了
				onRefresh();

				// Check for listener beans and register them.
				//注册ApplicationContextListener，并早期触发在prepareRefresh定义的早期事件
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				//设置默认的ConversionService（值类型转换服务），构建LoadTimeWeaverAware类型对象
				//冻结配置文件，从这里开始，配置的BeanDefinition不允许再更改，允许进行缓存，即使能够修改，修改的都是副本
				//构建eager的bean，内部调用的getBean方法会在第4小节分析
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				//初始化生命周期处理器LifecycleProcessor，调用实现了Lifecycle接口的bean的start方法
				//发送ContextRefreshedEvent事件，注册MBeanServer
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}
```

