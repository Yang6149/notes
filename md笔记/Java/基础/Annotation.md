# Annotation

Java 定义了一套注解，共7个，3个在java.lang中,剩下4个在java.lang.annotation中。

作用在代码上的注解：

* @Override - 检查该方法是否是重载方法，如果发现其弗雷，或者是引用的接口没有该方法，则报编译错误
* @Depressed - 标记过时方法。如果使用该方法，则会报编译警告
* SuppressWarnings - 指示编译器去忽略注解中声明的警告



作用在其他注解的注解（元注解）是：

* @Retention - 标识这个注解怎么保存，是只在代码中，还是编入class文件中，或者是在运行时可以通过反射访问。
* @Documented - 标记这些注解是否包含在用户文档中。
* @Target - 标记这个注解应该是哪种 Java 成员。
* @Inherited - 标记这个注解是继承于哪个注解类

从 Java 7 开始，额外添加了 3 个注解：

* @SafeVarargs - Java 7 开始支持， 忽略任何使用参数为泛型变量的方法或构造函数调用产生的警告。
* FunctionalInterface - Java 8 开始支持，标识一个匿名函数或函数式接口。
* @Repeatable - Java 8 开始支持，表示某注解可以在同一个声明上使用多次。

## 1、 Annotation 架构

![](https://www.runoob.com/wp-content/uploads/2019/08/28123151-d471f82eb2bc4812b46cc5ff3e9e6b82.jpg)

从中，我们可以看出：

**(01) 1 个 Annotation 和 1 个 RetentionPolicy 关联。**

可以理解为：每1个Annotation对象，都会有唯一的RetentionPolicy属性。

**(02) 1 个 Annotation 和 1~n 个 ElementType 关联。**

可以理解为：对于每 1 个 Annotation 对象，可以有若干个 ElementType 属性。

**(03) Annotation 有许多实现类，包括：Deprecated, Documented, Inherited, Override 等等。**

Annotation 的每一个实现类，都 "和 1 个 RetentionPolicy 关联" 并且 " 和 1~n 个 ElementType 关联"。

下面，我先介绍框架图的左半边(如下图)，即 Annotation, RetentionPolicy, ElementType；然后在就 Annotation 的实现类进行举例说明。

![img](https://www.runoob.com/wp-content/uploads/2019/08/28123653-84d14b886429482bb601dc97155220fb.jpg)

## 2、Annotation 组成部分

java Annotation 的组成中，有 3 个非常重要的主干类。它们分别是：

## Annotation.java

```java
package java.lang.annotation;
public interface Annotation {

  boolean equals(Object obj);

  int hashCode();

  String toString();

  Class<? extends Annotation> annotationType();
}
```



## ElementType.java

```java
package java.lang.annotation;

public enum ElementType {
  TYPE,        /* 类、接口（包括注释类型）或枚举声明  */

  FIELD,        /* 字段声明（包括枚举常量）  */

  METHOD,       /* 方法声明  */

  PARAMETER,      /* 参数声明  */

  CONSTRUCTOR,     /* 构造方法声明  */

  LOCAL_VARIABLE,   /* 局部变量声明  */

  ANNOTATION_TYPE,   /* 注释类型声明  */

  **PACKAGE**       /* 包声明  */
}
```





## RetentionPolicy.java

```java
package java.lang.annotation;
public enum RetentionPolicy {
  SOURCE,       /* Annotation信息仅存在于编译器处理期间，编译器处理完之后就没有该Annotation信息了  */

  CLASS,       /* 编译器将Annotation存储于类对应的.class文件中。默认行为   */

  RUNTIME       /* 编译器将Annotation存储于class文件中，并且可由JVM读入  */
}
```





说明：