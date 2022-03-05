

# Spring注解驱动开发笔记-扩展原理篇

BeanPostProcessor本质上就是在Bean对象的初始化之前做了什么，初始化之后又做了什么

## 一、BeanFactoryPostProcessor

BeanPostProcessor：Bean后置处理器，Bean创建对象初始化前后进行的拦截工作的，详情可看Spring注解驱动开发笔记-AOP篇的AOP原理部分有讲到

BeanFactoryPostProcessor：BeanFactory的后置处理器，在BeanFactory标准初始化之后调用（所有的Bean定义已经保存加载到BeanFactory中，但是Bean的实例并没有被创建），用来定制和修改BeanFactory中的内容。

BeanDefinitionRegistryPostProcessor(BeaPostProcessor子接口)：在BeanFactoryPostProcessor之前调用(所有Bean定义信息将要被加载到BeanFactory中)

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

![image-20220223011214937](C:\Users\Samsara\Desktop\学习资料和IDE\java资料\3_文档资料\API文档\SSM\Spring注解驱动开发\image-20220223011214937.png)

显然是在IoC容器创建（**Bean实例创建之前**）之前就执行了BeanFactoryPostProcessor



### 2.源码分析-MyBeanFactoryPostProcessor.postProcessBeanFactory()执行时机

#### 2.1创建IoC容器（调用AnnotationConfigApplicationContext的构造器）

```java
AnnotationConfigApplicationContext ioc = new AnnotationConfigApplicationContext(ExtConfig.class);
//AnnotationConfigApplicationContext的构造器
public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
   this();
   register(annotatedClasses);
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

                //执行BeanFactoryPostProcessors，这次看这里
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
    //这次走这里
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
    //调用本类的这个方法
   invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

 
   beanFactory.clearMetadataCache();
}
```

2.2.1.1.1调用本类下的 invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

```java
private static void invokeBeanFactoryPostProcessors(Collection<? extends BeanFactoryPostProcessor> postProcessors, ConfigurableListableBeanFactory beanFactory) {
   for (BeanFactoryPostProcessor postProcessor : postProcessors) {
       //遍历所有的BeanFactoryPostProcessor，调用到我们自己MyBeanFactoryPostProcessor重写目标方法
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

![image-20220223022303038](C:\Users\Samsara\Desktop\学习资料和IDE\java资料\3_文档资料\API文档\SSM\Spring注解驱动开发\image-20220223022303038.png)

显然是在所有Bean定义信息将要被加载到BeanFactory中（**BeanFactoryPostProcessor**）**之前**就执行了BeanDefinitionRegistryPostProcessor



### 2.源码分析-BeanDefinitionRegistryPostProcessor(BeaPostProcessor子接口).postProcessBeanDefinitionRegistry执行时机

#### 2.1创建IoC容器（调用AnnotationConfigApplicationContext的构造器）

```java
AnnotationConfigApplicationContext ioc = new AnnotationConfigApplicationContext(ExtConfig.class);
//AnnotationConfigApplicationContext的构造器
public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
   this();
   register(annotatedClasses);
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

                //执行BeanFactoryPostProcessors，这次也看这里
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
    //这次也走这里
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
       //在beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class)这里面创建、初始化、注册BeanFactoryPostProcessor
       //类似于BeanPostProcessors创建，这个在Spring注解驱动开发笔记-aop篇三.4中有具体解析
						currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
						processedBeans.add(ppName);
						reiterate = true;
					}
				}
				sortPostProcessors(currentRegistryProcessors, beanFactory);
				registryProcessors.addAll(currentRegistryProcessors);
                //调用本类的这个方法执行自己写的postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry)
				invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
				currentRegistryProcessors.clear();
			}

			//当上个方法invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry)调用完后
    		//2.2.1.1.2走这里
    		//这个方法就是调用自己写的postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory)
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
