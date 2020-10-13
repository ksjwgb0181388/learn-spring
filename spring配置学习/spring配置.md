# 一、基本概念

官方管这两种模式分别叫：

`Full @Configuration`和`lite @Bean mode`，口语上我习惯把它称为Spring配置的Full模式和Lite模式更易沟通。





# 二、@Configuration和@Bean

Spring新的配置体系中最为重要的构件是：`@Configuration`标注的类，`@Bean`标注的方法。

~~~java
// @since 3.0
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Configuration {

	@AliasFor(annotation = Component.class)
	String value() default "";
	// @since 5.2
	boolean proxyBeanMethods() default true;
	
}
~~~



用`@Configuration`注解标注的类表明其主要目的是作为bean定义的**源**。此外，`@Configuration`类允许通过调用同一类中的其他`@Bean` method方法来定义bean之间的依赖关系（下有详解）。

~~~java
// @since 3.0
@Target({ElementType.METHOD, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Bean {

	@AliasFor("name")
	String[] value() default {};
	@AliasFor("value")
	String[] name() default {};
	@Deprecated
	Autowire autowire() default Autowire.NO;
	// @since 5.1
	boolean autowireCandidate() default true;
	String initMethod() default "";
	String destroyMethod() default AbstractBeanDefinition.INFER_METHOD;
	
}
~~~

`@Bean`注解标注在方法上，用于指示方法实例化、配置和初始化要由Spring IoC容器管理的新对象。对于熟悉Spring的`<beans/>`XML配置的人来说，`@Bean`注解的作用与`<bean/>`元素相同。您可以对任何Spring的@Component组件使用`@Bean`注释的方法代替（注意：这是理论上，实际上比如使用@Controller标注的组件就不能直接使用它代替）。

简单粗暴理解：`@Configuration`标注的类等同于一个xml文件，`@Bean`标注的方法等同于xml文件里的一个`<bean/>`标签



## 2.1 Full模式和Lite模式

**Full模式和Lite模式均是针对于Spring配置类而言的**，和xml配置文件无关。值得注意的是：判断是Full模式 or Lite模式的前提是，首先你得是个容器组件。至于一个实例是如何“晋升”成为容器组件的，可以用注解也可以没有注解，本文就不展开讨论了，这属于Spring的基础知识。



## 2.2 Lite模式

当`@Bean`方法在没有使用`@Configuration`注释的类中声明时，它们被称为**在Lite模式下处理**。它包括：在`@Component`中声明的`@Bean`方法，甚至只是在一个非常普通的类中声明的Bean方法，都被认为是Lite版的配置类。`@Bean`方法是一种通用的工厂方法（`factory-method`）机制。

和Full模式的`@Configuration`不同，Lite模式的`@Bean`方法**不能声明Bean之间的依赖关系**。因此，这样的`@Bean`方法**不应该调用其他@Bean方法**。每个这样的方法实际上**只是一个特定Bean引用的工厂方法(factory-method)**，没有任何特殊的运行时语义。



### 何时为Lite模式

官方定义为：在没有标注`@Configuration`的类里面有`@Bean`方法就称为Lite模式的配置。透过源码再看这个定义是不完全正确的，而应该是有如下case均认为是Lite模式的配置类：



1. 类上标注有`@Component`注解
2. 类上标注有`@ComponentScan`注解
3. 类上标注有`@Import`注解
4. 类上标注有`@ImportResource`注解
5. **若类上没有任何注解**，但类内存在@Bean方法



标注有`@Configuration(proxyBeanMethods = false)`，注意：此值默认是true哦，需要显示改为false才算是Lite模式



细心的你会发现，自Spring5.2（对应Spring Boot 2.2.0）开始，内置的几乎所有的`@Configuration`配置类都被修改为了`@Configuration(proxyBeanMethods = false)`，目的何为？答：以此来降低启动时间，为Cloud Native继续做准备。



### 优缺点

**优点**：

- 运行时不再需要给对应类生成CGLIB子类，提高了运行性能，降低了启动时间
- 可以该配置类当作一个普通类使用喽：也就是说@Bean方法 **可以是private、可以是final**

**缺点**：

- 不能声明@Bean之间的依赖，也就是说不能通过方法调用来依赖其它Bean
- （其实这个缺点还好，很容易用其它方式“弥补”，比如：把依赖Bean放进方法入参里即可）



### 小总结

- 该模式下，配置类本身不会被CGLIB增强，放进IoC容器内的就是本尊
- 该模式下，对于内部类是没有限制的：可以是Full模式或者Lite模式
- 该模式下，配置类内部**不能通过方法调用**来处理依赖，否则每次生成的都是一个新实例而并非IoC容器内的单例
- 该模式下，配置类就是一普通类嘛，所以@Bean方法可以使用`private/final`等进行修饰（static自然也是阔仪的）



## 2.3 Full模式

在常见的场景中，`@Bean`方法都会在标注有`@Configuration`的类中声明，以确保总是使用“Full模式”，这么一来，交叉方法引用会被重定向到容器的生命周期管理，所以就可以更方便的管理Bean依赖。



### 何时为Full模式

标注有`@Configuration`注解的类被称为full模式的配置类。自Spring5.2后这句话改为下面这样我觉得更为精确些：

- 标注有`@Configuration`或者`@Configuration(proxyBeanMethods = true)`的类被称为Full模式的配置类
- （当然喽，proxyBeanMethods属性的默认值是true，所以一般需要Full模式我们只需要标个注解即可）

### 优缺点

**优点**：

- 可以支持通过常规Java调用相同类的@Bean方法而保证是容器内的Bean，这有效规避了在“Lite模式”下操作时难以跟踪的细微错误。特别对于萌新程序员，这个特点很有意义

**缺点**：

- 运行时会给该类生成一个CGLIB子类放进容器，有一定的性能、时间开销（这个开销在Spring Boot这种拥有大量配置类的情况下是不容忽视的，这也是为何Spring 5.2新增了`proxyBeanMethods`属性的最直接原因）
- 正因为被代理了，所以@Bean方法 **不可以是private、不可以是final**



### 小总结

- 该模式下，配置类会被CGLIB增强(生成代理对象)，放进IoC容器内的是代理
- 该模式下，对于内部类是没有限制的：可以是Full模式或者Lite模式
- 该模式下，配置类内部**可以通过方法调用**来处理依赖，并且能够保证是同一个实例，都指向IoC内的那个单例
- 该模式下，@Bean方法不能被`private/final`等进行修饰（很简单，因为方法需要被复写嘛，所以不能私有和final。defualt/protected/public都可以哦），否则启动报错（其实IDEA编译器在编译器就提示可以提示你了）：



## 2.4 使用建议

了解了Spring配置类的Full模式和Lite模式，那么在工作中我该如何使用呢？这里A哥给出使用建议，仅供参考：

- 如果是在公司的业务功能/服务上做开发，使用Full模式
- 如果你是个容器开发者，或者你在开发中间件、通用组件等，那么使用Lite模式是一种更被推荐的方式，它对Cloud Native更为友好



# 三、源码分析

判断一个Bean是否是**Bean的后置处理器**很方便，只需看它是否实现了`BeanPostProcessor`接口即可；

那么如何去确定一个Bean是否是@Configuration配置Bean呢？若是，如何区分是Full模式还是Lite模式呢？这便就是本文将要讨论的内容。



## 3.1 如何判断一个组件是否是@Configuration配置？

1. 首先需要明确：`@Configuration`配置前提必须是IoC管理的一个组件（也就是常说的Bean）。

2. Spring使用***BeanDefinitionRegistry***注册中心管理着所有的Bean定义信息，那么对于这些Bean信息哪些属于`@Configuration`配置呢，这是需要甄选出来的。

3. 判断一个Bean是否是`@Configuration`配置类这个逻辑统一交由**ConfigurationClassUtils**这个工具类去完成。



## 3.2 ConfigurationClassUtils工具类

见名之意，它是和配置有关的一个工具类，提供几个静态工具方法供以使用。

它是`Spring 3.1`新增，对于它的作用，官方给的解释是：**用于标识`@Configuration`类的实用程序(Utilities)。**

它主要提供了一个方法：`checkConfigurationClassCandidate()`用于检查给定的Bean定义是否是配置类的候选对象（或者在配置/组件类中声明的嵌套组件类），**并做相应的标记**。



### checkConfigurationClassCandidate()

~~~java
ConfigurationClassUtils：

	public static boolean checkConfigurationClassCandidate(BeanDefinition beanDef, MetadataReaderFactory metadataReaderFactory) {
		...
		// 根据Bean定义信息，拿到器对应的注解元数据
		AnnotationMetadata metadata = xxx;
		...
		
		// 根据注解元数据判断该Bean定义是否是配置类。若是：那是Full模式还是Lite模式
		Map<String, Object> config = metadata.getAnnotationAttributes(Configuration.class.getName());
		if (config != null && !Boolean.FALSE.equals(config.get("proxyBeanMethods"))) {
			beanDef.setAttribute(CONFIGURATION_CLASS_ATTRIBUTE, CONFIGURATION_CLASS_FULL);
		} else if (config != null || isConfigurationCandidate(metadata)) {
			beanDef.setAttribute(CONFIGURATION_CLASS_ATTRIBUTE, CONFIGURATION_CLASS_LITE);
		} else {
			return false;
		}
		
		...
		
		// 到这。它肯定是一个完整配置（Full or Lite） 这里进一步把@Order排序值放上去
		Integer order = getOrder(metadata);
		if (order != null) {
			beanDef.setAttribute(ORDER_ATTRIBUTE, order);
		}

		return true;
	}
~~~



步骤总结：

1. 根据Bean定义信息解析成为一个注解元数据对象 **AnnotationMetadata metadata**

   1. 可能是个`AnnotatedBeanDefinition`，也可能是个`StandardAnnotationMetadata`

2. 根据注解元数据metadata判断是否是个 **@Configuration**

   配置类，有如下三种可能case：

   1. 标注有`@Configuration`注解**并且**该注解的`proxyBeanMethods = false`，那么mark一下它是**Full模式**的配置。否则进入下一步判断
   2. 标注有`@Configuration`注解**或者**符合Lite模式的条件（上文有说一共有5种可能是Lite模式，源码处在`isConfigurationCandidate(metadata)`这个方法里表述），那么mark一下它是**Lite模式**的配置。否则进入下一步判断
   3. 不是配置类，并且返回结果`return false`

3. 能进行到这一步，说明该Bean肯定是个配置类了（Full模式或者Lite模式），那就取出其`@Order`值（若有的话），然后mark进Bean定义里面去

**这个mark动作很有意义：后面判断一个配置类是Full模式还是Lite模式，甚至判断它是否是个配置类均可通过`beanDef.getAttribute(CONFIGURATION_CLASS_ATTRIBUTE)`这样完成判断**。



#### 方法使用处

知晓了`checkConfigurationClassCandidate()`能够判断一个Bean(定义)是否是一个配置类，那么它在什么时候会被使用呢？通过查找可以发现它被如下两处使用到：

- 使用处：`ConfigurationClassPostProcessor.processConfigBeanDefinitions()`处理配置Bean定义阶段。

~~~java
ConfigurationClassPostProcessor：

	public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
		
		// 拿出当前所有的Bean定义信息，一个个的检查是否是配置类	
		String[] candidateNames = registry.getBeanDefinitionNames();
		for (String beanName : candidateNames) {
			BeanDefinition beanDef = registry.getBeanDefinition(beanName);
			if (beanDef.getAttribute(ConfigurationClassUtils.CONFIGURATION_CLASS_ATTRIBUTE) != null) {
				logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);
			}
			// 如果该Bean定义不是配置类，那就继续判断一次它是否是配置类，若是就加入结果集合里
			else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
				configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
			}
		}
		...
	}
~~~

- `ConfigurationClassPostProcessor`是个`BeanDefinitionRegistryPostProcessor`，会在`BeanFactory` **准备好后**执行生命周期方法。因此自然而然的，`checkConfigurationClassCandidate()`会在此阶段调用，用于区分出哪些是配置Bean。

**值得注意的是**：`ConfigurationClassPostProcessor`的执行时期是非常早期的（`BeanFactory`准备好后就执行嘛），这个时候容器内的Bean定义**很少**。这个时候只有**主配置类**才被注册了进来，那些想通过`@ComponentScan`扫进来的配置类都还没到“时间”，这个时间节点很重要，请注意区分。为了方便你理解，我分别把Spring和Spring Boot在此阶段的Bean定义信息截图展示如下：



![](C:\Users\Admin\Desktop\learn-spring\图片\20200516172229283.png)

以上是Spring环境，对应代码为：



使用处：`ConfigurationClassParser.doProcessConfigurationClass()` **解析** `@Configuration`配置类阶段。所处的大阶段同上使用处，仍旧是`ConfigurationClassPostProcessor#postProcessBeanDefinitionRegistry()`阶段

~~~java
ConfigurationClassParser：

	@Nullable
	protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass) throws IOException {
		... // 先解析nested内部类（内部类会存在@Bean方法嘛~）
		... // 解析@PropertySource资源，加入到environment环境
		... // 解析@ComponentScan注解，把组件扫描进来
		scannedBeanDefinitions = ComponentScanAnnotationParser.parse(componentScan, ...);
			// 把扫描到的Bean定义信息依旧需要一个个的判断，是否是配置类	
			// 若是配置类，就继续当作一个@Configuration配置类来解析parse() 递归嘛
			for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
				...
				if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
					parse(bdCand.getBeanClassName(), holder.getBeanName());
				}
			}
		... // 解析@Import注解
		... // 解析@ImportResource注解
		... // 解析当前配置里配置的@Bean方法
		... // 解析接口默认方法（因为配置类可能实现接口，然后接口默认方法可能标注有@Bean ）
		... // 处理父类（递归，直到父类为java.打头的为止）
	}
~~~

这个方法是Spring对配置类解析的**最核心步骤**，

通过它顺带也能够解答你的疑惑了吧：为何你仅需在类上标注一个`@Configuration`注解即可让它成为一个配置类？因为被Scan扫描进去了嘛~



通过以上**两个使用处**的分析和对比，对于`@Configuration`配置类的理解，你至少应该掌握了如下讯息：

1. `@Configuration`配置类肯定是个组件，存在于IoC容器里
2. `@Configuration`配置类是**有主次之分**的，主配置类是驱动整个程序的入口，可以是一个，也可以是多个（若存在多个，支持使用@Order排序）
3. 我们平时一般只书写**次配置类**（而且一般写多个），它**一般**是借助主配置类的`@ComponentScan`能力完成加载进而解析的（当然也可能是`@Import`、又或是被其它次配置类驱动的）
4. 配置类可以存在嵌套（如内部类），继承，实现接口等特性

聊完了最为重要的`checkConfigurationClassCandidate()`方法，当然还有必要看看`ConfigurationClassUtils`的另一个工具方法`isConfigurationCandidate()`。



### isConfigurationCandidate()

它是一个public static工具方法，通过给定的注解元数据信息来判断它是否是一个`Configuration`。

~~~java
ConfigurationClassUtils：

	static {
		candidateIndicators.add(Component.class.getName());
		candidateIndicators.add(ComponentScan.class.getName());
		candidateIndicators.add(Import.class.getName());
		candidateIndicators.add(ImportResource.class.getName());
	}

	public static boolean isConfigurationCandidate(AnnotationMetadata metadata) {
		// 不考虑接口 or 注解 说明：注解的话也是一种“特殊”的接口哦
		if (metadata.isInterface()) {
			return false;
		}
		// 只要该类上标注有以上4个注解任意一个，都算配置类
		for (String indicator : candidateIndicators) {
			if (metadata.isAnnotated(indicator)) {
				return true;
			}
		}
		// 若一个注解都没标注，那就看有木有@Bean方法 若有那也算配置类
		return metadata.hasAnnotatedMethods(Bean.class.getName());
	}
~~~

步骤总结：

1. 若是接口类型（含注解类型），直接不予考虑，返回false。否则继续判断
2. 若此类上标注有`@Component、@ComponentScan、@Import、@ImportResource`任意一个注解，就判断成功返回true。否则继续判断
3. 到此步，就说明**此类上没有标注任何注解**。若存在@Bean方法，返回true，否则返回false。



**需要特别特别特别注意的是：此方法它的并不考虑`@Configuration`注解，是“轻量级”判断，这是它和`checkConfigurationClassCandidate()`方法的最主要区别**。当然，后者依赖于前者，依赖它来根据注解元数据判断是否是Lite模式的配置。



## 3.3 Spring 5.2.0版本变化说明

因为本文的讲解和代码均是基于`Spring 5.2.2.RELEASE`的，而并不是所有小伙伴都会用到这么新的版本。关于此部分的实现，以Spring 5.2.0版本为分界线实现上有些许差异，所以在此处做出说明。

### proxyBeanMethods属性的作用

`proxyBeanMethods`属性是Spring 5.2.0版本为`@Configuration`注解新增加的一个属性：

~~~java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Configuration {
	@AliasFor(annotation = Component.class)
	String value() default "";
	// @since 5.2
	boolean proxyBeanMethods() default true;
}
~~~

它的作用是：是否允许代理@Bean方法。说白了：决定此配置使用Full模式还是Lite模式。为了保持向下兼容，`proxyBeanMethods`的默认值是true，使用Full模式配置。

Spring 5.2提出了这个属性项，是期望你在已经了解了它的作用之后，显示的把它置为false的，因为在云原生将要到来的今天，启动速度方面Spring一直在做着努力，也希望你能配合嘛。这不`Spring Boot`就“配合”得很好，它在2.2.0版本（依赖于Spring 5.2.0）起就把它的所有的自动配置类的此属性改为了false，即`@Configuration(proxyBeanMethods = false)`。



### Full模式/Lite模式实现上的差异

由于Spring 5.2.0新增了`proxyBeanMethods`属性来控制模式，因此实现上也有些许诧异，请各位注意甄别：



## 思考题？

1. 既然`isConfigurationCandidate()`判断方法是为`checkConfigurationClassCandidate()`服务，那Spring为何也把它设计为public static呢？
2. `ConfigurationClassUtils`里还存在对`@Order`顺序的解析方法，不是说Spring的Bean是无序的吗？这又如何理解呢？





## 3.4 何时创建代理？

我们已然知道Full模式的配置类最终会被CGLIB字节码提升，从而最终放一个代理类对象到Spring容器里。那么我们先来弄清楚创建代理的时机在哪儿~

Spring容器在`refresh()`启动步骤的`AbstractApplicationContext#invokeBeanFactoryPostProcessors`这一步会执行所有的`BeanFactoryPostProcessor`处理器，而此时`BeanFactory`才刚刚准备好，容器内除了`ConfigurationClassPostProcessor`之外，并无任何其它`BeanFactoryPostProcessor`，截图示例如下：



![](C:\Users\Admin\Desktop\learn-spring\图片\20200519080351869.png)



既然这样，那么接下来就会会`ConfigurationClassPostProcessor`这个后置处理器喽。



## 3.4 ConfigurationClassPostProcessor

用于引导处理`@Configuration`配置类。该后置处理器的优先级是较高的，属于`PriorityOrdered`分类。

**说明：`PriorityOrdered`的优先级肯定比`Order`接口的高**



~~~java
// @since 3.0
public class ConfigurationClassPostProcessor implements BeanDefinitionRegistryPostProcessor,
		PriorityOrdered, ResourceLoaderAware, BeanClassLoaderAware, EnvironmentAware {
		...
}
~~~



### postProcessBeanDefinitionRegistry()

从注册进来的配置类（可能是Full模式，可能是Lite模式）里进一步派生bean定义。简而言之：收集到所有的`BeanDefinition`（后简称为bd）存储起来，包括`@Import、@Component`等等组件。**并且做出标注：是Full模式的还是Lite模式的配置类**（若非配置组件就不标注哦）。

~~~~java
ConfigurationClassPostProcessor：

	@Override
	public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
		// 生成一个id，放置后面再重复执行
		int registryId = System.identityHashCode(registry);
		// 若重复执行  就抛出异常
		if (this.registriesPostProcessed.contains(registryId)) {
			throw new IllegalStateException("postProcessBeanDefinitionRegistry already called on this post-processor against " + registry);
		}
		if (this.factoriesPostProcessed.contains(registryId)) {
			throw new IllegalStateException("postProcessBeanFactory already called on this post-processor against " + registry);
		}

		// 表示此registry里的bd收集动作，已经做了  避免再重复收集此registry
		this.registriesPostProcessed.add(registryId);

		// 根据配置类，收集到所有的bd信息
		// 并且做出mark标注：是Full模式还是Lite模式，和很重要很重要
		processConfigBeanDefinitions(registry);
	}
~~~~

执行完此方法，已经完成了bd的收集和标记，那接下来就是本文的主菜了：**帮你解答上面case的结果**。



### postProcessBeanFactory()

此方法的作用用一句话可概括为：**为Full模式的Bean使用CGLIB做字节码提升，确保最终生成的是代理类实例放进容器内**。

~~~java
ConfigurationClassPostProcessor：

	@Override
	public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		int factoryId = System.identityHashCode(beanFactory);
		// 防止重复处理
		if (this.factoriesPostProcessed.contains(factoryId)) {
			throw new IllegalStateException("postProcessBeanFactory already called on this post-processor against " + beanFactory);
		}
		this.factoriesPostProcessed.add(factoryId);

		// 在执行postProcessBeanDefinitionRegistry方法的时就已经将
		// 这个id添加到registriesPostProcessed集合中了
		// 所以到这里就不会再重复执行配置类的解析了（解析@Import、@Bean等）
		if (!this.registriesPostProcessed.contains(factoryId)) {
			processConfigBeanDefinitions((BeanDefinitionRegistry) beanFactory);
		}

		// 从名字上看，这个方法应该就是为配置类创建代理用的喽
		enhanceConfigurationClasses(beanFactory);
		
		// 添加了一个后置处理器 它是个SmartInstantiationAwareBeanPostProcessor
		// 它不是本文重点，略
		beanFactory.addBeanPostProcessor(new ImportAwareBeanPostProcessor(beanFactory));
	}
~~~

达到这一步之前，已经完成了bd的收集和标记（见上一步）。对bd进行实例化之前，针对于**Full模式的配置类**这步骤里会做增强处理，那就是`enhanceConfigurationClasses(beanFactory)`这个方法。



### enhanceConfigurationClasses(beanFactory)

对一个`BeanFactory`进行增强，先查找配置类`BeanDefinition`，再根据Bean定义信息（元数据信息）来决定配置类是否应该被`ConfigurationClassEnhancer`增强。具体处理代码如下：

~~~java
ConfigurationClassPostProcessor：

	public void enhanceConfigurationClasses(ConfigurableListableBeanFactory beanFactory) {
		// 最终需要做增强的Bean定义们
		Map<String, AbstractBeanDefinition> configBeanDefs = new LinkedHashMap<>();
		
		for (String beanName : beanFactory.getBeanDefinitionNames()) {
			BeanDefinition beanDef = beanFactory.getBeanDefinition(beanName);
			Object configClassAttr = beanDef.getAttribute(ConfigurationClassUtils.CONFIGURATION_CLASS_ATTRIBUTE);
		
			... // 省略其它情况以及异常情况的处理代码
			
			// 如果是Full模式，才会放进来
			if (ConfigurationClassUtils.CONFIGURATION_CLASS_FULL.equals(configClassAttr)) {
				configBeanDefs.put(beanName, (AbstractBeanDefinition) beanDef);
			}
		}
		if (configBeanDefs.isEmpty()) {
			// nothing to enhance -> return immediately
			return;
		}

		// ConfigurationClassEnhancer就是对配置类做增强操作的核心类，下面详解
		ConfigurationClassEnhancer enhancer = new ConfigurationClassEnhancer();
		// 对每个Full模式的配置类，一个个做enhance()增强处理
		for (Map.Entry<String, AbstractBeanDefinition> entry : configBeanDefs.entrySet()) {
			AbstractBeanDefinition beanDef = entry.getValue();
			
			// 如果代理了@Configuration类，则始终代理目标类
			// 该属性和自动代理时是相关的，具体参见Spring的自动代理章节描述
			beanDef.setAttribute(AutoProxyUtils.PRESERVE_TARGET_CLASS_ATTRIBUTE, Boolean.TRUE);
			
			// CGLIB是给父类生成子类对象的方式实现代理，所以这里指定“父类”类型
			Class<?> configClass = beanDef.getBeanClass();
			// 做增强处理，返回enhancedClass就是一个增强过的子类
			Class<?> enhancedClass = enhancer.enhance(configClass, this.beanClassLoader);
			// 不相等，证明代理成功，那就把实际类型设置进去
			// 这样后面实例化配置类的实例时，实际实例化的就是增强子类喽
			if (configClass != enhancedClass) {
				beanDef.setBeanClass(enhancedClass);
			}
		}
	}
~~~

**值得注意的是，虽然此方法被设计为public的，但是只被一处使用到。Spring这么做是为了给提供钩子，方便容器开发者做扩展时使用**















































































