# 一、Spring注解驱动开发

![](C:\Users\HANG\Desktop\微信图片_20200922211509.png)





# 二、组件注册

## 2.1@Configuration和@Bean给容器添加组件

~~~java
//配置类==配置文件
@Configuration  //告诉spring这是一个配置文件
public class Config {

    //给容器注册一个bean；类型为返回值类型，【id】用方法名作为id
    @Bean
    public User user(){
        return new User();
    }

}
~~~



## 2.2@ConponentScan自动扫描组件&指定扫描规则

~~~java
@ComponentScan(value = "com.jhang",excludeFilters = {
        @ComponentScan.Filter(type = FilterType.ANNOTATION,classes = Controller.class)
},useDefaultFilters = false)
//value:指定扫描的包
//excludeFilters=Filter[]:指定扫描的时候按照什么规则排除哪些组件
//includeFilters=Filter[]:指定扫描的时候只需要包含哪些组件
//userDefaultFilters=false:去除默认规则匹配
~~~



## 2.3 自定义TypeFilter指定过滤规则

~~~java
public class MyTypeFilter implements TypeFilter {

    /**
     * metadataReader:读取的当前正在扫描类的信息
     * metadataReaderFactory:可以获取其他任何类信息
     *
     */
    public boolean match(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory) throws IOException {

        //获取当前类注解的信息
        AnnotationMetadata annotationMetadata = metadataReader.getAnnotationMetadata();

        //获取当前正在扫描类的类信息
        ClassMetadata classMetadata = metadataReader.getClassMetadata();

        //获取当前类资源（类的路径）
        Resource resource = metadataReader.getResource();

        String className = classMetadata.getClassName();

        System.out.println(className);

        if (className.contains("er")) return true;

        return false;
    }
}
~~~



~~~java
//FilterType.ANNOTATION:按照注解
//FilterType.ASSIGNABLE_TPYE:按照给定类型
//FilterType.ASPECTJ:使用ASPECTJ表达式
//FilterType.REGEX:使用正则表达式
//FilterType.CUSTOM:使用自定义规则，必须实现Filter接口

@ComponentScan(value = "com.jhang",includeFilters = {
        @ComponentScan.Filter(type = FilterType.CUSTOM,classes = MyTypeFilter.class)
},useDefaultFilters = false)
~~~



## 2.4 @Scopre设置组件作用域

~~~java
@Configuration
public class Config2 {

    /**
     * scope可选值：
     *     prototype：多例：ioc容器不会将对象放到容器中，每次调用创建一个
     *     singleton：单例:ioc容器启动会创建对象放到ico容器，
     *                以后每次获取都是从容器中拿去（从map中获取）
     *     ----------------------------------
     *     request：同一次请求创建一个实例
     *     session：同一个session创建一个实例
     * @return
     */
    @Bean
    @Scope(value = "singleton")
    public User user(){
        return new User();
    }
}
~~~



## 2.5 @Lazy-bean 懒加载

懒加载：

- 单实例bean：默认容器启动的时候创建对象
- 懒加载：容器启动不会创建对象，第一次获取bean再创建对象，并进行一些初始化



## 2.6 @Condiational——按照条件注册bean

springboot底层大量使用的注解，按照一定条件进行判断。满足条件给容器注册bean

~~~java
//获取bean类型，返回的是Map集合
Map<String, User> beansOfType = applicationContext.getBeansOfType(User.class);

System.out.println(beansOfType);


//动态获取环境变量的值
Environment environment = applicationContext.getEnvironment();
String property = environment.getProperty("os.name");
System.out.println(property);
~~~



~~~java
//判断是否是window系统
public class WindowsCondition implements Condition {

    /**
     * conditionContext:判断条件能使用的上下文（环境）
     * annotatedTypeMetadata：注释信息
     */
    public boolean matches(ConditionContext conditionContext, AnnotatedTypeMetadata annotatedTypeMetadata) {

        // 是否是window系统
        //1、能获取ioc使用的beanfactory
        ConfigurableListableBeanFactory beanFactory = conditionContext.getBeanFactory();

        //2、获取类加载器
        ClassLoader classLoader = conditionContext.getClassLoader();

        //3、获取当前环境信息
        Environment environment = conditionContext.getEnvironment();

        //4、获取bean定义的注册类,可以动态注册一个bean
        //可以注册bean 删除bean 查询bean 判断容器是否有bean
        BeanDefinitionRegistry registry = conditionContext.getRegistry();

        String property = environment.getProperty("os.name");

        if (property.contains("windows")){
            return false;
        }
        return false;
    }
}
~~~



## 2.7 @Import给容器快速导入一个组件

导入第三方包里面的组件

快速的给容器导入一个组件

~~~java
@Configuration
@Import({Color.class, User.class})
public class Config3 {
}
~~~

容器会自动注册这个组件，id默认是全类名



### ImportSelector

这是一个接口，返回需要导入的组件的全类名数组

~~~java
@Configuration
@Import({Color.class, MyImportSelector.class})
public class Config3 {
}


public class MyImportSelector implements ImportSelector {
    /**
     * annotationMetadata:当前标注@Import类的所有注解信息，不仅获取@Import注解还能获取其他注解
     *
     * 返回值：就是要注册进容器的组件
     */
    public String[] selectImports(AnnotationMetadata annotationMetadata) {
        //方法不能返回null值
        return new String[]{"com.jhang.bean.User"};
    }
}
~~~





### ImportBeanDefinitionRegistrar

~~~java
public class MyImportBeanRegistrar implements ImportBeanDefinitionRegistrar {

    /**
     * annotationMetadata:当前标注@Import注解 类的注解信息
     *
     * beanDefinitionRegistry:BeanDefinitionRegistry注册类；
     *             把所有需要添加到容器的bean:调用 BeanDefinitionRegistry.registerBeanDefinition手工注册进来
     *
     */
    public void registerBeanDefinitions(AnnotationMetadata annotationMetadata, BeanDefinitionRegistry beanDefinitionRegistry) {

        //判断容器是否有符合的bean
        boolean red = beanDefinitionRegistry.containsBeanDefinition("com.jhang.bean.Color");

        if (red){
            //指定bean名
            RootBeanDefinition rootBeanDefinition = new RootBeanDefinition(Yellow.class);
            beanDefinitionRegistry.registerBeanDefinition("rainBow",rootBeanDefinition);
        }

    }
}



@Configuration
@Import({Color.class, MyImportBeanRegistrar.class})
public class Config3 {}
~~~





# BeanPostProcessor的使用









# 三、属性赋值



## 3.1 @Value属性赋值

使用@Value赋值：

- 基本数值
- Spel：#{}
- ${}：取出配置文件的值（再运行环境变量里面的值）



## 3.2 @PropertySource

此注解主要用来获取外置配置文件中的属性，保存到运行环境变量中

@PropertySource(value = {"classpath:/person.properties"})



还可以用其他方式获取配置文件中的值

~~~java
Environment env = ApplicationContext.getEnvironment();

//直接获取ket值的属性值，因为配置文件已经加载进环境中了
String property = env.getProperty("person.nickname");

sout(property)

~~~



## 案例

~~~java
@Component
@PropertySource(value={"classpath:/person.properties"})
public class Person{
    
    @Value("${person.name}")
    String name;
    
    @Value("#{20+22}")
    Integer age;
    
    @Value("2020")
    Integer id;
}
~~~









# 四、自动装配

spring利用依赖注入（DI），完成对IOC容器中各个组件的依赖关系赋值



## 4.1 @Autowired

1. 默认优先按照类型去容器找对应的组件：applicationContext.getBean(Book.class)
2. 如果找到多个相同类型的组件：讲属性的名称【名字】作为组件id去容器中查找



## 4.2 @Qualifier()

用来指定哪个组件被装配

@Qualfier("book")



## 4.3 @Auwowired(required=false)

使用required=false：指明当前组件是非必须的装配的。



## 4.4 @Primary

让spring进行自动装配的时候，默认使用首选的bean;也可以继续使用@Qualifierz



## 4.5 @Resource

Spring还支持JSR250（@Resource）和@JSR330（@Inject）

默认是按照组件【名称】进行装配，没有按照@Primary功能和reqiured=false



### @Inject

需要导入javax.inject的包，和Autowired的功能一样。没有reqiured=false



## 4.6 AutowiredAnnotationBeanPostProcessor

用来解析自动装配功能



~~~java
@Component
public class Car{
    
}
~~~







## 4.7 方法、构造器位置的自动注入

@Autowired：构造器，参数，方法，属性

~~~java

//标注在方法，spring容器创建当前对象，就会调用方法，完成赋值
//方法使用的参数，自定义类型的值从ioc容器获取
@Autowired
public void setCar(Car car){
    this.car = car;
}
~~~



默认加在ioc容器中的组件，容器启动会调用无参构造器，创建对象，然后进行初始化赋值操作

~~~java
//构造器要用的组件，都是从容器中获取
@Autowired
public Boss(Car car){
    this.car = car;
    sout("Boss.....有参构造器")
}
~~~



## 4.8 Aware注入spring底层组件&原理

自定义组件如果想要使用spring容器底层的一些组件（ApplicationContext、BeanFactory）

自定义组件实现xxxAware：在创建对象时，会调用接口规定的方法。注入相关的组件Aware

~~~java
public class Preson implements ApplicationContextAware,BeanNameAware,
							   EmbeddedValueResolverAware{
    
    private ApplicationContext context;
    
    public void setApplicationContext(ApplicationContext application){
        this.context = application;
    }
    
    
    public void setBeanName(String name){
        sout("当前bean的名字":name)
    }
        
                                   
    public void setEmbeddedValueResolver(StringValueResolver resolver){
        String value = resolver.resolverStringValue("你好${os.name} 我是#{20*18}");
    }                               
}
~~~



### xxxAware的工作原理

功能使用的事xxxProcessor

ApplicationContextAware ===> ApplicationContextAwareProcessor



## 4.9 @Profile环境搭建

@Profile：spring为我们提供的根据当前环境，动态的切换和激活一些列bean的功能

**注解可以标注在方法上和类上**

**没有标注环境标识的bean，任何环境都可以加载**



开发中特别好：

- 开发环境、测试环境、生产环境
- 数据源的配置的数据源切换
- 根据环境切换组件，根据当前环境动态的切换组件



~~~java
/**
 * 配置数据源的方式很多种
 *
 * 可以直接写死的方式进行注入
 *
 * 可以使用EmbeddedValueResolverAware的值解析器解析数据源信息
 *
 * 可以直接使用${} 直接获取数据源的配置信息
 */
@Configuration
@PropertySource("classpath:/db.properties")
public class Config implements EmbeddedValueResolverAware {

    @Value("${db.username}")
    private String username;

    private StringValueResolver valueResolver;

    /**
     * @Prop：指定组件在哪个环境中才能被注册到容器中，不指定任何环境都可以注册这个组件
     *
     * @Profile("default")
     * 1. 加了环境标识的bean，只有这个环境被激活的时候才能注册到容器中。默认是default环境
     *
     */
    @Profile("dev")
    @Bean("dev")
    public DataSource dataSourceTest(@Value("${db.pwd}") String password) throws Exception {

        ComboPooledDataSource comboPooledDataSource = new ComboPooledDataSource();

        comboPooledDataSource.setUser(username);

        comboPooledDataSource.setPassword(password);

        String s = valueResolver.resolveStringValue("${db.driverClass}");

        comboPooledDataSource.setDriverClass(s);

        comboPooledDataSource.setJdbcUrl("jdbc:mysql://localhost:3306/test");

        return  comboPooledDataSource;
    }

    @Profile("prop")
    @Bean
    public DataSource dataSourcePro(@Value("${db.pwd}") String password) throws Exception {

        ComboPooledDataSource comboPooledDataSource = new ComboPooledDataSource();

        comboPooledDataSource.setUser(username);

        comboPooledDataSource.setPassword(password);

        String s = valueResolver.resolveStringValue("${db.driverClass}");

        comboPooledDataSource.setDriverClass(s);

        comboPooledDataSource.setJdbcUrl("jdbc:mysql://localhost:3306/prop");

        return  comboPooledDataSource;
    }


    /**
     * 可以通过 值解析器进行解析
     * @param stringValueResolver
     */
    public void setEmbeddedValueResolver(StringValueResolver stringValueResolver) {
        this.valueResolver = stringValueResolver;
    }
}
~~~



使用代码的方式进行激活

~~~java
//创建IOC容器
AnnotationConfigApplicationContext configApplicationContext = new AnnotationConfigApplicationContext();

//设置需要激活的环境
configApplicationContext.getEnvironment().setActiveProfiles("test","prop");

//注册主配置类
configApplicationContext.register(Config.class);

//启动刷新容器
configApplicationContext.refresh();
~~~





# 五、APO核心原理

## 5.1 如何使用springAOP



### 切面配置类

~~~java
/**
 * AOP : 指在程序运行期间动态的将某段代码切入到指定方法位置进行的编程方式
 *
 *     在业务逻辑将日志进行打印（方法运行之前，方法运行结束，方法出现异常）
 *
 * 1。导入aop模块：Spring AOP（spring-aspects）
 * 2. 定义一个日志切面类(LogAspects)：切面类里面的方法。需要动态感知
 *       通知方法；
 *            前置通知(@Before)：logStart：在目标方法运行之前通知
 *            后置通知(@After)：logEnd：在目标方法运行结束后通知
 *            返回通知(@AfterReturning)：logReturn：在目标方法正常返回之后运行
 *            异常通知(@AfterThrowing)：logException：在目标方法出现异常后来运行
 *            环绕通知(@Around)：动态代理，手动推进目标方法运行（joinPoint.procced()）
 *
 * 3. 给切面类的目标方法标注何时何地进行运行
 * 4. 将切面类和业务类都加入到容器中
 * 5. 必须告诉spring哪个类是切面类
 * 6. 给当前配置类加一个 EnableAspectAutoProxy 告诉spring 开启注解的AOP模式
 *
 * 7. 在spring中会有很多@Enablexxx功能，就是开启某些配置
 *
 *
 */
@EnableAspectJAutoProxy
@Configuration
public class AopConfig {

    @Bean
    public AopService aopService(){
        return new AopService();
    }

    //切面类加入到容器中
    @Bean
    public LogAspects logAspects(){
        return new LogAspects();
    }

}
~~~



### 切面类

~~~java
//给切面类加入一个注解：告诉spring当前类是一个切面类
@Aspect
public class LogAspects {

    /**
     * 抽取公共切入点表达式
     * 1.本类引用
     * 2.其他的切面引入 全类名限定
     */
    @Pointcut("execution(public * com.baidu.service.*.*(..))")
    public void pointCut(){}

    //在目标方法执行之前切入
    @Before("pointCut()")
    public void logStart(JoinPoint joinPoint){
        Object[] args = joinPoint.getArgs();
        System.out.println("方法名："+joinPoint.getSignature().getName()+"触发了。。。参数列表是：{"+ Arrays.asList(args)+"}");
    }


    @After("pointCut()")
    public void logEnd(){
        System.out.println("触发结束了。。。");
    }

    @AfterReturning(value = "pointCut()",returning = "result")
    public void LogReturn(Object result){
        System.out.println("正常返回，运行结果是:{"+result+"}");
    }

    /**
     * JoinPoint 一定要出现在参数表的第一位
     */
    @AfterThrowing(value = "pointCut()",throwing = "exception")
    public void logException(JoinPoint joinPoint,Exception exception){
        System.out.println("异常出现了.....异常信息：{"+exception+"}");
    }

}
~~~



### 业务类

~~~java
public class AopService {
    public int div(int i,int j){
        return i/j;
    }
}

public static void main(String[] args) {

    AnnotationConfigApplicationContext annotationConfigApplicationContext = new AnnotationConfigApplicationContext(AopConfig.class);

    AopService aopService = annotationConfigApplicationContext.getBean("aopService", AopService.class);

    int div = aopService.div(1, 1);

    System.out.println(div);

    annotationConfigApplicationContext.close();
}

~~~



## 5.2@EnableAspectJAutoProxy 的原理

原理：关注给容器注册了什么组件



给容器中添加一个组件

```java
@Import({AspectJAutoProxyRegistrar.class})
```



利用AspectJAutoProxyRegistrar自定义给容器中注册bean

~~~java
class AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar {
    AspectJAutoProxyRegistrar() {
    }

    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);
        AnnotationAttributes enableAspectJAutoProxy = AnnotationConfigUtils.attributesFor(importingClassMetadata, EnableAspectJAutoProxy.class);
        if (enableAspectJAutoProxy != null) {
            if (enableAspectJAutoProxy.getBoolean("proxyTargetClass")) {
                AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
            }

            if (enableAspectJAutoProxy.getBoolean("exposeProxy")) {
                AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
            }
        }

    }
}
~~~



利用AspectJAutoProxyRegistrar自定义给容器中注册bean

~~~java
private static BeanDefinition registerOrEscalateApcAsRequired(
			Class<?> cls, BeanDefinitionRegistry registry, @Nullable Object source) {}
~~~

internalAutoProxyCreator==AnnotationAwareAspectJAutoProxyCreator

启动给容器注册一个AnnotationAwareAspectJAutoProxyCreator



### AnnotationAwareAspectJAutoProxyCreator是什么？

继承关系：

- AspectJAwareAdvisorAutoProxyCreator
  - AbstractAdvisorAutoProxyCreator
    - AbstractAutoProxyCreator 
      - **implements** SmartInstantiationAwareBeanPostProcessor, BeanFactoryAware
      - 关注后置处理器的工作（在bean初始化完成前后做的事情）、自动装配BeanFactory



- AbstractAutoProxyCreator 

AbstractAutoProxyCreator.setBeanFactory（）

AbstractAutoProxyCreator.postProcessBeforeInstantiation（）

AbstractAutoProxyCreator.postPorcessAfterInitialization（）





- AbstractAdvisorAutoProxyCreator

 AbstractAdvisorAutoProxyCreator.setBeanFactory（）=》initBeanFactory（）

 

- AnnotationAwareAspectJAutoProxyCreator

  AnnotationAwareAspectJAutoProxyCreator.initBeanFactory（）

  

  

  #### 方法调用流程

  1. 传入配置类，创建IOC容器

     （AnnotationConfigApplicationContext anno = new AnnotationConfigApplication()）

  2. 注册配置类，调用refresh() 刷新容器

  3. registerBeanPostProcessor（beanFactory）：注册bean的后置处理器来方便拦截bean的创建

     1. 先获取ioc容器已经定义的需要创建的对象所有beanPostPorcessor
     2. 给容器中加别的BeanPostPorcessor
     3. 优先注册实现了PriorityOrdered接口的BeanPostPorcessor
     4. 再给容器注册实现了Ordered接口的BeanPostPorcessor
     5. 最后注册没有实现优先级接口的BeanPostProcessor

  

















































































