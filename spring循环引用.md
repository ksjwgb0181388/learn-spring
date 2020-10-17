# spring循环依赖

spring当中的循环依赖是怎么解决的？

1. spring当中单例模式是支持循环依赖



依赖注入功能——在初始化的时候

1. 初始化bean——bean有一个初始化过程——spring生命周期！



## 如何关闭spring循环依赖

~~~java

AnnotationConfigApplication context  = new AnnotationConfigApplication();

context.register(AppConfig.class);
AbstractAutowireCapableBeanFactroy cap = (AbstractAutowireCapableBeanFactroy)context.getBeanFactroy();

cap.setAllowCircularReferences(false);

context.refresh();

~~~



## spring循环依赖的三个map

1. singletonObjects **单例池** spring容器 一级

   1. 这个缓存 缓存的就是实例化之后的bean

      getBean的背后支持 singletonObjects.get("indexService");

2. singletonFactories  工厂 二级

3. earlySingletonObjects 三级缓存































































