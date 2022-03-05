# Spring注解驱动开发-IOC篇

# Bean的注册：

## 一、组件注册---使用@Configuration、@Bean给容器中注册组件

### 1.导入Spring所需依赖

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.wxr</groupId>
    <artifactId>spring-annotation</artifactId>
    <version>1.0-SNAPSHOT</version>
    <dependencies>
        <!-- https://mvnrepository.com/artifact/org.springframework/spring-context -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>4.3.12.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-aspects</artifactId>
            <version>4.3.12.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-jdbc</artifactId>
            <version>4.3.12.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
            <scope>test</scope>
        </dependency>
        <!-- https://mvnrepository.com/artifact/javax.inject/javax.inject -->
        <dependency>
            <groupId>javax.inject</groupId>
            <artifactId>javax.inject</artifactId>
            <version>1</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/c3p0/c3p0 -->
        <dependency>
            <groupId>c3p0</groupId>
            <artifactId>c3p0</artifactId>
            <version>0.9.1.2</version>
        </dependency> 
        <!-- https://mvnrepository.com/artifact/mysql/mysql-connector-java -->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.44</version>
        </dependency>
    </dependencies>
</project>
```



### 2.创建配置类（MyConfig）

```java
package com.wxr.config;
import com.wxr.bean.Person;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**配置类==beans.xml
 * @author FatterShadystart
 * @create 2022-02-12 17:37
 */
//告诉spring这是一个配置类
@Configuration
public class MyConfig {
    //给容器中注册一个Bean，类型为返回值的类型，id是默认方法名作为的id
    @Bean
    public Person person(){
        return new Person("lisi",18);
    }

}

```



### 3.在测试类里面获取注册的Bean对象以及获取所有Bean对象的名字

```java
public class MainTest {
    public static void main(String[] args){
        //配置文件的方式
//        ClassPathXmlApplicationContext ioc = new ClassPathXmlApplicationContext("classpath:beans.xml");
//        Person person = ioc.getBean("person", Person.class);
//        System.out.println(person);
        //配置类
        AnnotationConfigApplicationContext ioc = new AnnotationConfigApplicationContext(MyConfig.class);
        Person person = ioc.getBean("person", Person.class);
        System.out.println(person);
        String[] beanNamesForType = ioc.getBeanNamesForType(Person.class);
        for (String name : beanNamesForType) {
            System.out.println(name);
        }
    }
}

```



### 4.输出结果

![image-20220212210836165](C:\Users\MateBook\Desktop\IDE、数据库系统等工具\API文档\SSM\Spring注解驱动开发\image-20220212210836165.png)







## 二、组件注册---@ComponentScan自动扫描组件和指定扫描规则

### 1.和xml配置一样context:component-scan,只要标注了@Controller、@Service、@Repository、@Component，都会被这个注解扫描到（在MyConfig配置类中写）

#### 1.1excludeFilters

```java
/**配置类==beans.xml
 * @author FatterShadystart
 * @create 2022-02-12 17:37
 */
//告诉spring这是一个配置类
@Configuration
//@ComponentScan,value:指定要扫描的包
//excludeFilters=Filter[]：指定扫描的时候按照什么规则排除组件
//includeFilters=Filter[]：指定扫描的时候只需要包含符合规则的组件
@ComponentScan(value = "com.wxr",excludeFilters = {
        //不扫描Controller注解
        @ComponentScan.Filter(type = FilterType.ANNOTATION,classes = {Controller.class})
})
public class MyConfig {
    //给容器中注册一个Bean，类型为返回值的类型，id是默认方法名作为的id
    @Bean
    public Person person(){
        return new Person("lisi",18);
    }

}

```

#### 1.2includeFilters

```java
/**配置类==beans.xml
 * @author FatterShadystart
 * @create 2022-02-12 17:37
 */
//告诉spring这是一个配置类
@Configuration
//@ComponentScan,value:指定要扫描的包
//excludeFilters=Filter[]：指定扫描的时候按照什么规则排除组件
//includeFilters=Filter[]：指定扫描的时候只需要包含符合规则的组件
@ComponentScan(value = "com.wxr",includeFilters = {
        //不扫描Controller和Service注解
        @ComponentScan.Filter(type = FilterType.ANNOTATION,classes = {Controller.class})},
        //使用includeFilters的时候要禁用掉默认的过滤规则
        useDefaultFilters = false
)
public class MyConfig {
    //给容器中注册一个Bean，类型为返回值的类型，id是默认方法名作为的id
    @Bean
    public Person person(){
        return new Person("lisi",18);
    }
}
```

#### 1.3自定义TypeFilter指定过滤规则

##### 1.3.1配置类的写法

```java
//告诉spring这是一个配置类
@Configuration
//@ComponentScan,value:指定要扫描的包
//excludeFilters=Filter[]：指定扫描的时候按照什么规则排除组件
//includeFilters=Filter[]：指定扫描的时候只需要包含符合规则的组件
@ComponentScan(value = "com.wxr",includeFilters = {
        @ComponentScan.Filter(type = FilterType.CUSTOM,classes = {MyTypeFilter.class})
        },useDefaultFilters = false
)
public class MyConfig {
    //给容器中注册一个Bean，类型为返回值的类型，id是默认方法名作为的id
    @Bean
    public Person person(){
        return new Person("lisi",18);
    }

}

```

##### 1.3.2要实现自定义指定过滤规则，需要新建一个类然后实现TypeFilter，在match方法里面自定义过滤规则

```java
public class MyTypeFilter implements TypeFilter {
    /**
     * @param metadataReader        读取到当前正在扫描的类的信息
     * @param metadataReaderFactory 可以获取到其他任何类的信息
     * @return
     * @throws IOException
     */
    @Override
    public boolean match(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory) throws IOException {
        //获取当前正在扫描的类的类信息
        ClassMetadata classMetadata = metadataReader.getClassMetadata();
        //获取当前扫描类的类名
        String className = classMetadata.getClassName();
        //如果类名包含"er"那么就扫描成功
        if (className.contains("er")) {
            return true;
        }
        return false;
    }
}
```

### 2.在测试类里面获取所有的Bean对象的名字

```java
 	@Test
    public void test01(){
        AnnotationConfigApplicationContext ioc = new AnnotationConfigApplicationContext(MyConfig.class);
        String[] definitionNames = ioc.getBeanDefinitionNames();
        for (String name : definitionNames) {
            System.out.println(name);
        }
    }
```



### 3.输出结果

#### 1.1的结果

![1644673962(1)](C:\Users\MateBook\Desktop\IDE、数据库系统等工具\API文档\SSM\Spring注解驱动开发\1644673962(1).jpg)

#### 1.2的结果

![1644673859(1)](C:\Users\MateBook\Desktop\IDE、数据库系统等工具\API文档\SSM\Spring注解驱动开发\1644673859(1).jpg)

#### 1.3的结果

![1644676638(1)](C:\Users\MateBook\Desktop\IDE、数据库系统等工具\API文档\SSM\Spring注解驱动开发\1644676638(1).jpg)









## 三、组件注册---@Scope设置组件作用域和@Lazy实现Bean懒加载

### 1.创建配置类MyConfig2

##### 	1.1单实例（默认值）

```java
@Configuration
public class MyConfig2 {
    @Bean("person")
    /**
     * prototype: 多实例的：ioc容器启动并不会去调用方法创建对象放到ioc容器中，而是每次获取的时候才会调用方法创建对象。
     * singleton：单实例的（默认值）：ioc容器启动的时候会调用方法创建对象放到ioc容器中，以后每次获取都直接从容器中拿。
     */
    @Scope("singleton")
    public Person person(){
        System.out.println("给容器中添加....person对象");
        return new Person("zhangsan",25);
    }
}
```

##### 	1.2多实例

```java
@Configuration
public class MyConfig2 {
    @Bean("person")
    /**
     * prototype: 多实例的：ioc容器启动并不会去调用方法创建对象放到ioc容器中，而是每次获取的时候才会调用方法创建对象。
     * singleton：单实例的（默认值）：ioc容器启动的时候会调用方法创建对象放到ioc容器中，以后每次获取都直接从容器中拿。
     */
    @Scope("prototype")
    public Person person(){
        System.out.println("给容器中添加....person对象");
        return new Person("zhangsan",25);
    }
}
```

##### 	1.3单实例配合懒加载

```java
@Configuration
public class MyConfig2 {

    @Bean("person")
    /**
     * prototype: 多实例的：ioc容器启动并不会去调用方法创建对象放到ioc容器中，而是每次获取的时候才会调用方法创建对象。
     * singleton：单实例的（默认值）：ioc容器启动的时候会调用方法创建对象放到ioc容器中，以后每次获取都直接从容器中拿。
     * 懒加载：
     *      单实例bean：默认在容器启动的时候创建对象
     *      懒加载：容器启动不创建对象。第一次使用（获取）Bean的时候创建对象，并且初始化。
     */
    @Lazy
    @Scope("singleton")
    public Person person(){
        System.out.println("给容器中添加....person对象");
        return new Person("zhangsan",25);
    }
}
```



### 2.在测试类里面测试Bean对象的获取

```java
@Test
public void test02(){
    AnnotationConfigApplicationContext ioc = new AnnotationConfigApplicationContext(MyConfig2.class);
    System.out.println("ioc容器创建完成。。。。");
    Person person = ioc.getBean("person", Person.class);
    Person person1 = ioc.getBean("person", Person.class);
    System.out.println(person==person1);
 }
```



### 3.输出结果

##### 		1.1的结果（单实例默认在ioc容器启动的时候就创建对象放到ioc容器中，每次获取的时候直接从容器中拿）

![1644683100(1)](C:\Users\MateBook\Desktop\IDE、数据库系统等工具\API文档\SSM\Spring注解驱动开发\1644683100(1).jpg)



##### 		1.2的结果（ioc容器启动并不会去调用方法创建对象放到ioc容器中，而是每次获取的时候才会调用方法创建对象。）

![1644683223(1)](C:\Users\MateBook\Desktop\IDE、数据库系统等工具\API文档\SSM\Spring注解驱动开发\1644683223(1).jpg)



##### 		1.3的结果（添加@Lazy懒加载注解后，ioc容器启动时不创建对象。第一次使用（获取）Bean的时候创建对象，并且初始化。）

![1644683313(1)](C:\Users\MateBook\Desktop\IDE、数据库系统等工具\API文档\SSM\Spring注解驱动开发\1644683313(1).jpg)





## 四、组件注册---@Conditional 按条件注册Bean

### 1.创建配置类MyConfig2

```java
@Configuration
//@Conditional加在类上的话，要满足当前条件，这个类中配置的所有bean注册才会生效
//@Conditional({WindowsCondition.class})
public class MyConfig2 {

    /**
     *@Conditional
     * 如果是windows系统，就给容器注册Windows
     * 如果是linux系统，就给容器注册Linux
     *
     * @return
     */
    @Conditional({LinuxCondition.class})
    @Bean("Linux")
    public Person person02(){
        return new Person("Linux liuns",21);
    }

    @Conditional({WindowsCondition.class})
    @Bean("Windows")
    public Person person01(){
        return new Person("Windows Bill",20);
    }
}
```

### 2.创建条件类，实现Condition接口

```java
/**判断是否是Linux系统
 * @author FatterShadystart
 * @create 2022-02-13 14:57
 */
public class LinuxCondition implements Condition {
    /**
    
     * @param context 判断条件能使用的上下文环境
     * @param metadata 当前标注了condition注解的注释信息
     * @return
     */
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        //获取ioc的BeanFactory
        ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
        //获取类加载器
        ClassLoader classLoader = context.getClassLoader();
        //获取当前环境信息
        Environment environment = context.getEnvironment();
        //获取bean定义的注册类
        BeanDefinitionRegistry registry = context.getRegistry();

        String property = environment.getProperty("os.name");
        if(property.contains("linux")){
            return true;
        }
        return false;
    }
}
```



```java
/**判断是否是Windows系统
 * @author FatterShadystart
 * @create 2022-02-13 14:57
 */
public class WindowsCondition implements Condition{
    /**
     * @param context 判断条件能使用的上下文环境
     * @param metadata 当前标注了condition注解的注释信息
     * @return
     */
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        //获取ioc的BeanFactory
        ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
        //获取类加载器
        ClassLoader classLoader = context.getClassLoader();
        //获取当前环境信息
        Environment environment = context.getEnvironment();
        //获取bean定义的注册类
        BeanDefinitionRegistry registry = context.getRegistry();

        String property = environment.getProperty("os.name");
        if(property.contains("Windows")){
            return true;
        }
        return false;
    }
}
```



### 3.编写测试类，输出注册的Bean对象名和Bean对象

```java
@Test
public void test03(){
    AnnotationConfigApplicationContext ioc = new AnnotationConfigApplicationContext(MyConfig2.class);
    String[] beanDefinitionNames = ioc.getBeanNamesForType(Person.class);
    for (String name : beanDefinitionNames) {
        System.out.println(name);
    }
    System.out.println("*********************************");
    Map<String, Person> persons = ioc.getBeansOfType(Person.class);
    System.out.println(persons);

}
```



### 4.输出结果

![1644736467(1)](C:\Users\MateBook\Desktop\IDE、数据库系统等工具\API文档\SSM\Spring注解驱动开发\1644736467(1).jpg)







## 五、组件注册方法总结（@Import和FactoryBean的使用）

### 1.给容器中注册组件的方法：

1.1包扫描（ComponentScan）+组件标注注解（@Controller/@Service/@Repository/@Component）



1.2@Bean：导入第三方包未写组件标注注解的组件



1.3@Import：根据无参构造快速给容器中导入一个组件

​			1.3.1@Import（要导入到容器中的组件），容器就会自动注册这个组件，id是默认的全类名

​			1.3.2 ImportSelector：返回需要导入的组件的全类名数组

​			1.3.3 ImportBeanDefinitionRegistrar：手动注册bean到容器中



1.4使用Spring提供的FactoryBean（工厂Bean）

​			默认获取到的是工厂bean调用getObject创建的对象。如果要获取工厂本身就在getBean()的id前面加一个“&”标识



### 2.创建配置类MyConfig3

####  1.3.1的实现（只需要创建MyConfig3配置类）

```java
@Configuration
//@Import导入组件，id默认是组件的全类名
@Import({Color.class, Red.class})
public class MyConfig3 {
}
```

#### 1.3.2的实现（创建MyConfig3配置类后，再创建MyImportSelector实现ImportSelector接口）

```java
@Configuration
//@Import导入组件，MyImportSelector是自定义返回需要导入的组件的全类名数组
@Import({MyImportSelector.class})
public class MyConfig3 {
}
```

```java
public class MyImportSelector implements ImportSelector {
    /**
     *
     * @param importingClassMetadata 当前标注Import注解的类的所有注解信息
     * @return 导入到容器中组件的全类名
     */
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        return new String[]{"com.wxr.bean.Blue","com.wxr.bean.Yellow"};
    }
}

```

#### 1.3.3的实现（创建MyConfig3配置类后，再创建MyImportBeanDefinitionRegistrar实现ImportBeanDefinitionRegistrar接口）

```java
@Configuration
//@Import导入组件，MyImportBeanDefinitionRegistrar自定义手动注册bean到容器中
@Import({Blue.class,Red.class,MyImportBeanDefinitionRegistrar.class})
public class MyConfig3 {
}
```

```java
public class MyImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {


    /**
     *
     * @param importingClassMetadata 当前类的注解信息
     * @param registry BeanDefinition注册类：把所有需要添加到容器中的Bean（调用BeanDefinitionRegistry.registerBeanDefinition手动注册进来）
     */
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        //全类名
        boolean definition1 = registry.containsBeanDefinition("com.wxr.bean.Red");
        boolean definition2 = registry.containsBeanDefinition("com.wxr.bean.Blue");
        if(definition1&&definition2){
            //指定Bean定义信息（Bean的类型，Bean的作用域...）
            RootBeanDefinition rootBeanDefinition = new RootBeanDefinition(RainBow.class);
            //注册一个Bean，指定bean名
            registry.registerBeanDefinition("rainBow", rootBeanDefinition);
        }
    }
}
```

#### 1.4的实现（创建MyConfig3配置类后，再创建ColorFactoryBean实现FactoryBean<T>接口）

```java
@Configuration
public class MyConfig3 {
    @Bean
    public ColorFactoryBean colorFactoryBean(){
        return new ColorFactoryBean();
    }
}
```

```java
/**
 * 创建一个Spring定义的工厂Bean
 */
public class ColorFactoryBean implements FactoryBean<Color>{
    /**
     * @return 返回一个Color对象这个对象会添加到容器中
     * @throws Exception
     */
    @Override
    public Color getObject() throws Exception {
        return new Color();
    }
    
    /**
     * @return 返回对象的类型
     */
    @Override
    public Class<?> getObjectType() {
        return Color.class;
    }

    /**
     * @return  true:单实例  false：多实例
     */
    @Override
    public boolean isSingleton() {
        return true;
    }
}

```



### 3.在测试类里面获取所有的Bean对象的名字

#### 1.3的测试类

```java
    @Test
    public void test04(){
        AnnotationConfigApplicationContext ioc = new AnnotationConfigApplicationContext(MyConfig3.class);
        String[] beanDefinitionNames = ioc.getBeanDefinitionNames();
        for (String name : beanDefinitionNames) {
            System.out.println(name);
        }
    }

```

#### 1.4的测试类

```java
 @Test
    public void test05(){
        AnnotationConfigApplicationContext ioc = new AnnotationConfigApplicationContext(MyConfig3.class);
        String[] beanDefinitionNames = ioc.getBeanDefinitionNames();
        for (String name : beanDefinitionNames) {
            System.out.println(name);
        }
        //工厂Bean获取的是调用getObject创建的对象
        Object colorFactoryBean = ioc.getBean("colorFactoryBean");
        System.out.println("bean的类型："+colorFactoryBean.getClass());
        //如果要获取工厂Bean就在前面加上&前缀
        Object colorFactoryBean1 = ioc.getBean("&colorFactoryBean");
        System.out.println("beanFactory的类型："+colorFactoryBean1.getClass());
    }


```





### 4.输出结果

####   1.3.1的结果

![ZC@H2{`LV%Z9O`_H%P8FC6N](C:\Users\MateBook\Desktop\IDE、数据库系统等工具\API文档\SSM\Spring注解驱动开发\ZC@H2{`LV%Z9O`_H%P8FC6N.png)

####   1.3.2的结果

![1644742586(1)](C:\Users\MateBook\Desktop\IDE、数据库系统等工具\API文档\SSM\Spring注解驱动开发\1644742586(1).jpg)

####   1.3.3的结果

![image-20220219211214155](C:\Users\MateBook\Desktop\IDE、数据库系统等工具\API文档\SSM\Spring注解驱动开发\image-20220219211214155.png)

####  1.4的结果

![1644744586(1)](C:\Users\MateBook\Desktop\IDE、数据库系统等工具\API文档\SSM\Spring注解驱动开发\1644744586(1).jpg)









# 二、Bean的生命周期

## 一、指定初始化和销毁方法---@Bean

### 1.bean的生命周期

 bean创建--->初始化---->销毁的过程

bean的生命周期是通过容器进行管理：我们可以自定义初始化和销毁方法，当容器在bean进行到当前生命周期的时候来调用我们自定义的初始化和销毁方法



### 2.指定初始化和销毁方法---基于XML配置方式

```xml
    <bean id="person" class="com.wxr.bean.Person" scope="prototype" init-method="" destroy-method="">
```



### 3.指定初始化和销毁方法---基于注解配置方式

#### 3.1通过@Bean的initMethod、destroyMethod进行调用实现

##### 3.1.1创建配置类MainConfigOfLifeCycle

```java
@Configuration
public class MainConfigOfLifeCycle {
    //指定初始化和销毁方法
    @Bean(initMethod = "init", destroyMethod = "destroy")
    public Car car() {
        return new Car();
    }
}

```

##### 3.1.2创建Car类，包含构造器，init，destroy方法

```java
public class Car {
    public Car() {
        System.out.println("car的构造器.........");
    }
    public void init() {
        System.out.println("car的初始化方法........");
    }
    public void destroy() {
        System.out.println("car的销毁方法.........");
    }
}
```

##### 3.1.3测试代码

```java
	/**
     * @Scope("singleton")
     */
	@Test
    public void test01(){
        AnnotationConfigApplicationContext ioc = new AnnotationConfigApplicationContext(MainConfigOfLifeCycle.class);
        System.out.println("ioc容器创建完成");
        ioc.close();
    }
```

```java
	/**
     * @Scope("prototype")
     */
	@Test
   public void test02(){
        AnnotationConfigApplicationContext ioc = new AnnotationConfigApplicationContext(MainConfigOfLifeCycle.class);
        System.out.println("ioc容器创建完成");
        Object car = ioc.getBean("car");
        ioc.close();
 }
```

##### 3.1.4输出结果

test01:                                            										test02:

![1644757060(1)](C:\Users\MateBook\Desktop\IDE、数据库系统等工具\API文档\SSM\Spring注解驱动开发\1644757060(1).jpg)![image-20220213210955789](C:\Users\MateBook\Desktop\IDE、数据库系统等工具\API文档\SSM\Spring注解驱动开发\image-20220213210955789.png)

**说明：**

1.构造器（对象的创建）：

​											单实例：在容器启动的时候创建对象

​											多实例：在每次获取的时候创建对象

2.初始化：在对象创建完成（实例化），并赋值以后（初始化），调用初始化方法（initMethod）

3.销毁：

​			单实例：容器关闭的时候

​			多实例：容器不会管理这个bean，容器不会调用销毁方法





#### 3.2通过Bean实现InitializingBean（定义初始化逻辑）,DisposableBean（定义销毁逻辑） 接口实现

##### 3.2.1创建配置类MainConfigOfLifeCycle

```java
@Configuration
@ComponentScan("com.wxr")
public class MainConfigOfLifeCycle {
}
```

##### 3.2.2创建Computer类，包含构造器，实现InitializingBean,DisposableBean接口

```java
@Component
public class Computer implements InitializingBean,DisposableBean {
    public Computer() {
        System.out.println("Computer的构造器.........");
    }

    /**
     * 和自己写的init方法一样
     *
     * @throws Exception
     */
    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("Computer的afterPropertiesSet（初始化方法）.........");
    }
    /**
     * 和自己写的destroy方法一样
     *
     * @throws Exception
     */
    @Override
    public void destroy() throws Exception {
        System.out.println("Computer的destroy（销毁方法）.........");
    }
}
```

3.2.3测试代码

```java
@Test
public void test03(){
    AnnotationConfigApplicationContext ioc = new AnnotationConfigApplicationContext(MainConfigOfLifeCycle.class);
    System.out.println("ioc容器创建完成");
    ioc.close();
}
```

3.2.4输出结果

![image-20220213212332471](C:\Users\MateBook\Desktop\IDE、数据库系统等工具\API文档\SSM\Spring注解驱动开发\image-20220213212332471.png)



#### 3.3通过JSR250的@PostConstruct（在bean创建完成并且复制完成的时候执行），@PreDestroy（在容器销毁bean之前通知用户进行清理工作）实现

##### 3.3.1创建配置类MainConfigOfLifeCycle

```java
@Configuration
@ComponentScan("com.wxr")
public class MainConfigOfLifeCycle {
}
```

##### 3.3.2创建Computer类，包含构造器

```java
@Component
public class Student {
    public Student() {
        System.out.println("Student的构造器.........");
    }

    //对象创建并赋值后调用
    @PostConstruct
    public void init() {
        System.out.println("Student的@PostConstruct（init方法）.........");
    }

    //容器移除对象之前
    @PreDestroy
    public void destroy() {
        System.out.println("Student的@PreDestroy（destroy方法）..........");
    }
}

```

##### 3.3.3测试代码

```java
@Test
public void test03(){
    AnnotationConfigApplicationContext ioc = new AnnotationConfigApplicationContext(MainConfigOfLifeCycle.class);
    System.out.println("ioc容器创建完成");
    ioc.close();
}
```

##### 3.3.4输出结果

![image-20220213214336799](C:\Users\MateBook\Desktop\IDE、数据库系统等工具\API文档\SSM\Spring注解驱动开发\image-20220213214336799.png)





#### 3.4通过自定义 BeanPostProcessor（bean的后置处理器）接口实现

##### 3.4.1创建一个Car对象

```java
@Component
public class Car {

    public Car() {
        System.out.println("car的构造器.........");
    }
    public void init() {
        System.out.println("car的初始化方法........");
    }
    public void destroy() {
        System.out.println("car的销毁方法.........");
    }
}
```

##### 3.4.2创建配置类MainConfigOfLifeCycle

```java
@Configuration
//导入自定义后置处理器
@Import(MyBeanPostProcessor.class)
public class MainConfigOfLifeCycle {
		//创建一个Car对象实现init，和destroy方法
        @Bean(initMethod = "init",destroyMethod = "destroy")
        public Car car(){
            return new Car();
        }

}

```

##### 3.4.3创建MyBeanPostProcessor类，实现BeanPostProcessor接口

```java
@Component
public class MyBeanPostProcessor implements BeanPostProcessor {
  
    /**
     * @param bean 刚创建的bean实例对象
     * @param beanName 刚创建的bean实例对象名字
     * @return 返回源对象或者是包装好的对象
     * @throws BeansException
     */
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("postProcessBeforeInitialization..."+beanName+"-->"+bean);
        return bean;
    }
    /**
     * @param bean 刚创建的bean实例对象
     * @param beanName 刚创建的bean实例对象名字
     * @return 返回源对象或者是包装好的对象
     * @throws BeansException
     */
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("postProcessAfterInitialization..."+beanName+"-->"+bean);
        return bean;
    }
}


```

##### 3.4.4测试代码

```java
@Test
   public void test03(){
        AnnotationConfigApplicationContext ioc = new AnnotationConfigApplicationContext(MainConfigOfLifeCycle.class);
        System.out.println("ioc容器创建完成");
        ioc.close();
 }
```

##### 3.4.5输出结果

![image-20220219002326026](C:\Users\MateBook\Desktop\IDE、数据库系统等工具\API文档\SSM\Spring注解驱动开发\image-20220219002326026.png)

**说明：后置处理器执行位置**

构造器（对象的创建）

**BeanPostProcessor.postProcessBeforeInitialization ：在bean初始化之前工作**

初始化：在对象创建完成（实例化），并赋值以后（初始化），调用初始化方法（initMethod）

**BeanPostProcessor.postProcessAfterInitialization：在bean初始化之后工作**

销毁





## 二、属性赋值---@Value的使用

### 1.创建配置类MyConfigOfPropertyValues

```java
//使用@PropertySource读取外部配置文件中的k/v保存到运行的环境变量中
@PropertySource(value = {"classpath:Person.properties"})
@Configuration
public class MyConfigOfPropertyValues {
    @Bean
    public Person person(){
        return new Person();
    }
}

```

### 2.在要赋值的bean的属性中加入@Value

```java
public class Person {
    /**
     * @Value 可以写的值：
     *                   1.基本数值
     *                   2.SpEL:#{}
     *                   3.取出配置文件中的值:${}
     *
     */
    @Value("张三")
    private String name;
    @Value("#{20-2}")
    private Integer age;
    @Value("${person.nickname}")
    private String nickName;
    public Person() {
    }
    public Person(String name, Integer age, String nickName) {
        this.name = name;
        this.age = age;
        this.nickName = nickName;
    }
    public Person(String name, Integer age) {
        this.name = name;
        this.age = age;

    } 
    @Override
    public String toString() {
        return "Person{" +
                "name='" + name + '\'' +
                ", age=" + age +
                ", nickName='" + nickName + '\'' +
                '}';
    }
    public String getName() {
        return name;
    }
    public void setName(String name) {
        this.name = name;
    }
    public Integer getAge() {
        return age;
    }
    public void setAge(Integer age) {
        this.age = age;
    }
}

```

#### 在自定义配置文件Person.properties中写值

```properties
person.nickname=\u5C0F\u5f20\u4E09
```

### 3.测试代码

```java
  @Test
    public void test04(){
        AnnotationConfigApplicationContext ioc = new AnnotationConfigApplicationContext(MyConfigOfPropertyValues.class);
        Person person = ioc.getBean("person", Person.class);
        System.out.println(person);
    }

```

### 4.输出结果

![image-20220214171809874](C:\Users\MateBook\Desktop\IDE、数据库系统等工具\API文档\SSM\Spring注解驱动开发\image-20220214171809874.png)













## 三、自动装配---@Autowired、@Qualifier的使用

### 1.自动装配概念

​	 Spring利用依赖注入(DI)，完成对IOC容器中各个组件的依赖关系赋值;

​	 **AutowiredAnnotationBeanPostProcessor：解析完成自动装配功能**

### 2.使用自动装配

#### 2.1@Autowired、@Qualifier的使用

##### 2.1.1创建配置类MyConfigAutowired

###### 2.1.1.1创建配置类MyConfigAutowired（没有再次获得相同类型的组件）

```java
@Configuration
@ComponentScan(value = {"com.wxr.service","com.wxr.dao","com.wxr.controller"})
public class MyConfigAutowired {
 
}
```

###### 2.1.1.2创建配置类MyConfigAutowired（获得相同类型的组件）

```java
@Configuration
@ComponentScan(value = {"com.wxr.service","com.wxr.dao","com.wxr.controller"})
public class MyConfigAutowired {
    @Bean("bookDao2")
    public BookDao bookDao() {
        BookDao bookDao2 = new BookDao();
        bookDao2.setLabel("2");
        return  bookDao2;
    }
}
```

##### 2.1.2创建BookController、BookService、BookDao类

###### 2.1.2.1创建BookController、BookService、BookDao类（没有再次获得相同类型的组件）

```java
@Controller
public class  BookController {
    @Autowired
    private BookService bookService;
}


@Service
public class BookService {
    @Autowired
    private BookDao bookDao;
    
    @Override
    public String toString() {
        return "BookService{" +
                "bookDao=" + bookDao +
                '}';
    }
}

@Repository
public class BookDao {
    private String label="1";

    public String getLabel() {
        return label;
    }

    public void setLabel(String label) {
        this.label = label;
    }

    @Override
    public String toString() {
        return "BookDao{" +
                "label='" + label + '\'' +
                '}';
    }
}

```

###### 2.1.2.1创建BookController、BookService、BookDao类（获得相同类型的组件）

```java
@Controller
public class  BookController {
    @Autowired
    private BookService bookService;
}

@Service
public class BookService {
    @Autowired 
    @Qualifier("bookDao2")
    private BookDao bookDao;
    @Override
    public String toString() {
        return "BookService{" +
                "bookDao=" + bookDao +
                '}';
    }
}

@Repository
public class BookDao {
    private String label="1";

    public String getLabel() {
        return label;
    }

    public void setLabel(String label) {
        this.label = label;
    }

    @Override
    public String toString() {
        return "BookDao{" +
                "label='" + label + '\'' +
                '}';
    }
}

```

##### 2.1.3测试代码

```java
      /**
     * 自动注入:
     			1.默认优先按照 类型 去容器中找对应的组件：相当于ioc.getBean(BookDao.class)；找到就赋值
     			2.如果找到 多个相同类型 的组件，再将属性的名称作为组件的id去容器中查找：相当于ioc.getBean(“bookDao”)
     			  一般来说，如果找到多个相同类型的组件后，我们都要搭配着@Qualifier进行使用
     			3.@Qualifier指定需要装配的组件id，而不是使用属性名
                        BookService{
                                    @Autowired
     								BookDao    bookDao
                                    
       @Autowired的使用位置：构造器（如果组件只有一个有参构造器，这个有参构造器的@Autowired可以省略）、参数、方法、属性
       
     */
@Test
    public void test01(){
        AnnotationConfigApplicationContext ioc = new AnnotationConfigApplicationContext(MyConfigAutowired.class);
        BookService beanBookService = ioc.getBean(BookService.class);
        System.out.print("自动注入的dao："+beanBookService);
        System.out.println();
        BookDao beanBookDao = ioc.getBean("bookDao",BookDao.class);
        System.out.println("主动获取的dao："+beanBookDao);
        ioc.close();
    }
```

##### 2.1.4输出结果

没有多个类型相同的组件的时候：

![image-20220215212111261](C:\Users\MateBook\Desktop\IDE、数据库系统等工具\API文档\SSM\Spring注解驱动开发\image-20220215212111261.png)

获得了多个相同类型的组件的时候并且使用了@Qualifier后：

![image-20220215212033306](C:\Users\MateBook\Desktop\IDE、数据库系统等工具\API文档\SSM\Spring注解驱动开发\image-20220215212033306.png)



## 四.自动装配---Aware注入Spring底层组件

### 1.创建配置类MyConfigAutowired

```java
/**
 * 自定义组件想要使用Spring容器底层的一些组件（ApplicationContext,BeanFactory,...）
 * 需要自定义组件实现xxxAware：在创建对象的时候，会调用接口规定的方法注入相关组件把Spring底层的一些组件注入到自定义的组件当中
 * xxxAware：功能是被xxxProcessor实现的：
 *               ApplicationContextAware==>ApplicationContextAwareProcessor
 *
 * @author FatterShadystart
 * @create 2022-02-14 17:24
 */
@Configuration
@ComponentScan(value = {"com.wxr.bean"})
public class MyConfigAutowired {

}
```

### 2.创建自定义组件实现XXXAware的类Red

```java
@Component
public class Red implements ApplicationContextAware,BeanNameAware,EmbeddedValueResolverAware {

    private ApplicationContext applicationContext;

    @Override
    public void setBeanName(String name) {
        System.out.println("当前bean的名字："+name);
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        System.out.println("传入的ioc容器："+applicationContext);
        this.applicationContext=applicationContext;
    }

    @Override
    public void setEmbeddedValueResolver(StringValueResolver resolver) {
        String s = resolver.resolveStringValue("你好${os.name}");
        System.out.println("解析的字符串是："+s);
    }
}

```

### 3.测试代码

```java
   @Test
    public void test01(){
        AnnotationConfigApplicationContext ioc = new AnnotationConfigApplicationContext(MyConfigAutowired.class);
        ioc.close();
    }
```

### 4.输出结果

![image-20220215231653711](C:\Users\MateBook\Desktop\IDE、数据库系统等工具\API文档\SSM\Spring注解驱动开发\image-20220215231653711.png)







## 五、自动装配---@Profile根据环境注册bean

### 1.创建db.properties文件

```properties
db.user=root
db.password=wxr.1005
db.driverClass=com.mysql.jdbc.Driver
```

### 2.创建配置类MyConfigOfProfile

```java
/**
 * 一、Profile:
 *        Spring为我们提供的可以根据当前环境，动态地激活和切换一系列组件的功能
 *环境：
 *     开发环境、测试环境、生产环境
 *数据源:
 * (/A库)(/B库)(/C库)
 *
 * 二、@Profile:  指定组件在哪个环境的情况下才能被注册到容器中，如果不指定，任何环境下都可以注册这个组件
 *              1.加了环境标识的bean，只有在这个环境被激活的时候才能注册到容器中
 *              2.写在配置类上，只有是指定环境的时候，整个配置类的环境才会生效
 *              3.没有标注环境标识的bean，在任何情况下都会加载
 *
 * @author FatterShadystart
 * @create 2022-02-15 23:23
 */
//使用@PropertySource读取外部配置文件中的k/v保存到运行的环境变量中
@PropertySource(value = {"classpath:dbconfig.properties"})
@Configuration
public class MyConfigOfProfile implements EmbeddedValueResolverAware{

    @Value("${db.user}")
    private String user;

    private StringValueResolver valueResolver;

    private String driverClass;

    @Profile("test")
    @Bean("testDatasource")
    public DataSource datasourceTest(@Value("${db.password}")String pwd) throws Exception {
        ComboPooledDataSource cpds = new ComboPooledDataSource();
        cpds.setUser(user);
        cpds.setPassword(pwd);
        cpds.setJdbcUrl("jdbc:mysql://localhost:3306/test");
        cpds.setDriverClass(driverClass);
        return cpds;
    }

    @Profile("develop")
    @Bean("developDatasource")
    public DataSource datasourceDevelop(@Value("${db.password}")String pwd) throws Exception {
        ComboPooledDataSource cpds = new ComboPooledDataSource();
        cpds.setUser(user);
        cpds.setPassword(pwd);
        cpds.setJdbcUrl("jdbc:mysql://localhost:3306/ssm_crud");
        cpds.setDriverClass(driverClass);
        return cpds;
    }
    @Profile("Produce")
    @Bean("ProduceDatasource")
    public DataSource datasourceProduce(@Value("${db.password}")String pwd) throws Exception {
        ComboPooledDataSource cpds = new ComboPooledDataSource();
        cpds.setUser(user);
        cpds.setPassword(pwd);
        cpds.setJdbcUrl("jdbc:mysql://localhost:3306/book");
        cpds.setDriverClass(driverClass);
        return cpds;
    }

    /**
     * 任何环境下 未标识@Profile
     * @return
     */
    @Bean
    public Color color(){
        return  new Color();
    }

    @Override
    public void setEmbeddedValueResolver(StringValueResolver resolver) {
        this.valueResolver=resolver;
        driverClass=valueResolver.resolveStringValue("${db.driverClass}");
    }
}

```



### 3.测试代码(输出所有Bean对象的名字)

```java
 @Test
    public void test01(){
        //创建IoC容器
        AnnotationConfigApplicationContext ioc = new AnnotationConfigApplicationContext();
        //设置需要激活的环境
        ioc.getEnvironment().setActiveProfiles("test","develop");
        //注册主配置类
        ioc.register(MyConfigOfProfile.class);
        //启动刷新容器
        ioc.refresh();
        String[] DataSourceForType = ioc.getBeanNamesForType(DataSource.class);
        for (String name : DataSourceForType) {
            System.out.println(name);
        }
        String[] ColorForType = ioc.getBeanNamesForType(Color.class);
        for (String name : ColorForType) {
            System.out.println(name);
        }
        ioc.close();
    }
```

### 4.输出结果

![image-20220216002009900](C:\Users\MateBook\Desktop\IDE、数据库系统等工具\API文档\SSM\Spring注解驱动开发\image-20220216002009900.png)

 
