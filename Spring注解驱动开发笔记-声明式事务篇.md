# Spring注解驱动开发笔记-声明式事务篇

## 一、什么是声明式事务

简言之就是Spring通过注解，简化数据库事务的开发，是建立在AOP之上的。其本质是对方法前后进行拦截，然后在目标方法开始之前创建或者加入一个事务，在执行完目标方法之后根据执行情况提交或者回滚事务。声明式事务最大的优点就是不需要通过编程的方式管理事务，这样就不需要在业务逻辑代码中掺杂事务管理的代码，只需在配置文件中做相关的事务规则声明(或通过基于@Transactional注解的方式)，便可以将事务规则应用到业务逻辑中。



## 二、声明式事务使用案例（除了1，其余都在com.wxr.tx包下）

### 1.导入相关依赖：数据源、数据库驱动、Mybatis

```xml
	   <!--c3p0数据源 -->
        <dependency>
            <groupId>c3p0</groupId>
            <artifactId>c3p0</artifactId>
            <version>0.9.1.2</version>
        </dependency>
        <!-- mysql数据库驱动 -->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.44</version>
        </dependency>
        <!-- mybatis核心包 -->
        <dependency>
            <groupId>org.mybatis</groupId>
            <artifactId>mybatis</artifactId>
            <version>3.2.6</version>
        </dependency>
        <!-- mybatis/spring包 -->
        <dependency>
            <groupId>org.mybatis</groupId>
            <artifactId>mybatis-spring</artifactId>
            <version>1.2.2</version>
        </dependency>
```

### 2.创建数据库表

```mysql
CREATE TABLE `tbl_user` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `username` varchar(50) DEFAULT NULL,
  `age` int(20) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=17 DEFAULT CHARSET=utf8
```

### 3.创建JavaBean---User

```java
public class User {
    private int id;
    private String username;
    private int age;

    @Override
    public String toString() {
        return "User{" +
                "id=" + id +
                ", username='" + username + '\'' +
                ", age=" + age +
                '}';
    }

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }
}
```

### 	4.创建sql映射文件，UserMapper.xml

```xml-dtd
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.wxr.tx.UserMapper">
    <insert id="save" parameterType="com.wxr.tx.User">
       insert into `tbl_user`(username,age)values(#{username},#{age})
    </insert>
</mapper>
```

### 5.创建UserMapper类

```java
@Repository
public interface UserMapper {
    void save(@Param("username") String username,@Param("age")int age);
}
```

### 6.创建UserService类

```java
@Service
public class UserService {
    @Autowired
    private UserMapper userMapper;
    @Transactional
    public void addUser(){
        String username = UUID.randomUUID().toString().substring(0, 5);
        userMapper.save(username,18);
        System.out.println("插入完成");
         //事务异常
        int i=10/0;、
            
            
            
    }
```

### 7.创建配置类SpringConfig

```java
//注解扫描
@ComponentScan(basePackages = "com.wxr.tx")
//声明当前类为配置类
@Configuration
//扫描mapper接口
@MapperScan("com.wxr.tx")
@EnableTransactionManagement
public class SpringConfig {
    @Bean
    public DataSource dataSource() throws Exception {
        ComboPooledDataSource dataSource = new ComboPooledDataSource();
        dataSource.setUser("root");
        dataSource.setPassword("wxr.1005");
        dataSource.setDriverClass("com.mysql.jdbc.Driver");
        dataSource.setJdbcUrl("jdbc:mysql://localhost:3306/test");
        return dataSource;
    }

    @Bean
    public SqlSessionFactory sqlSessionFactory() throws Exception {
        //在spring和Mybatis整合中采用mybatis提供的SQLSessionFactoryBean对象
        SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
        //为sqlSessionFatoryBean设置连接池属性
        //Spring对@configuration类会做特殊处理，给容器中添加组件的方法，多次调用都只是从容器中获取组件（单例）
        sqlSessionFactoryBean.setDataSource(dataSource());
        //获取PathMatchingResourcePatternResolver对象为扫描mapper文件做准备
        PathMatchingResourcePatternResolver path = new PathMatchingResourcePatternResolver();
        //设置mapper文件位置
        sqlSessionFactoryBean.setMapperLocations(path.getResources("classpath:UserMapper.xml"));
        //返回SqlSessionFactory对象
        return  sqlSessionFactoryBean.getObject();
    }

    @Bean
    public PlatformTransactionManager transactionManager() throws Exception {
        return new DataSourceTransactionManager(dataSource());
    }
```

### 8.测试代码+输出结果

```java
@Test
public void test01(){
    AnnotationConfigApplicationContext ioc = new AnnotationConfigApplicationContext(SpringConfig.class);
    UserService userService = ioc.getBean(UserService.class);
    userService.addUser();
    ioc.close();
}
```

#### 8.1 当5中事务未异常，插入完成：

![image-20220222183607737](C:\Users\MateBook\Desktop\IDE、数据库系统等工具\API文档\SSM\Spring注解驱动开发\image-20220222183607737.png)

![image-20220222183632414](C:\Users\MateBook\Desktop\IDE、数据库系统等工具\API文档\SSM\Spring注解驱动开发\image-20220222183632414.png)

#### 8.2当5中事务未异常，插入完成，但是回滚：

![image-20220222183816950](C:\Users\MateBook\Desktop\IDE、数据库系统等工具\API文档\SSM\Spring注解驱动开发\image-20220222183816950.png)



![image-20220222183807443](C:\Users\MateBook\Desktop\IDE、数据库系统等工具\API文档\SSM\Spring注解驱动开发\image-20220222183807443.png)****