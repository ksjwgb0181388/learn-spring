

# 一、什么是循环依赖

相互引用了对方，也就是我们常常的说的循环依赖，spring是允许这样的循环依赖(前提是单例的情况下的,非构造方法注入的情况下)

spring的属性注入属于spring bean的生命周期一部分



两个概念——spring bean（一下简称bean）和对象；

1. **spring bean**——受spring容器管理的对象，可能经过了完整的spring bean生命周期（为什么是可能？难道还有bean是没有经过bean生命周期的？答案是有的，具体我们后面文章分析），最终存在spring容器当中；一个bean一定是个对象
2. **对象**——任何符合java语法规则实例化出来的对象，但是一个对象并不一定是spring bean；



所谓的bean的生命周期就是磁盘上的类通过spring扫描，然后实例化，跟着初始化，继而放到容器当中的过程；

<img src="C:\Users\Admin\Desktop\learn-spring\图片\20191118210559319.png"  />



1：实例化一个ApplicationContext的对象；

2：调用bean工厂后置处理器完成扫描；

3：循环解析扫描出来的类信息；

4：实例化一个BeanDefinition对象来存储解析出来的信息；

5：把实例化好的beanDefinition对象put到`beanDefinitionMap`当中缓存起来，以便后面实例化bean；

6：再次调用bean工厂后置处理器；

7：当然spring还会干很多事情，比如国际化，比如注册BeanPostProcessor等等，如果我们只关心如何实例化一个bean的话那么这一步就是spring调用`finishBeanFactoryInitialization`方法来实例化单例的bean，实例化之前spring要做验证，需要遍历所有扫描出来的类，依次判断这个bean是否Lazy，是否prototype，是否abstract等等；

8：如果验证完成spring在实例化一个bean之前需要推断构造方法，因为spring实例化对象是通过构造方法反射，故而需要知道用哪个构造方法；

9：推断完构造方法之后spring调用构造方法反射实例化一个**对象**；注意我这里说的是对象、对象、对象；这个时候对象已经实例化出来了，但是并不是一个完整的bean，最简单的体现是这个时候实例化出来的对象属性是没有注入，所以不是一个完整的bean；

10：spring处理合并后的beanDefinition(合并？是spring当中非常重要的一块内容，后面的文章我会分析)；

11：判断是否支持循环依赖，如果支持则提前把一个工厂存入singletonFactories——map；

12：判断是否需要完成属性注入
13：如果需要完成属性注入，则开始注入属性

14：判断bean的类型回调Aware接口

15：调用生命周期回调方法

16：如果需要代理则完成代理

17：put到单例池——bean完成——存在spring容器当中



~~~java
package com.shadow.service;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;
import org.springframework.stereotype.Component;

import javax.annotation.PostConstruct;

@Component
public class Z implements ApplicationContextAware {
	@Autowired
	X x;//注入X

    //构造方法
	public Z(){
		System.out.println("Z create");
	}

    //生命周期初始化回调方法
	@PostConstruct
	public void zinit(){
		System.out.println("call z lifecycle init callback");
	}

	//ApplicationContextAware 回调方法
	@Override
	public void setApplicationContext(ApplicationContext ac) {
		System.out.println("call aware callback");
	}
}



~~~































































































