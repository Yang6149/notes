# Spring AOP

- Aspect: A modularization of a concern that cuts across multiple classes. Transaction management is a good example of a crosscutting concern in enterprise Java applications. In Spring AOP, aspects are implemented by using regular classes (the [schema-based approach](https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/core.html#aop-schema)) or regular classes annotated with the `@Aspect` annotation (the [@AspectJ style](https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/core.html#aop-ataspectj)).
- Join point: A point during the execution of a program, such as the execution of a method or the handling of an exception. In Spring AOP, a join point always represents a method execution.
- Advice: Action taken by an aspect at a particular join point. Different types of advice include “around”, “before” and “after” advice. (Advice types are discussed later.) Many AOP frameworks, including Spring, model an advice as an interceptor and maintain a chain of interceptors around the join point.
- Pointcut: **要增强的多个方法的集合** A predicate that matches join points. Advice is associated with a pointcut expression and runs at any join point matched by the pointcut (for example, the execution of a method with a certain name). The concept of join points as matched by pointcut expressions is central to AOP, and Spring uses the AspectJ pointcut expression language by default.
- Introduction: Declaring additional methods or fields on behalf of a type. Spring AOP lets you introduce new interfaces (and a corresponding implementation) to any advised object. For example, you could use an introduction to make a bean implement an `IsModified` interface, to simplify caching. (An introduction is known as an inter-type declaration in the AspectJ community.)
- Target object: An object being advised by one or more aspects. Also referred to as the “advised object”. Since Spring AOP is implemented by using runtime proxies, this object is always a proxied object.
- AOP proxy: An object created by the AOP framework in order to implement the aspect contracts (advise method executions and so on). In the Spring Framework, an AOP proxy is a JDK dynamic proxy or a CGLIB proxy.
- Weaving: linking aspects with other application types or objects to create an advised object. This can be done at compile time (using the AspectJ compiler, for example), load time, or at runtime. Spring AOP, like other pure Java AOP frameworks, performs weaving at runtime.

### AspectJ

Spring AOP 2.5 语法复杂，所以借助了 AspectJ 的语法

代理对象在init spring初始化的时候就已经存在了，而不是使用getBean调用的时候创建，getBean本质是调用了getSingleton(beanName)，有的话返回、没有调用 doCreateBean

```java
doCreateBean(){
    //创建目标对象
    //创建代理对象并返回
}
```

```java
wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean,beanName)
```

通过一个 bean 循环扫描一系列beanPostProcessor的实现类，选一个进行处理，并返回一个代理对象具体方法为 cglib或jdk动态代理 如**果bean有接口的话用动态代理、如果没有接口的话使用cglib**

### JDK Proxy

如果一个类实现了一个接口，那么默认使用的是 jdk proxy 的方法，通过实现一个实现相同接口的类来代理目标类。

### cglib

cglib 动态代理是利用 asm 开源包，对代理对象类的 Class 文件进行加载，通过修改其字节码生成子类来处理。