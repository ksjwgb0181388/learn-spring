# 一、扩展原理

## 1.1 BeanFactoryPostPorcessor

- BeanPostPorcessor：bean后置处理器，在bean创建对象初始化前后进行拦截操作

- BeanFactoryPostPorcessor：beanFactory的后置处理器

   在beanFactory标准初始化之后调用，所有的bean定义已经保存加载到beanFactory中，但是bean的实例还未创建
   
- 在refresh（）方法中的invokeBeanFactoryPostPorcessors(beanFactory)



### 1.执行时机

1. ioc容器创建对象

2. invokeBeanFactoryPostPorcessors（beanFactory）：执行BeanFactoryPostPorcessors

   ​	如何找到所有的BeanFactoryPostPorcessor并执行他们的方法？

   		1. 直接在beanFactory中找到所有类型是BeanFactoryPostPorcessor的组件，并执行他们的方法
   
     		2. 在初始化创建其他组件前面执行





## 1.2 BeanDefinitionRegistryPostProcessor

BeanDefinitionRegistryPostProcessor extends BeanFactoryPostPorcessor



接口方法：postPorcessBeanDefinitionRegistry（BeanDefinitionRegistry registry）

BeanDefinitionRegistry：bean定义的存储中心，以后beanFactory就是按照BeanDefinitionRegistry里面保存的每一个bean定义信息创建bean实例（单例还是多例，类型是什么。。。。）

也可以注册一个新的bean

~~~java
RootBeanDefinition root = new RootBeanDefinition();
registry.registerBeanDefinition("hello",root);
~~~

~~~java
AbstractBeanDefinition bean = 
BeanDefinitionBuilder.rootBeanDefinition(Blue.class).getBeanDefinition();

registry.registerBeanDefinition("hello",bean);
~~~









### 1. 执行时机

在所有bean定义信息将要被加载，bean实例还未创建 在BeanPostPorcessor之前

1. IOC容器创建

2. 调用refresh（）里面的invokeBeanFactoryPostPorcessors（beanFactroy）

3. 先从容器中获取所有的BeanDefinitionRegistryPostPorcessor组件。

   1. 依次触发所有的postPorcessBeanDefinitionRegistry（）
   2. 再触发possPorcessBeanFactroy（）

4. 再来从容器中找到BeanFactroyPostPorcessor组件，然后依次触发postPorcessBeanFactroy（）方法

   



## 1.3 ApplicationListener

监听容器中发布的事件，完成事件驱动开发



public interface ApplicationListener<E extends ApplicationEvent>{}

监听ApplicationEvent及其子事件



### 1. 基于事件开发

1. 写一个监听器来监听某个事件（ApplicationEvent下面的及其子类）

2. 将监听器加入到容器中

3. 只有有相应类型的发布，就能监听到这个事件

   ​	ContextRefreshEvent：容器刷新完成（所有bean都已经完全创建）会发布这个事件

4. 发布一个事件

   ~~~java
   public static void main(String[] args) {
           AnnotationConfigApplicationContext configApplicationContext = new AnnotationConfigApplicationContext();
   
           //发布一个事件
           configApplicationContext.publishEvent(new ApplicationEvent(new String()) {
           });
   
   }
   ~~~



~~~java
@Component
public class MyApplicationListener implements ApplicationListener<ApplicationEvent> {

    //当容器中发布此事件以后，方法会触发
    public void onApplicationEvent(ApplicationEvent applicationEvent) {
        System.out.println("收到事件"+applicationEvent);
    }
}
~~~



### 1.2 内部原理

1. ContextRefreshedEvent事件：

   1. 容器创建，调用refresh方法

   2. 调用finishRefresh（）：容器刷新完成执行

   3. 也可以自己发布事件，都会走下面的流程

      

      

      事件发布流程：

      - 调用pubilshEvent(new ContextRefreshedEvent（this）)

       1. 获取事件的多播器（派发器）：getApplicationEventMulticaster（）

       2. multicasterEvent（）：派发事件

       3. 获取到所有的ApplicationListener

          for（final ApplicationListener<?> listener ： getApplicationListeners(event,type)）

          1. 如果有Executor，可以支持使用Executor进行异步派发

          2. 否则使用同步方法，直接执行listener方法

             拿到listener回调onApplicationEvent方法



【事件多播器（ApplicationEventMulticaster）】

ApplicationEventMulticaster

1. 容器创建对象：refresh（）

2. 调用initApplicationEventMulticaster（）：初始化ApplicationEventMulticaster

   1. 先去容器找有没有id="applicationEventMulticaster"的组件；

   2. 如果没有创建一个SimpleApplicationEventMulticaster（beanFactroy）

      并且加入到容器中，就可以在其他组件要派发事件，自动注入这个applicationEventMultister



【容器中有哪些监听器】

- refresh（）方法中

- registerListener（）【check for listener beans and register them】

  - 从容器中拿到所有的监听器，把他们注册到applicationEventMulticaster中 

    //获取所有类型的监听器

    String [] listenerBeanNames = getBeanNamesForType(ApplicationListener.class,true,false)

    //将listener注册到ApplicationEventMulticaster中

    getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName)



## 1.4 @EventListener和SmartInitalizingSingleton

@EventListener：可以将业务层进行监听事件的发生

~~~Java
@Service
public class UserService{
    
    
    //定义一个监听方法，用来监听事件的发生
    @EventListener(classes = {ApplicationEvent.class})
    public void listener(ApplicationEvent event){
        sout(event);
    }
    
}
~~~



注解原理：利用EventListenerMethodProcessor来进行处理



SmartInitalizingSingleton：接口

方法：afterSingletonsInstantiated

​	会在单实例bean创建阶段完成之后



EventListenerMethodPorcessor处理器来解析@EventListener



### 1. 执行流程

1. ioc容器创建对象并refresh容器

2. finishBeanFactroyInitialzation(beanFactroy)：初始化剩下单实例bean

   1. 先创建所有的单实例bean；getBean（）

   2. 获取所有创建好的单实例bean，判断是否是SmartInitalizingSingleton类型的

      如果是就调用afterSingletonsInstantiated（）







# 二、spring容器源码

## 1.1 BeanFactroy预处理

1. prepareRefresh（） 刷新预处理

   1. initPropertySources（）：初始化一些属性设置；此方法是一个空实现。留给子类自定义属性个性化的属性设置
   2. getEnviroment().validateRequiredProperties()：校验属性的合法等
   3. earlyApplicationEvents = new LinkedHashSet<ApplicationEvent>：保存容器中早期的事件

2. obtainFreshBeanFactory()：获取beanFactroy

   1. refreshBeanFactroy：刷新beanFactroy。核心是创建了一个beanFactroy对象 

      创建了一个this.beanFactroy = new DefaultListableBeanFactory()

   2. getBeanFactory：返回刚才GenericApplicationContext创建的BeanFactory对象

   3. 将创建的BeanFactory返回 【默认DefaultListableBeanFactory】

3. prepareBeanFactory(beanFactory)：对BeanFactory的预准备工作（BeanFactroy进行一些设置）

   1. 设置BeanFactroy的类加载器、支持表达式..............

   2. 添加部分BeanPostPorcessor【ApplicationContextAwareProcessor】

   3. 设置忽略的自动装配的接口EnvironmentAware、EmbeddedValueResolverAware、xxxx

   4. 注册可以解析的自动装配；可以在任何组件中自动注入:BeanFactroy、ResourceLoader、ApplicationEventPublisher、ApplicationContext

   5. 添加BeanPostPorcessor【ApplicationListenerDetector】

   6. 添加编译时的AspectJ支持

   7. 给BeanFactroy中注册一些能用的组件；

      environment【ConfigurationEnvironment】

      systemProperties【Map<String,Object>】

      systemEnvironment【Map<String,Object>】

   ~~~java
   	protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
   		// Tell the internal bean factory to use the context's class loader etc.
   		beanFactory.setBeanClassLoader(getClassLoader());
   		beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
   		beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));
   
   		// Configure the bean factory with context callbacks.
   		beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
   		beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
   		beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
   		beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
   		beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
   		beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
   		beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
   
   		// BeanFactory interface not registered as resolvable type in a plain factory.
   		// MessageSource registered (and found for autowiring) as a bean.
   		beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
   		beanFactory.registerResolvableDependency(ResourceLoader.class, this);
   		beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
   		beanFactory.registerResolvableDependency(ApplicationContext.class, this);
   
   		// Register early post-processor for detecting inner beans as ApplicationListeners.
   		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));
   
   		// Detect a LoadTimeWeaver and prepare for weaving, if found.
   		if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
   			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
   			// Set a temporary ClassLoader for type matching.
   			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
   		}
   
   		// Register default environment beans.
   		if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
   			beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
   		}
   		if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
   			beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
   		}
   		if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
   			beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
   		}
   	}
   ~~~

   

4. postProcessBeanFactory(beanFactroy)：BeanFactroy准备工作完成后进行的后置处理工作
   
   1. 子类通过重写这个方法来在BeanFactroy创建并预准备完成以后做进一步的设置





## 1.2执行BeanFactoryPostPorcessor

1. invokeBeanFactroyPostProcessors(beanFactory)：执行beanFactroyPostPorcessor的方法

   beanFactroyPostPorcessor：BeanFactroy的后置处理器；在beanFactroy标准初始化之后执行的

   两个子接口：BeanFactroyProcessor、BeanDefinitionRegistryPostPorcessor

   执行顺序：

   1. PostProcessorRegistrationDelegate.invokeBeanFactoryPostPorcessors:执行BeanFactroyPorcessor方法

      先执行BeanDefinitionRegistryPostProcessor

      1. 获取所有的BeanDefinitionRegistryPostProcessor；

      2. 先执行实现了 PriorityOrdered优先级接口的BeanDefinitionRegistryPostPorcessor

         执行postProcessor.postProcessBeanDefinitionRegistry(registry)

      3. 再执行实现了Ordered顺序接口的BeanDefinitionRegistryPostProcessor

         执行postProcessor.postProcessBeanDefinitionRegistry(registry)

      4. 最后执行没有实现任何优先级顺序的BeanDefinitionRegistryPostProcessors；

         执行postProcessor.postProcessBeanDefinitionRegistry(registry)

   2. 再执行BeanFactoryPostProcessor的方法

      1. 获取所有的BeanDefinitionRegistryPostProcessor；

      2. 先执行实现了 PriorityOrdered优先级接口的BeanFactoryPostProcessor

         执行postProcessor.postProcessBeanDefinitionRegistry(registry)

      3. 再执行实现了Ordered顺序接口的BeanFactoryPostProcessor

         执行postProcessor.postProcessBeanDefinitionRegistry(registry)

      4. 最后执行没有实现任何优先级顺序的BeanFactoryPostProcessor；

         执行postProcessor.postProcessBeanDefinitionRegistry(registry)



## 1.3 注册BeanPostProcessor

作用：注册BeanPostProcessor（Bean的后置处理器）【intercept bean creation】

不同接口类型的BeanPostProcessor：在bean创建的前后执行时机是不一样的

​	BeanPostProcessor

​	DestructionAwareBeanPostProcessor（销毁方法的后置处理器）

​	InstantiationAwareBeanPostProcessor

​	SmartInstantiationAwareBeanPostProcessor

​	MergedBeanDefinitionPostProcessor【记录在internalPostProcessors】



1. 获取所有的 BeanPostProcessor；后置处理器都有优先级 PriorityOrdered、Ordered

2. 先注册PriorityOrdered；把每一个BeanPostProcessor添加到BeanFactroy中

   beanFactroy.addBeanPostProcessor(postProcessor)

3. 再注册实现了Ordered接口的

4. 最后注册，没有实现任何优先级接口的

5. 最终注册MergedBeanDefinitionPostProcessor；

6. 最终还注册一个ApplicationListenerDetector；来检查bean创建完成后，是否是ApplicationListener

   applicationContext.addApplicationListener((ApplicationListener<?>) bean)





## 1.4 initMessageSource

初始化messageSource组件（国际化功能，消息绑定和消息解析）

1. 获取BeanFactory

2. 看容器中是否有ID为messageSource组件

3. 如果有就拿来用，如果没有就拿一个默认的DelegatingMessageSource

   MessageSource：取出国际化配置文件中的某个key值；能按照区域信息获取

4. 把创建好的MessageSource注册到容器中，以后获取国际化配置文件值的时候，可以注入MessageSource中

   beanFactroy.registrySingleton(MESSAGE_SOURCE_BEAN_NAME，this.messageSource)

   调用 String getMessage（String code，Object[] args，String defaultMessage，Locale locale）





## 1.5 initApplicationEventMuticaster

初始化事件派发器

1. 获取BeanFactroy
2. 从BeanFactroy中获取applicationEventMulticaster的组件
3. 如果没有配置，就创建一个SimpleApplicationEventMulticaster
4. 将创建的ApplicationEventMulticaster注入到容器中，以后其他组件直接自动注入即可



## 1.6 onRefresh

作用：留给子容器的

1. 重写onRefresh方法，容器刷新的时候自定义逻辑



## 1.7 registerListeners

给容器中将所有项目的监听器ApplicationListener注册进来

1. 从容器中拿到所有的ApplicationListener组件

2. 将每个监听器添加到事件派发器

   getApplicationEventMulticaster.addApplicationListenerBean(listenerBeanName)

3. 派发之前步骤产生的事件；

   



## 1.8 finishBeanFactroyInitialization(beanFactroy) 重点！！！！！

初始化所有剩下的单实例bean

1. beanFactroy.preInstantiateSingletons()：初始化所有剩下的单实例Bean

   1. List<String> beanNames = new ArrayList<String>(this.beanDefinitionNames);获取所有的容器的bean，依次进行初始化和创建对象

   2. 拿到Bean的定义信息 RootBeanDefinition db = getMergedLocalBeanDefinition(beanName)

      Bean不是抽象的，是单实例的，是非懒加载的

   3. 判断是否是FactroyBean；是否是实现FactroyBean接口的Bean，如果是调用getObject方法

      如果不是工厂Bean，利用getBean创建对象

      getBean(beanName)的步骤

      1. 调用doGetBean

      2. 先获取缓存中保存的单实例bean，如果能获取到说明以前创建过。（所有创建的单实例Bean都会被缓存）

         从final Map<String,Object> singletonObjects = new ConcurrentHashMap<Stirng,Object>(256)获取的

      3. 缓存是空的。开始bean的创建流程

      4. 标记当前Bean已经被创建 markBeanAsCreated防止多线程同时操作

      5. 获取Bean的定义信息 

         final RootBeanDefinition mdb = getMergedLocalBeanDefinition(beanName)

      6. 获取当前Bean依赖的其他Bean

         1. 什么叫依赖bean

            ```xml
            <bean id="person" class="com.atguigu.bean.Person" depends-on="user,book">
            
            决定创建顺序
            ```

      7. 启动单实例Bean的创建流程

         1. 调用createBean(beanName,mbd,args)

         ~~~java
         sharedInstance = getSingleton(beanName,new ObjectFactroy<Object>){
             @Override
             public Object getObject throws BeansException{
                 try{
                     return createBean(beanName,mbd,args);
                 }
                 catch(BeanException ex){
                     destroySingleton(beanName);
                     throw ex;
                 }
             }
         }
         ~~~

         2. Object bean = resolveBeforeInstantiation(beanName,mbdToUse)

            给BeanPostProcessors 一个机会 返回一个bean代理对象

            是InstantiationAwareBeanPostProcessor：提前执行的逻辑

            先触发：postProcessBeforeInstantiation（）

            如果有返回值：触发postProcessAfterInitialization（）

         3. 如果前面的InstantiationAwareBeanPostProcessor没有返回代理对象

         4. Object beanInstance = doCreateBean（beanName，mbdToUse，args）

            1. 【**创建Bean实例**】，createBeanInstance（beanName，mbd，args ）

               利用工厂方法或者对象的构造器创建出Bean实例

            2. applyMergedBeanDefinitionPostProcessors（mbd，beanType，beanName）

               调用MergedBeanDefinitionPostProcessor的postProcessMergedBeanDefinition方法

            3. 【**为bean的属性赋值**】populateBean（beanName，mbd，instanceWrapper）：

               赋值之前

               1. 拿到InstantiationAwareBeanPostProcessor后置处理器；

                  执行postProcessAfterInstantiation（）

               2. 再拿到InstantiationAwareBeanPostProcessor后置处理器；

                  执行postProcessPropertyValues（）

               3. **applyPropertyValues**（beanName，mbd，bw，pvs）：为属性setter方法等赋值

            4. 【**Bean初始化**】initializeBean（beanName，exposedObject，mbd）

               1. 【执行Aware接口】invokeAwareMethods（beanName，bean）：执行xxxAware接口的方法

                  BeanNameAware、BeanClassLoaderAware、BeanFactroyAware

               2. 【后置处理器初始化之前】applyBeanPostProcessorsBeforeInitialization（wrappedBean，beanName） 

                  BeanPostPorcessor.postPorcessBeforeInitialization（）

               3. 【执行初始化方法】invokeInitMethods（beanName，wrappedBean，mbd）

                  1. 是否是InitializingBean接口的实现；执行接口规定的初始化
                  2. 是否自定义初始化方法

               4. 【执行初始化之后方法】applyBeanPostProcessorsAfterInitialization

                  BeanPostProcessor.postProcessAfterInitialization（）

            5. 注册Bean的销毁方法

         5. 将创建的Bean添加到缓存中singletonObjects；

            ioc容器就是这些Map；很多的Map保存了单实例bean，环境信息。。。。

   4. 所有的Bean都利用getBean创建完成以后

      检查所有的Bean是否是SmartInitializingSingleton接口的；

      如果是就执行 **afterSingletonsInstantiated**方法



## 1.9 容器创建完成 finishRefresh

完成BeanFactroy的初始化创建工作；IOC容器彻底创建完成





1. initLifecycleProcessor（）；初始化生命周期有关的后置处理器；LifecycleProcessor

   默认从容器中找是否有lifecycleProcessor的组件【LifecycleProcessor】；

   如果没有使用默认的 new DefaultLifecycleProcessor（）

   写一个LifecycleProcessor的实现类，可以在BeanFactroy刷新完成或关闭之后进行一些调用。

   void onRefresh（）、void onClose（）

2. getLifecycleProcessor.onRefresh()：拿到前面定义的生命周期处理器（监听BeanFactroy）；回调onRefresh

3. publishEvent(new ContextRefreshedEvent(this))：发布容器刷新完成事件

4. LiveBeanView.registerApplicationContext(this)





# 三、Spring元数据Metadata的使用

注解编程之**AnnotationMetadata**，**ClassMetadata**、**MetadataReaderFactory**

### 前言

`Spring`在2.0的时候就支持了基于`XML Schema`的扩展机制，让我们可以自定义的对xml配置文件进行扩展（四大步骤，有兴趣的可以自己学习），比如鼎鼎大名的`Dubbo`它就扩展了xml，用它来引入服务或者导出服务。
随着`Spring3.0+`的发展，xml慢慢的淡出了我们的视野，特别是`Spring Boot`的流行让xml彻底消失，所有的xml配置都使用注解的方式进行了代替。



### 元数据

**元数据：数据的数据**。比如**Class**就是一种元数据。`Metadata`在`org.springframework.core.type`包名下，还有用于读取的子包`classreading`也是重要知识点。此体系大致的类结构列出如下图：

![](C:\Users\Admin\Desktop\spring源码\图片\20190323182923947.png)



![20190323183004773](C:\Users\Admin\Desktop\spring源码\图片\20190323183004773.png)



![](C:\Users\Admin\Desktop\spring源码\图片\20190323183029887.png)



可以看到顶层接口有两个：`ClassMetadata`和`AnnotatedTypeMetadata`

**注解上的注解**，`Spring`将其定义为元注解(`meta-annotation`),如 `@Component`标注在 `@Service`上，`@Component`就被称作为元注解。后面我们就将注解的注解称为元注解。



### ClassMetadata：对`Class`的抽象和适配

此接口的所有方法，基本上都跟Class有关。

```java
// @since 2.5
public interface ClassMetadata {

	// 返回类名（注意返回的是最原始的那个className）
	String getClassName();
	boolean isInterface();
	// 是否是注解
	boolean isAnnotation();
	boolean isAbstract();
	// 是否允许创建  不是接口且不是抽象类  这里就返回true了
	boolean isConcrete();
	boolean isFinal();
	// 是否是独立的(能够创建对象的)  比如是Class、或者内部类、静态内部类
	boolean isIndependent();
	// 是否有内部类之类的东东
	boolean hasEnclosingClass();
	@Nullable
	String getEnclosingClassName();
	boolean hasSuperClass();
	@Nullable
	String getSuperClassName();
	// 会把实现的所有接口名称都返回  具体依赖于Class#getSuperclass
	String[] getInterfaceNames();

	// 基于：Class#getDeclaredClasses  返回类中定义的公共、私有、保护的内部类
	String[] getMemberClassNames();
}

```



![](C:\Users\Admin\Desktop\spring源码\图片\2019100718182382.png)





### StandardClassMetadata

基于Java标准的（Standard）反射实现元数据的获取。

```java
// @since 2.5
public class StandardClassMetadata implements ClassMetadata {
	// 用于内省的Class类
	private final Class<?> introspectedClass;
	
	// 唯一构造函数：传进来的Class，作为内部的内省对象
	public StandardClassMetadata(Class<?> introspectedClass) {
		Assert.notNull(introspectedClass, "Class must not be null");
		this.introspectedClass = introspectedClass;
	}
	... // 后面所有的方法实现，都是基于introspectedClass,类似代理模式。举例如下：

	public final Class<?> getIntrospectedClass() {
		return this.introspectedClass;
	}
	@Override
	public boolean isInterface() {
		return this.introspectedClass.isInterface();
	}
	@Override
	public String[] getMemberClassNames() {
		LinkedHashSet<String> memberClassNames = new LinkedHashSet<>(4);
		for (Class<?> nestedClass : this.introspectedClass.getDeclaredClasses()) {
			memberClassNames.add(nestedClass.getName());
		}
		return StringUtils.toStringArray(memberClassNames);
	}
}

```

它有个非常重要的子类：`StandardAnnotationMetadata`它和注解密切相关，在文章下半部分重点分析。

### MethodsMetadata：新增访问方法元数据的接口

它是子接口，主要增加了从Class里获取到`MethodMetadata`们的方法：

```java
// @since 2.1 可以看到它出现得更早一些
public interface MethodsMetadata extends ClassMetadata {
	// 返回该class所有的方法
	Set<MethodMetadata> getMethods();
	// 方法指定方法名的方法们（因为有重载嘛~）
	Set<MethodMetadata> getMethods(String name);
}
```

名字上请不要和`MethodMetadata`搞混了，`MethodMetadata`是`AnnotatedTypeMetadata`的子接口，代表具体某一个Type（方法上的注解）；而此类是个`ClassMetadata`，它能获取到本类里所有的方法Method（MethodMetadata）~



### AnnotatedTypeMetadata：对注解元素的封装适配

什么叫注解元素(`AnnotatedElement`)？比如我们常见的`Class、Method、Constructor、Parameter`等等都属于它的子类都属于注解元素。简单理解：只要能在上面标注注解都属于这种元素。`Spring4.0`新增的这个接口提供了对注解统一的、便捷的访问，使用起来更加的方便高效了。

```java
// @since 4.0
public interface AnnotatedTypeMetadata {

	// 此元素是否标注有此注解~~~~
	// annotationName：注解全类名
	boolean isAnnotated(String annotationName);
	
	// 这个就厉害了：取得指定类型注解的所有的属性 - 值（k-v）
	// annotationName：注解全类名
	// classValuesAsString：若是true表示 Class用它的字符串的全类名来表示。这样可以避免Class被提前加载
	@Nullable
	Map<String, Object> getAnnotationAttributes(String annotationName);
	@Nullable
	Map<String, Object> getAnnotationAttributes(String annotationName, boolean classValuesAsString);

	// 参见这个方法的含义：AnnotatedElementUtils.getAllAnnotationAttributes
	@Nullable
	MultiValueMap<String, Object> getAllAnnotationAttributes(String annotationName);
	@Nullable
	MultiValueMap<String, Object> getAllAnnotationAttributes(String annotationName, boolean classValuesAsString);
}

```

它的继承树如下：

![](C:\Users\Admin\Desktop\spring源码\图片\20191007195953382.png)



两个子接口相应的都提供了标准实现以及基于`ASM`的`Visitor`模式实现。

> ASM 是一个通用的 Java 字节码操作和分析框架。它可以用于修改现有类或直接以二进制形式动态生成类。 ASM 虽然提供与其他 Java 字节码框架如 Javassist，CGLIB类似的功能，但是其设计与实现小而快，且性能足够高。

> Spring 直接将 ASM 框架核心源码内嵌于 Spring-core中，目前`Spring 5.1使用ASM 7 版本。



### MethodMetadata：方法描述

子接口，用来描述`java.lang.reflect.Method`。

```java
// @since 3.0
public interface MethodMetadata extends AnnotatedTypeMetadata {
	String getMethodName();
	String getDeclaringClassName();
	String getReturnTypeName();
	boolean isAbstract();
	boolean isStatic();
	boolean isFinal();
	boolean isOverridable();
}

```

##### StandardMethodMetadata

基于反射的标准实现，略



##### MethodMetadataReadingVisitor

基于`ASM`的实现的，继承自`ASM``的org.springframework.asm.MethodVisitor`采用`Visitor`的方式读取到元数据。

```java
// @since 3.0
public class MethodMetadataReadingVisitor extends MethodVisitor implements MethodMetadata {
	...
}
```



### `AnnotationMetadata`（重要）

这是理解`Spring`注解编程的必备知识，它是`ClassMetadata`和`AnnotatedTypeMetadata`的子接口，具有两者共同能力，并且新增了访问注解的相关方法。可以简单理解为它是对注解的抽象。

> 经常这么使用得到注解里面所有的属性值：
> `AnnotationAttributes attributes = AnnotationConfigUtils.attributesFor(annoMetadata, annType);`



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

同样的它提供了两种实现方式。



#### StandardAnnotationMetadata

继承了`StandardClassMetadata`，很明显关于`ClassMetadata`的实现部分就交给此父类了，自己只关注于`AnnotationMetadata`接口的实现。

```java
// @since 2.5
public class StandardAnnotationMetadata extends StandardClassMetadata implements AnnotationMetadata {
	
	// 很显然它是基于标准反射类型：java.lang.annotation.Annotation
	// this.annotations = introspectedClass.getAnnotations()
	private final Annotation[] annotations;
	private final boolean nestedAnnotationsAsMap;
	...

	
	// 获取本Class类上的注解的元注解们
	@Override
	public Set<String> getMetaAnnotationTypes(String annotationName) {
		return (this.annotations.length > 0 ?
				AnnotatedElementUtils.getMetaAnnotationTypes(getIntrospectedClass(), annotationName) : Collections.emptySet());
	}

	@Override
	public boolean hasAnnotation(String annotationName) {
		for (Annotation ann : this.annotations) {
			if (ann.annotationType().getName().equals(annotationName)) {
				return true;
			}
		}
		return false;
	}
	...
}

```



#### AnnotationMetadataReadingVisitor

继承自`ClassMetadataReadingVisitor`，同样的`ClassMetadata`部分实现交给了它。

> 说明：`ClassMetadataReadingVisitor`是`org.springframework.core.type.classreading`包下的类，同包的还有我下面重点讲述的`MetadataReader`。此实现类最终委托给`AnnotationMetadataReadingVisitor`来做的，而它便是`ClassMetadataReadingVisitor`的子类（`MetadataReader`的底层实现就是它，使用的ASM的`ClassVisitor`模式读取元数据）。



~~~java
// @since 2.5
public class AnnotationMetadataReadingVisitor extends ClassMetadataReadingVisitor implements AnnotationMetadata {
	...
}
~~~









# 总结

1. spring容器在启动的时候，先会保存所有注册进来的Bean的定义信息；

   1. xml注册Bean<bean>
   2. 注解注册Bean @Service @Component @Bean

2. spring容器会在合适的时机创建这些bean

   1. 用到Bean的时候；利用getBean创建Bean，创建好以后保存到容器中
   2. 统一创建剩下所有Bean的时候；finishBeanFactroyInitialization（）

3. 后置处理器（BeanPostProcessor）

   1. 每一个Bean创建完成都会使用各种后置处理器进行处理；增强Bean的功能

      AutowiredAnnotationBeanPostProcessor：处理自动注入功能

      AnnotationAwareAspectJAutoProxyCreator：来做AOP功能

4. 事件驱动模型

   ApplicationListener：事件监听

   ApplicationEventMulticaster；事件派发











































































































