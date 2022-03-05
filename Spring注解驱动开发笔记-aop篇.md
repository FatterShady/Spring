

# Spring注解驱动开发笔记-aop篇

## 一、什么是AOP

AOP【底层动态代理】:指在程序运行期间动态地将某段代码切入到指定方法的指定位置进行运行的编程方式

​	

## 二、AOP的使用案例

##### 1.导入aop模块：Spring AOP：（spring-aspects）

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-aspects</artifactId>
    <version>4.3.12.RELEASE</version>
</dependency>
```



##### 2.定义一个业务逻辑类（MathCalculator）

```java
public class MathCalculator {
    public int div(int i,int j){
        System.out.println("MathCalculator.div执行了。。。");
        return i/j;
    }
}
```



##### 3.定义日志切面类（LogAspect）：切面类里的方法需要动态感知MathCalculator.div运行到哪里

###### 	3.1给切面类的目标方法标注通知注解

###### 	3.2告诉Spring谁是切面类（@Aspect）

```java
/**
新增功能：在业务逻辑运行的时候将日志进行打印（方法之前、方法运行结束、方法出现异常。。。）
通知方法：

前置通知(@Before)
后置通知(@After)
返回通知(@AfterReturning)
异常通知(@AfterThrowing)
环绕通知(@Around)：动态代理，手动推进目标方法的运行（joinPoint.proceed）

*/

//告诉Spring当前类是一个切面类
@Aspect
public class LogAspect {

    /**
     * 抽取公共的切入点表达式
     */
    @Pointcut("execution(public int com.wxr.aop.MathCalculator.div(int,int))")
    public void pointCut(){
    }

    //前置通知(@Before)：logStart，在MathCalculator.div运行之前运行
    @Before("com.wxr.aop.LogAspect.pointCut()")
    public void logStart(){
        System.out.println("MathCalculator.div的@Before...");
    }

    //后置通知(@After)：logEnd，在MathCalculator.div运行结束运行(无论方法是正常结束还是异常结束)
    @After("com.wxr.aop.LogAspect.pointCut()")
    public void logEnd(){
        System.out.println("MathCalculator.div的@After...");
    }

    //返回通知(@AfterReturning)：logReturn，在MathCalculator.div运行正常返回运行
    @AfterReturning("com.wxr.aop.LogAspect.pointCut()")
    public void logReturn(){
        System.out.println("MathCalculator.div的@AfterReturning...");
    }

    //异常通知(@AfterThrowing)：logException，在MathCalculator.div运行出现异常后运行
    @AfterThrowing("com.wxr.aop.LogAspect.pointCut()")
    public void logException(){
        System.out.println("MathCalculator.div的@AfterThrowing...");
    }
}
```



##### 4.创建配置类MyConfigOfAop

###### 	4.1将切面类和业务逻辑类都加入到配置类(容器)中

###### 	4.2给配置类中加入@EnableAspectJAutoProxy，开启基于注解的aop模式

```java
//在Spring中有很多的@EnableXXX，关于开启某一项功能替代xml配置
//开启基于注解的aop模式
@EnableAspectJAutoProxy
@Configuration
public class MyConfigOfAop {

    //业务逻辑类加入到容器中
    @Bean
    public MathCalculator mathCalculator(){
        return new MathCalculator();
    }

    //切面类加入到容器中
    @Bean
    public LogAspect logAspect(){
        return new LogAspect();
    }
    
}
```



##### 5.测试代码+输出结果

版本1：

```java
@Test
public void test01(){
    AnnotationConfigApplicationContext ioc = new AnnotationConfigApplicationContext(MyConfigOfAop.class);
    MathCalculator mathCalculator = ioc.getBean(MathCalculator.class);
    mathCalculator.div(1,1 );
    ioc.close();
}
```

![image-20220218230604999](C:\Users\MateBook\Desktop\IDE、数据库系统等工具\API文档\SSM\Spring注解驱动开发\image-20220218230604999.png)



版本2：

```java
@Test
public void test02(){
    AnnotationConfigApplicationContext ioc = new AnnotationConfigApplicationContext(MyConfigOfAop.class);
    MathCalculator mathCalculator = ioc.getBean(MathCalculator.class);
    mathCalculator.div(1,0 );
    ioc.close();
}
```

##### ![image-20220218230531801](C:\Users\MateBook\Desktop\IDE、数据库系统等工具\API文档\SSM\Spring注解驱动开发\image-20220218230531801.png)







































## 三、AOP原理（已忽略无关紧要的部分代码）

### 1.@EnableAspectJAutoProxy

```java
//关键看它
@Import(AspectJAutoProxyRegistrar.class)
public @interface EnableAspectJAutoProxy {
}
```

### 2.AspectJAutoProxyRegistrar(手动组件注册)

```java
/**在《Spring注解驱动开发笔记-ioc篇的Bean的注册：五、1.3.3》
 *也使用过这个ImportBeanDefinitionRegistrar接口
 */
/**
重写registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry)
用于自定义手动注册一些组件到bean中
*/
class AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar {
	@Override
	public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        //重点关注
		AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);
        //多余代码省略
	}
}


public abstract class AopConfigUtils {
public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry) {
       //调用本类的registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry, null)
		return registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry, null);
}
public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry, Object  source) {
    //调用本类的 registerOrEscalateApcAsRequired(Class<?> cls, BeanDefinitionRegistry registry, Object source)
   
	return registerOrEscalateApcAsRequired(AnnotationAwareAspectJAutoProxyCreator.class, registry, source);
}

/**
 *AopConfigUtils.registerOrEscalateApcAsRequired(Class<?> cls, BeanDefinitionRegistry registry, @Nullable Object source)
 * 目的：
 *  向容器中注册一个名字叫internalAutoProxyCreator，类型是AnnotationAwareAspectJAutoProxyCreator的组件
 */
private static BeanDefinition registerOrEscalateApcAsRequired(Class<?> cls, BeanDefinitionRegistry registry, @Nullable Object source) {
	//利用AnnotationAwareAspectJAutoProxyCreator的class类型新建RootBeanDefinition
	RootBeanDefinition beanDefinition = new RootBeanDefinition(cls);
	//向beanDefinition对象中加入其它一些属性
	beanDefinition.setSource(source);
	beanDefinition.getPropertyValues().add("order", Ordered.HIGHEST_PRECEDENCE);
	beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
	//将新定义的beanDefinition注入到注册中心
	//注意：AUTO_PROXY_CREATOR_BEAN_NAME其实就是org.springframework.aop.config.internalAutoProxyCreator（）
	registry.registerBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME, beanDefinition);
    //返回的beanDefinition现在就是internalAutoProxyCreator（AnnotationAwareAspectJAutoProxyCreator）
	return beanDefinition;
	}
}
```

### 3.AnnotationAwareAspectJAutoProxyCreator类（本质上就是BeanPostProcessor）

#### UML类图重点关注红色部分：

![image-20220221163717378](C:\Users\MateBook\Desktop\IDE、数据库系统等工具\API文档\SSM\Spring注解驱动开发\image-20220221163717378.png)

BeanPostProcessor: 后置处理器，在bean初始化完成的前后做事情

​		Spring底层对BeanPostProcessor的使用：bean赋值、注入其他组件、@Autowired、生命周期注解功能、@Async、......

​		BeanFactoryAware:自动装配Spring底层的BeanFactory组件

**所以AnnotationAwareAspectJAutoProxyCreator类本质上就是个BeanPostProcessor**



#### 3.1AbstractAutoProxyCreator（AnnotationAwareAspectJAutoProxyCreator的顶级父类）

```java
/**
 *1.从继承图关系可以看出，要关注BeanFactoryAware、BeanPostProcessor接口
 *  就需要看AnnotationAwareAspectJAutoProxyCreator类的父类AbstractAutoProxyCreator
 *2.在《Spring注解驱动开发笔记-ioc篇的Bean的生命周期：四、自动装配---Aware注入Spring底层组件》
 * 《Spring注解驱动开发笔记-ioc篇的Bean的生命周期：一、指定初始化和销毁方法---@Bean的3.4通过自定义 BeanPostProcessor（bean的后置处理器）接口实现》
 * 实现过这种XXXAware的类似接口,也实现过BeanPostProcessor的接口
 */



/**
 *只保留需要关注的方法
 */
public abstract class AbstractAutoProxyCreator extends ProxyProcessorSupport
      implements SmartInstantiationAwareBeanPostProcessor, BeanFactoryAware {
    
    
    	@Override
	public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
		Object cacheKey = getCacheKey(beanClass, beanName);
		if (beanName == null || !this.targetSourcedBeans.contains(beanName)) {
			if (this.advisedBeans.containsKey(cacheKey)) {
				return null;
			}
			if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
				this.advisedBeans.put(cacheKey, Boolean.FALSE);
				return null;
			}
		}
		if (beanName != null) {
			TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
			if (targetSource != null) {
				this.targetSourcedBeans.add(beanName);
				Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
				Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
				this.proxyTypes.put(cacheKey, proxy.getClass());
				return proxy;
			}
		}

		return null;
	}
    
    
	@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		if (bean != null) {
			Object cacheKey = getCacheKey(bean.getClass(), beanName);
			if (!this.earlyProxyReferences.contains(cacheKey)) {
				return wrapIfNecessary(bean, beanName, cacheKey);
			}
		}
		return bean;
	}
    
    @Override
    public void setBeanFactory(BeanFactory beanFactory) {
		this.beanFactory = beanFactory;
	}
    
}
```

#### 3.2AbstractAdvisorAutoProxyCreator（AbstractAutoProxyCreator的子类）

```java
public abstract class AbstractAdvisorAutoProxyCreator extends AbstractAutoProxyCreator {
    
    
	/**
	 *只保留和BeanPostProcessor、BeanFactoryAware有关的方法
	 */
    
    /**
     *重写了AbstractAutoProxyCreator.setBeanFactory(BeanFactory beanFactory) 那么就调用的是它重写的子类
     */
    
	@Override
	public void setBeanFactory(BeanFactory beanFactory) {
		super.setBeanFactory(beanFactory);
		if (!(beanFactory instanceof ConfigurableListableBeanFactory)) {
			throw new IllegalArgumentException(
					"AdvisorAutoProxyCreator requires a ConfigurableListableBeanFactory: " + beanFactory);
		}
       //会调用下面的initBeanFactory((ConfigurableListableBeanFactory) beanFactory);
		initBeanFactory((ConfigurableListableBeanFactory) beanFactory);
	}
    
    protected void initBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		this.advisorRetrievalHelper = new BeanFactoryAdvisorRetrievalHelperAdapter(beanFactory);
	}

}
```

#### 3.3AspectJAwareAdvisorAutoProxyCreator类（AbstractAdvisorAutoProxyCreator的子类）

#### 3.4回到AnnotationAwareAspectJAutoProxyCreator类（AspectJAwareAdvisorAutoProxyCreator的子类）

```java
/**
 * 重写了父类AbstractAdvisorAutoProxyCreator.initBeanFactory(ConfigurableListableBeanFactory beanFactory)
 *那么就调用的是该类的initBeanFactory(ConfigurableListableBeanFactory beanFactory) 
 */

@Override
protected void initBeanFactory(ConfigurableListableBeanFactory beanFactory) {
   super.initBeanFactory(beanFactory);
   if (this.aspectJAdvisorFactory == null) {
      this.aspectJAdvisorFactory = new ReflectiveAspectJAdvisorFactory(beanFactory);
   }
   this.aspectJAdvisorsBuilder =
         new BeanFactoryAspectJAdvisorsBuilderAdapter(beanFactory, this.aspectJAdvisorFactory);
}
```



### 4.AOP原理---创建、初始化并注册AnnotationAwareAspectJAutoProxyCreator

上一篇着重讲了@EnableAspectJAutoProxy的作用，以及分析了AnnotationAwareAspectJAutoProxyCreator的相关类的关系这篇我们开始讲解AnnotationAwareAspectJAutoProxyCreator的创建。

#### 4.1创建IoC容器（调用AnnotationConfigApplicationContext的构造器）

```java
AnnotationConfigApplicationContext ioc = new AnnotationConfigApplicationContext(MyConfigOfAop.class);

//AnnotationConfigApplicationContext的构造器
public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
   this();
   register(annotatedClasses);
    //4.2看这里
   refresh();
}
```

#### 4.2调用AbstractApplicationContext.refresh()刷新容器

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

				invokeBeanFactoryPostProcessors(beanFactory);

                //调用AbstractApplicationContext.registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory)
                //注册bean的后置处理器，4.2.1看这里
				registerBeanPostProcessors(beanFactory);
              
                //调用本类的finishBeanFactoryInitialization(beanFactory)完成BeanFactory的初始化工作，在第5点会具体提到
				finishBeanFactoryInitialization(beanFactory);
 			}
    }
```

##### 4.2.1调用AbstractApplicationContext.registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory)注册Bean的后置处理器

```java
/**
 *这是AbstractApplicationContext.registerBeanPostProcessors(beanFactory);
 */        
    protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
		PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
	}
            
```

##### 4.2.1.1调用PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this)对BeanPostProcessor进行创建、初始化和注册

```java
 /**
   *这是PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
   */            
 public static void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext)
 { 
     //拿到IoC容器中已经定义了的需要创建对象的所有的BeanPostProcessor
	String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);
	int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
    //给容器中添加其他的BeanPostProcessor
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

	//优先注册实现了PriorityOrdered接口的BeanPostProcessor
	sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
	registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

	List<BeanPostProcessor> orderedPostProcessors = new ArrayList<BeanPostProcessor>();
	for (String ppName : orderedPostProcessorNames) {
        //使用getBean(ppName, BeanPostProcessor.class)创建+初始化BeanPostProcessor（因为它也是一个特殊的bean所以调用getBean）
        //4.2.1.1.1看这里
		BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
		orderedPostProcessors.add(pp);
		if (pp instanceof MergedBeanDefinitionPostProcessor) {
			internalPostProcessors.add(pp);
		}
	}
	sortPostProcessors(orderedPostProcessors, beanFactory);
     //注册实现了Ordered接口的BeanPostProcessor（AnnotationAwareAspectJAutoProxyCreator本次就是走这个方法）
     //4.2.1.1.2看这里
	registerBeanPostProcessors(beanFactory, orderedPostProcessors);
	//多余代码已省略
     
}           
```

###### 4.2.1.1.1调用beanFactory.getBean(ppName, BeanPostProcessor.class)创建+初始化BeanPostProcessor

```java
/**
 *AbstractBeanFactory.getBean(String name, Class<T> requiredType) ;
 */	
@Override
public <T> T getBean(String name, Class<T> requiredType) throws BeansException {
	return doGetBean(name, requiredType, null, false);
}

/**
 *然后调用到AbstractBeanFactory.doGetBean(name, requiredType, null, false);
 */
protected <T> T doGetBean(final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly) throws BeansException {
       //多余代码已省略
        /*
         *提前检查缓存中是否注册过了单实例bean,如果能获取到说明bean是之前被创建过的，直接使用
         */
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
    
       /*
         *如果缓存中没有获取到单实例bean：internalAutoProxyCreator（AnnotationAwareAspectJAutoProxyCreator），那么再创建
         */
        else {
				if (mbd.isSingleton()) {
             //这里就是在创建internalAutoProxyCreator（AnnotationAwareAspectJAutoProxyCreator）
    	    //又会调用DefaultSingletonBeanRegistry.getSingleton(String beanName, ObjectFactory<?> singletonFactory)
					sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
						@Override
						public Object getObject() throws BeansException {
							try {
                                  //如果获取不到就创建一个Bean实例
                                //createBean就是真正用于创建BeanPostProcessor对象
                                //这里就是创建internalAutoProxyCreator（AnnotationAwareAspectJAutoProxyCreator）
                                //4.2.1.1.1.1看这里
								return createBean(beanName, mbd, args);
							}
							catch (BeansException ex) {
								destroySingleton(beanName);
								throw ex;
							}
						}
					});
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}
         }
         
}
	

/**
 *再调用到DefaultSingletonBeanRegistry.getSingleton(String beanName, ObjectFactory<?> singletonFactory)
 */
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
    	try {
            		//如果获取不到就创建Bean实例，又回到上一段代码
					singletonObject = singletonFactory.getObject();
					newSingleton = true;
				}
}


```

4.2.1.1.1.1调用AbstractAutowireCapableBeanFactory.createBean(String beanName, RootBeanDefinition mbd, Object[] args)用于真正创建bean实例

```java
	@Override
	protected Object createBean(String beanName, RootBeanDefinition mbd, Object[] args) throws BeanCreationException {
        try {
 			//希望后置处理器在这里能够返回一个代理对象，这次不走这里
			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
			if (bean != null) {
				return bean;
			}
		}
        
		//第一次走这里4.2.1.1.1.1.1看这里 又调用了本类下的doCreateBean(beanName, mbdToUse, args)
		Object beanInstance = doCreateBean(beanName, mbdToUse, args);
	}
```

4.2.1.1.1.1.1又调用了本类（AbstractAutowireCapableBeanFactory）下的doCreateBean(beanName, mbdToUse, args)用于bean的各种创建、属性赋值、初始化

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)
		throws BeanCreationException {
		
    //再次调用本类下的createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args)
    //创建Bean（internalAutoProxyCreator（AnnotationAwareAspectJAutoProxyCreator））的实例
	instanceWrapper = createBeanInstance(beanName, mbd, args);
    //这个bean就是（internalAutoProxyCreator（AnnotationAwareAspectJAutoProxyCreator））
	final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);
    /**
    	还调用本类下的populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw)
    	给Bean（internalAutoProxyCreator（AnnotationAwareAspectJAutoProxyCreator））的属性赋值
    	
    	最后调用本类下的initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd)
   		给Bean（internalAutoProxyCreator（AnnotationAwareAspectJAutoProxyCreator））初始化
   	*/
		Object exposedObject = bean;
		populateBean(beanName, mbd, instanceWrapper);
    //4.2.1.1.1.1.1.1看这里
		exposedObject = initializeBean(beanName, exposedObject, mbd);
    
    
    //返回创建好的Bean实例（internalAutoProxyCreator（AnnotationAwareAspectJAutoProxyCreator））
    		return exposedObject;
	}
   

```

4.2.1.1.1.1.1.1调用本类下的initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) 

用于bean（internalAutoProxyCreator（AnnotationAwareAspectJAutoProxyCreator））的初始化

```java
/**
  * 本类下的initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd)   
  */
     protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
    //多余代码已省略
    //再次调用本类下的该方法用于处理Aware接口的方法回调,这里是设置BeanFactory
    invokeAwareMethods(beanName, bean);		         
    //拿到所有初始化之前的后置处理器
    ppedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    //执行初始化方法的方法
      invokeInitMethods(beanName, wrappedBean, mbd);
    //拿到所有初始化之后的后置处理器
      wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
     }
        
    /*
     *本类下的invokeAwareMethods(final String beanName, final Object bean)
     */
     private void invokeAwareMethods(final String beanName, final Object bean) {
    if (bean instanceof Aware) {
    	if (bean instanceof BeanNameAware) {
    		((BeanNameAware) bean).setBeanName(beanName);
    	}
    	if (bean instanceof BeanClassLoaderAware) {
    		((BeanClassLoaderAware) bean).setBeanClassLoader(getBeanClassLoader());
    	}
    	if (bean instanceof BeanFactoryAware) {
            //调用AbstractAdvisorAutoProxyCreator. setBeanFactory(BeanFactory beanFactory),设置BeanFactory
            //4.2.1.1.1.1.1.1.1看这里
    		((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
    	}
    }

    
```

4.2.1.1.1.1.1.1.1调用AbstractAdvisorAutoProxyCreator. setBeanFactory(BeanFactory beanFactory)

```java
@Override
public void setBeanFactory(BeanFactory beanFactory) {
    //调用父类的AbstractAutoProxyCreator.setBeanFactory(beanFactory)
   super.setBeanFactory(beanFactory);
    //因为AnnotationAwareAspectJAutoProxyCreator子类重写过这个方法，所以会调用重写过的方法
    //4.2.1.1.1.1.1.1.2看这里
   initBeanFactory((ConfigurableListableBeanFactory) beanFactory);
}
```

4.2.1.1.1.1.1.1.2调用AnnotationAwareAspectJAutoProxyCreator.initBeanFactory(ConfigurableListableBeanFactory beanFactory)

```java
@Override
protected void initBeanFactory(ConfigurableListableBeanFactory beanFactory) {
   super.initBeanFactory(beanFactory);
   if (this.aspectJAdvisorFactory == null) {
      // 利用反射得到了 AdvisorFactory工厂
      this.aspectJAdvisorFactory = new ReflectiveAspectJAdvisorFactory(beanFactory);
   }
    //将之前创建的通知工厂和Bean工厂包装了一次,利用适配器获取 AdvisorsBuilder建造器
   this.aspectJAdvisorsBuilder =
         new BeanFactoryAspectJAdvisorsBuilderAdapter(beanFactory, this.aspectJAdvisorFactory);
}
```

###### 4.2.1.1.2逐级返回到调用PostProcessorRegistrationDelegate.registerBeanPostProcessors

```java
/**
 *调用PostProcessorRegistrationDelegate.registerBeanPostProcessors
 * 把Bean（internalAutoProxyCreator（AnnotationAwareAspectJAutoProxyCreator））注册到BeanFactory中
 */

private static void registerBeanPostProcessors(
			ConfigurableListableBeanFactory beanFactory, List<BeanPostProcessor> postProcessors) {
		for (BeanPostProcessor postProcessor : postProcessors) {
            //把BeanPostProcessor注册到BeanFactory中
			beanFactory.addBeanPostProcessor(postProcessor);
		}
	}
```

#### 4.3至此完成AnnotationAwareAspectJAutoProxyCreator的创建、初始化、注册



### 5.AOP原理---寻找AnnotationAwareAspectJAutoProxyCreator的执行时机

上一篇主要介绍了AnnotationAwareAspectJAutoProxyCreator是如何被注册到spring容器中的，这一篇主要介绍AnnotationAwareAspectJAutoProxyCreator执行的时机，也就是它所起的作用

#### 5.1UML类图

![image-20220221163717378](C:\Users\MateBook\Desktop\IDE、数据库系统等工具\API文档\SSM\Spring注解驱动开发\image-20220221163717378.png)

根据类图可知，AnnotationAwareAspectJAutoProxyCreator本质上是InstantiationAwareBeanPostProcessor



#### 5.2调用AbstractApplicationContext.refresh()里面的finishBeanFactoryInitialization(beanFactory)完成BeanFactory的初始化工作实例化剩下的单实例bean

```java
/**
 *这是AbstractApplicationContext.refresh()
 */
@Override
	public void refresh() throws BeansException, IllegalStateException {
            //多余代码已经省略
            //调用AbstractApplicationContext.registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory)
            //注册bean的后置处理器，在第4点已经做完了
   		   registerBeanPostProcessors(beanFactory);    
            //调用本类的finishBeanFactoryInitialization(beanFactory)完成BeanFactory的初始化工作，
        	//实例化剩余的单实例bean到容器中       
		   finishBeanFactoryInitialization(beanFactory);
   }
/**
 *这是AbstractApplicationContext.finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory)
 */     
    protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
        //多余代码已经省略
		//创建所有剩余的（非懒加载）单例，依次创建Bean实例,就是在创建bean的对象、
        //5.2.1看这里
		beanFactory.preInstantiateSingletons();
	}

```

##### 5.2.1调用DefaultListableBeanFactory.preInstantiateSingletons() 创建单实例Bean对象

```java
@Override
	public void preInstantiateSingletons() throws BeansException {
		//多余代码已经省略
        
		List<String> beanNames = new ArrayList<String>(this.beanDefinitionNames);


		//遍历获取容器中所有的Bean，依次创建对象
		// Trigger initialization of all non-lazy singleton beans...
		for (String beanName : beanNames) {
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
                //判断是否是工厂bean的逻辑
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
                    //调用AbstractBeanFactory.getBean()创建Bean
                    //5.2.1.1看这里
					getBean(beanName);
				}
			}
		}
```

###### 5.2.1.1调用AbstractBeanFactory.getBean()

```java
/**
 *调用AbstractBeanFactory.getBean()
 */
@Override
public Object getBean(String name) throws BeansException {
   return doGetBean(name, null, null, false);
}


/**
 *调用AbstractBeanFactory.doGetBean(final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
 */
	protected <T> T doGetBean(final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)throws BeansException {
        //多余代码已省略
        /*
         *提前检查缓存中是否注册过了单实例bean,如果能获取到说明bean是之前被创建过的，直接使用
         */
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
         /*
         *如果缓存中没有获取到单实例bean，那么再创建
         */
        else {
				if (mbd.isSingleton()) {
					sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
						@Override
						public Object getObject() throws BeansException {
							try {
                                //5.2.1.1.1走这里
								return createBean(beanName, mbd, args);
							}
							catch (BeansException ex) {
								destroySingleton(beanName);
								throw ex;
							}
						}
					});
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}
         }
        
        /**
         *再调用到DefaultSingletonBeanRegistry.getSingleton(String beanName, ObjectFactory<?> singletonFactory)
         */
    public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
                try {
                           //如果获取不到就创建Bean实例，又回到上一段代码
                            singletonObject = singletonFactory.getObject();
                            newSingleton = true;
                        }
        }

}
```

5.2.1.1.1调用AbstractAutowireCapableBeanFactory.createBean(String beanName, RootBeanDefinition mbd, Object[] args)

```java
	/**
	 *AbstractAutowireCapableBeanFactory.createBean(String beanName, RootBeanDefinition mbd, Object[] args)
	 */	
	@Override
	protected Object createBean(String beanName, RootBeanDefinition mbd, Object[] args) throws BeanCreationException {
		try {
			//5.2.1.1.1.1走这里 希望后置处理器在这里能够返回一个代理对象，如果能返回就使用
			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
			if (bean != null) {
				return bean;
			}
		}
			//如果不能就进入 doCreateBean(beanName, mbdToUse, args)，用于真正的创建一个bean实例，和4.2.1.1.1后的流程一样        
        	Object beanInstance = doCreateBean(beanName, mbdToUse, args);
	}



```

5.2.1.1.1.1调用AbstractAutowireCapableBeanFactory.resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) 

```java
/**
 *AbstractAutowireCapableBeanFactory.resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) 
 */
protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
         if (targetType != null) {
             //使用后置处理器尝试返回对象（实例化前）
             //5.2.1.1.1.1.1走这里
            bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
            if (bean != null) {
                //实例化后调用AbstractAutoProxyCreator.applyBeanPostProcessorsAfterInitialization
               bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
            }
         }
   mbd.beforeInstantiationResolved = (bean != null);
   return bean;
}
```

5.2.1.1.1.1.1调用AbstractAutowireCapableBeanFactory. applyBeanPostProcessorsBeforeInstantiation(Class<?> beanClass, String beanName) 

**结论：**

​     1.BeanPostProcessor是在Bean对象创建完成初始化前后调用的
​     2.**InstantiationAwareBeanPostProcessor(也就是AnnotationAwareAspectJAutoProxyCreator（他的父类实现了这个接口）)是在创建Bean对象之前先尝试用后置处理器返回对象**

```java
protected Object applyBeanPostProcessorsBeforeInstantiation(Class<?> beanClass, String beanName) {
    //遍历所有的BeanPostProcessor
    for (BeanPostProcessor bp : getBeanPostProcessors()) {
     //如果属于InstantiationAwareBeanPostProcessor，就执行它本类的postProcessBeforeInstantiation
      if (bp instanceof InstantiationAwareBeanPostProcessor) {
         InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
          //最后来到AbstractAutoProxyCreator.postProcessBeforeInstantiation(Class<?> beanClass, String beanName)
          //这里就是AnnotationAwareAspectJAutoProxyCreator的执行时机
          //6.1走这里
         Object result = ibp.postProcessBeforeInstantiation(beanClass, beanName);
         if (result != null) {
            return result;
         }
      }
   }
   return null;
}
```





### 6.AOP原理---AnnotationAwareAspectJAutoProxyCreator如何创建AOP代理对象

在5中讲解了AnnotationAwareAspectJAutoProxyCreator的执行时机，它是在普通bean的创建过程中，尝试创建一个bean的代理对象替代bean去注册到容器中。这个尝试过程首先调用了postProcessBeforInstantiation方法。

这一篇开始讲解这个方法，并往下调试分析aop代理对象的创建过程。

#### 6.1调用AbstractAutoProxyCreator.postProcessBeforeInstantiation(Class<?> beanClass, String beanName)（实例化前）

```java
/**
 *本次测试是按照二、AOP使用案例进行测试
 *由于前面写的MathCalculator这个bean是需要aop代理的bean，因此我们就以MathCalculator为例来分析这个bean的创建过程
 */

@Override
public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
    Object cacheKey = getCacheKey(beanClass, beanName);

   if (beanName == null || !this.targetSourcedBeans.contains(beanName)) {
       //判断当前bean（MathCalculator）是否在advisedBean（保存了所有需要增强的bean）当中
      if (this.advisedBeans.containsKey(cacheKey)) {
         return null;
      }
       /**1.判断当前bean是否是基础类型isInfrastructureClass(beanClass)的
          Advice、Pointcut、Advisor、AopInfrastructureBean或者是否是切面类（@Aspect）
        *在这里是false
        *
        *2.以及是否需要跳过shouldSkip(beanClass, beanName)
        *    2.1获取候选增强器（切面类中的通知方法）【List<Advisor> candidateAdvisors】
      	*	    在这里每一个封装的通知方法增强器是InstantiationModelAwarePointcutAdvisor
      	*	    判断每一个增强器是否是AspectJPointcutAdvisor类型（显然不是）的
      	*在这里是false
        */
      if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
         this.advisedBeans.put(cacheKey, Boolean.FALSE);
         return null;
      }
   }

   if (beanName != null) {
      TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
      if (targetSource != null) {
         this.targetSourcedBeans.add(beanName);
         Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
         Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
         this.proxyTypes.put(cacheKey, proxy.getClass());
         return proxy;
      }
   }
	//本次全部返回null，所以并没有拿到bean实例，会走正常创建bean的流程和4.2.1.1.1后的流程一样    
   return null;
}


protected boolean isInfrastructureClass(Class<?> beanClass) {
		boolean retVal = Advice.class.isAssignableFrom(beanClass) ||
				Pointcut.class.isAssignableFrom(beanClass) ||
				Advisor.class.isAssignableFrom(beanClass) ||
				AopInfrastructureBean.class.isAssignableFrom(beanClass);
		if (retVal && logger.isTraceEnabled()) {
			logger.trace("Did not attempt to auto-proxy infrastructure class [" + beanClass.getName() + "]");
		}
		return retVal;
	}

protected boolean shouldSkip(Class<?> beanClass, String beanName) {
        //获取候选增强器（切面类中的通知方法）
		List<Advisor> candidateAdvisors = findCandidateAdvisors();
        //判断每一个增强器是否是AspectJPointcutAdvisor类型的
		for (Advisor advisor : candidateAdvisors) {
			if (advisor instanceof AspectJPointcutAdvisor) {
				if (((AbstractAspectJAdvice) advisor.getAdvice()).getAspectName().equals(beanName)) {
					return true;
				}
			}
		}
		return super.shouldSkip(beanClass, beanName);
	}

    /**
     *在这里返回的false
     */
	protected boolean shouldSkip(Class<?> beanClass, String beanName) {
		return false;
	}


```

#### 6.2调用AbstractAutowireCapableBeanFactory. applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)（初始化后）

当MathCalculator的bean对象创建完成前后（走的正常bean创建流程后），又执行bean的初始化方法的前后执行所有BeanPostProcessor后置处理器的方法。而我们的AnnotationAwareAspectJAutoProxyCreator就是一个BeanPostProcessor，因此最终也会执行到它的postProcessBeforeInitialization（因为是空实现就不看了），postProcessAfterInitialization这个两个方法。

```java
/**
 *AbstractAutowireCapableBeanFactory. applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
 *
 */
	@Override
	public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
			throws BeansException {
		Object result = existingBean;
		for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
            //进行初始化后的方法，用于创建bean的动态代理对象
            //6.2.1走这里
			result = beanProcessor.postProcessAfterInitialization(result, beanName);
			if (result == null) {
				return result;
			}
		}
		return result;
	}


```

##### 6.2.1调用AbstractAutoProxyCreator. postProcessAfterInitialization(Object bean, String beanName)用于创建bean的动态代理对象

```java


/**
 *AbstractAutoProxyCreator. postProcessAfterInitialization(Object bean, String beanName)
 *用于创建bean的动态代理对象
 */
public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
   if (bean != null) {
      Object cacheKey = getCacheKey(bean.getClass(), beanName);
      if (!this.earlyProxyReferences.contains(cacheKey)) {
         /**调用本方法下的wrapIfNecessary(bean, beanName, cacheKey)如果必要的话进行包装
           *执行
           *作用：
           *一、
                1.获取当前bean的候选的所有增强器（通知方法）
                2.获取到能在bean中使用的增强器
                3.给增强器进行排序
            二、保存当前bean在advisedBeans中
            三、如果当前bean需要增强，创建当前bean的代理对象
            */
          return wrapIfNecessary(bean, beanName, cacheKey);
       }
   }
   return bean;
}

/**
 *AbstractAutoProxyCreator. wrapIfNecessary(Object bean, String beanName, Object cacheKey)
 */
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    //省略部分代码

 		 /**
            *作用一、
                    1.获取当前bean的候选的所有增强器（通知方法）
                    2.获取到能在bean中使用的增强器
                    3.给增强器进行排序
      	  */
        //6.2.1.1走这里
        Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
      	if (specificInterceptors != DO_NOT_PROXY) {
                  // 作用二、保存当前bean在advisedBeans中
      		this.advisedBeans.put(cacheKey, Boolean.TRUE);
                  //作用三、如果当前bean需要增强，创建当前bean的代理对象
            //6.2.1.2走这里
      		Object proxy = createProxy(
      				bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
      		this.proxyTypes.put(cacheKey, proxy.getClass());
      		return proxy;
      	}
      	this.advisedBeans.put(cacheKey, Boolean.FALSE);
      	return bean;
     }

```

###### 6.2.1.1作用一、调用AbstractAdvisorAutoProxyCreator.getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null)

```java
/**
 *AbstractAdvisorAutoProxyCreator.getAdvicesAndAdvisorsForBean(Class<?> beanClass, String beanName, TargetSource targetSource)
 */
	@Override
protected Object[] getAdvicesAndAdvisorsForBean(Class<?> beanClass, String beanName, TargetSource targetSource) {
    //调用本类下的abstractAdvisorAutoProxyCreator.findEligibleAdvisors(Class<?> beanClass, String beanName)
		List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
		if (advisors.isEmpty()) {
			return DO_NOT_PROXY;
		}
		return advisors.toArray();
	}

/**
 *AbstractAdvisorAutoProxyCreator.findEligibleAdvisors(Class<?> beanClass, String beanName)
 */·
protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
        //拿到所有的增强器
		List<Advisor> candidateAdvisors = findCandidateAdvisors();
        //看这一行，找到能在当前bean使用的增强器（找到切入Bean的通知方法有哪些）
		List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
		extendAdvisors(eligibleAdvisors);
		if (!eligibleAdvisors.isEmpty()) {
            //给可用的增强器排序
			eligibleAdvisors = sortAdvisors(eligibleAdvisors);
		}
		return eligibleAdvisors;
	}

/**
 *AbstractAdvisorAutoProxyCreator.findAdvisorsThatCanApply
 */
	protected List<Advisor> findAdvisorsThatCanApply(
		List<Advisor> candidateAdvisors, Class<?> beanClass, String beanName) {
		ProxyCreationContext.setCurrentProxiedBeanName(beanName);
		try {
            //找到能在当前bean使用的增强器（找到切入Bean的通知方法有哪些）
			return AopUtils.findAdvisorsThatCanApply(candidateAdvisors, beanClass);
		}
		finally {
			ProxyCreationContext.setCurrentProxiedBeanName(null);
		}
	}

/**
 *AopUtils.findAdvisorsThatCanApply(List<Advisor> candidateAdvisors, Class<?> clazz)
 */
	public static List<Advisor> findAdvisorsThatCanApply(List<Advisor> candidateAdvisors, Class<?> clazz) {
		if (candidateAdvisors.isEmpty()) {
			return candidateAdvisors;
		}
        //拿到所有的增强器来创建一个可用的集合
		List<Advisor> eligibleAdvisors = new LinkedList<Advisor>();
        /**
         *进行增强器是否可用逻辑的判断
         */
		for (Advisor candidate : candidateAdvisors) {
			if (candidate instanceof IntroductionAdvisor && canApply(candidate, clazz)) {
				eligibleAdvisors.add(candidate);
			}
		}
		boolean hasIntroductions = !eligibleAdvisors.isEmpty();
		for (Advisor candidate : candidateAdvisors) {
			if (candidate instanceof IntroductionAdvisor) {
				// already processed
				continue;
			}
			if (canApply(candidate, clazz, hasIntroductions)) {         
				eligibleAdvisors.add(candidate);
			}
		}
        //返回能用的增强器
		return eligibleAdvisors;
	}



```

###### 6.2.1.2作用三、AbstractAutoProxyCreator.createProxy(Class<?> beanClass, String beanName, Object[] specificInterceptors, TargetSource targetSource)

说明：

​		1.给容器中返回当前组件使用的cglib增强了的代理对象

​		2.以后容器中获取到的就是这个组件的代理对象

```java
/**
 *AbstractAutoProxyCreator.createProxy(Class<?> beanClass, String beanName, Object[] specificInterceptors, TargetSource targetSource)
 */
protected Object createProxy(Class<?> beanClass, String beanName, Object[] specificInterceptors, TargetSource targetSource) 
{
   if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
      AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
   }

    //创建代理工厂
   ProxyFactory proxyFactory = new ProxyFactory();
   proxyFactory.copyFrom(this);

   if (!proxyFactory.isProxyTargetClass()) {
      if (shouldProxyTargetClass(beanClass, beanName)) {
         proxyFactory.setProxyTargetClass(true);
      }
      else {
         evaluateProxyInterfaces(beanClass, proxyFactory);
      }
   }

    //获取所有增强器
   Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
    //将增强器添加到proxyFactory代理工厂中
   proxyFactory.addAdvisors(advisors);
   proxyFactory.setTargetSource(targetSource);
   customizeProxyFactory(proxyFactory);

   proxyFactory.setFrozen(this.freezeProxy);
   if (advisorsPreFiltered()) {
      proxyFactory.setPreFiltered(true);
   }
	//用proxyFactory代理工厂创建aop代理对象
   return proxyFactory.getProxy(getProxyClassLoader());
}


/**
 *ProxyFactory.getProxy(ClassLoader classLoader)
 */
	public Object getProxy(ClassLoader classLoader) {
		return createAopProxy().getProxy(classLoader);
	}


/**
 *ProxyCreatorSupport.createAopProxy()
 *
 */
	protected final synchronized AopProxy createAopProxy() {
		if (!this.active) {
			activate();
		}
        //创建AOP代理
		return getAopProxyFactory().createAopProxy(this);
	}

/**
  *DefaultAopProxyFactory.createAopProxy()
  *
  */
@Override
	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class<?> targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
                //返回jdk动态代理（实现了接口）
				return new JdkDynamicAopProxy(config);
			}
            //返回cglib动态代理（未实现接口）本次返回的是这个
			return new ObjenesisCglibAopProxy(config);
		}
		else {
			return new JdkDynamicAopProxy(config);
		}
	}



```

#### 6.3回到6.2返回bean的动态代理对象

```java
/**
 *AbstractAutowireCapableBeanFactory. applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
 *
 */
	@Override
	public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
			throws BeansException {
		Object result = existingBean;
		for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {       
			result = beanProcessor.postProcessAfterInitialization(result, beanName);
			if (result == null) {
				return result;
			}
		}
        //返回动态代理对象
		return result;
	}

```





### 7.AOP原理---AOP代理对象执行bean的目标方法的过程

在6中主要讲了上一篇主要讲解AnnotationAwareAspectJAutoProxyCreator是如何创建AOP代理对象的。

这一篇开始讲解AOP代理对象执行bean的目标方法的整个过程。

#### 7.1单元测试进入MathCalculator.div（int i,int j）方法

```java
/**
 *在容器创建之后，获取MathCalculator对象，是一个cglib代理对象。这个对象里面保存了详细信息，比如增强器、目标对象等，而增强器则是前面分析的几个通知方法。
 */
@Test
public void test02(){
    AnnotationConfigApplicationContext ioc = new AnnotationConfigApplicationContext(MyConfigOfAop.class);
    MathCalculator mathCalculator = ioc.getBean(MathCalculator.class);
    //7.2走这里
    mathCalculator.div(1,1 );
    ioc.close();
}
```

#### 7.2调用CglibAopProxy.intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy)执行代理对象的方法

拦截器链：每一个通知方法被包装为方法拦截器，利用MethodInterceptor机制依次执行

```java
@Override
public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
   Object oldProxy = null;
   boolean setProxyContext = false;
   Class<?> targetClass = null;
   Object target = null;
       //根据this.advised(ProxyFactory对象)获取将要执行的目标方法的拦截器链
       //7.2.1走这里
      List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
      Object retVal;

      //判断当前拦截器链是否为空
      if (chain.isEmpty() && Modifier.isPublic(method.getModifiers())) {
         //如果拦截器链为空，则直接执行目标方法
         Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
         retVal = methodProxy.invoke(target, argsToUse);
      }
      else {
         //如果拦截器链不为空，把需要执行的目标对象、目标方法、拦截器链等信息传入创建一个CglibMethodInvocation对象，并且调用它的proceed()方法
          //这里就是开始执行代理对象的增强方法了
          //7.2.3走这里
         retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
      }
      retVal = processReturnType(proxy, target, method, retVal);
      return retVal;
   }
   finally {
      if (target != null) {
         releaseTarget(target);
      }
      if (setProxyContext) {
         // Restore old proxy.
         AopContext.setCurrentProxy(oldProxy);
      }
   }
}
```

##### 7.2.1调用AdvisedSupport.getInterceptorsAndDynamicInterceptionAdvice(Method method, Class<?> targetClass)获取拦截器链

```java
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Method method, Class<?> targetClass) {
    //先从缓存中获取
   MethodCacheKey cacheKey = new MethodCacheKey(method);
   List<Object> cached = this.methodCache.get(cacheKey);
   if (cached == null) {
 //如果缓存为空，则调用拦截器链工厂DefaultAdvisorChainFactory. getInterceptorsAndDynamicInterceptionAdvice(Advised config, Method method, Class<?> targetClass)
       //7.2.1.1走这里
      cached = this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(
            this, method, targetClass);
      this.methodCache.put(cacheKey, cached);
   }
   return cached;
}
```

###### 7.2.1.1调用DefaultAdvisorChainFactory. getInterceptorsAndDynamicInterceptionAdvice(Advised config, Method method, Class<?> targetClass)

```java
@Override
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(
      Advised config, Method method, Class<?> targetClass) {

    //保存所有拦截器，长度为4个增强器+1个默认的ExposeInvocationInterceptor组成
   List<Object> interceptorList = new ArrayList<Object>(config.getAdvisors().length);
   Class<?> actualClass = (targetClass != null ? targetClass : method.getDeclaringClass());
   boolean hasIntroductions = hasMatchingIntroductions(config, actualClass);
   AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();

    //遍历增强器集合
   for (Advisor advisor : config.getAdvisors()) {
      if (advisor instanceof PointcutAdvisor) {
         PointcutAdvisor pointcutAdvisor = (PointcutAdvisor) advisor;
         if (config.isPreFiltered() || pointcutAdvisor.getPointcut().getClassFilter().matches(actualClass)) {
             //调用DefaultAdvisorAdapterRegistry.getInterceptors(Advisor advisor) 
             //构建Interceptor数组
             //7.2.1.1.1走这里
            MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
            MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
            if (MethodMatchers.matches(mm, method, actualClass, hasIntroductions)) {
               if (mm.isRuntime()) {
                  // Creating a new object instance in the getInterceptors() method
                  // isn't a problem as we normally cache created chains.
                  for (MethodInterceptor interceptor : interceptors) {
                     interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptor, mm));
                  }
               }
               else {
                  interceptorList.addAll(Arrays.asList(interceptors));
               }
            }
         }
      }
      else if (advisor instanceof IntroductionAdvisor) {
         IntroductionAdvisor ia = (IntroductionAdvisor) advisor;
         if (config.isPreFiltered() || ia.getClassFilter().matches(actualClass)) {
            Interceptor[] interceptors = registry.getInterceptors(advisor);
            interceptorList.addAll(Arrays.asList(interceptors));
         }
      }
      else {
         Interceptor[] interceptors = registry.getInterceptors(advisor);
         interceptorList.addAll(Arrays.asList(interceptors));
      }
   }
	//返回拦截器集合
   return interceptorList;
}

```

7.2.1.1.1调用DefaultAdvisorAdapterRegistry.getInterceptors(Advisor advisor) 构建interceptor数组

```java
@Override
public MethodInterceptor[] getInterceptors(Advisor advisor) throws UnknownAdviceTypeException {
    //创建 List<MethodInterceptor> 集合
   List<MethodInterceptor> interceptors = new ArrayList<MethodInterceptor>(3);
   Advice advice = advisor.getAdvice();
    //如果是MethodInterceptor，直接加入到集合中
   if (advice instanceof MethodInterceptor) {
      interceptors.add((MethodInterceptor) advice);
   }
    //如果不是，则使用AdvisorAdapter将增强器转为MethodInterceptor
   for (AdvisorAdapter adapter : this.adapters) {
      if (adapter.supportsAdvice(advice)) {
         interceptors.add(adapter.getInterceptor(advisor));
      }
   }
   if (interceptors.isEmpty()) {
      throw new UnknownAdviceTypeException(advisor.getAdvice());
   }
    //返回拦截器数组
   return interceptors.toArray(new MethodInterceptor[interceptors.size()]);
}
```

##### 7.2.2依次返回给7.2的chain对象



##### 7.2.3调用ReflectiveMethodInvocation.proceed()拦截器链的执行过程

拦截器链的执行就是触发拦截器链的调用过程：

1.如果没有拦截器执行目标方法，或者拦截器的索引和拦截去数组-1大小一样（指定到了最后一个拦截器）

2.链式获取每一个拦截器，拦截器执行invoke方法，每一个拦截器等待下一个拦截器执行完成返回以后再来执行

3.拦截器链的机制：保证通知方法和目标方法的执行顺序。

###### 7.2.3.1首次调用ReflectiveMethodInvocation.proceed()

```java
@Override
public Object proceed() throws Throwable {
   // 如果没有拦截器执行目标方法，或者拦截器的索引和拦截去数组-1大小一样（指定到了最后一个拦截器）
    //第一次currentInterceptorIndex=-1
   if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
      return invokeJoinpoint();
   }
//执行++currentInterceptionIdex，也就是currentInterceptionIdex=0，开始获取第一个拦截器，也就是ExposeInvocationInterceptor
   Object interceptorOrInterceptionAdvice =
         this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
   if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
      InterceptorAndDynamicMethodMatcher dm =
            (InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
      if (dm.methodMatcher.matches(this.method, this.targetClass, this.arguments)) {
         return dm.interceptor.invoke(this);
      }
      else {
         return proceed();
      }
   }
   else {
       //上面的逻辑不走，直接调用ExposeInvocationInterceptor.invoke(MethodInvocation mi)
       //7.2.3.1.1走这里
      return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
   }
}
```

7.2.3.1.1调用ExposeInvocationInterceptor.invoke(MethodInvocation mi)

```java
@Override
public Object invoke(MethodInvocation mi) throws Throwable {
   MethodInvocation oldInvocation = invocation.get();
   invocation.set(mi);
   try {
       //继续调用第二次的ReflectiveMethodInvocation.proceed()
      return mi.proceed();
   }
   finally {
      invocation.set(oldInvocation);
   }
}
```

###### 7.2.3.2第二次调用ReflectiveMethodInvocation.proceed()

```java
@Override
public Object proceed() throws Throwable {
   // 如果没有拦截器执行目标方法，或者拦截器的索引和拦截去数组-1大小一样（指定到了最后一个拦截器）
   //第二次currentInterceptorIndex=0
   if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
      return invokeJoinpoint();
   }
//执行++currentInterceptionIdex，也就是currentInterceptionIdex=1，开始获取第二个拦截器，AspectJAfterThrowingAdvice
   Object interceptorOrInterceptionAdvice =
         this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
   if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
      InterceptorAndDynamicMethodMatcher dm =
            (InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
      if (dm.methodMatcher.matches(this.method, this.targetClass, this.arguments)) {
         return dm.interceptor.invoke(this);
      }
      else {
         return proceed();
      }
   }
   else {
       //7.2.3.2.1走这里
       //上面的逻辑不走，继续调用AspectJAfterThrowingAdvice.invoke(MethodInvocation mi)
      return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
   }
}
```

7.2.3.2.1调用AspectJAfterThrowingAdvice.invoke(MethodInvocation mi)

```java
@Override
public Object invoke(MethodInvocation mi) throws Throwable {
   try {
      return mi.proceed();
   }
   catch (Throwable ex) {
      if (shouldInvokeOnThrowing(ex)) {
         invokeAdviceMethod(getJoinPointMatch(), null, ex);
      }
      throw ex;
   }
}
```

###### 7.2.3.3第三次调用ReflectiveMethodInvocation.proceed()

7.2.3.3.1调用AfterReturningAdviceInterceptor.invoke(MethodInvocation mi)

```java
@Override
public Object invoke(MethodInvocation mi) throws Throwable {
   Object retVal = mi.proceed();
   this.advice.afterReturning(retVal, mi.getMethod(), mi.getArguments(), mi.getThis());
   return retVal;
}
```

###### 7.2.3.4第四次调用ReflectiveMethodInvocation.proceed()

7.2.3.4.1调用AspectJAfterAdvice.invoke(MethodInvocation mi)

```java
@Override
public Object invoke(MethodInvocation mi) throws Throwable {
   try {
      return mi.proceed();
   }
   finally {
      invokeAdviceMethod(getJoinPointMatch(), null, null);
   }
}
```

###### 7.2.3.5第五次调用ReflectiveMethodInvocation.proceed()

7.2.3.5.1调用MethodBeforeAdviceInterceptor.invoke(MethodInvocation mi)

```java
@Override
public Object invoke(MethodInvocation mi) throws Throwable {
    //先调用的前置通知方法
   this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis() );
   return mi.proceed();
}
```

###### 7.2.3.6第六次调用ReflectiveMethodInvocation.proceed()

```java
@Override
public Object proceed() throws Throwable {
   // 如果没有拦截器执行目标方法，或者拦截器的索引和拦截去数组-1大小一样（指定到了最后一个拦截器）
   // 最后currentInterceptionIdex=4，直接走这个方法
   if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
      
      return invokeJoinpoint();
   }

    //多余代码省略
}

protected Object invokeJoinpoint() throws Throwable {
		return AopUtils.invokeJoinpointUsingReflection(this.target, this.method, this.arguments);
	}


public static Object invokeJoinpointUsingReflection(Object target, Method method, Object[] args)
			throws Throwable {
    //多余代码省略
            //直接调用原方法
			return method.invoke(target, args);
	//多余代码省略
}

```



## 四、AOP原理总结

1.@EnableAspectJAutoProxy开启AOP功能

2.@EnableAspectJAutoProxy给容器中注册一个组件AnnotationAwareAspectJAutoProxyCreator后置处理器

3.容器创建流程：

​     3.1注册bean的后置处理器：registerBeanPostProcessors(beanFactory);

​	 3.2初始化剩下的单实例bean：finishBeanFactoryInitialization(beanFactory);		

​			3.2.1创建业务逻辑组件和切面组件

​			3.2.2AnnotationAwareAspectJAutoProxyCreator会拦截组件创建过程

​		        	3.2.2.1组件创建完成后，判断组件是否需要增强：是，就把切面通知方法包装成增强器，给业务逻辑组件创建一个代理对象（cglib）

4.执行目标方法：

​	4.1代理对象执行目标方法

​         4.1.1调用CglibAopProxy.intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) 

​		          4.1.1.1得到目标方法的拦截器链（增强器包装成MethodInterceptor）

​		          4.1.1.2利用拦截器的链式机制，依次进入每一个拦截器进行执行

​                  4.1.1.3效果：

​									正常执行：前置通知-目标方法-后置通知-返回通知

​									异常执行：前置通知-目标方法-后置通知-异常通知

![image-20220221221818669](C:\Users\MateBook\Desktop\IDE、数据库系统等工具\API文档\SSM\Spring注解驱动开发\image-20220221221818669.png)
