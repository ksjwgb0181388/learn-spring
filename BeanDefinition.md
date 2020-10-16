# 一、问题

为什么spring不能用Class来建立bean呢?

因为Class无法完成bean的抽象，比如bean的作用域，bean的注入模型，bean是否是懒加载等等信息，Class是无法抽象出来的，故而需要一个BeanDefinition类来抽象这些信息，以便于spring能够完美的实例化一个bean



> 上述文字可以简单理解spring当中的BeanDefinition就是java当中的Class
> Class可以用来描述一个类的属性和方法等等其他信息
> BeanDefintion可以描述springbean当中的scope、lazy，以及属性和方法等等其他信息



用一副图来说明java实例化一个对象的基本流程

![](C:\Users\Admin\Desktop\learn-spring\图片\20191018150846696.jpg)



但是spring的bean实例化过程和一个普通java对象的实例化过程还是有区别的，同样用一幅图来说明一下

![](C:\Users\Admin\Desktop\learn-spring\图片\20191018151632339.png)







# 二、启动流程

1. 当spring容器启动的时候会去调用ConfigurationClassPostProcessor这个bean工厂的后置处理器完成扫描
2. spring设计了一个BeanDefintion的类用来存储这些信息。故而当spring读取到类的信息之后
   1. 会实例化一个BeanDefinition的对象，继而调用这个对象的各种set方法存储信息；
   2. 每扫描到一个符合规则的类，spring都会实例化一个BeanDefinition对象，然后把根据类的类名生成一个bean的名字（比如一个类IndexService，spring会根据类名IndexService生成一个bean的名字`indexService`,spring内部有一套默认的名字生成规则，但是程序员可以提供自己的名字生成器覆盖spring内置的
   3. 继而spring会把这个beanDefinition对象和生成的beanName放到一个map当中，key=beanName，value=beanDefinition对象；至此上图的第①②③步完成。
3. 当spring把类所对应的beanDefintion对象存到map之后，spring会调用程序员提供的bean工厂后置处理器。
   1. 在spring的代码级别是用一个接口来表示BeanFactoryPostProcessor，只要实现这个接口便是一个bean工厂后置处理器了
   2. `ConfigurationClassPostProcessor`便是一个spring自己实现的bean工厂后置处理器，这个类笔者认为是阅读spring源码当中`最重要`的类，没有之一



![](C:\Users\Admin\Desktop\learn-spring\图片\20191018204649189.jpg)



## 接口的介绍

**只需关注两个接口`BeanDefinitionRegistryPostProcessor`和`BeanFactoryPostProcessor`**



因为spring完成上述①②③步的功能就是调用`BeanDefinitionRegistryPostProcessor`的`postProcessBeanDefinitionRegistry`方法完成的



到了第④步spring首先会调用`ConfigurationClassPostProcessor`的`BeanFactoryPostProcessor`的`postProcessBeanFactory`的方法

然后在调用程序员提供的`BeanFactoryPostProcessor`的`postProcessBeanFactory`方法



即使程序员没有提供自己扩展的`BeanFactoryPostProcessor`，spring也会执行内置的`BeanFactoryPostProcessor`也就是`ConfigurationClassPostProcessor`，所以上图画的并不标准，少了一步；即spring执行内置的`BeanFactoryPostProcessor`



## 重点总结

1. 如果是直接实现`BeanFactoryPostProcessor`的类是在spring完成扫描类之后（所谓的扫描包括把类变成beanDefinition然后put到map之中），在实例化bean（第⑤步）之前执行；
2. 如果是实现`BeanDefinitionRegistryPostProcessor`接口的类；诚然这种也叫bean工厂后置处理器他的执行时机是在执行直接实现`BeanFactoryPostProcessor`的类之前，和扫描（上面①②③步）是同期执行
   1. 假设你的程序扩展一个功能，需要在**这个时期**做某个功能则可以实现这个接口
   2. mybatis的最新代码里面便是扩展这个类来实现的





# 三、BeanFactoryPostProcessor的作用和意义

## 3.1 接口的方法签名

~~~java
void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
~~~

Modify the application context’s internal bean factory after its standard initialization
在应用程序上下文的标准初始化之后修改它的内部bean工厂



1. 其实是spring提供的一个扩展点（spring提供很多扩展点，学习spring源码的一个非常重要的原因就是要学会这些扩展点，以便对spring做二次开发或者写出优雅的插件）
2. 可以让程序员干预bean工厂的**初始化过程**,注意不是==实例化过程==
3. 整个容器初始化过程就是spring各种后置处理器调用过程;各种后置处理器当中大体分为两种
   1. 一种关于实例化的后置处理器
   2. 一种是关于初始化的后置处理器
4. beanFactory怎么new出来的（实例化）`BeanFactoryPostProcessor`是干预不了的，但是beanFactory new出来之后各种属性的填充或者修改（初始化）是可以通过`BeanFactoryPostProcessor`来干预；
5. 可以看到`BeanFactoryPostProcessor`里唯一的方法`postProcessBeanFactory`中唯一的参数就是一个标准的beanFactory对象——**ConfigurableListableBeanFactory**；





# 四、BeanFactoryPostProcessor后置处理器的流程

![](C:\Users\Admin\Desktop\learn-spring\图片\20191028142612376.jpg)



1. 启动main方法，在main方法里面调用

2. AnnotationConfigApplicationContext的无参构造方法，然后②-1在无参构造方法里面首先实例化

3. 继而②-2spring又实例化了一个ClassPathBeanDefinitionScanner对象，这个对象顾名思义就是能够用来完成spring的扫描功能，但是这里提一句——spring内部完成扫描功能并不是用的这个对象，而是在扫描的时候又new了一个新的ClassPathBeanDefinitionScanner对象；
4. 调用register(Appconfig.class);首先会把Appconfig类解析成为一个beanDefintion对象
5. 给解析出来的beanDefinition对象设置一些默认属性，继而put到beanDefintionMap当中.spring后面会遍历这个map根据map当中的beanDefinition来实例化bean
6. Appconfig类的beanDefintion存在在map当中那么他必然会被spring容器实例化称为一个bean
   1. Appconfig当中有很多加了@Bean的方法，这些方法需要被调用，故而需要实例化，但是Appconfig类的实例化很复杂比一般类实例化过程复杂很多，涉及到代理涉及到cglib等等



## 4.1 AnnotatedBeanDefinitionReader这个对象实例化出来有什么用？

1. 可用于编程式动态注册一个带注解的bean
   1. 比如我们有一个类A存在com.shadow包下面，并且是一个加注解的类。比如加了@Component，正常情况下这个类A一般是被spring扫描出来的，但是有不正常情况，比如spring并没有扫描到com.shadow包，那么类A就无法被容器实例化。有人可能会问为什么没有扫描到com.shadow包？扫描情况下不会扫描到？其实很简单，假设你的这个类是动态生成，在容器实例化的时候不存在那么肯定不存在，再或者这个包下面有N多类但是只有一个类加了注解，那么其实你不需要去扫描，只需要添加这一个加了注解的类即可，再或者一个类是你和第三方系统交互后得到的。那么这个时候我们可以把这个类通过AnnotatedBeanDefinitionReader的register(Class clazz)方法把一个带注解的类注册给spring
2. 可以代替ClassPathBeanDefinitionScanner这个类.ClassPathBeanDefinitionScanner是spring完成扫描的核心类
3. spring完成扫描主要是依靠ClassPathBeanDefinitionScanner这个类的对象，但是AnnotatedBeanDefinitionReader可以替代他完成相同的注解解析
   1. 通过ClassPathBeanDefinitionScanner扫描出来的类A和通过AnnotatedBeanDefinitionReader显示注册的类A在spring内部会一套相同的解析规则
4. AnnotatedBeanDefinitionReader注册我们的**配置类**(配置类就是那个加了@Configuration和@ComponentScan的那个类)
   1. 为什么配置类需要手动注册呢？
      1. spring完成扫描是需要解析Appconfig.java当中的@ComponentScan注解的值（一般是一个包名），得到这个值之后去扫描这个值所代表的包下面的所有bean
      2. spring得到Appconfig.class之后把他解析成BeanDefintion对象，继而去获取@ComponentScan的值然后才开始扫描其他bean





## 4.2 总结

AnnotatedBeanDefinitionReader 作用：

1. 主要是可以动态、显示的注册一个bean
2. 具备解析一个类的功能；和扫描解析一个类的功能相同



使用场景：

1. 可以显示、动态注册一个程序员提供的bean
2. 在初始化spring容器的过程中他完成了对配置类的注册和解析功能；



初始化spring容器的时候代码有很多种写法

~~~java
AnnotationConfigApplicationContext ac =
				new AnnotationConfigApplicationContext();
				
		//动态注册一个配置类
		ac.register(Appconfig.class);
		
		//调用refresh方法
		//这里很多资料都叫做刷新spring容器
		//但是我觉得不合适，这是硬核翻译，比较生硬
		//笔者觉得理解为初始化spring容器更加精准
		//至于为什么以后慢慢更新再说
		ac.refresh();

~~~



~~~java
AnnotationConfigApplicationContext ac =
		new AnnotationConfigApplicationContext(Appconfig.class);
~~~



1. 这两种写法都初始化spring容器
2. 代码上的区别无非就是第一种写法是调用`AnnotationConfigApplicationContext()；`的无参构造方法，第二种写法是调用了`AnnotationConfigApplicationContext（Class<?>... annotatedClasses）`有参构造方法；



推荐使用第一种写法：

1. 首先第二种方法是在spring容器完成初始化之后的到的ac对象，容器已经初始化了，这个时候得到这个对象能干了事情少了很多；
2. 第一种方法在初始化之前得到的，那么能干的事情可多了
   1. 比如我们可以在容器初始化之前动态注册一个自己的bean，就是上文提到的 AnnotatedBeanDefinitionReader的应用场景
   2. 再比如可以利用ac对象来关闭或者开启spring的循环依赖
   3. 可以在容器初始化之前注册自己实例化的BeanDefinition对象



































































































































































