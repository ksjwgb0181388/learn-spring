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





# 三、总结

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











































































































