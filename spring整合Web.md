# 一、servlet3.0

## 1. 传统servlet的使用

~~~java
public class HelloServlet extends HttpServlet{
    
    protected void doGet(HttpServletRequest req,HttpServletResponse res){
        res.getWriter().write("hello....");
    }
}

然后再web.xml使用
~~~



## 2. 使用Spring整合Web应用

~~~java
@WebServlet("/hello")
public class HelloServlet extends HttpServlet{
    
    protected void doGet(HttpServletRequest req,HttpServletResponse res){
        res.getWriter().write("hello....");
    }
}
~~~



## 1.3 ServletContainerInitializer

1. Shared libraries（共享库）
2. runtimes pluggability（运行时插件）



1. Servlet容器启动会扫描，当前应用里面每一个jar包的**ServletContainerInitializer**的实现

2. 提供**ServletContainerInitializer**的实现类

   必须绑定在META-INF/services/javax.servlet.ServletContainerInitializer

   文件的内容就是**ServletContainerInitializer**实现的全类名



总结：容器在启动应用的时候，会扫描当前应用每一个jar包里面

META-INF/services/javax.servlet.ServletContainerInitializer 指定的实现类，启动并运行这个实现类的方法



~~~java

//容器启动的时候，会将@HandlesTypes指定的这个类型下面的子类（实现类，子接口等）传递过来
//传入自定义感兴趣的【类型】
@HandlesType(value = {
    HelloService.class
})
public class MyServletContainerInitializer implements ServletContainerInitializer{
    
    //应用启动的时候，会启动onStartup方法
    //Set<Class<?>>：获取注解中感兴趣的【子类型】
    //ServletContext：代表当前Web应用的ServletContext；一个Web应用代表一个ServletContext
    public void onStartup(Set<Class<?>> set,ServletContext context){
        
    }
}
~~~



## 1.4 ServletContext注册三大组件

三大组件：**Servlet、Filter、Listener**

~~~java
public class MyServletContainerInitializer implements ServletContainerInitializer{
    
    public void onStartup(Set<Class<?>> set,ServletContext context){
        
        //注册组件 ServletRegistration的
        Dynamic mic = context.addServlet("userServlet",new UserServlet());
    	//配置servlet映射信息
        mic.addMapping("/user");
        
        
        //注册监听器
        sc.addListener(UserListener.class);
        
        //注册filter FilterRegistration的
        Dynamic filter = sc.addFilter("userFilter",UserFilter.class);
        //配置filter的映射信息
        filter.addMappingForUrlPatterns(EnumSet.of(DispatcherType.REQUEST),true,"/*");
        
    
    }
}


public class PersonServlet extends HttpServlet{
    
    protected void doGet(HttpServletRequest req,HttpServletResponse res){
        res.getWriter().write("tomcat.....")
    }
}


public class PersonFilter implements Filter{
    
    public void destroy(){
        
    }
    public void doFilter(ServletRequest req,ServletReponse res){
        //过滤请求
        sout("filter生效了....")
        //放行    
        filter.doFilter(req,res);
    }
    public void init(FilterConfig config) throws ServletException{
        
    }
}


public class PersonListener implements ServletContextListener{
    
    //监听ServletContext销毁
    public void contextDestroyed(ServletContextEvent event){
        
    }
    
    //监听ServletContext启动初始化
    public void contextInitialized(ServletContextEvent event){
        sout("")
    }
}

~~~



## 1.5 servlet3.0 与 springMVC整合

1. 导入mvc的maven坐标

2. 利用ServletContainerInitializer来实现

3. web容器在启动的时候，会扫描每个jar包中的META-INF下的文件

4. spring的应用启动会加载感兴趣的WebApplicationInitializer接口下的所有组件

5. 并且为WebApplicationInitializer组件创建对象（组件不是接口，不是抽象类）

   1. AbstractContextLoaderInitializer：创建根容器（createRootApplicationContext）

   2. AbstractDispatcherServletInitializer：

      创建一个Web的IOC容器:createServletApplicationContext()

      创建一个DispatcherServlet：createDispatcherServlet（）

      将创建的DispatcherServlet添加到ServletContext中

      ​	getServletMappings（）

   3. AbstractAnnotationConfigDispatcherServletInitializer：注解方式配置DispatchServlet初始化器

      创建根容器：createRootApplicationContext

      ​					    getRootConfigClasses（）：传入一个配置类

      创建web的ioc容器：createServletApplicationContext

      ​						获取配置类：getServletConfigClasses（）



总结：

​	以注解方法来启动SpringMVC，继承AbstractAnnotationConfiDispatcherServletInitializer

​	实现抽象方法指定DispatcherServlet配置信息；



## 1.6 spring-mvc整合

~~~java
//web容器启动的时候创建对象；调用方法进行初始化容器，以及前端控制器
public class MyWebApplication extends AbstractAnnotationConfiDispatcherServletInitializer{
    
    //根据根容器的配置类：（Spring的配置文件） 父容器
    protected Class<?>[] getRootConfigClasses(){
        
    }
    
    
    //获取web容器的配置类（SpringMVC配置文件） 子容器
    protected Class<?>[] getServletConfigClasses(){
        
    }
    
    //获取DispatherServlet的映射信息
    protected String<?>[] getServletMappings(){
        
        //拦截所有请求，包括静态资源，但是不包括jsp
        return new String[]{"/"}
    }
    
}
~~~

写两个配置类

~~~java
//Spring的容器不扫描controller；父容器
@Component(value="com.包路径",excludeFilters={
    @Filter(type=FilterType.ANNOTATION,classes={Controller.class})
})
public class RootConfig{
    
}
~~~



~~~java
//Spring的容器只扫描controller；子容器
@Component(value="com.包路径",includeFilters={
    @Filter(type=FilterType.ANNOTATION,classes={Controller.class})
},useDefaultFilters=false)
public class AppConfig{
    
}

~~~



## 1.7 定制SpringMVC

~~~java
@EnableWebMvc：开启springMVC的定制功能
~~~



配置组件（视图解析器、视图映射、静态资源映射、拦截器。。。）

实现接口：WebMvcConfigurer

~~~java
@EnableWebMvc
public class AppConfig implements WebMvcConfigurer{
    
    //定制环节
    //视图解析器
    public void configureViewResolvers(ViewResolverRegistry registry){
        //默认所有的页面都从WEB-INF下找jsp文件
        registry.jsp();
    }
    
    //静态资源访问
    public void configureDefaultServletHandling(DefaultServletHandlerConfigurer con){
        //开启静态资源访问
        con.enable();
    }
    
}

~~~





































