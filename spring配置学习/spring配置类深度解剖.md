# 一、版本约定

本文内容若没做特殊说明，均基于以下版本：

- JDK：`1.8`
- Spring Framework：`5.2.2.RELEASE`



# 二、相关类

- `@Configuration`：标注在类上，表示该类是个Full模式的配置类

  - 自Spring 5.2.0版本后它加了个`proxyBeanMethods`属性来显示控制Full模式还是Lite模式，默认是true表示Full模式

    

- `@Bean`：标注在方法上，表示方法生成一个由Spring容器管理的Bean

  

- `ConfigurationClassPostProcessor`：用于引导处理@Configuration配置类的后置处理器。注意：它只是引导处理，并不是实际处理

  

- `ConfigurationClassUtils`：内部工具类。用于判断组件是否是配置类，又或是Full模式/Lite模式，然后在bd元数据里打上标记

  - 它还会处理一件小事：获取@Configuration配置类上标注的@Order排序值并放进bd里

    

- `BeanMethod`：内部使用的类。用于封装标注有@Bean注解的方法

  

- `ConfigurationClass`：内部使用的类。每一个@Configuration配置类都会被封装为它，内部会包含多个@Bean方法（BeanMethod）

  

- `ConfigurationClassParser`：解析@Configuration配置类，最终以ConfigurationClass对象的形式展示，并且填充它：因为一个配置类可以`@Import`导入另外一个（或者N多个）其它配置类，所以需要填充

  

- `ConfigurationClassBeanDefinitionReader`：内部使用的类。读取给定的已经解析好的`Set<ConfigurationClass>`集合，把里面的bd信息注册到`BeanDefinitionRegistry`里去（这里决定了bd的有序和无序相关问题）

  

- `ConfigurationClassEnhancer`：内部使用的类。配置类增强器，用于对@Configuration类（Full模式）使用CGLIB增强，生成一个代理子类字节码Class对象

  

- `EnhancedConfiguration`：被增强器增强过的配置类，都会自动的让实现此接口（实际是个`BeanFactoryAware`）接口

  

- `SpringNamingPolicy`：使用CGLIB生成字节码类名名称生成策略 -> 名称中会有`BySpringCGLIB`字样

  

- `BeanFactoryAwareMethodInterceptor`：CGLIB代理对象拦截器。作用：拦截代理类的`setBeanFactory()`方法，给对应属性赋值

  

- `BeanMethodInterceptor`：CGLIB代理对象拦截器。作用：拦截所有@Bean方法的执行，以支持可以通过直接调用@Bean方法来管理依赖关系（当然也支持`FactoryBean`模式）



# 三、配置类解析流程图

1. **配置类的解析均是交由`ConfigurationClassPostProcessor`来引导**。

2. 在Spring Framework里（非Spring Boot）里，它是`BeanDefinitionRegistryPostProcessor`处理器的**唯一**实现类，用于引导处理`@Configuration`配置类。

3. 解析入口是`postProcessBeanDefinitionRegistry()`方法，实际处理委托给了`processConfigBeanDefinitions()`方法。



<img src="C:\Users\Admin\Desktop\spring源码\图片\20200523101507366.png" style="zoom:50%;" />







# 四、配置类增强流程图

如果一个配置类是Full模式，那么它就需要被CGLIB字节码提升。增强动作委托给`enhanceConfigurationClasses(beanFactory)`去完成。



<img src="C:\Users\Admin\Desktop\spring源码\图片\20200523103748156.png" style="zoom:50%;" />





# 五、生成增强子类字节码流程图

针对于Full模式配置类的字节码生成，委托给`ConfigurationClassEnhancer`增强器去完成，最终得到一个CGLIB提升过的子类Class**字节码对象**。

<img src="C:\Users\Admin\Desktop\spring源码\图片\20200523105718734.png" style="zoom:50%;" />



字节码实际是由Enhancer生成，就不用再深入了，那属于CGLIB（甚至ASM）的范畴，很容易头晕，也并无必要。



## 拦截器执行流程图

拦截器是完成增强**实际逻辑**的核心部件，因此它的执行流程**需要引起重视**。一共有两个“有用”的拦截器，分别画出。



# 六、BeanFactoryAwareMethodInterceptor拦截流程图

拦截`setBeanFactory()`方法的执行

<img src="C:\Users\Admin\Desktop\spring源码\图片\20200523112931198.png" style="zoom:50%;" />



### BeanMethodInterceptor拦截流程图

拦截`@Bean`方法的执行

<img src="C:\Users\Admin\Desktop\spring源码\图片\20200523112931198.png" style="zoom:50%;" />



### 配置类在Full模式下的“能力”展示

配置类（标注有@Configuration注解，属于Full模式）：

```java
@Configuration
public class AppConfig {
}
```

测试案例

```java
    public static void main(String[] args) {    

        ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
        AppConfig appConfig = context.getBean(AppConfig.class);
        System.out.println(appConfig.getClass());    
        System.out.println(appConfig.getClass().getSuperclass() == AppConfig.class);    		                 System.out.println(AopUtils.isCglibProxy(appConfig));

    }


=============输出结果===================================================================

class com.yourbatman.fullliteconfig.config.AppConfig$$EnhancerBySpringCGLIB$$d38ead10
true
false
```

结果解释：

1. Full模式的配置类被CGLIB增强了，所以最终放进容器内的实际是代理对象

2. 代理类是由CGLIB生成的子类，所以父类必然就是**目标类**

3. 这个为何是false？？？其实这个和`AopUtils.isCglibProxy()`的实现有关（建议你源码点进去瞄一眼一切都明白了），这个配置类仅仅是被CGLIB代理了，和AOP没毛关系































