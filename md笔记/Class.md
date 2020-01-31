# Class类的三种实例化模式  
### 类的定义
`public final class Class<T>extends Object implements Serializable,GenericDeclaration,Type,AnnotatedElement`  
### 第一种 Object类支持：public final Class<?> getClass()  
```java
Person per = new Person();
Class<? extends Person > cls = per.getClass();
System.out.println(cls);# class 包.类
System.out.println(cls).getName();# 包.类
```  
如果只是想获得Class类对象必须得先获得指定类的对象

### JVM支持  
```java
Class<? extends Person > cls = Person.Class;
System.out.println(cls);# class 包.类
System.out.println(cls).getName();# 包.类
```  
  
### Class类支持  
```java
#Person包没有导入
Class<? > cls = Class.forName("包.类");
System.out.println(cls);# class 包.类
System.out.println(cls).getName();# 包.类
```  

### 工厂设计模式升级
```java
package com.Runtim;

import java.lang.reflect.InvocationTargetException;

public class TestRun {
    public static void main(String[] args) {
        Houseservice h= new Houseservice();
        Class<? extends Houseservice> cls= h.getClass();
        System.out.println(cls);
        System.out.println("******************");
        try {
            Cloudservice c =Factory.get("com.Runtim.Cloudservice",Cloudservice.class);
            c.send();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        }
    }

}
interface service{
    public void send();
}
class Houseservice implements service{
    public Houseservice(){
        System.out.println(this+"构造方法");
    }
    @Override
    public void send() {
        System.out.println("House send");
    }
}
class Cloudservice implements service{
    public Cloudservice(){
        System.out.println(this+"构造方法");
    }
    @Override
    public void send() {
        System.out.println("Cloud send");
    }
}
class Factory{
    public static <T> T get(String stringname,Class<T> clazz) throws ClassNotFoundException, NoSuchMethodException, IllegalAccessException, InvocationTargetException, InstantiationException {
        T instance=null;
        instance =(T)Class.forName(stringname).getDeclaredConstructor().newInstance();
        return instance;
    }
}
```  
可以指定任意类进行返回实例，而不修改工厂类