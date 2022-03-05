#    Spring注解驱动开发笔记-Spring容器的创建

## 一、环境搭建

### 1.创建配置类ExtConfig

```java
@ComponentScan("com.wxr.ext")
@Configuration
public class ExtConfig {


}
```

### 2测试代码，debug入口

```java
public class IocTest {
    @Test
    public void test01(){
        AnnotationConfigApplicationContext ioc = new AnnotationConfigApplicationContext(ExtConfig.class);
        ioc.close();
    }
}
```



## 二、AnnotationConfigApplicationContext的UML图

![image-20220223235531282](C:\Users\MateBook\Desktop\IDE、数据库系统等工具\API文档\SSM\Spring注解驱动开发\image-20220223235531282.png)











## 三、源码分析

### 1.创建IoC容器（调用AnnotationConfigApplicationContext的构造器）

```java
AnnotationConfigApplicationContext ioc = new AnnotationConfigApplicationContext(ExtConfig.class);
//AnnotationConfigApplicationContext的构造器
public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
    //3.2.1的时候会调用一下这里创建BeanFactory
   this();
    //注册多个注解Bean定义类  
   register(annotatedClasses);
    //2.看这里
   refresh();
}


1、AnnotationConfigApplicationContext.register
public void register(Class<?>... annotatedClasses) {
        Assert.notEmpty(annotatedClasses, "At least one annotated class must be specified");
        this.reader.register(annotatedClasses);
    }
2、AnnotatedBeanDefinitionReader.register
public void register(Class<?>... annotatedClasses) {
        for (Class<?> annotatedClass : annotatedClasses) {
            registerBean(annotatedClass);
        }
    }
3、AnnotatedBeanDefinitionReader.registerBean
public void registerBean(Class<?> annotatedClass, String name,@SuppressWarnings("unchecked") Class<? extends Annotation>... qualifiers) {
			//该类继承GenericBeanDefinition并实现AnnotatedBeanDefinition，
			//AnnotatedGenericBeanDefinition 描述了一个注解bean实例，它具有属性值，构造函数参数值。
        AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(annotatedClass);
			//根据{@code @Conditional}注释确定是否应跳过某个项目
        if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
            return;
        }
			//ScopeMetadata 描述了一个Spring管理的bean，包括范围名称和范围的代理行为范围的特点。(也就是spring的作用域）
        ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
        abd.setScope(scopeMetadata.getScopeName());
			//获取传入注解配置类的bean
        String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));
			//处理该注解bean上的一些其他注解属性，比如@lazy，@Primary等注解
			//有的话加入到AnnotatedGenericBeanDefinition 中
		AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
			//@qualifiers注解
        if (qualifiers != null) {
            for (Class<? extends Annotation> qualifier : qualifiers) {
                if (Primary.class == qualifier) {
                    abd.setPrimary(true);
                }
                else if (Lazy.class == qualifier) {
                    abd.setLazyInit(true);
                }
                else {
                    abd.addQualifier(new AutowireCandidateQualifier(qualifier));
                }
            }
        }
		//BeanDefinitionHolder拥有名称和别名的BeanDefinition的持有者。可以注册为内部bean的占位符。
        BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
        definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
			//在BeanDefinitionRegistry中注册相应的获取的bean，definitionHolder.getBeanName()作为beanName
		//definitionHolder.getBeanDefinition()作为value，如果有别名还要进行别名注册
    BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
    }

```

### 2.调用AbstractApplicationContext.refresh()刷新容器

```java
public void refresh() throws BeansException, IllegalStateException {
   synchronized (this.startupShutdownMonitor) {
      //刷新的预处理工作，3.1走这里
      prepareRefresh();

      //获取BeanFactory，3.2走这里
      ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

      //对BeanFactory做一些预准备工作（对BeanFactory做一些设置），3.3走这里
      prepareBeanFactory(beanFactory);

      try {
         //空方法，但是子类通过重写这个方法来在BeanFactory创建并预准备完成以后做进一步的设置 ，3.4走这里
         postProcessBeanFactory(beanFactory);

         //执行BeanFactoryPostProcessors，3.5走这里
         //在BeanFactory标准初始化之后调用（所有的Bean定义已经保存加载到BeanFactory中，但是Bean的实例并没有被创建）
         invokeBeanFactoryPostProcessors(beanFactory);

         // 注册BeanPostProcessor(Bean后置处理器)在Bean创建对象初始化前后进行的拦截工作的，3.6走这里         
          registerBeanPostProcessors(beanFactory);

         // 初始化MessageSource组件（做国际化功能；消息绑定，消息解析），3.7走这里
         initMessageSource();

         // 初始化事件的多播器（派发器），3.8走这里
         initApplicationEventMulticaster();

         //空方法，但是子类通过重写这个方法来初始化特殊的bean，3.9走这里
         onRefresh();

         //给容器中将所有项目里面的ApplicationListener注册进来，3.10看这里
         registerListeners();

         //调用本类的finishBeanFactoryInitialization(beanFactory)完成BeanFactory的初始化工作，
         //实例化剩余的单实例bean到容器中 ，3.11看这里
         finishBeanFactoryInitialization(beanFactory);

         //完成BeanFactory的初始化创建工作，IoC容器创建完成，3.12看这里
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

### 3.refresh的每个方法详细分析

#### 3.1调用AbstractApplicationContext.prepareRefresh()---刷新的预处理工作

```java
@Override
protected void prepareRefresh() {
    //清理缓存
   this.scanner.clearCache();
    //父类的刷新,3.1.1看这里
   super.prepareRefresh();
}
```

##### 3.1.1调用父类的AbstractApplicationContext.prepareRefresh()

```java
protected void prepareRefresh() {
    //记录时间
   this.startupDate = System.currentTimeMillis();
    //设置容器是否关闭
   this.closed.set(false);
    //设置容器是否激活
   this.active.set(true);

    //在控制台打印容器刷新日志
   if (logger.isInfoEnabled()) {
      logger.info("Refreshing " + this);
   }

  //初始化一些属性设置，在子容器中自定义一些属性设置
   initPropertySources();

  //进行属性校验
   getEnvironment().validateRequiredProperties();

  //创建一个Set集合用于保存容器中的一些事件
   this.earlyApplicationEvents = new LinkedHashSet<ApplicationEvent>();
}
```



#### 3.2调用AbstractApplicationContext.obtainFreshBeanFactory()---获取BeanFactory

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    //刷新BeanFactory，3.2.1看这里
   refreshBeanFactory();
    //获取BeanFactory
   ConfigurableListableBeanFactory beanFactory = getBeanFactory();
   if (logger.isDebugEnabled()) {
      logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
   }
    //返回BeanFactory
   return beanFactory;
}
```

##### 3.2.1调用GenericApplicationContext. refreshBeanFactory()-刷新BeanFactory，给beanFactory设置了一个序列化ID

```java
//有一个DefaultListableBeanFactory beanFactory
private final DefaultListableBeanFactory beanFactory;
//在1.创建IoC容器的时候会调用一个this();
//它会调用本类AnnotationConfigApplicationContext的空参构造器
//然后调用父类也就是GenericApplicationContext的空参构造器，也就是下面这个方法，创建了这个DefaultListableBeanFactory beanFactory
public GenericApplicationContext() {
		this.beanFactory = new DefaultListableBeanFactory();
	}


@Override
protected final void refreshBeanFactory() throws IllegalStateException {
   if (!this.refreshed.compareAndSet(false, true)) {
      throw new IllegalStateException(
            "GenericApplicationContext does not support multiple refresh attempts: just call 'refresh' once");
   }
    //给beanFactory设置了一个序列化ID
   this.beanFactory.setSerializationId(getId());
}
```



#### 3.3调用AbstractApplicationContext.prepareBeanFactory(ConfigurableListableBeanFactory beanFactory)---对BeanFactory做一些预准备工作

```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
  
    //设置beanFactory类加载器
   beanFactory.setBeanClassLoader(getClassLoader());
    //设置beanFactory表达式解析器
   beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
   beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

    //添加部分的BeanPostProcessor（ApplicationContextAwareProcessor）
   beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
    //设置忽略的自动装配的接口EnvironmentAware、EmbeddedValueResolverAware、xxx；
   beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
   beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
   beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
   beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
   beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
   beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

  //注册可以解析的自动装配；我们能直接在任何组件中自动注入：BeanFactory、ResourceLoader、ApplicationEventPublisher、ApplicationContext
   beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
   beanFactory.registerResolvableDependency(ResourceLoader.class, this);
   beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
   beanFactory.registerResolvableDependency(ApplicationContext.class, this);

   //添加部分的BeanPostProcessor（ApplicationListenerDetectorr）
   beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

   // 添加编译时的AspectJ
   if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
      beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
      // Set a temporary ClassLoader for type matching.
      beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
   }

   // 给BeanFactory中注册一些能用的组件：
    //environment【ConfigurableEnvironment】、systemProperties【Map<String, Object>】、systemEnvironment【Map<String, Object>】
   if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
      beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
   }
   if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
      beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
   }
   if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
      beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
   }
}
```



#### 3.4调用AbstractApplicationContext.postProcessBeanFactory(beanFactory)---完成后置处理工作

```java
//子类通过重写这个方法来在BeanFactory创建并预准备完成以后做进一步的设置
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    }
```



#### 3.5调用AbstractApplicationContext. invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory)---执行BeanFactory的后置处理器

```java
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
    //3.5.1走这里
   PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());
   if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
      beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
      beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
   }
}
```

##### 3.5.1走这里调用PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

```java
public static void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {
    
		   //多余代码省略
//=====================================先执行BeanDefinitionRegistryPostProcessor接口的BeanPostProcessor===========================
    	   //获取所有的实现该BeanDefinitionRegistryPostProcessor的名称
			String[] postProcessorNames =
					beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
           //看先执行实现了PriorityOrdered优先级接口的BeanDefinitionRegistryPostProcessor
			for (String ppName : postProcessorNames) {
				if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
					currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
					processedBeans.add(ppName);
				}
			}  
			sortPostProcessors(currentRegistryProcessors, beanFactory);
			registryProcessors.addAll(currentRegistryProcessors);
    		//在这里把标注@Configuration等注解的类注册进来
			invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
			currentRegistryProcessors.clear();

			//再执行实现了Ordered顺序接口的BeanDefinitionRegistryPostProcessor
			postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
			for (String ppName : postProcessorNames) {
				if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
					currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
					processedBeans.add(ppName);
				}
			}
			sortPostProcessors(currentRegistryProcessors, beanFactory);
			registryProcessors.addAll(currentRegistryProcessors);
			invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
			currentRegistryProcessors.clear();

			// 最后执行没有实现任何优先级或者是顺序接口的BeanDefinitionRegistryPostProcessor
			boolean reiterate = true;
			while (reiterate) {
				reiterate = false;
				postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
				for (String ppName : postProcessorNames) {
					if (!processedBeans.contains(ppName)) {
						currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
						processedBeans.add(ppName);
						reiterate = true;
					}
				}
				sortPostProcessors(currentRegistryProcessors, beanFactory);
				registryProcessors.addAll(currentRegistryProcessors);
				invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
				currentRegistryProcessors.clear();
			}

			// Now, invoke the postProcessBeanFactory callback of all processors handled so far.
			invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
			invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
		}

		else {
			// Invoke factory processors registered with the context instance.
			invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
		}
    
    
  
//===========================================再执行BeanFactoryPostProcessor接口的方法===========================================
		//从这里获取到类型是BeanFactoryPostProcessor的所有名字
		String[] postProcessorNames =beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

		List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<BeanFactoryPostProcessor>();
		List<String> orderedPostProcessorNames = new ArrayList<String>();
		List<String> nonOrderedPostProcessorNames = new ArrayList<String>();
        //遍历拿到所有BeanFactoryPostProcessor的名字
        //根据实现排序接口与否加入到不同的地方
		for (String ppName : postProcessorNames) {
			if (processedBeans.contains(ppName)) {
			}
			else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
				priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
			}
			else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
				orderedPostProcessorNames.add(ppName);
			}
			else {
				nonOrderedPostProcessorNames.add(ppName);
			}
		}

   
   // 先调用执行实现了PriorityOrdered优先级接口的BeanFactoryPostProcessor
   sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
   invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

   //再执行实现了Ordered顺序接口的BeanFactoryPostProcessor
   List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<BeanFactoryPostProcessor>();
   for (String postProcessorName : orderedPostProcessorNames) {
      orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
   }
   sortPostProcessors(orderedPostProcessors, beanFactory);
   invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

   //最后执行没有实现任何优先级或者是顺序接口的BeanFactoryPostProcessor
   List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<BeanFactoryPostProcessor>();
   for (String postProcessorName : nonOrderedPostProcessorNames) {
       //遍历拿到所有的BeanFactoryPostProcessor
       //在beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class)这里面创建、初始化、注册BeanFactoryPostProcessor
       //类似于BeanPostProcessors创建，这个在Spring注解驱动开发笔记-aop篇三.4中有具体解析
      nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
   }
   
   invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

 
   beanFactory.clearMetadataCache();
}
```



#### 3.6调用AbstractApplicationContext.registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory)---注册BeanPostProcessor(Bean后置处理器)

```java
/**
 *这是AbstractApplicationContext.registerBeanPostProcessors(beanFactory);
 */        
    protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
		PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
	}
            
```

##### 3.6.1调用PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this)对BeanPostProcessor进行创建、初始化和注册

```java
 /**
   *这是PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
   *说明：

		BeanPostProcessor的一些子接口：在Bean初始化前后的执行时机是不一样的

		DestructionAwareBeanPostProcessor

		InstantiationAwareBeanPostProcessor

		SmartInstantiationAwareBeanPostProcessor

		MergedBeanDefinitionPostProcessor[internalPostProcessors]
   *
   *
   *
   */            
 public static void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext)
 { 
     //拿到IoC容器中已经定义了的需要创建对象的所有的BeanPostProcessor
	String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);
	int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
    //给容器中添加其他的BeanPostProcessor（BeanPostProcessorChecker）
	beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));
	//对BeanPostProcessor按照优先级分类
	List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<BeanPostProcessor>();
	List<BeanPostProcessor> internalPostProcessors = new ArrayList<BeanPostProcessor>();
	List<String> ordered = new ArrayList<String>();
	List<String> nonOrderedPostProcessorNames = new ArrayList<String>();
	for (String ppName : postProcessorNames) {
		if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			priorityOrderedPostProcessors.add(pp);
            //如果是MergedBeanDefinitionPostProcessor就添加到internalPostProcessors集合中
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
			orderedPostProcessorNames.add(ppName);
		}
		else {
			nonOrderedPostProcessorNames.add(ppName);
		}
	}

	
	sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
     //优先注册实现了PriorityOrdered接口的BeanPostProcessor
     //把每一个BeanPostProcessor添加到BeanFactory中
	registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);
 
	 //再注册Ordered接口的
     //把每一个BeanPostProcessor添加到BeanFactory中
        List<BeanPostProcessor> orderedPostProcessors = new ArrayList<BeanPostProcessor>();
        for (String ppName : orderedPostProcessorNames) {
            BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
            orderedPostProcessors.add(pp);
            if (pp instanceof MergedBeanDefinitionPostProcessor) {
                internalPostProcessors.add(pp);
            }
        }
        sortPostProcessors(beanFactory, orderedPostProcessors);
        registerBeanPostProcessors(beanFactory, orderedPostProcessors);
     
     //最后注册没有实现任何优先级接口的
     //把每一个BeanPostProcessor添加到BeanFactory中
        List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<BeanPostProcessor>();
        for (String ppName : nonOrderedPostProcessorNames) {
            BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
            nonOrderedPostProcessors.add(pp);
            if (pp instanceof MergedBeanDefinitionPostProcessor) {
                internalPostProcessors.add(pp);
            }
        }
        registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);
   //最终注册 internal BeanPostProcessors.（比如MergedBeanDefinitionPostProcessor）
        sortPostProcessors(beanFactory, internalPostProcessors);
        registerBeanPostProcessors(beanFactory, internalPostProcessors);
    //在Bean创建完成后检查是否是ApplicationListener，如果是注册一个
        beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));

     
}           
```



#### 3.7调用AbstractApplicationContext.initMessageSource() 初始化MessageSource组件---做国际化功能；消息绑定，消息解析

```java
protected void initMessageSource() {
		//获取BeanFactory
        ConfigurableListableBeanFactory beanFactory = getBeanFactory();
       //看容器中是否有id为messageSource的，类型是MessageSource的组件，如果有赋值给messageSource，如果没有自己创建一个DelegatingMessageSource
        if (beanFactory.containsLocalBean(MESSAGE_SOURCE_BEAN_NAME)) {
            this.messageSource = beanFactory.getBean(MESSAGE_SOURCE_BEAN_NAME, MessageSource.class);
            if (this.parent != null && this.messageSource instanceof HierarchicalMessageSource) {
                HierarchicalMessageSource hms = (HierarchicalMessageSource) this.messageSource;
                if (hms.getParentMessageSource() == null) {
                    hms.setParentMessageSource(getInternalParentMessageSource());
                }
            }
            if (logger.isDebugEnabled()) {
                logger.debug("Using MessageSource [" + this.messageSource + "]");
            }
        }
        else {
            DelegatingMessageSource dms = new DelegatingMessageSource();
            dms.setParentMessageSource(getInternalParentMessageSource());
            this.messageSource = dms;
	 //把创建好的MessageSource注册在容器中，以后获取国际化配置文件的值的时候，可以自动注入MessageSource；          
            beanFactory.registerSingleton(MESSAGE_SOURCE_BEAN_NAME, this.messageSource);
            if (logger.isDebugEnabled()) {
                logger.debug("Unable to locate MessageSource with name '" + MESSAGE_SOURCE_BEAN_NAME +
                        "': using default [" + this.messageSource + "]");
            }
        }
    }
```



#### 3.8调用AbstractApplicationContext.initApplicationEventMulticaster()---初始化事件的多播器（派发器）

```java
protected void initApplicationEventMulticaster() {
   ConfigurableListableBeanFactory beanFactory = getBeanFactory();
    //判断在beanFactory中是否有id="applicationEventMulticaster"的组件
   if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
       //如果有就直接获取
      this.applicationEventMulticaster =        
            beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
      if (logger.isDebugEnabled()) {
         logger.debug("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
      }
   }
   else {
       //如果没有那么就自己创建一个新的id="applicationEventMulticaster"并注册到容器中
       //我们可以再其他组件要派发事件的时候，自动注入这个applicationEventMulticaster
      this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
      beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
      if (logger.isDebugEnabled()) {
         logger.debug("Unable to locate ApplicationEventMulticaster with name '" +
               APPLICATION_EVENT_MULTICASTER_BEAN_NAME +
               "': using default [" + this.applicationEventMulticaster + "]");
      }
   }
}
```



#### 3.9调用AbstractApplicationContext.onRefresh()---空方法，但是子类通过重写这个方法来初始化特殊的bean

```java
protected void onRefresh() throws BeansException {
   // For subclasses: do nothing by default.
}
```



#### 3.10调用AbstractApplicationContext.registerListeners()---给容器中将所有项目里面的ApplicationListener注册进来

```java
protected void registerListeners() {
		//在事件派发器中注册静态指定的事件监听器
        for (ApplicationListener<?> listener : getApplicationListeners()) {
    getApplicationEventMulticaster().addApplicationListener(listener);
        }
        //从容器中拿到所有的ApplicationListener并遍历
        String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
        for (String listenerBeanName : listenerBeanNames) {
		//将每个监听器添加到事件派发器中             
            getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
        }

		//派发之前步骤产生的事件
        Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
        this.earlyApplicationEvents = null;
        if (earlyEventsToProcess != null) {
            for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
                getApplicationEventMulticaster().multicastEvent(earlyEvent);
            }
        }
    }
```



#### 3.11调用AbstractApplicationContext.finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory)---初始化所有剩下的单实例bean

```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
        //多余代码已经省略
		//创建所有剩余的（非懒加载）单例，依次创建Bean实例,就是在创建bean的对象、
        //5.2.1看这里
		beanFactory.preInstantiateSingletons();
}

public void preInstantiateSingletons() throws BeansException {
		if (this.logger.isDebugEnabled()) {
			this.logger.debug("Pre-instantiating singletons in " + this);
		}
	
		//通过beanDefinitionNames，获取容器中的所有Bean的id，依次进行实例化
        List<String> beanNames = new ArrayList<String>(this.beanDefinitionNames);

        
        for (String beanName : beanNames) {
		//获取Bean的定义信息；RootBeanDefinition
            RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
		//Bean不是抽象的，是单实例的，不是懒加载；
            if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
        //判断是否是FactoryBean；是否是实现FactoryBean接口的Bean
                if (isFactoryBean(beanName)) {
                    final FactoryBean<?> factory = (FactoryBean<?>) getBean(FACTORY_BEAN_PREFIX + beanName);
                    boolean isEagerInit;
                    if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
                        isEagerInit = AccessController.doPrivileged(new PrivilegedAction<Boolean>() {
                            @Override
                            public Boolean run() {
                                return ((SmartFactoryBean<?>) factory).isEagerInit();
                            }
                        }, getAccessControlContext());
                    }
                    else {
                        isEagerInit = (factory instanceof SmartFactoryBean &&
                                ((SmartFactoryBean<?>) factory).isEagerInit());
                    }
                    if (isEagerInit) {
                        getBean(beanName);
                    }
                }
                else {
                    //不是工厂Bean。利用getBean(beanName);创建对象，3.11.1看这里
                    //这个getBean(beanName)和在测试类里面写的ioc.getBean(ExtConfig.class)本质上是一样的
                    getBean(beanName);
                }
            }
        }
   
        //所有bean都利用getBean创建完成后，拿到所有的bean对象
		for (String beanName : beanNames) {
			Object singletonInstance = getSingleton(beanName);
            //判断所有bean是否是SmartInitializingSingleton
            //如果是就执行afterSingletonsInstantiated()
			if (singletonInstance instanceof SmartInitializingSingleton) {
				final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
				if (System.getSecurityManager() != null) {
					AccessController.doPrivileged(new PrivilegedAction<Object>() {
						@Override
						public Object run() {
							smartSingleton.afterSingletonsInstantiated();
							return null;
						}
					}, getAccessControlContext());
				}
				else {
					smartSingleton.afterSingletonsInstantiated();
				}
			}
		}
	}


```

##### 3.11.1调用AbstractBeanFactory.getBean(String name)

```java
@Override
public Object getBean(String name) throws BeansException {
   return doGetBean(name, null, null, false);
}

protected <T> T doGetBean(
			final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
			throws BeansException {

		final String beanName = transformedBeanName(name);
		Object bean;

		//先获取缓存中保存的单实例Bean。如果能获取到说明这个Bean之前被创建过（所有创建过的单实例Bean都会被缓存起来）
        //3.11.1.1看这里
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
			if (logger.isDebugEnabled()) {
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}

		else {
			//缓存中获取不到，开始Bean的创建对象流程
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			//获取BeanFactory
			BeanFactory parentBeanFactory = getParentBeanFactory();
            //获取是否有父工厂的存在
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				// Not found -> check parent.
				String nameToLookup = originalBeanName(name);
				if (args != null) {
					// Delegation to parent with explicit args.
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else {
					// No args -> delegate to standard getBean method.
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
			}

			if (!typeCheckOnly) {
                //标记当前bean已经被创建，避免多线程的时候创建了两个bean               
				markBeanAsCreated(beanName);
			}

			try {
                //获取Bean的定义信息；
				final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);

				//获取当前Bean依赖的其他Bean;如果有，按照getBean()把依赖的Bean先创建出来
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dep : dependsOn) {
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
						registerDependentBean(dep, beanName);
						getBean(dep);
					}
				}

				//启动单实例Bean的创建流程
				if (mbd.isSingleton()) {
                    //3.11.1.3回调看这里
					sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
						@Override
						public Object getObject() throws BeansException {
							try {
                                //在这里创建单实例Bean，3.11.1.2看这里
								return createBean(beanName, mbd, args);
							}
							catch (BeansException ex) {
								// Explicitly remove instance from singleton cache: It might have been put there
								// eagerly by the creation process, to allow for circular reference resolution.
								// Also remove any beans that received a temporary reference to the bean.
								destroySingleton(beanName);
								throw ex;
							}
						}
					});
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}

				//多余代码省略
		return (T) bean;
	}
```

###### 3.11.1.1调用DefaultSingletonBeanRegistry. getSingleton(String beanName)

```java
@Override
public Object getSingleton(String beanName) {
   return getSingleton(beanName, true);
}

protected Object getSingleton(String beanName, boolean allowEarlyReference) {
        //就是从这个this.singletonObjects里面
       //private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(256);
       //获取到的所有保存下来的单实例Bean的
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			synchronized (this.singletonObjects) {
				singletonObject = this.earlySingletonObjects.get(beanName);
				if (singletonObject == null && allowEarlyReference) {
					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
					if (singletonFactory != null) {
						singletonObject = singletonFactory.getObject();
						this.earlySingletonObjects.put(beanName, singletonObject);
						this.singletonFactories.remove(beanName);
					}
				}
			}
		}
		return (singletonObject != NULL_OBJECT ? singletonObject : null);
	}

```



###### 3.11.1.2调用AbstractAutowireCapableBeanFactory.createBean(String beanName, RootBeanDefinition mbd, Object[] args)

```java
@Override
protected Object createBean(String beanName, RootBeanDefinition mbd, Object[] args) throws BeanCreationException {
   if (logger.isDebugEnabled()) {
      logger.debug("Creating instance of bean '" + beanName + "'");
   }
    //拿到 RootBeanDefinition
   RootBeanDefinition mbdToUse = mbd;

   //多余代码省略
   try {
      //让BeanPostProcessor[InstantiationAwareBeanPostProcessor]先拦截，看是否能返回代理对象
       //类似解析在Spring注解驱动开发笔记-aop篇三.5.2.1.1.1有详解
      Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
      if (bean != null) {
         return bean;
      }
   }
   catch (Throwable ex) {
      throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
            "BeanPostProcessor before instantiation of bean failed", ex);
   }

    //如果没有返回bean的代理对象那么就实际创建指定的bean，3.11.1.2.1走这里
   Object beanInstance = doCreateBean(beanName, mbdToUse, args);
   if (logger.isDebugEnabled()) {
      logger.debug("Finished creating instance of bean '" + beanName + "'");
   }
    //返回bean实例
   return beanInstance;
}
```

3.11.1.2.1调用AbstractAutowireCapableBeanFactory.doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)---实际创建指定的bean

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)
      throws BeanCreationException {

   // Instantiate the bean.
   BeanWrapper instanceWrapper = null;
   if (mbd.isSingleton()) {
      instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
   }
   if (instanceWrapper == null) {
       //创建bean实例，当这个方法执行完成后，bean的对象就创建出来了，3.11.1.2.1.1走这里
      instanceWrapper = createBeanInstance(beanName, mbd, args);
   }
   final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);
   Class<?> beanType = (instanceWrapper != null ? instanceWrapper.getWrappedClass() : null);
   mbd.resolvedTargetType = beanType;

   //允许一些BeanPostProcessor修改Bean定义信息
   synchronized (mbd.postProcessingLock) {
      if (!mbd.postProcessed) {
         try {
             //调用MergedBeanDefinitionPostProcessor的postProcessMergedBeanDefinition处理
            applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
         }
         catch (Throwable ex) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                  "Post-processing of merged bean definition failed", ex);
         }
         mbd.postProcessed = true;
      }
   }

  //多余代码省略
    
   // Initialize the bean instance.
   Object exposedObject = bean;
   try {
       //为Bean赋值，依赖注入，3.11.1.2.1.2走这里
      populateBean(beanName, mbd, instanceWrapper);
      if (exposedObject != null) {
          //进行bean的初始化，3.11.1.2.1.3走这里
         exposedObject = initializeBean(beanName, exposedObject, mbd);
      }
   }
   //多余代码省略

   //注册bean的销毁方法
   try {
      registerDisposableBeanIfNecessary(beanName, bean, mbd);
   }
   catch (BeanDefinitionValidationException ex) {
      throw new BeanCreationException(
            mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
   }

   return exposedObject;
}
```



3.11.1.2.1.1调用AbstractAutowireCapableBeanFactory.createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args)---创建bean实例，当这个方法执行完成后，bean的对象就创建出来了

```java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args) {
   //解析创建bean的类型
   Class<?> beanClass = resolveBeanClass(mbd, beanName);
	//多余代码省略

   if (mbd.getFactoryMethodName() != null)  {
       //利用工厂方法或者对象的构造器创建对象
      return instantiateUsingFactoryMethod(beanName, mbd, args);
   }
   //多余代码省略
}
```



3.11.1.2.1.2调用AbstractAutowireCapableBeanFactory.populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw)---为Bean赋值，依赖注入

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) {
   //多余代码省略   
   if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {      
      for (BeanPostProcessor bp : getBeanPostProcessors()) {
          //在赋值之前遍历InstantiationAwareBeanPostProcessors
         //然后执行postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)
         if (bp instanceof InstantiationAwareBeanPostProcessor) {
            InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
            if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
               continueWithPropertyPopulation = false;
               break;
            }
         }
      }
   }
   //多余代码省略 
   if (hasInstAwareBpps || needsDepCheck) {
      PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
      if (hasInstAwareBpps) {
         for (BeanPostProcessor bp : getBeanPostProcessors()) {
             //在赋值之前遍历InstantiationAwareBeanPostProcessors
              //然后执行postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName)
            if (bp instanceof InstantiationAwareBeanPostProcessor) {
               InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
               pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
               if (pvs == null) {
                  return;
               }
            }
         }
      }
      if (needsDepCheck) {
         checkDependencies(beanName, mbd, filteredPds, pvs);
      }
   }
    
//==========================================================赋值之前============================================================   

    //为属性Bean利用setter方法进行赋值，依赖注入
   applyPropertyValues(beanName, mbd, bw, pvs);
}
```



3.11.1.2.1.3调用AbstractAutowireCapableBeanFactory. initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd)---进行bean的初始化操作

```java
protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
       //多余代码已省略
    //执行xxxAware接口的方法
      invokeAwareMethods(beanName, bean);
    
    //再次调用本类下的该方法用于处理XXXAware接口的方法回调
    invokeAwareMethods(beanName, bean);		      
    
    //拿到所有初始化之前的后置处理器，并执行
    ppedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    
    //执行初始化方法的方法
      invokeInitMethods(beanName, wrappedBean, mbd);
    
    //拿到所有初始化之后的后置处理器，并执行
      wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    //多余代码已省略
}
```



###### 3.11.1.3调用DefaultSingletonBeanRegistry.getSingleton(String beanName, ObjectFactory<?> singletonFactory)---把创建初始化好的bean变成单例

```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
  		//多余代码省略
        //把创建好的bean实例添加到单实例bean中去
            addSingleton(beanName, singletonObject);
       	//多余代码省略
      return (singletonObject != NULL_OBJECT ? singletonObject : null);
   }
}
```

##### 接下来逐级返回，bean实例就创建完成了



#### 3.12调用AbstractApplicationContext.finishRefresh()完成BeanFactory的初始化创建工作，IoC容器创建完成

```java
 protected void finishRefresh() {
        //初始化和生命周期有关的后置处理器；LifecycleProcessor
        //默认从容器中找是否有LifecycleProcessor的组件，如果没有就默认创建一个加入到IoC容器中
        initLifecycleProcessor();

        // 拿到前面定义的生命周期处理器（BeanFactory）；回调onRefresh()；
        getLifecycleProcessor().onRefresh();

        //发布容器刷新完成事件
        publishEvent(new ContextRefreshedEvent(this));

        //Participate in LiveBeansView MBean, if active.
        LiveBeansView.registerApplicationContext(this);
    }
```

