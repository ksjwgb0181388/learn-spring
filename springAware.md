# 一、接口继承关系

![](C:\Users\Admin\Desktop\learn-spring\图片\3397380-6ef519bbc705ce28.webp)



先举个`BeanNameAware`的例子，实现`BeanNameAware`接口，可以让该`Bean`感知到自身的`BeanName`

`BeanNameAware`接口的定义

~~~~java
public interface BeanNameAware extends Aware {
      void setBeanName(String name);
}
~~~~



# 二、spring源码分析Aware子类的使用场景

1. 在spring容器初始化过程中，会执行AbstractApplicationContext类的prepareBeanFactory方法，这里面会创建一个bean后置处理器ApplicationContextAwareProcessor，如下图红框所示：

   ![](C:\Users\Admin\Desktop\learn-spring\图片\20180813060028561.jpg)



2. 在bean被初始化之前，所有的bean后置处理器的postProcessBeforeInitialization方法都会被执行，如下图红框所示：

   ![](C:\Users\Admin\Desktop\learn-spring\图片\20180813061248409.jpg)



3. 由以上两步可以确定：对于每个bean后置处理器来说，它的postProcessBeforeInitialization方法会在每个bean的初始化之前被调用一次；



4. 来看看ApplicationContextAwareProcessor类的postProcessBeforeInitialization方法，按照前面的分析，该方法在每个bean被初始化之前都会被执行，如下图红框所示，invokeAwareInterfaces方法会被调用，这是我们要关注的重点：





~~~java
private void invokeAwareInterfaces(Object bean) {
        if (bean instanceof Aware) {
            if (bean instanceof EnvironmentAware) {
                ((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
            }
            if (bean instanceof EmbeddedValueResolverAware) {
                ((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(
                        new EmbeddedValueResolver(this.applicationContext.getBeanFactory()));
            }
            if (bean instanceof ResourceLoaderAware) {
                ((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
            }
            if (bean instanceof ApplicationEventPublisherAware) {
                ((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
            }
            if (bean instanceof MessageSourceAware) {
                ((MessageSourceAware) bean).setMessageSource(this.applicationContext);
            }
            if (bean instanceof ApplicationContextAware) {
                ((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
            }
        }
    }
~~~



从上述代码可以看出，如果当前的bean实现了某个接口，那么它的某个对应的方法就会被调用，例如我们创建了一个bean实现了ApplicationContextAware接口，那么这个bean的setApplicationContext方法就会被调用，入参是applicationContext成员变量，这样我们的bean就能得到applicationContext对象了；

以上就是Aware的接口使用原理：业务按需要实现特定的Aware接口，spring容器会主动找到该bean，然后调用特定的方法，将特定的参数传递给bean；



# 三、BeanNameAware接口的调用场景

BeanNameAware也是Aware的子接口，不过它的调用场景和前面分析的几个Aware子接口不同，并未出现在ApplicationContextAwareProcessor类的invokeAwareInterfaces方法中，我们来看看它是如何被调用的；

1. 如下图所示，红框中就是BeanNameAware接口被调用的地方，而绿框中的applyBeanPostProcessorsBeforeInitialization方法就是前面我们分析的那些Aware子接口被调用的位置：

![](C:\Users\Admin\Desktop\learn-spring\图片\20180813065411978.jpg)



2. 方法invokeAwareMethods如下所示，和前面的讨论一样，特定类型的bean，其特定的方法被调用，传入特定的入参：

   ~~~java
   private void invokeAwareMethods(final String beanName, final Object bean) {
           if (bean instanceof Aware) {
               if (bean instanceof BeanNameAware) {
                   ((BeanNameAware) bean).setBeanName(beanName);
               }
               if (bean instanceof BeanClassLoaderAware) {
                   ((BeanClassLoaderAware) bean).setBeanClassLoader(getBeanClassLoader());
               }
               if (bean instanceof BeanFactoryAware) {
                   ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
               }
           }
       }
   ~~~

   

# 四、各个Aware的作用

## 4.1 ImportAware应用

**该接口同样也是需要配合@Import注解进行使用**，其主要作用就是配合@Enable××通过开关的形式开启某个功能时进行各项属性值的初始化工作。

其中比较典型的应用场景就是@EnableRedissonHttpSession

 ~~~java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
@Import({RedissonHttpSessionConfiguration.class})
@Configuration
public @interface EnableRedissonHttpSession {
    int maxInactiveIntervalInSeconds() default 1800;
 
    String keyPrefix() default "";
}


 ~~~

只要我们开启了RedissonHttpSession功能，spring就会自动导入RedissonHttpSessionConfiguration.class；该注解中提供了两个参数：

1. maxInactiveIntervalInSeconds：会话超时时间
2. keyPrefix：key的前缀



其最终就是在RedissonHttpSessionConfiguration中处理，并应用在配置类中：

~~~java
@Configuration
public class RedissonHttpSessionConfiguration extends SpringHttpSessionConfiguration implements ImportAware {
    private Integer maxInactiveIntervalInSeconds;
    private String keyPrefix;
 
...
    public void setImportMetadata(AnnotationMetadata importMetadata) {
        Map<String, Object> map = importMetadata.getAnnotationAttributes(EnableRedissonHttpSession.class.getName());
        AnnotationAttributes attrs = AnnotationAttributes.fromMap(map);
        this.keyPrefix = attrs.getString("keyPrefix");
        this.maxInactiveIntervalInSeconds = (Integer)attrs.getNumber("maxInactiveIntervalInSeconds");
    }
}
~~~

比如需要开发自己的插件，整合到spring时就可以基于这种模式，如下简单模拟下：

自定一个插件配置类：

~~~java
@Component
public class MyDb implements ImportAware {


    private int maxConnections;


    @Override
    public void setImportMetadata(AnnotationMetadata annotationMetadata) {
        Map<String, Object> attributesMap = annotationMetadata.getAnnotationAttributes(EnableMyDb.class.getName());
        AnnotationAttributes attrs = AnnotationAttributes.fromMap(attributesMap);
        this.maxConnections = attrs.getNumber("maxConnections");
        System.out.println(this.maxConnections);
    }


    public void store(){
        System.out.println(this.maxConnections);
    }
~~~

~~~java
@Retention(RetentionPolicy.RUNTIME)
@Import(MyDb.class)
public @interface EnableMyDb {
    int maxConnections() default 1000;
}
~~~



























































































