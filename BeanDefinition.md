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









































































































































































