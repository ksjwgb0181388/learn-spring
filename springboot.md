# 一、YAML基本语法

1. 使用缩进表示层级关系
2. 缩进时不允许时候TAB键，只允许使用空格
3. 缩进的空格数目不重要，只要相同层级的元素左侧对齐即可
4. 大小写敏感



## 1.1 YAML支持的三种数据结构

### 1. 字面量

普通值的值（数字、字符串、布尔）

字符串默认不用加上单引号和双引号。

双引号：不会转义字符串里面的特殊字符，特殊字符会作为本身想表示意思

单引号：会转义特殊字符，特殊字符最终只是一个普通的字符串





### 2. 对象、Map（属性和值）

~~~yaml
friends:
	lastName: zhangsan
	age: 30
~~~

行内写法

~~~yaml
friends: {lastName:zhangsan,age:30}
~~~



### 3. 数组（List、Set）

用-值表示数组中的一个元素

~~~yaml
pets:
	- cat
	- dog
	- pig
~~~

行内写法

~~~yaml
pets: [cat,dog,pig]
~~~



# 二、读取YAML配置文件的值

## 方式一:@ConfigurationProperties

1. 配置一个实体类组件，配置文件中每一个属性的值

   ~~~java
   @ConfigurationProperties:告诉springboot本类的所有属性都是配置文件
   					 prefix:配置文件中哪个下面的所有属性一一映射
   
   @Component                         
   @ConfigurationProperties(prefix = "person")    
   public class Person{
       
       private String lastName;
       private Date birth;
       private Boolean boss;
       private Map<String,Object> map;
       private List<Object> list;
       private Dog dog;
       
   }
   
   ~~~



**使用配置文件需要导入依赖**

spring默认使用yml中的配置，但有时候要用传统的xml或properties配置，就需要使用spring-boot-configuration-processor了

~~~xml
<dependency>
    <groupId>org.springframework.boot<groupId>
    <artifactId>spring-boot-configuration-processor<artifactId>
    <optional>true<optional>    
<dependency>    
~~~



**只有这个组件是容器中的组件，才能使用容器提供的@ConfigurationProperties功能**



## 方式二:@Value

~~~java
@Component
public class Person{
    
    //@Value等同于以下配置
    <bean class = "Person">
        <property name="lastName" value="?" />
    </bean>    
    
    @Value("字面量、${key}从环境变量、配置文件中获取/#{SPEL}")
    private String name;
    
    private Map<String,Object> map;
    
    private List<Object> list;
    
    private Boolean boss;
}

~~~





## @Value和ConfigurationProperties区别

|                      | @Configuration           | @Value       |
| -------------------- | ------------------------ | ------------ |
| 功能                 | 批量注入配置文件中的属性 | 一个一个指定 |
| 松散绑定(松散语法)   | 支持                     | 不支持       |
| Spel                 | 不支持                   | 支持         |
| JSR303数据校验       | 支持                     | 不支持       |
| 复杂类型封装 例如map | 支持                     | 不支持       |



### @Validated

用来验证每个值的校验 只能和 @ConfigurationProperties 使用



## @EnableConfigurationProperites

使用 @ConfigurationProperties 注解的类生效。



**说明**：如果一个配置类只配置@ConfigurationProperties注解，而没有使用@Component，那么在IOC容器中是获取不到properties 配置文件转化的bean。



当`@EnableConfigurationProperties`注解应用到你的`@Configuration`时， 

任何被`@ConfigurationProperties`注解的beans将自动被Environment属性配置。 这种风格的配置特别适合与SpringApplication的外部YAML配置进行配合使用。



```java
@ConfigurationProperties(prefix = "service.properties")
public class HelloServiceProperties {
    private static final String SERVICE_NAME = "test-service";

    private String msg = SERVICE_NAME;
       set/get
}


@Configuration
@EnableConfigurationProperties(HelloServiceProperties.class)
@ConditionalOnClass(HelloService.class)
@ConditionalOnProperty(prefix = "hello", value = "enable", matchIfMissing = true)
public class HelloServiceAutoConfiguration {

}

@RestController
public class ConfigurationPropertiesController {

    @Autowired
    private HelloServiceProperties helloServiceProperties;

    @RequestMapping("/getObjectProperties")
    public Object getObjectProperties () {
        System.out.println(helloServiceProperties.getMsg());
        return myConfigTest.getProperties();
    }
}
```



注意：

如果@ConfigurationProperties是在第三方包中，那么@component是不能注入到容器的。只有@EnableConfigurationProperties才可以注入到容器。



spring boot版本2.2.0.M5起，@ConfigurationProperties与@Component不能同时存在，否则会出现2次注入，

也就是说，这个@EnableConfigurationProperties 基本没什么用了。。。



 

## 什么时候用value和configurationproeprties

- 只是在某个业务逻辑中需要获取一下配置文件中的某项值，使用@Value
- 专门编写了一个javabean来和配置文件进行映射；使用@configurationproperties



## 配置properties配置文件编码问题

~~~yaml
person.lastName=张三
person.age=18
person.birth=2017/12/15
person.lists=a,b,c
person.dog.name=dog

person.map.name=zhang
person.map.age=22

#Idea使用编码是UTF-8 
~~~



# 三、spring高级配置读取

**@PropertySource、@ImportResource、@Bean**



1. **@PropertySource**：加载指定的配置文件，只能读取 **.properties**文件

~~~java
@PropertySource(value = {classpath:person.properties})

@PropertySource(value = {"classpath:static/config/authorSetting.properties"},
        ignoreResourceNotFound = false, encoding = "UTF-8", name = "authorSetting.properties")

1.value：指明加载配置文件的路径。 
2.ignoreResourceNotFound：指定的配置文件不存在是否报错，默认是false。当设置为 true 时，若该文件不存在，程序不会报错。实际项目开发中，最好设置 ignoreResourceNotFound 为 false。 
3.encoding：指定读取属性文件所使用的编码，我们通常使用的是UTF-8。

~~~

- 如果要读取yml和yaml文件需要重新配置信息

  ~~~java
  public class YamlPropertyResourceFactory implements PropertySourceFactory {
      @Override
      public PropertySource<?> createPropertySource(String name, EncodedResource encodedResource) throws IOException {
          String resourceName = Optional.ofNullable(name).orElse(encodedResource.getResource().getFilename());
          if (resourceName.endsWith(".yml") || resourceName.endsWith(".yaml")){
              List<PropertySource<?>> load = new YamlPropertySourceLoader().load(resourceName, encodedResource.getResource());
             return load.get(0);
          }else{
              return new PropertiesPropertySource(resourceName,new Properties());
          }
      }
  }
  
  @ConfigurationProperties(prefix = "person")
  @PropertySource(value = {"classpath:person.yml"},factory = YamlPropertyResourceFactory.class) //此处指明了要使用的配置文件
  @Component
  public class Person {
  
      private String name;
      private Integer age;
      private Date birth;
      private List<Object> list;
      private Map<String,Object> map;
  }    
  ~~~

  





2. **@ConfigurationProperties**：从全局配置文件中获取



3. **@ImportResource**：导入Spring的配置文件，让配置文件里面的内容生效

~~~xml
<bean id="helloService" class="com.atguigu.springboot.service">
	 
<bean>
    
@Autowired
ApplicationContext ioc;
    
~~~

- springboot里面没有spring的配置文件，我们自己编写的配置文件也不能识别

- 需要标注在主配置类上

  ~~~java
  @ImportResource(localtions = {"classpath:beans.xml"})
  导入spring的配置文件让其生效
  ~~~



springboot推荐给容器中添加组件的方式：推荐使用全注解的方式

1. 配置类 ==== spring配置文件
2. @Configuration:指明当前类是一个配置类，就是代替之前的spring文件

~~~java
@Configuration
public class MyAppConfig{
    
    //将方法的返回值添加到容器中，容器的默认id就是方法名字
    @Bean
    public HelloService helloService(){
        return new HelloService();
    }
}
~~~



## 配置文件的占位符

1. **RandomValuePropertySource**:配置文件中可以使用随机数

   - ${random.value}
   - ${random.int}
   - ${ramdom.long}
   - ${random.int(10)}
   - ${random.int[1024,65535]}

2. **属性配置占位符**

   ~~~yaml
   app.name= MyApp
   app.description= ${app.name} is Spring Boot application
   ~~~

   - 可以在配置文件中引用前面配置过的属性（优先级前面配置过的也可以用）
   - ${app.name:默认值}：来指定找不到的属性时的默认值



# 四、Profile

Profile是spring对不同环境提供不同配置功能的支持，可以通过激活、指定参数等方式快速切换环境

1. 多profile文件形式：
   - 格式：application-{profile}.properties
   - application-dev.properites、application-prod.properties
2. 多profile文档块模式
3. 激活方式：
   - 命令行：--spring.profiles.active=dev
   - 配置文件：spring.profiles.active=dev
   - jvm参数：-Dspring.profile.active=dev





## 多Profile文件

在主配置文件编写的时候，文件名可以是 application-{profile}.properties/yml

默认：使用application.properties的配置

- application-dev.properties
- application-prod.properties





## 激活指定Profile

1. 在配置文件中指定 spring.profiles.active=dev





## yml支持多文档块方式

不用写很多配置文件了

~~~yaml
server:
	port: 8080
spring:
	profiles:
		active: dev
	
--- //这是文档块
server:
	port: 8084
spring:
	profiles: dev

---	//这是文档块
server:
	port :8083
spring:
	profiles: prod
~~~



## 命令行模式

在Program argument中配置命令行

**Program argument**: --spring.profiles.active=dev



# 五、配置文件加载位置

- springboot 启动会扫描以下位置的application.properties或者application.yml文件作为spring boot的默认配置文件
  - file：./config/
  - file：./
  - classpath：/config/
  - classpath：/
  - 以上是按照【优先级从高到低】的顺序，所有位置的文件都会被加载，【高优先级配置内容】会覆盖低优先级内容
  - 也可以通过spring.config.location来改变默认配置
- 互补配置
  - 用高优先级配置部分内容，低优先级全部内容



### spring.config.location改变配置文件

项目打包之后，使用命令行参数的形式，启动项目的时候指定配置文件的新位置；

指定的配置文件和默认加载的配置文件会一起起作用形成互补配置

~~~yaml
java -jar 打包文件 --spring.config.location=配置文件路径
~~~



# 六、外部配置文件加载顺序

从以下位置加载配置，优先级从高到低



1. 命令行参数
2. 来自java:comp/env的JNDI属性
3. 操作系统环境变量
4. RandomValuePropertySource配置的random.*属性
5. jar包外部的application-{profile}.properties或application.yml(带spring.profile)文件
6. jar包内部的application-{profile}.properties或application.yml（带spring.profile）配置文件
7. jar包外部的application.properties或application.yml（不带spring.profile）配置文件
8. jar包内部的application.properties或application.yml（不带spring.profile）配置文件
9. @Configuration注解类上的@PropertySource
10. 通过SpringApplication.setDefaultProperties指定的默认属性



# 七、自动配置原理

1. springboot启动的时候加载主配置类，开启了自动配置功能@EnableAutoConfiguration
2.  @EnableAutoConfiguration
   - 利用EnableAutoConfigurationImportSelector给容器中导入一些组件







# 八、Spring的资源访问接口---Resource

前言：JDK提供的访问资源的类（File等）不能很好满足各种某些资源的访问需求。比如缺少从类路径和Web容器的上下文中获取资源的资源操作类

Spring的Resource接口提供了更好用的资源访问能力。Spring使用Resource访问各种资源文件，配置文件资源，国际化属性资源等。



**Resource.java**

~~~java
package org.springframework.core.io;

import java.io.File;
import java.io.IOException;
import java.net.URI;
import java.net.URL;
import org.springframework.core.io.InputStreamSource;

public interface Resource extends InputStreamSource {
    boolean exists();//资源是否存在

    boolean isReadable();

    boolean isOpen();//资源是否打开

    URL getURL() throws IOException;//如果底层资源可以表示成URL，则返回对应的URL对象

    URI getURI() throws IOException;

    File getFile() throws IOException;//如果底层资源是一个文件，则返回对应的File对象

    long contentLength() throws IOException;

    long lastModified() throws IOException;

    Resource createRelative(String var1) throws IOException;

    String getFilename();

    String getDescription();
}
~~~

Resource 继承了 InputStreamSource



## 最常用的Resource实现类：

- **WritableResource**：可写资源接口。两个实现类PathResource和FileSystemResource
- **ClassPathResource**：用于加载类路径下的资源
- **FileSystemResource**：用于加载文件系统资源，如：D:/conf/test.xml
- **UrlResource**：访问任何可以通过URL表示的资源，包括文件系统资源、HTTP资源、FTP资源
- **PathResource**：可以访问任何通过URL、Path、系统文件路径表示的资源





## 资源访问Demo - FileSourceExample

~~~java
package springResources;

import org.springframework.core.io.ClassPathResource;
import org.springframework.core.io.PathResource;
import org.springframework.core.io.Resource;
import org.springframework.core.io.WritableResource;

import java.io.*;

/**
 * Created by Mistra-WR on 2018/3/18/018.
 */
public class FileSourceExample {
    public static void main(String [] args){
        try {
            String  filePath = "D:/Working/Spring/src/main/resources/conf/resourceFileTest.txt";
            //使用系统文件路径方式加载文件
            WritableResource res1 = new PathResource(filePath);
            //使用类路径方式加载文件
            Resource res2 = new ClassPathResource("conf/resourceFileTest.txt");
            
            EncodedResource encRes = new EncodedResource(res2,"UTF-8");//可以对资源进行特定编码
            File file1 = res1.getFile();//获得File对象
            File file2 = encRes.getResource().getFile();//获得File对象
            
            OutputStream stream1 = res1.getOutputStream();//获取文件输出流
            stream1.write("我是小王瑞".getBytes());
            stream1.close();

            //使用Resource接口读资源文件
            InputStream ins1 = res1.getInputStream();//获取文件输入流
            InputStream ins2 = res2.getInputStream();

            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            int i;
            while ((i = ins1.read())!=-1){
                baos.write(i);
            }
            System.out.println(baos.toString());

            System.out.println("res1:" + res1.getFilename());//获得文件名
            System.out.println("res2:" + res2.getFilename());
        }catch (IOException e){
            e.printStackTrace();
        }
    }
}
~~~



### **资源类型的地址前缀：**

| 前缀       | 例子                           | 说明                                 |
| ---------- | ------------------------------ | ------------------------------------ |
| classpath: | classpath:com/myapp/config.xml | 从classpath中加载。                  |
| file:      | file:/data/config.xml          | 作为 `URL` 从文件系统中加载。        |
| http:      | http://myserver/logo.png       | 作为 `URL` 加载。                    |
| (none)     | /data/config.xml               | 根据 `ApplicationContext` 进行判断。 |



## Spring加载resource时classpath*:与classpath:的区别

- classpath*:的出现是为了从多个jar文件中加载相同的文件

- classpath:只能加载找到的第一个文件.



**比如 resource1.jar中的package 'com.test.rs' 有一个 'jarAppcontext.xml' 文件,内容如下:**

<bean name="ProcessorImplA" class="com.test.spring.di.ProcessorImplA" />

**resource2.jar中的package 'com.test.rs' 也有一个 'jarAppcontext.xml' 文件,内容如下:**

<bean id="ProcessorImplB" class="com.test.spring.di.ProcessorImplB" />



**通过使用下面的代码则可以将两个jar包中的文件都加载进来**

`ApplicationContext ctx = new ClassPathXmlApplicationContext( "classpath*:com/test/rs/jarAppcontext.xml");`



**而如果写成下面的代码,就只能找到其中的一个xml文件(顺序取决于jar包的加载顺序)**

`ApplicationContext ctx = new ClassPathXmlApplicationContext( "classpath:com/test/rs/jarAppcontext.xml");`



## PathMatchingResourcePatternResolver

扫描虽有类路径下及JAR包中对应com.smart类包下的路径，读取所有以.xml为后缀的文件资源。

~~~java
package springResources;

import org.springframework.core.io.Resource;
import org.springframework.core.io.support.PathMatchingResourcePatternResolver;
import org.springframework.core.io.support.ResourcePatternResolver;
import org.testng.annotations.Test;

/**
 * Created by Mistra-WR on 2018/3/20/020.
 */
public class PatternResolverTest {

    @Test
    public void getResources() throws Throwable{
        ResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();

        Resource resource[] = resolver.getResources("classpath*:com/smart/**/*.xml");

        for (Resource resource1:resource){
            System.out.println(resource1.getDescription());
        }
    }
}
~~~



# SpringBoot整合Druid问题细节

1. 使用**DataSourceBuilder**创建DataSource时候需要指定Type类型

   `DataSourceBuilder.create().type(DruidDataSource.class).build();`

   



# 九、SpringBoot 向容器注册Bean的多种方式

**向Spring容器注册Bean有多种方式**

- @ComponentScan
- @Bean
- @Import



## 通过@ComponentScan注册Bean

Spring容器会扫描@ComponentScan配置的包路径，找到标记@Component注解的类加入到Spring容器。

效果等同于XML配置文件中的`<context:component-scan base-package="包名">`



| 常用属性名     | 类型     | 说明                       |
| -------------- | -------- | -------------------------- |
| includeFilters | Filter[] | 指定扫描导入类型的过滤规则 |
| excludeFilters | Filter[] | 指定扫描排除类型的过滤规则 |

**默认是加载和Application类所在同一个目录下的所有类，包括所有子目录下的类。**

**当启动类和**@Component**分开时，如果启动类在某个包下，需要在启动类中增加注解@ComponentScan，配置需要扫描的包名。**



## @Component说明

常见继承：
\- @Configuration：标记类为配置类，常与@ComponentScan或@Bean注解一起使用
\- @Controller
\- @Repository
\- @Service





## 通过@Bean注册Bean

标记在方法上，将方法返回值注册到Spring容器，类型为返回值类型，id默认为方法名。

效果等同于XML配置文件中的`<bean id="beanName" class="className"/>`



## 通过@Import注册Bean

 容器中就会自动注册这个组件，id默认是全类名

- 直接注册指定类

```java
// 启动类
@Import({ ImportTest.class })
public class RegistryBean {

    public static void main(String[] args) throws Exception {
        ConfigurableApplicationContext context = SpringApplication.run(RegistryBean.class, args);
        String[] beanNames = context.getBeanDefinitionNames();
        for (String beanName : beanNames) {
            System.out.println(beanName);
        }
    }

}

public class ImportTest {
}
```



- 配合ImportSelector接口注册指定类,**返回需要导入的组件的全类名数组；**
- 他是ImportSelector的一个扩展实现。基本是在所有的Configuration中所有bean实例化完成后，才会触发。在Import的类是Conditional的时候，这个类型的Selector特别有用。
- 主要实现方法是selectImports，返回对应的全类名。

```java
//自定义逻辑返回需要导入的组件
public class MyImportSelector implements ImportSelector {
 
    //返回值，就是到导入到容器中的组件全类名
    //AnnotationMetadata:当前标注@Import注解的类的所有注解信息
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        // TODO Auto-generated method stub
        //importingClassMetadata
        //方法不要返回null值
        //当前类的所有注解
        Set<String> annotationTypes = importingClassMetadata.getAnnotationTypes();
        System.out.println("当前配置类的注解信息："+annotationTypes);
        //注意不能返回null,不然会报NullPointException
        return new String[]{"com.paopaoedu.springboot.bean.User01","com.paopaoedu.springboot.bean.User02"};
    }
}
 
public class User01 {
	public String username;
 
    public User01() {
        System.out.println("user01...constructor");
    }
}
 
public class User02 {
    public String username;
 
    public User02() {
        System.out.println("user02...constructor");
    }
}
 
@Configuration
@Import({ImportDemo.class, MyImportSelector.class})
public class ImportConfig {
 
    /**
     * 给容器中注册组件；
     * 1）、包扫描+组件标注注解（@Controller/@Service/@Repository/@Component）[自己写的类]
     * 2）、@Bean[导入的第三方包里面的组件]
     * 3）、@Import[快速给容器中导入一个组件]
     * 		1）、@Import(要导入到容器中的组件)；容器中就会自动注册这个组件，id默认是全类名
     * 		2）、ImportSelector:返回需要导入的组件的全类名数组；
     * 		3）、ImportBeanDefinitionRegistrar:手动注册bean到容器中
     * 4）、使用Spring提供的 FactoryBean（工厂Bean）;
     * 		1）、默认获取到的是工厂bean调用getObject创建的对象
     * 		2）、要获取工厂Bean本身，我们需要给id前面加一个&，&userFactoryBean
     */
 
    @Bean
    public User user(){
        return new User("Lily");
    }
}
 
@RestController
public class ImportDemoController {
 
    @Autowired
    private User user;
 
    @Autowired
    private ImportDemo importDemo;
 
    @Autowired
    private User01 user01;
 
    
 
    @RequestMapping("/importDemo")
    public String demo() throws Exception {
        importDemo.doSomething();
        user01.username = "user01";
        String s = user.username;
        String s1 = user01.username;
 
        return "ImportDemo@SpringBoot " + s + " " + s1;
    }
}
```

同时修改启动类上注解`@Import({ ImportTest.class })`为`@Import({ ImportTest.class，ImportSelectorTest .class })`

 

- 配合ImportBeanDefinitionRegisterar接口注册指定类,手动注册bean到容器中

```java
public class MyImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {
 
    /**
     * AnnotationMetadata：当前类的注解信息
     * BeanDefinitionRegistry:BeanDefinition注册类；
     * 		把所有需要添加到容器中的bean；调用
     * 		BeanDefinitionRegistry.registerBeanDefinition手工注册进来
     */
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
 
        boolean definition = registry.containsBeanDefinition("com.paopaoedu.springboot.bean.User01");
        boolean definition2 = registry.containsBeanDefinition("com.paopaoedu.springboot.bean.User02");
        if(definition && definition2){
            //指定Bean定义信息作用域都可以在这里定义；（Bean的类型，Bean。。。）
            RootBeanDefinition beanDefinition = new RootBeanDefinition(User03.class);
            //注册一个Bean，指定bean名
            registry.registerBeanDefinition("User03", beanDefinition);
        }
    }
 
}
```

同时修改启动类上注解`@Import({ ImportTest.class })`为`@Import({ ImportTest.class,ImportSelectorTest.class, ImportBeanDefinitionRegistrarTest.class })`

 

## 使用Spring提供的 FactoryBean(工厂Bean)

- 默认获取到的是工厂bean调用getObject创建的对象
- 要获取工厂Bean本身，我们需要给id前面加一个&& xxxFactoryBean 注意类名是X，这里就是小写的x？



~~~java
public class UserFactoryBean implements FactoryBean<User04> {
    @Override
    public User04 getObject() throws Exception {
        // TODO Auto-generated method stub
        System.out.println("UserFactoryBean...getObject...");
        return new User04("User04");
    }
 
    @Override
    public Class<?> getObjectType() {
        // TODO Auto-generated method stub
        return User04.class;
    }
 
    //是否单例？
    //true：这个bean是单实例，在容器中保存一份
    //false：多实例，每次获取都会创建一个新的bean；
    @Override
    public boolean isSingleton() {
        // TODO Auto-generated method stub
        return true;
    }
}
 
 
public class User04 {
    public String username;
    public User04(String s) {
        String nowtime= DateUtil.now();
        username=s+" "+nowtime;
    }
}
 
 
 
@Configuration
@Import({ImportDemo.class, MyImportSelector.class, MyImportBeanDefinitionRegistrar.class})
public class ImportConfig {
 
    /**
     * 给容器中注册组件；
     * 1）、包扫描+组件标注注解（@Controller/@Service/@Repository/@Component）[自己写的类]
     * 2）、@Bean[导入的第三方包里面的组件]
     * 3）、@Import[快速给容器中导入一个组件]
     * 		1）、@Import(要导入到容器中的组件)；容器中就会自动注册这个组件，id默认是全类名
     * 		2）、ImportSelector:返回需要导入的组件的全类名数组；
     * 		3）、ImportBeanDefinitionRegistrar:手动注册bean到容器中
     * 4）、使用Spring提供的 FactoryBean（工厂Bean）;
     * 		1）、默认获取到的是工厂bean调用getObject创建的对象
     * 		2）、要获取工厂Bean本身，我们需要给id前面加一个&，&userFactoryBean
     */
    @Bean
    public UserFactoryBean userFactoryBean(){
        return new UserFactoryBean();
    }
 
    @Bean
    public User user(){
        return new User("Lily");
    }
}
 
 
@RestController
public class ImportDemoController {
 
    @Autowired
    private User user;
 
    @Autowired
    private ImportDemo importDemo;
 
    @Autowired
    private User01 user01;
 
    @Autowired
    private UserFactoryBean userFactoryBean;
 
    @RequestMapping("/importDemo")
    public String demo() throws Exception {
        importDemo.doSomething();
        user01.username = "user01";
        String s = user.username;
        String s1 = user01.username;
        String s4 = userFactoryBean.getObject().username;
 
        return "ImportDemo@SpringBoot " + s + " " + s1 + " " + s4;
    }
}
 
 
@SpringBootApplication
public class SpringBootLearningApplication {
 
    public static void main(String[] args) {
        SpringApplication.run(SpringBootLearningApplication.class, args);
 
        AnnotationConfigApplicationContext context =
                new AnnotationConfigApplicationContext("com.paopaoedu.springboot.config");
        ImportDemo importDemo = context.getBean(ImportDemo.class);
        importDemo.doSomething();
        printClassName(context);
 
        Object bean1 = context.getBean("userFactoryBean");
        Object bean2 = context.getBean("userFactoryBean");
        System.out.println(bean1 == bean2);
    }
 
    private static void printClassName(AnnotationConfigApplicationContext annotationConfigApplicationContext){
        String[] beanDefinitionNames = annotationConfigApplicationContext.getBeanDefinitionNames();
        for (int i = 0; i < beanDefinitionNames.length; i++) {
            System.out.println("匹配的类"+beanDefinitionNames[i]);
        }
    }
}
~~~





# AnnotationMetadata

~~~java
// @since 2.5
public interface AnnotationMetadata extends ClassMetadata, AnnotatedTypeMetadata {

	//拿到当前类上所有的注解的全类名（注意是全类名）
	Set<String> getAnnotationTypes();
	// 拿到指定的注解类型
	//annotationName:注解类型的全类名
	Set<String> getMetaAnnotationTypes(String annotationName);
	
	// 是否包含指定注解 （annotationName：全类名）
	boolean hasAnnotation(String annotationName);
	//这个厉害了，用于判断注解类型自己是否被某个元注解类型所标注
	//依赖于AnnotatedElementUtils#hasMetaAnnotationTypes
	boolean hasMetaAnnotation(String metaAnnotationName);
	
	// 类里面只有有一个方法标注有指定注解，就返回true
	//getDeclaredMethods获得所有方法， AnnotatedElementUtils.isAnnotated是否标注有指定注解
	boolean hasAnnotatedMethods(String annotationName);
	// 返回所有的标注有指定注解的方法元信息。注意返回的是MethodMetadata 原理基本同上
	Set<MethodMetadata> getAnnotatedMethods(String annotationName);
}

~~~

### 定义一个注解

~~~java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(MyHttpDeferredImportSelector.class)
public @interface MyHttp {
    String name() default "";
    String value() default "";

}
~~~



### 注解实现类

~~~java
@Slf4j
public class MyHttpDeferredImportSelector implements DeferredImportSelector {
    @Override public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        importingClassMetadata.getAllAnnotationAttributes(MyHttp.class.getName(),true)
            .forEach((k,v) -> {
                log.info(importingClassMetadata.getClassName());
                log.info("k:{},v:{}",k,String.valueOf(v));
            });
        return new String[0];
    }
}
~~~



### 使用注解

~~~java
@Slf4j
@MyHttp(name = "myc1",value = "myc1-value")
public class MyConfiguration1 {
    public MyConfiguration1() {
      log.info("MyConfiguration1 construct...");
    }

    public void execute() {
        log.info("MyConfiguration1 execute...");
    }
}
~~~



我们运行的时候能看到如下结果：

![](C:\Users\Admin\Desktop\5675167-cde649438b87353f.webp)



**我们能取到使用注解类的所有信息。这样子，在自定义一些处理规则的时候，会方便很多。**





# RootBeanDefinition、ChildBeanDefinition、GenericBeanDefinition的区别

1. RootBeanDefinition可以单独作为一个BeanDefinition，也可以作为其他BeanDefinition的父类。但是他不能作为其他BeanDefinition的子类
2. ChildBeanDefinition相当于一个子类，不可以单独存在，必须要依赖一个父BeanDetintion。（最大的区别他的parentName属性是通过构造方法设置的，而且并没有提供一个无参构造方法给我们。)





# @Configuration 和 @Component 区别

> 一句话概括就是 `@Configuration` 中所有带 `@Bean` 注解的方法都会被动态代理，因此调用该方法返回的都是同一个实例。
>



## `@Configuration` 注解：

~~~java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Configuration {
    String value() default "";
}
~~~

从定义来看， `@Configuration` 注解本质上还是 `@Component`，因此 `<context:component-scan/>` 或者 `@ComponentScan` 都能处理`@Configuration` 注解的类。















# 十、@Conditional&自动配置报告

**@Conditional派生注解**（spring注解原生的@Conditional作用）

作用：必须是@Conditional指定的条件成立，才能给容器添加组件，配置里面的内容才生效



| @Conditional扩展                | 作用（判断是否满足当前指定条件）           |
| ------------------------------- | ------------------------------------------ |
| @ConditionalOnJava              | 系统的java版本是否符合要求                 |
| @ConditionalOnBean              | 容器中存在指定Bean                         |
| @ConditionalOnMissingBean       | 容器中不存在指定Bean                       |
| @ConditionalOnExpression        | 满足Spel表达式指定                         |
| @ConditionalOnClass             | 系统中指定的类                             |
| @ConditionalOnMissingClass      | 系统中没有指定的类                         |
| @ConditionalOnSingleCandidate   | 系统中只有一个Bean，或者这个Bean是首选Bean |
| @ConditionalOnResource          | 类路径下是否存在指定资源文件               |
| @ConditionalOnProperty          | 系统中指定的属性是否有指定的值             |
| @ConditionalOnWebApplication    | 当前是Web环境                              |
| @ConditionalOnNotWebApplication | 当前不是Web环境                            |
| @ConditionalOnJndi              | JDNI存在指定项                             |
|                                 |                                            |



**自动配置类必须在一定条件下才能生效**

如何知道哪些自动配置类生效？

配置文件中写上：`debug=true`





# 十一、嵌入式Servlet容器配置修改

springboot默认是用嵌入式Servlet容器（Tomcat）



1. 如何定制和修改Servlet容器的相关配置

   ServerProeprties

   ~~~yaml
   server.port= 8081
   server.context-path=/crud
   ~~~



​	编写一个EmbeddedServletContainerCustomizer：嵌入式的Servlet容器的定制器

```java
@Bean
public EmbeddedServletContainerCustomizer customer(){
    return new EmbeddedServletContainerCustomizer(){
        
        //定制嵌入式的Servlet容器相关规则
        public void customize(ConfigurableEmbeddedServletContainer  							 container){
            
            container.setPort(8033);
        }
    }
}
```

​	

​	springboot里面有很多的xxxCustomizer帮助我们进行定制配置



springboot能否支持其他servlet容器



## 配置嵌入式Servlet容器

注册Servlet、Filter、Listener

1. ServletRegistrationBean
2. FilterRegistrationBean
3. ServletListenerRegistrationBean



注册Servlet

~~~java
public class MyServlet extends HttpServlet{
    
    //处理get请求
    protected void doGet(HttpServletRequest rq,HttpServletReponse 						   resp){
        doPost(req,resp);
    }
}
~~~

~~~java
@Configuration
public class MyServerConfig{
    //注册三大组件
    @Bean
    public ServletRegistrationBean myServlet(){
        return new ServletRegistrationBean(new 														MyServlet,"myServlte");
    }
}

~~~



注册Listener

~~~java
public class MyListener implements ServletContextListener{
    
    public void contextInitialized(ServletContextEvent sce){
        sout("web容器启动");
    }
    
    public void contextDestoryed(ServletContextEvent sce){
        sout("web容器销毁");
    }
}

~~~

~~~java
@Bean
public ServletListenerRegistrationBean myListener(){
    return new ServletListenerRegistration<MyListener>(new 													MyListener);
}

~~~



## 切换其他Servlet容器

1. Jetty(长连接)：应用于聊天等
2. Undertow（不支持JSP）：适合做高并发



# 十二、数据访问

> 前言：对于数据访问层，无论是SQL还是NOSQL，SpringBoot默认采用整合SpringData的方式进行统一处理，添加了大量的自动配置，屏蔽了很多设置。
>
> 引入了xxxTemplate，xxxRepository来简化我们对数据库访问层的操作。对我们来说只需要简单的设置即可



## JDBC&自动配置原理

1. 引入starter ： spring-boot-starter-jdbc
2. 配置application.yml
3. 测试
4. 高级配置：使用druid数据源
   1. 引入druid
   2. 配置属性
5. 配置druid数据源监控



~~~yaml
spring:
	datasource:
		username: root
		password: root
		url: jdbc:mysql://localhost:3306/e_spring
		driver-class-name: com.mysql.jdbc.Driver
~~~

- 默认是用org.apache.tomcat.jdbc.pool.DataSource作为数据源
- 数据源的相关配置都在DataSourceProperties里面
- 自动配置原理
  
  - 参考DataSourceConfiguraion：根据配置创建数据源，默认使用Tomcat连接池。可以使用spring.datasource.type指定数据源
  
  - 自定义数据源
  
    ~~~java
    @ConditionalOnMissingBean(DataSource.class)
    @ConditionalOnProperty(name="spring.datasource.type")
    static class Generic{
        
        @Bean
        public DataSource dataSource(DataSourceProperties properties){
            //使用DataSourceBuilder创建数据源，利用反射创建响应type的数据源
        }
    }
    ~~~
  
    







## SpringBoot读取SQL脚本

1. DataSourceInitializer：ApplicationListener

   1. runSchemaScripts：运行建表语句
   2. runDataScripts：运行插入数据的语句

2. 默认文件命名

   ~~~yml
   schema-*.sql、data-*.sql
   默认规则：schema.sql，schema-all.sql
   可以使用：
   spring: 
   	schema:
   		- classpath：department.sql
   ~~~

3. 操作数据库：自动配置了JDBCTemplate



## 整合Mybatis（一）

1. @Mapper:这是一个操作数据库的mapper
2. @Options:用来指定数据库主键生成策略
   1. useGenerateKeys=true
   2. keyProperty="id"



3. 配置Mybatis的配置信息

   ~~~java
   @Configuration
   public class MyBatisConfig{
       
       @Bean
       public ConfigurationCustomizer configurationCustomer(){
           
           return new ConfigurationCustomizer(){
               
               public void customizer(Configuration c){
                   c.setMapUnderscoreToCamelCase(true);
               }
           }
       }
   }
   ~~~

4. 配置文件版本

   ~~~java
   mybatis:
   	config-location: classpath:mybatis-config.xml
       mapper-locations:classpath:mapper/*.xml
   ~~~

   

## 整合JPA

### springdata

springdata是springboot底层默认采用的技术，包括了非关系型数据库、Map-Reduce框架、云服务等等，另外还包括了对关系型数据库的访问支持



### 统一的Repository接口

1. Repository<T,ID extends Serializable>：统一接口
2. RevisionRepository<T,ID extends Serializable,N extends Number& Comparable<N>> :基于乐观锁机制
3. CrudRepository<T,ID extends Serializable>：基本CRUD操作
4. PagingAndSortingRepository<T，ID extends Serializabel>：基本CRUD分页



## JPA配置

1. 导入JPA的Maven坐标

2. 编写一个实体类和数据表进行映射，并且配置好关系

   ~~~java
   @Entity // 告诉JPA这是一个实体类（和数据表映射的类）
   @Table(name="person")// 告诉指定和哪个数据表对应；如果省略默认就是类名小写
   public class Person{
       
       @Id //这是主键
       //自增
       @GeneratedValue(starategy = GenerationType.IDENTITY)
       private String name;
       
       @Column(name="age",length=50)// 和数据库列名
       private Integer age;
       private String email;
   }
   ~~~

3. 编写DAO

   ~~~java
   public interface UserRepository 
       		extends JpaRepository<User,Integer>{
       
       
   }
   ~~~

4. 编写application.yml

   ~~~yaml
   spring:
   	jpa:
   		hibernate:
   			ddl-auto:update
   		show-sql: true
   ~~~

   



# 十三、SpringBoot启动配置原理

## 重要的事件回调机制

- ApplicationContextInitializer
- SpringApplicationRunListener
- ApplicationRunner
- CommandLineRunner





# 十四、**自定义属性源工厂**

如果想要更加灵活的自定义属性源，比如实现从中心化的配置服务加载配置，可以通过实现PropertySourceFactory接口，并通过配置PropertySource注解的factory参数来实现。



~~~java
@Configuration
@PropertySource(value = ""/*placeholder*/,
    factory = CompositePropertySourceFactory.class)
public class CompositeConfigAutoConfiguration {
}
~~~

value字段用于指定配置源对应的资源文件，如果不需要使用资源文件，可以配置为任意值，参数值将会被传递到factory参数的createPropertySource方法。



PropertySourceFactory接口的定义如下：

~~~java
/**
 * Strategy interface for creating resource-based {@link PropertySource} wrappers.
 *
 * @author Juergen Hoeller
 * @since 4.3
 * @see DefaultPropertySourceFactory
 */
public interface PropertySourceFactory {
 
 /**
 * Create a {@link PropertySource} that wraps the given resource.
 * @param name the name of the property source
 * @param resource the resource (potentially encoded) to wrap
 * @return the new {@link PropertySource} (never {@code null})
 * @throws IOException if resource resolution failed
 */
 PropertySource<?> createPropertySource(String name, EncodedResource resource) throws IOException;
 
}
~~~



**需要注意的是PropertySourceFactory的加载时机早于Spring Beans容器，因此实现上不能依赖于Spring的IOC。**



PropertySourceFactory要求实现类返回PropertySource。PropertySource是Spring属性（或者说配置）功能的核心接口，有很多实现，比如：

- ResourcePropertySource 从Resource加载PropertySource
- PropertiesPropertySource 从properties文件加载PropertySource
- SystemEnvironmentPropertySource 从系统环境变量加载PropertySource
- MapPropertySource 包装一个Map为PropertySource（Adapter模块）
- CompositePropertySource 支持将若干PropertySource进行组合（Composite模式）



在自定义属性源时比较常用的是MapPropertySource和CompositePropertySource。

MapPropertySource可以用于将自己加载的属性数据包装，参考其构造方法。































