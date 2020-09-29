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

Bean的后置处理器，在bean初始化前后进行一些处理工作。

postProcessBeforeInitialization:初始化之前进行处理工作

postProcessAfterInitialization：初始化之后进行处理工作



执行流程：

遍历得到容器中所有的BeanPostPorcessor；挨个执行，一旦方法返回null，跳出for循环

populateBean(beanName，mbd，instanceWrapper)：给bean进行属性赋值

initializeBean

applyBeanPostPorcessorsBeforeInitialization（wrappedBean）

invokeInitMethods（bean，wrappedBean，mdb）：执行初始化

applyBeanPostProcessorAfterInitialzation（wrappedBean,beanName）



~~~java
/**
 * 后置处理器：初始化前后进行处理
 */
@Component
public class MyBeanPostProcessor implements BeanPostProcessor {

    /**
     *
     * @param bean : Bean
     * @param beanName：Bean的名字
     * @return
     * @throws BeansException
     */
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("postProcessBeforeInitialization" +beanName + "=>" + bean);
        return bean;
    }

    /**
     *
     * @param bean:Bean
     * @param beanName:Bean的名字
     * @return
     * @throws BeansException
     */
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("postProcessAfterInitialization"  +beanName + "=>" + bean);
        return bean;
    }
}
~~~



## spring底层的使用

- ApplicationContextAwareProcessor帮组件注册IOC容器

  ~~~java
  @Component
  public class Preson implements ApplicationContextAware{
      
      private ApplicationContext context;
      
      public void setApplicationContext(ApplicationContext applicationContext){
          
          this.context = applicationContext;
          
      }
  }
  ~~~



ApplicationContextAware：上级接口就是BeanPostProcessor 

注意bean赋值，注入其他组件，数据校验，生命周期注解功能。都是用BeanPostProcessor的使用



# 三、属性赋值



## @Value属性赋值

使用@Value赋值：

- 基本数值
- Spel：#{}
- ${}：取出配置文件的值（再运行环境变量里面的值）



## @PropertySource

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



## @Autowired

1. 默认优先按照类型去容器找对应的组件：applicationContext.getBean(Book.class)
2. 如果找到多个相同类型的组件：讲属性的名称【名字】作为组件id去容器中查找



## @Qualifier()

用来指定哪个组件被装配

@Qualfier("book")



## @Auwowired(required=false)

使用required=false：指明当前组件是非必须的装配的。



## @Primary

让spring进行自动装配的时候，默认使用首选的bean;也可以继续使用@Qualifierz



## @Resource

Spring还支持JSR250（@Resource）和@JSR330（@Inject）

默认是按照组件【名称】进行装配，没有按照@Primary功能和reqiured=false



### @Inject

需要导入javax.inject的包，和Autowired的功能一样。没有reqiured=false



## AutowiredAnnotationBeanPostProcessor

用来解析自动装配功能



~~~java
@Component
public class Car{
    
}
~~~







## 方法、构造器位置的自动注入

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

































































































