

# Spring注解驱动开发笔记-扩展原理篇

BeanPostProcessor本质上就是在Bean对象的初始化之前做了什么，初始化之后又做了什么

## 一、BeanFactoryPostProcessor

BeanPostProcessor：Bean后置处理器，Bean创建对象初始化前后进行的拦截工作的，详情可看Spring注解驱动开发笔记-AOP篇的AOP原理部分有讲到

BeanFactoryPostProcessor：BeanFactory的后置处理器，在BeanFactory标准初始化之后调用（所有的Bean定义已经保存加载到BeanFactory中，但是Bean的实例并没有被创建），用来定制和修改BeanFactory中的内容。

BeanDefinitionRegistryPostProcessor(BeaPostProcessor子接口)：在BeanFactoryPostProcessor之前调用(所有Bean定义信息将要被加载到BeanFactory中之前)

### 1.环境搭建

#### 1.1创建实体类User

```java
public class User {
    public User() {
        System.out.println("User的构造器执行了。。。");
    }
    public void init(){
        System.out.println("User的init()执行了。。。");
    }
    public void destroy(){
        System.out.println("User的destroy()执行了。。。");
    }
}

```

#### 1.2创建配置类ExtConfig

```java
@ComponentScan("com.wxr.ext")
@Configuration
public class ExtConfig {
    @Bean
    public User user(){
        return new User();
    }

}
```

#### 1.3创建MyBeanFactoryPostProcessor实现BeanFactoryPostProcessor接口

```java
@Component
public class MyBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        System.out.println("MyBeanFactoryPostProcessor执行了postProcessBeanFactory");
        int cout=beanFactory.getBeanDefinitionCount();
        System.out.println("有"+cout+"个组件");
        String[] name = beanFactory.getBeanDefinitionNames();
        System.out.println("有这些个bean：");
        for (String s : name) {
            System.out.println(s);
        }
    }
}
```

#### 1.4测试代码和结果

```java
@Test
public void test01(){
    AnnotationConfigApplicationContext ioc = new AnnotationConfigApplicationContext(ExtConfig.class);
    System.out.println("ioc容器创建完成");
    ioc.close();
}
```

![image-20220223011214937](C:\Users\MateBook\Desktop\IDE、数据库系统等工具\API文档\SSM\Spring注解驱动开发\image-20220223011214937.png)

显然是在IoC容器创建（**Bean实例创建之前**）之前就执行了BeanFactoryPostProcessor



### 2.源码分析-MyBeanFactoryPostProcessor.postProcessBeanFactory()执行时机

#### 2.1创建IoC容器（调用AnnotationConfigApplicationContext的构造器）

```java
AnnotationConfigApplicationContext ioc = new AnnotationConfigApplicationContext(ExtConfig.class);
//AnnotationConfigApplicationContext的构造器
public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
   this();
   register(annotatedClasses);
    //2.2走这里
   refresh();
}
```

#### 2.2调用AbstractApplicationContext.refresh()刷新容器

因为invokeBeanFactoryPostProcessors(beanFactory)比finishBeanFactoryInitialization(beanFactory)先执行

所以BeanFactoryPostProcessor，在BeanFactory标准初始化之后调用（所有的Bean定义已经保存加载到BeanFactory中，但是Bean的实例并没有被创建）

```java
/**
 *这是AbstractApplicationContext.refresh()
 */
@Override
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			
			prepareRefresh();

			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			prepareBeanFactory(beanFactory);

			try {
				postProcessBeanFactory(beanFactory);

                //执行BeanFactoryPostProcessor
                //2.2.1走这里
				invokeBeanFactoryPostProcessors(beanFactory);

                //注册bean的后置处理器，这个在Spring注解驱动开发笔记-aop篇三.4中有具体解析
				registerBeanPostProcessors(beanFactory);
              
                //调用本类的finishBeanFactoryInitialization(beanFactory)完成BeanFactory的初始化工作
                //这个在Spring注解驱动开发笔记-aop篇三.5中有具体解析
				finishBeanFactoryInitialization(beanFactory);
                //多余代码省略
 			}
    }
```

##### 2.2.1调用AbstractApplicationContext. invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory)执行BeanFactory的后置处理器

```java
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
    //2.2.1.1走这里
   PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());
   if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
      beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
      beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
   }
}
```

###### 2.2.1.1调用PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

```java
public static void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {
    
		//多余代码省略（在BeanDefinitionRegistryPostProcessor的执行时候会详细讲解二、2.2.1.1）
    	
		//从这里获取到类型是BeanFactoryPostProcessor的所有名字
		String[] postProcessorNames =beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

		// Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
		// Ordered, and the rest.
		List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<BeanFactoryPostProcessor>();
		List<String> orderedPostProcessorNames = new ArrayList<String>();
		List<String> nonOrderedPostProcessorNames = new ArrayList<String>();
        //遍历拿到所有BeanFactoryPostProcessor的名字
        //根据实现排序接口与否加入到不同的地方
		for (String ppName : postProcessorNames) {
			if (processedBeans.contains(ppName)) {
				// skip - already processed in first phase above
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

   
   // 先调用实现priorityOrdered接口的
   sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
   invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

   // 然后调用实现Ordered接口的
   List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<BeanFactoryPostProcessor>();
   for (String postProcessorName : orderedPostProcessorNames) {
      orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
   }
   sortPostProcessors(orderedPostProcessors, beanFactory);
   invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

   // 最后调用没实现接口的 这次走这里
   List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<BeanFactoryPostProcessor>();
   for (String postProcessorName : nonOrderedPostProcessorNames) {
       //遍历拿到所有的BeanFactoryPostProcessor
       //在beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class)这里面创建、初始化、注册BeanFactoryPostProcessor
       //类似于BeanPostProcessors创建，这个在Spring注解驱动开发笔记-aop篇三.4中有具体解析
      nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
   }
    //2.2.1.1.1走这里
   invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

 
   beanFactory.clearMetadataCache();
}
```

2.2.1.1.1调用本类下的 invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

```java
private static void invokeBeanFactoryPostProcessors(Collection<? extends BeanFactoryPostProcessor> postProcessors, ConfigurableListableBeanFactory beanFactory) {
   for (BeanFactoryPostProcessor postProcessor : postProcessors) {
       //遍历所有的BeanFactoryPostProcessor，调用到我们自己MyBeanFactoryPostProcessor重写目标方法
       //2.2.1.1.1.1走这里
      postProcessor.postProcessBeanFactory(beanFactory);
   }
}
```

2.2.1.1.1.1调用到我们自己MyBeanFactoryPostProcessor重写目标方法，完成调用

```java
@Component
public class MyBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        System.out.println("MyBeanFactoryPostProcessor执行了postProcessBeanFactory");
        String[] name = beanFactory.getBeanDefinitionNames();
        System.out.println("有这些个bean：");
        for (String s : name) {
            System.out.println(s);
        }
    }
}
```

















## 二、BeanDefinitionRegistryPostProcessor(BeaPostProcessor子接口)

BeanDefinitionRegistryPostProcessor(BeaPostProcessor子接口)：在BeanFactoryPostProcessor之前调用(所有Bean定义信息将要被加载到BeanFactory中)

利用BeanDefinitionRegistryPostProcessor给容器中额外添加一些组件

### 1.环境搭建

#### 1.1创建实体类User

```java
public class Student {
    public Student() {
        System.out.println("Student的构造器执行了。。。");
    }
}

```

#### 1.2创建配置类ExtConfig

```java
@ComponentScan("com.wxr.ext")
@Configuration
public class ExtConfig {
    @Bean
    public User user(){
        return new User();
    }

}
```

#### 1.3创建MyBeanDefinitionRegistryPostProcessor实现BeanDefinitionRegistryPostProcessor接口

```java
@Component
public class MyBeanDefinitionRegistryPostProcessor implements BeanDefinitionRegistryPostProcessor {
    /***
     * BeanDefinitionRegistry:
     * Bean定义信息的保存中心，以后BeanFactory就是按照BeanDefinitionRegistry里面保存的每一个Bean定义信息创建bean实例
     *
     * @param registry
     * @throws BeansException
     */
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
        System.out.println("MyBeanDefinitionRegistryPostProcessor执行了postProcessBeanDefinitionRegistry");
        System.out.println("Bean的数量是："+registry.getBeanDefinitionCount());
        //利用BeanDefinitionRegistryPostProcessor给容器中额外添加一些组件
        AbstractBeanDefinition beanDefinition = BeanDefinitionBuilder.rootBeanDefinition(Student.class).getBeanDefinition();
        registry.registerBeanDefinition("student",beanDefinition );
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        System.out.println("MyBeanDefinitionRegistryPostProcessor执行了postProcessBeanFactory");
        String[] name = beanFactory.getBeanDefinitionNames();
        System.out.println("有这些个Bean：");
        for (String s : name) {
            System.out.println(s);
        }
    }
}

```

#### 1.4测试代码和结果

```java
 @Test
    public void test02(){
        AnnotationConfigApplicationContext ioc = new AnnotationConfigApplicationContext(ExtConfig.class);
        System.out.println("ioc容器创建完成");
        ioc.close();
   }
```

![image-20220223022303038](C:\Users\MateBook\Desktop\IDE、数据库系统等工具\API文档\SSM\Spring注解驱动开发\image-20220223022303038.png)

显然是在所有Bean定义信息将要被加载到BeanFactory中（**BeanFactoryPostProcessor**）**之前**就执行了BeanDefinitionRegistryPostProcessor



### 2.源码分析-BeanDefinitionRegistryPostProcessor(BeaPostProcessor子接口).postProcessBeanDefinitionRegistry执行时机

#### 2.1创建IoC容器（调用AnnotationConfigApplicationContext的构造器）

```java
AnnotationConfigApplicationContext ioc = new AnnotationConfigApplicationContext(ExtConfig.class);
//AnnotationConfigApplicationContext的构造器
public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
   this();
   register(annotatedClasses);
    //2.2走这里
   refresh();
}
```

#### 2.2调用AbstractApplicationContext.refresh()刷新容器

和一、2的前两步一样走invokeBeanFactoryPostProcessors(beanFactory);

所以BeanFactoryPostProcessor，在BeanFactory标准初始化之后调用（所有的Bean定义已经保存加载到BeanFactory中，但是Bean的实例并没有被创建）

```java
/**
 *这是AbstractApplicationContext.refresh()
 */
@Override
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			
			prepareRefresh();

			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			prepareBeanFactory(beanFactory);

			try {
				postProcessBeanFactory(beanFactory);

                //执行BeanFactoryPostProcessors
                //2.2.1走这里
				invokeBeanFactoryPostProcessors(beanFactory);

                //注册bean的后置处理器，这个在Spring注解驱动开发笔记-aop篇三.4中有具体解析
				registerBeanPostProcessors(beanFactory);
              
                //调用本类的finishBeanFactoryInitialization(beanFactory)完成BeanFactory的初始化工作
                //这个在Spring注解驱动开发笔记-aop篇三.5中有具体解析
				finishBeanFactoryInitialization(beanFactory);
                //多余代码省略
 			}
    }
```

##### 2.2.1调用AbstractApplicationContext. invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory)执行BeanFactory的后置处理器

```java
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
    //2.2.1.1走这里
   PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());
   if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
      beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
      beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
   }
}
```

###### 2.2.1.1调用PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

```java
public static void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {
    
			//多余代码省略

		   //从这里获取到类型是BeanDefinitionRegistryPostProcessor的所有名字
			List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<BeanDefinitionRegistryPostProcessor>();

			// 先调用实现priorityOrdered接口的
			String[] postProcessorNames =
					beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
			for (String ppName : postProcessorNames) {
				if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
					currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
					processedBeans.add(ppName);
				}
			}
			sortPostProcessors(currentRegistryProcessors, beanFactory);
			registryProcessors.addAll(currentRegistryProcessors);
			invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
			currentRegistryProcessors.clear();

			//  先调用实现Ordered接口的
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

			
            // 最后调用没实现接口的 这次走这里
			boolean reiterate = true;
			while (reiterate) {
				reiterate = false;
				postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
				for (String ppName : postProcessorNames) {
					if (!processedBeans.contains(ppName)) {
        //遍历拿到所有的BeanDefinitionRegistryPostProcessor
        //在beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class)这里面创建、初始化、注册                                      BeanDefinitionRegistryPostProcessor
       //类似于BeanPostProcessors创建，这个在Spring注解驱动开发笔记-aop篇三.4中有具体解析
						currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
						processedBeans.add(ppName);
						reiterate = true;
					}
				}
				sortPostProcessors(currentRegistryProcessors, beanFactory);
				registryProcessors.addAll(currentRegistryProcessors);
                //调用本类的这个方法执行自己写的postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry)
                //2.2.1.1.1走这里
				invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
				currentRegistryProcessors.clear();
			}

			//当上个方法invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry)调用完后
    		//这个方法就是调用自己写的postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory)
    		//2.2.1.1.2走这里
			invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
			invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
		}
		//多余代码已省略（详见一.2.2.1.1）
		//就是再来从容器中类型是BeanFactoryPostProcessor的所有名字和一.2.2.1.1流程一样
}
```

2.2.1.1.1调用本类下的 invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);

```java
	private static void invokeBeanDefinitionRegistryPostProcessors(Collection<? extends BeanDefinitionRegistryPostProcessor> postProcessors, BeanDefinitionRegistry registry) {
		for (BeanDefinitionRegistryPostProcessor postProcessor : postProcessors) {
              //遍历BeanDefinitionRegistryPostProcessor
 //调用我们自己写的MyBeanDefinitionRegistryPostProcessor重写目标方法postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry)
			//2.2.1.1.1.1走这里
            postProcessor.postProcessBeanDefinitionRegistry(registry);
		}
	}
```

2.2.1.1.1.1调用我们自己写的MyBeanDefinitionRegistryPostProcessor重写目标方法postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry)

```java
@Component
public class MyBeanDefinitionRegistryPostProcessor implements BeanDefinitionRegistryPostProcessor {
    /***
     * BeanDefinitionRegistry:
     * Bean定义信息的保存中心，以后BeanFactory就是按照BeanDefinitionRegistry里面保存的每一个Bean定义信息创建bean实例
     *
     * @param registry
     * @throws BeansException
     */
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
        System.out.println("MyBeanDefinitionRegistryPostProcessor执行了postProcessBeanDefinitionRegistry");
        System.out.println("Bean的数量是："+registry.getBeanDefinitionCount());
        //手动自己注册一个bean
        AbstractBeanDefinition beanDefinition = BeanDefinitionBuilder.rootBeanDefinition(Student.class).getBeanDefinition();
        registry.registerBeanDefinition("student",beanDefinition );
    }

    //多余代码已省略
}

```





2.2.1.1.2调用本类下的invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);

```java
private static void invokeBeanFactoryPostProcessors(
      Collection<? extends BeanFactoryPostProcessor> postProcessors, ConfigurableListableBeanFactory beanFactory) {
   for (BeanFactoryPostProcessor postProcessor : postProcessors) {
       //和2.2.1.1.1类似
      //遍历BeanDefinitionRegistryPostProcessor
//调用我们自己写的MyBeanDefinitionRegistryPostProcessor重写目标方法postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory)
     //2.2.1.1.2.1走这里
       postProcessor.postProcessBeanFactory(beanFactory);
   }
}
```

2.2.1.1.2.1调用我们自己写的MyBeanDefinitionRegistryPostProcessor重写目标方法postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory)

```java
@Component
public class MyBeanDefinitionRegistryPostProcessor implements BeanDefinitionRegistryPostProcessor {
    //多余代码已省略
    
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        System.out.println("MyBeanDefinitionRegistryPostProcessor执行了postProcessBeanFactory");
        String[] name = beanFactory.getBeanDefinitionNames();
        System.out.println("有这些个Bean：");
        for (String s : name) {
            System.out.println(s);
        }
    }
}

```









## 三、ApplicationListener

ApplicationListener：监听容器中发布的事件。事件模型驱动开发

![image-20220223212123619](C:\Users\MateBook\Desktop\IDE、数据库系统等工具\API文档\SSM\Spring注解驱动开发\image-20220223212123619.png)

例如：

ContextRefreshedEvent：容器刷新完成（所有bean都完成创建）会发布这个事件

ContextClosedEvent：关闭容器的时候会发布这个事件

### 1.环境搭建

#### 1.1创建配置类ExtConfig

```java
@ComponentScan("com.wxr.ext")
@Configuration
public class ExtConfig {
 
}
```

#### 1.2创建MyApplicationListener实现ApplicationListener接口

```java
@Component
public class MyApplicationListener implements ApplicationListener<ApplicationEvent> {

    /**
     * 当容器中发布此事件以后，该方法出发
     * 监听ApplicationEvent及下面的子事件
     * 根据ApplicationEvent的继承关系：
     * 例如：
	 *		ContextRefreshedEvent：容器刷新完成（所有bean都完成创建）会发布这个事件
     *      ContextClosedEvent：关闭容器的时候会发布这个事件
     * 步骤： 
     *      1.写一个监听器来监听某个事件（ApplicationEvent及其子类）
     *      2.把监听器加入到容器中
     *      3.只要容器中有相应类型的事件发布，我们就监听到这个事件
     * @param event
     */
    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        System.out.println("收到事件："+event);
    }
}

```

#### 1.3测试代码和结果

```java
public class IocTest {
    @Test
    public void test02(){
        AnnotationConfigApplicationContext ioc = new AnnotationConfigApplicationContext(ExtConfig.class);
        //发布事件
        ioc.publishEvent(new ApplicationEvent(new String("我发布的事件")) {
        });
        System.out.println("ioc容器创建完成");
        ioc.close();
    }
}

```

![image-20220223214007457](C:\Users\MateBook\Desktop\IDE、数据库系统等工具\API文档\SSM\Spring注解驱动开发\image-20220223214007457.png)





### 2.源码分析-ApplicationListener原理

根据三.1的输出结果，我们按照先后顺序从ContextRefreshedEvent，IocTest$1，ContextClosedEvent进行分析

#### 2.1ContextRefreshedEvent事件

##### 2.1.1创建IoC容器（调用AnnotationConfigApplicationContext的构造器）

```java
public class IocTest {
    @Test
    public void test02(){
        //2.1.1.1走这里
        AnnotationConfigApplicationContext ioc = new AnnotationConfigApplicationContext(ExtConfig.class);
        //发布事件      
        ioc.publishEvent(new ApplicationEvent(new String("我发布的事件")) {
        });
        System.out.println("ioc容器创建完成");
        ioc.close();
    }
}


//AnnotationConfigApplicationContext的构造器
public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
   this();
   register(annotatedClasses);
    //2.1.1.1走这里
   refresh();
}
```

###### 2.1.1.1调用AbstractApplicationContext.refresh()刷新容器

```java
/**
 *这是AbstractApplicationContext.refresh()
 */
@Override
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			
			prepareRefresh();

			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			prepareBeanFactory(beanFactory);

			try {
				postProcessBeanFactory(beanFactory);

                //执行BeanFactoryPostProcessors，这个在一.2.2.1和二.2.2.1中有具体解析
				invokeBeanFactoryPostProcessors(beanFactory);

                //注册bean的后置处理器，这个在Spring注解驱动开发笔记-aop篇三.4中有具体解析
				registerBeanPostProcessors(beanFactory);
                  
                // 初始化事件的多播器（派发器），在2.1.1.1.1.1.1会提到
				initApplicationEventMulticaster();
                
                
                // 注册所有的监听器，从容器中拿到所有的监听器，把他们注册到applicationEventMulticaster中
				registerListeners();
                
                            
                //调用本类的finishBeanFactoryInitialization(beanFactory)完成BeanFactory的初始化工作
                //实例化剩下的单实例Bean               
                //这个在Spring注解驱动开发笔记-aop篇三.5中有具体解析
				finishBeanFactoryInitialization(beanFactory);
                
                //调用本类的finishRefresh()，容器刷新完成
                //2.1.1.1.1走这里
				finishRefresh();
               //多余代码省略
 			}
    }
```

2.1.1.1.1调用AbstractApplicationContext.finishRefresh()完成容器刷新

```java
protected void finishRefresh() {
   // Initialize lifecycle processor for this context.
   initLifecycleProcessor();

   // Propagate refresh to lifecycle processor first.
   getLifecycleProcessor().onRefresh();

   //发布事件ContextRefreshedEvent刚刚好2.1ContextRefreshedEvent是一样的
   //调用本类下的publishEvent(new ContextRefreshedEvent(this))
    //2.1.1.1.1.1走这里
   publishEvent(new ContextRefreshedEvent(this));

   // Participate in LiveBeansView MBean, if active.
   LiveBeansView.registerApplicationContext(this);
}
```

2.1.1.1.1.1调用AbstractApplicationContext. publishEvent(new ContextRefreshedEvent(this)) 开始进行ContextRefreshedEvent事件发布

```java
@Override
public void publishEvent(ApplicationEvent event) {
   publishEvent(event, null);
}

protected void publishEvent(Object event, ResolvableType eventType) {
		Assert.notNull(event, "Event must not be null");
		if (logger.isTraceEnabled()) {
			logger.trace("Publishing event in " + getDisplayName() + ": " + event);
		}

		// Decorate event as an ApplicationEvent if necessary
		ApplicationEvent applicationEvent;
		if (event instanceof ApplicationEvent) {
			applicationEvent = (ApplicationEvent) event;
		}
		else {
			applicationEvent = new PayloadApplicationEvent<Object>(this, event);
			if (eventType == null) {
				eventType = ((PayloadApplicationEvent)applicationEvent).getResolvableType();
			}
		}

		// Multicast right now if possible - or lazily once the multicaster is initialized
		if (this.earlyApplicationEvents != null) {
			this.earlyApplicationEvents.add(applicationEvent);
		}
		else {
            //1.获取事件的多播器（派发器）:getApplicationEventMulticaster()
            //2.派发事件：multicastEvent(applicationEvent, eventType)
            //2.1.1.1.1.1.1走这里
            //2.1.1.1.1.1.2走这里
			getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);
		}

		// Publish event via parent context as well...
		if (this.parent != null) {
			if (this.parent instanceof AbstractApplicationContext) {
				((AbstractApplicationContext) this.parent).publishEvent(event, eventType);
			}
			else {
				this.parent.publishEvent(event);
			}
		}
	}
```

2.1.1.1.1.1.1调用getApplicationEventMulticaster()获取事件的多播器（派发器）

```java
ApplicationEventMulticaster getApplicationEventMulticaster() throws IllegalStateException {
   if (this.applicationEventMulticaster == null) {
      throw new IllegalStateException("ApplicationEventMulticaster not initialized - " +
            "call 'refresh' before multicasting events via the context: " + this);
   }
   return this.applicationEventMulticaster;
}
```

获取方法在refresh()的initApplicationEventMulticaster()中

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



2.1.1.1.1.1.2调用SimpleApplicationEventMulticaster. multicastEvent(final ApplicationEvent event, ResolvableType eventType) 进行派发事件

```java
@Override
public void multicastEvent(final ApplicationEvent event, ResolvableType eventType) {
   ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
    //拿到所有的ApplicationListener挨个遍历
   for (final ApplicationListener<?> listener : getApplicationListeners(event, type)) {
      Executor executor = getTaskExecutor();
       //如果有异步执行的功能，那么通过多线程的方式异步派发invokeListener(listener, event);
      if (executor != null) {
         executor.execute(new Runnable() {
            @Override
            public void run() {
               invokeListener(listener, event);
            }
         });
      }
      else {
          //否则用同步的方式直接调用本类下的invokeListener(listener, event);
          //2.1.1.1.1.1.2.1走这里
         invokeListener(listener, event);
      }
   }
}
```

2.1.1.1.1.1.2.1调用SimpleApplicationEventMulticaster.invokeListener(ApplicationListener<?> listener, ApplicationEvent event)

```java
protected void invokeListener(ApplicationListener<?> listener, ApplicationEvent event) {
   ErrorHandler errorHandler = getErrorHandler();
   if (errorHandler != null) {
      try {
         doInvokeListener(listener, event);
      }
      catch (Throwable err) {
         errorHandler.handleError(err);
      }
   }
   else {
       //这次走这里
      doInvokeListener(listener, event);
   }
}


//再次调用本类下的doInvokeListener(ApplicationListener listener, ApplicationEvent event)
private void doInvokeListener(ApplicationListener listener, ApplicationEvent event) {
		try {
            //2.1.1.1.1.1.2.1.1
            //走这个方法直接回调到我们自己写的MyApplicationListener. onApplicationEvent(ApplicationEvent event) 
			listener.onApplicationEvent(event);
		}
		catch (ClassCastException ex) {
			String msg = ex.getMessage();
			if (msg == null || msg.startsWith(event.getClass().getName())) {
				// Possibly a lambda-defined listener which we could not resolve the generic event type for
				Log logger = LogFactory.getLog(getClass());
				if (logger.isDebugEnabled()) {
					logger.debug("Non-matching event type for listener: " + listener, ex);
				}
			}
			else {
				throw ex;
			}
		}
	}
```

2.1.1.1.1.1.2.1.1回到我们自己写的MyApplicationListener. onApplicationEvent(ApplicationEvent event)完成事件发布

```java
@Component
public class MyApplicationListener implements ApplicationListener<ApplicationEvent> {
    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        System.out.println("收到事件："+event);
    }
}
```







#### 2.2IocTest$1事件

##### 2.2.1调用自己写的发布时间方法ioc.publishEvent(new ApplicationEvent(new String("我发布的事件")) {});

```java
public class IocTest {
    @Test
    public void test02(){
        AnnotationConfigApplicationContext ioc = new AnnotationConfigApplicationContext(ExtConfig.class);
        //2.2.1.1走这里       
        ioc.publishEvent(new ApplicationEvent(new String("我发布的事件")) {
        });
        System.out.println("ioc容器创建完成");
        ioc.close();
    }
}
```

###### 2.2.1.1直接调用AbstractApplicationContext. publishEvent(new ContextRefreshedEvent(this)) 开始进行ContextRefreshedEvent事件发布

```java
@Override
public void publishEvent(ApplicationEvent event) {
   publishEvent(event, null);
}

protected void publishEvent(Object event, ResolvableType eventType) {
		Assert.notNull(event, "Event must not be null");
		if (logger.isTraceEnabled()) {
			logger.trace("Publishing event in " + getDisplayName() + ": " + event);
		}

		// Decorate event as an ApplicationEvent if necessary
		ApplicationEvent applicationEvent;
		if (event instanceof ApplicationEvent) {
			applicationEvent = (ApplicationEvent) event;
		}
		else {
			applicationEvent = new PayloadApplicationEvent<Object>(this, event);
			if (eventType == null) {
				eventType = ((PayloadApplicationEvent)applicationEvent).getResolvableType();
			}
		}

		// Multicast right now if possible - or lazily once the multicaster is initialized
		if (this.earlyApplicationEvents != null) {
			this.earlyApplicationEvents.add(applicationEvent);
		}
		else {
            //1.获取事件的多播器（派发器）:getApplicationEventMulticaster()
            //2.派发事件：multicastEvent(applicationEvent, eventType)，这次走这里
			getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);
		}

		// Publish event via parent context as well...
		if (this.parent != null) {
			if (this.parent instanceof AbstractApplicationContext) {
				((AbstractApplicationContext) this.parent).publishEvent(event, eventType);
			}
			else {
				this.parent.publishEvent(event);
			}
		}
	}
```

2.2.1.1后面的逻辑和2.1.1.1.1.1.2之后一样了这里就不赘述了





#### 2.3ContextClosedEvent事件

##### 2.3.1调用ioc.close()

```java
public class IocTest {
    @Test
    public void test02(){
        AnnotationConfigApplicationContext ioc = new AnnotationConfigApplicationContext(ExtConfig.class);
        //发布事件 
        ioc.publishEvent(new ApplicationEvent(new String("我发布的事件")) {
        });   
        System.out.println("ioc容器创建完成");
        //2.3.1.1走这里
        ioc.close();
    }
}
```

###### 2.3.1.1调用AbstractApplicationContext.close()关闭容器

```java
@Override
public void close() {
   synchronized (this.startupShutdownMonitor) {
       //这次走这里
      doClose();
      // If we registered a JVM shutdown hook, we don't need it anymore now:
      // We've already explicitly closed the context.
      if (this.shutdownHook != null) {
         try {
            Runtime.getRuntime().removeShutdownHook(this.shutdownHook);
         }
         catch (IllegalStateException ex) {
            // ignore - VM is already shutting down
         }
      }
   }
}
protected void doClose() {
     //多余代码省略
      try {
         //发布ContextClosedEvent事件
          //2.3.1.1.1走这里
         publishEvent(new ContextClosedEvent(this));
      }
      //多余代码省略
}
```

2.3.1.1.1调用AbstractApplicationContext.publishEvent(ApplicationEvent event)发布ContextClosedEvent事件

2.3.1.1.1.1后面的逻辑和2.1.1.1.1.1.2之后一样了这里就不赘述了



## 四、@EventListener

@EventListener：把它添加在某一个类的方法上，也可以监听容器中发布的事件

### 1.环境搭建

#### 1.1创建配置类ExtConfig

```java
@ComponentScan("com.wxr.ext")
@Configuration
public class ExtConfig {
 
}
```

#### 1.2创建UserService，使用@EventListener

```java
@Service
public class UserService {
    @EventListener(classes = ApplicationEvent.class)
    public void linsten(ApplicationEvent event){
        System.out.println("UserService监听到的事件"+event);
    }

}
```

#### 1.3测试代码和结果

```java
public class IocTest {
    @Test
    public void test02(){
        AnnotationConfigApplicationContext ioc = new AnnotationConfigApplicationContext(ExtConfig.class);
        //发布事件
        ioc.publishEvent(new ApplicationEvent(new String("我发布的事件")) {
        });
        System.out.println("ioc容器创建完成");
        ioc.close();
    }
}

```

![image-20220223231143950](C:\Users\MateBook\Desktop\IDE、数据库系统等工具\API文档\SSM\Spring注解驱动开发\image-20220223231143950.png)



























