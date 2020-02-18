# Spring Bean 解析注册

1. 读取 XML 配置文件
2. XML 文件解析为 document 文档
3. 解析 bean
4. 注册 bean
5. 实例化 bean
6. 获取 bean

## 1. 读取 XML 配置文件

查看源码第一步是找到程序入口，再以入口为突破口，一步步进行源码跟踪。

Java Web 应用中的入口就是 web.xml

在 web.xml 找到 ContextLoaderListener，此 Listener 负责初始化 Spring IoC

contextConfigLocation 参数设置了 bean 定义文件地址。

```xml
<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath*:spring.xml</param-value>
</context-param>
```

 下面是ContextLoaderListener的官方定义： 

> public class ContextLoaderListener extends ContextLoader implements ServletContextListener
> Bootstrap listener to start up and shut down Spring's root WebApplicationContext. Simply delegates to ContextLoader as well as to ContextCleanupListener.
> [https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/context/ContextLoaderListener.html](https://link.zhihu.com/?target=https%3A//docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/context/ContextLoaderListener.html)

 翻译过来 ContextLoaderListener 作用就是负责启动和关闭 Spring root WebApplicationContext。 

 具体WebApplicationContext是什么？开始看源码。 

```java
package org.springframework.web.context;
public class ContextLoaderListener extends ContextLoader implements ServletContextListener {
    public ContextLoaderListener() {
    }
    public ContextLoaderListener(WebApplicationContext context) {
        super(context);
    }
    //servletContext初始化时候调用
    public void contextInitialized(ServletContextEvent event) {
        this.initWebApplicationContext(event.getServletContext();
    }
    //servletContext销毁时候调用
    public void contextDestroyed(ServletContextEvent event) {
        this.closeWebApplicationContext(event.getServletContext());
    }
}
```

 从源码看出此Listener主要有两个函数，一个负责初始化WebApplicationContext，一个负责销毁。 

 继续看initWebApplicationContext函数。 

```java
public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
//初始化Spring容器时如果发现servlet 容器中已存在根Spring容根器则抛出异常，证明rootWebApplicationContext只能有一个。
   if (servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null) {
      throw new IllegalStateException(
            "Cannot initialize context because there is already a root application context present - " +
            "check whether you have multiple ContextLoader* definitions in your web.xml!");
   }
   if (this.context == null) {
	//1.创建webApplicationContext实例
        this.context = createWebApplicationContext(servletContext);
   }
   if (this.context instanceof ConfigurableWebApplicationContext) {
        ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
	 //2.配置WebApplicationContext
        configureAndRefreshWebApplicationContext(cwac, servletContext);
    }
    //把生成的webApplicationContext 设置为root webApplicationContext。
    servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);
    return this.context; 

}
```

在上面的代码中主要有两个功能：

- （1）创建WebApplicationContext实例。
- （2）配置生成WebApplicationContext实例。

### 1.1 创建 WebApplicationContext 实例

进入 CreateWebApplicationContext 函数

```java
protected WebApplicationContext createWebApplicationContext(ServletContext sc) {
   //得到ContextClass类,默认实例化的是XmlWebApplicationContext类
   Class<?> contextClass = determineContextClass(sc);
   //实例化Context类
   return (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);
}
```

进入 determineContextClass(sc) 函数

```java
protected Class<?> determineContextClass(ServletContext servletContext) {
   // 此处CONTEXT_CLASS_PARAM = "contextClass"
   String contextClassName = servletContext.getInitParameter(CONTEXT_CLASS_PARAM);
   if (contextClassName != null) {
         //若设置了contextClass则使用定义好的ContextClass。
         return ClassUtils.forName(contextClassName, ClassUtils.getDefaultClassLoader());
      }
   else {
      //此处获取的是在Spring源码中ContextLoader.properties中配置的org.springframework.web.context.support.XmlWebApplicationContext类。
      contextClassName = defaultStrategies.getProperty(WebApplicationContext.class.getName());
      return ClassUtils.forName(contextClassName, ContextLoader.class.getClassLoader());
}
```

### 1.2 配置 WebApplicationContext

 进入configureAndReFreshWebApplicaitonContext函数。 

```java
protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc) {
  //webapplicationContext设置servletContext.
   wac.setServletContext(sc);
   // 此处CONFIG_LOCATION_PARAM = "contextConfigLocation"，即读即取web.xm中配设置的contextConfigLocation参数值，获得spring bean的配置文件.
   String configLocationParam = sc.getInitParameter(CONFIG_LOCATION_PARAM);
   if (configLocationParam != null) {
      //webApplicationContext设置配置文件路径设。
      wac.setConfigLocation(configLocationParam);
   }
   //开始处理bean
   wac.refresh();
}
```

## 2. 解析 XML 文件

上面wac变量声明为ConfigurableWebApplicationContext类型，ConfigurableWebApplicationContext又继承了WebApplicationContext。

WebApplicationContext有很多实现类。 但从上面determineContextClass得知此处wac实际上是XmlWebApplicationContext类，因此进入XmlWebApplication类查看其继承的refresh()方法。

沿方法调用栈一层层看下去。

```java
public void refresh() throws BeansException, IllegalStateException {
   synchronized (this.startupShutdownMonitor) {
      //获取beanFactory
      ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
     // 实例化所有声明为非懒加载的单例bean 
      finishBeanFactoryInitialization(beanFactory);
    }
}
```

获取 beanFactory

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
     //初始化beanFactory
     refreshBeanFactory();
     return beanFactory;
}
```

beanFactory 初始化

```java
@Override
protected final void refreshBeanFactory() throws BeansException {
      DefaultListableBeanFactory beanFactory = createBeanFactory();
      //加载bean定义
      loadBeanDefinitions(beanFactory);
      synchronized (this.beanFactoryMonitor) {
      this.beanFactory = beanFactory;
      }
}
```

 加载bean。 

```java
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
   //创建XmlBeanDefinitionReader实例来解析XML配置文件
   XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
   initBeanDefinitionReader(beanDefinitionReader);
   //解析XML配置文件中的bean。
   loadBeanDefinitions(beanDefinitionReader);
}
```

读取 XML 配置文件

```java
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws IOException {
//此处读取的就是之前设置好的web.xml中配置文件地址
   String[] configLocations = getConfigLocations();
   if (configLocations != null) {
      for (String configLocation : configLocations) {
         //调用XmlBeanDefinitionReader读取XML配置文件
         reader.loadBeanDefinitions(configLocation);
      }
   }
}
```

 XmlBeanDefinitionReader 读取 XML 文件中的 bean 定义。 

```java
public int loadBeanDefinitions(String location, Set<Resource> actualResources) throws BeanDefinitionStoreException {
   ResourceLoader resourceLoader = getResourceLoader();
      Resource resource = resourceLoader.getResource(location);
      //加载bean
      int loadCount = loadBeanDefinitions(resource);
      return loadCount;
   }
}
```

 继续查看loadBeanDefinitons函数调用栈，进入到XmlBeanDefinitioReader类的loadBeanDefinitions方法。 

```java
public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
      //获取文件流
      InputStream inputStream = encodedResource.getResource().getInputStream();
      InputSource inputSource = new InputSource(inputStream);
     //从文件流中加载定义好的bean。
      return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
   }
}
```