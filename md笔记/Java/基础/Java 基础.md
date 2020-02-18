# Java 基础

## 一、数据类型

### 基本类型

- byte/8
- char/16
- short/16
- int/32
- float/32
- long/64
- double/64
- boolean/~

 boolean 只有两个值：true、false，可以使用 1 bit 来存储，但是具体大小没有明确规定。JVM 会在编译时期将 boolean 类型的数据转换为 int，使用 1 来表示 true，0 表示 false。JVM 支持 boolean 数组，但是是通过读写 byte 数组来实现的。 

### 包装类型

基本类型都有对应的包装类型，基本类型与其对应的包装类型之间的赋值使用自动装箱与拆箱完成。

```java
Integer x = 2;     // 装箱 调用了 Integer.valueOf(2)
int y = x;         // 拆箱 调用了 X.intValue()
```

- [Autoboxing and Unboxing](https://docs.oracle.com/javase/tutorial/java/data/autoboxing.html)

### 缓存池

new Integer(123) 与 Integer.valueOf(123) 的区别在于：

- new Integer(123) 每次都会新建一个对象；
- Integer.valueOf(123) 会使用缓存池中的对象，多次调用会取得同一个对象的引用。

```java
Integer x = new Integer(123);
Integer y = new Integer(123);
System.out.println(x == y);    // false
Integer z = Integer.valueOf(123);
Integer k = Integer.valueOf(123);
System.out.println(z == k);   // true
```

valueOf() 方法的实现比较简单，就是先判断值是否在缓存池中，如果在的话就直接返回缓存池的内容。

```java
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```

在 Java 8 中，Integer 缓存池的大小默认为 -128~127。

```java
static final int low = -128;
static final int high;
static final Integer cache[];

static {
    // high value may be configured by property
    int h = 127;
    String integerCacheHighPropValue =
        sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
    if (integerCacheHighPropValue != null) {
        try {
            int i = parseInt(integerCacheHighPropValue);
            i = Math.max(i, 127);
            // Maximum array size is Integer.MAX_VALUE
            h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
        } catch( NumberFormatException nfe) {
            // If the property cannot be parsed into an int, ignore it.
        }
    }
    high = h;

    cache = new Integer[(high - low) + 1];
    int j = low;
    for(int k = 0; k < cache.length; k++)
        cache[k] = new Integer(j++);

    // range [-128, 127] must be interned (JLS7 5.1.7)
    assert IntegerCache.high >= 127;
}
```

编译器会在自动装箱过程调用 valueOf() 方法，因此多个值相同且值在缓存池范围内的 Integer 实例使用自动装箱来创建，那么就会引用相同的对象。

```
Integer m = 123;
Integer n = 123;
System.out.println(m == n); // true
```

基本类型对应的缓冲池如下：

- boolean values true and false
- all byte values
- short values between -128 and 127
- int values between -128 and 127
- char in the range \u0000 to \u007F

在使用这些基本类型对应的包装类型时，如果该数值范围在缓冲池范围内，就可以直接使用缓冲池中的对象。

在 jdk 1.8 所有的数值类缓冲池中，Integer 的缓冲池 IntegerCache 很特殊，这个缓冲池的下界是 - 128，上界默认是 127，但是这个上界是可调的，在启动 jvm 的时候，通过 -XX:AutoBoxCacheMax=<size> 来指定这个缓冲池的大小，该选项在 JVM 初始化的时候会设定一个名为 java.lang.IntegerCache.high 系统属性，然后 IntegerCache 初始化的时候就会读取该系统属性来决定上界。

## 二、String

## 概览

String 被声明为 final，因此它不可被继承。(Integer 等包装类也不能被继承）

在 Java 8 中，String 内部使用 char 数组存储数据。

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    /** The value is used for character storage. */
    private final char value[];
}
```

在 Java 9 之后，String 类的实现改用 byte 数组存储字符串，同时使用 `coder` 来标识使用了哪种编码。

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    /** The value is used for character storage. */
    private final byte[] value;

    /** The identifier of the encoding used to encode the bytes in {@code value}. */
    private final byte coder;
}
```

### 不可变的好处

#### 1. 可以缓存 hash 值

 因为 String 的 hash 值经常被使用，例如 String 用做 HashMap 的 key。不可变的特性可以使得 hash 值也不可变，因此只需要进行一次计算。 

#### 2. String poll 的需要

 如果一个 String 对象已经被创建过了，那么就会从 String Pool 中取得引用。只有 String 是不可变的，才可能使用 String Pool。 

 ![img](https://camo.githubusercontent.com/152a310f7698bb23ae201081c5c497b5c1d396d9/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f696d6167652d32303139313231303030343133323839342e706e67) 

#### 3. 安全性

 String 经常作为参数，String 不可变性可以保证参数不可变。例如在作为网络连接参数的情况下如果 String 是可变的，那么在网络连接过程中，String 被改变，改变 String 的那一方以为现在连接的是其它主机，而实际情况却不一定是。 

#### 4. 线程安全

 String 不可变性天生具备线程安全，可以在多个线程中安全地使用。 

 [Program Creek : Why String is immutable in Java?](https://www.programcreek.com/2013/04/why-string-is-immutable-in-java/) 

### String、StringBuffer、StringBuilder

#### 1. 可变性

- String 不可变
- StringBuffer 和 StringBuilder 可变

#### 2. 线程安全性

- String 不可变，因此是线程安全的
- StringBuilder 不是线程安全的
- StringBuffer 是线程安全的，内部使用 synchronized 进行同步

### String Pool

 字符串常量池（String Pool）保存着所有字符串字面量（literal strings），这些字面量在编译时期就确定。不仅如此，还可以使用 String 的 intern() 方法在运行过程将字符串添加到 String Pool 中。 

当一个字符串调用 intern() 方法时，如果 String Pool 中已经存在一个字符串和该字符串值相等（使用 equals() 方法进行确定），那么就会返回 String Pool 中字符串的引用；否则，就会在 String Pool 中添加一个新的字符串，并返回这个新字符串的引用。

下面示例中，s1 和 s2 采用 new String() 的方式新建了两个不同字符串，而 s3 和 s4 是通过 s1.intern() 方法取得同一个字符串引用。intern() 首先把 s1 引用的字符串放到 String Pool 中，然后返回这个字符串引用。因此 s3 和 s4 引用的是同一个字符串。

```java
String s1 = new String("aaa");
String s2 = new String("aaa");
System.out.println(s1 == s2);           // false
String s3 = s1.intern();
String s4 = s1.intern();
System.out.println(s3 == s4);     
```

 如果是采用 "bbb" 这种字面量的形式创建字符串，会自动地将字符串放入 String Pool 中。 

```java
String s5 = "bbb";
String s6 = "bbb";
System.out.println(s5 == s6);  // true
```

 在 Java 7 之前，String Pool 被放在运行时常量池中，它属于永久代。而在 Java 7，String Pool 被移到堆中。这是因为永久代的空间有限，在大量使用字符串的场景下会导致 OutOfMemoryError 错误。 

- [StackOverflow : What is String interning?](https://stackoverflow.com/questions/10578984/what-is-string-interning)
- [深入解析 String#intern](https://tech.meituan.com/in_depth_understanding_string_intern.html)

### new String("abc")

 使用这种方式一共会创建两个字符串对象（前提是 String Pool 中还没有 "abc" 字符串对象）。 

- "abc" 属于字符串字面量，因此编译时期会在 String Pool 中创建一个字符串对象，指向这个 "abc" 字符串字面量；
- 而使用 new 的方式会在堆中创建一个字符串对象。

创建一个测试类，其 main 方法中使用这种方式来创建字符串对象。

```java
public class NewStringTest {
    public static void main(String[] args) {
        String s = new String("abc");
    }
}
```

 使用 javap -verbose 进行反编译，得到以下内容： 

```
// ...
Constant pool:
// ...
   #2 = Class              #18            // java/lang/String
   #3 = String             #19            // abc
// ...
  #18 = Utf8               java/lang/String
  #19 = Utf8               abc
// ...

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=3, locals=2, args_size=1
         0: new           #2                  // class java/lang/String
         3: dup
         4: ldc           #3                  // String abc
         6: invokespecial #4                  // Method java/lang/String."<init>":(Ljava/lang/String;)V
         9: astore_1
// ...
```

在 Constant Pool 中，#19 存储这字符串字面量 "abc"，#3 是 String Pool 的字符串对象，它指向 #19 这个字符串字面量。在 main 方法中，0: 行使用 new #2 在堆中创建一个字符串对象，并且使用 ldc #3 将 String Pool 中的字符串对象作为 String 构造函数的参数。

以下是 String 构造函数的源码，可以看到，在将一个字符串对象作为另一个字符串对象的构造函数参数时，并不会完全复制 value 数组内容，而是都会指向同一个 value 数组。

```java
public String(String original) {
    this.value = original.value;
    this.coder = original.coder;//jdk1.9
    this.hash = original.hash;
}
```

## 三、运算

### 参数传递

 Java 的参数是以值传递的形式传入方法中，而不是引用传递。 

 以下代码中 Dog dog 的 dog 是一个指针，存储的是对象的地址。在将一个参数传入一个方法时，本质上是将对象的地址以值的方式传递到形参中。 

```java
public class Dog {

    String name;

    Dog(String name) {
        this.name = name;
    }

    String getName() {
        return this.name;
    }

    void setName(String name) {
        this.name = name;
    }

    String getObjectAddress() {
        return super.toString();
    }
}
```

 在方法中改变对象的字段值会改变原对象该字段值，因为引用的是同一个对象。 

```java
class PassByValueExample {
    public static void main(String[] args) {
        Dog dog = new Dog("A");
        func(dog);
        System.out.println(dog.getName());          // B
    }

    private static void func(Dog dog) {
        dog.setName("B");
    }
}
```

 但是在方法中将指针引用了其它对象，那么此时方法里和方法外的两个指针指向了不同的对象，在一个指针改变其所指向对象的内容对另一个指针所指向的对象没有影响。 

```java
public class PassByValueExample {
    public static void main(String[] args) {
        Dog dog = new Dog("A");
        System.out.println(dog.getObjectAddress()); // Dog@4554617c
        func(dog);
        System.out.println(dog.getObjectAddress()); // Dog@4554617c
        System.out.println(dog.getName());          // A
    }

    private static void func(Dog dog) {
        System.out.println(dog.getObjectAddress()); // Dog@4554617c
        dog = new Dog("B");
        System.out.println(dog.getObjectAddress()); // Dog@74a14482
        System.out.println(dog.getName());          // B
    }
}
```

### float 与 double

 Java 不能隐式执行向下转型，因为这会使得精度降低。 

 1.1 字面量属于 double 类型，不能直接将 1.1 直接赋值给 float 变量，因为这是向下转型。 

```java
// float f = 1.1;
```

 1.1f 字面量才是 float 类型。 

```java
float f = 1.1f;
```

### 隐式类型转换

 因为字面量 1 是 int 类型，它比 short 类型精度要高，因此不能隐式地将 int 类型向下转型为 short 类型。 

 但是使用 += 或者 ++ 运算符会执行隐式类型转换。 

```java
short s1 = 1;
// s1 = s1 + 1;
s1 += 1;
s1++;
```

 上面的语句相当于将 s1 + 1 的计算结果进行了向下转型： 

```java
s1 = (short) (s1 + 1);
```

### switch

 从 Java 7 开始，可以在 switch 条件判断语句中使用 String 对象。 

```java
String s = "a";
switch (s) {
    case "a":
        System.out.println("aaa");
        break;
    case "b":
        System.out.println("bbb");
        break;
}
```

 switch 不支持 long，是因为 switch 的设计初衷是对那些只有少数几个值的类型进行等值判断，如果值过于复杂，那么还是用 if 比较合适。 

## 四、关键字

### final

1. 数据

   声明数据为常量，可以是编译时常量，也可以是在运行时被初始化后不能被改变的常量。

   * 对于基本对象，final 使数值不变；
   * 对于引用类型，final 使引用不变，也就不能引用其他对象，但是被引用的对象本身是可以修改的。

2. 方法

   声明方法不能被子类重写。

   private 方法隐式地被指定为 final，如果在子类中定义地方法和基类中地一个 private 方法签名相同，此时子类地方法不是重写基类方法，而是在子类中定义了一个新的方法。

3. 类

    声明类不允许被继承。 

### static

1. 静态变量
   * 静态变量：又称为类变量，也就是说这个变量属于类的，类所有的实例都共享静态变量，可以直接通过类名来访问它。静态变量在内存中只存在一份。
   * 实例变量：每创建一个实例就会产生一个实例变量，它与该实例同生共死。

2. 静态方法

    静态方法在类加载的时候就存在了，它不依赖于任何实例。所以静态方法必须有实现，也就是说它不能是抽象方法。 

   ```java
   public abstract class A {
       public static void func1(){
       }
       // public abstract static void func2();  // Illegal combination of modifiers: 'abstract' and 'static'
   }
   ```

    只能访问所属类的静态字段和静态方法，方法中不能有 this 和 super 关键字，因此这两个关键字与具体对象关联。 

   ```java
   public class A {
   
       private static int x;
       private int y;
   
       public static void func1(){
           int a = x;
           // int b = y;  // Non-static field 'y' cannot be referenced from a static context
           // int b = this.y;     // 'A.this' cannot be referenced from a static context
       }
   }
   ```

3. 静态语句块

   [类加载](https://www.cnblogs.com/ityouknow/p/5603287.html)

   ```java
   public class A {
       static {
           System.out.println("123");
       }
   
       public static void main(String[] args) {
           A a1 = new A();
           A a2 = new A();
       }
   }
   ```

4. 静态内部类

    非静态内部类依赖于外部类的实例，也就是说需要先创建外部类实例，才能用这个实例去创建非静态内部类。而静态内部类不需要。 

   ```java
   public class OuterClass {
   
       class InnerClass {
       }
   
       static class StaticInnerClass {
       }
   
       public static void main(String[] args) {
           // InnerClass innerClass = new InnerClass(); // 'OuterClass.this' cannot be referenced from a static context
           OuterClass outerClass = new OuterClass();
           InnerClass innerClass = outerClass.new InnerClass();
           StaticInnerClass staticInnerClass = new StaticInnerClass();
       }
   }
   ```

    静态内部类不能访问外部类的非静态的变量和方法。 

5. 静态导包

   在使用静态变量和方法时不用再指明 ClassName，从而简化代码，但可读性大大降低。

   ```java
   import static com.xxx.ClassName.*
   ```

6. 初始化顺序

    静态变量和静态语句块优先于实例变量和普通语句块，静态变量和静态语句块的初始化顺序取决于它们在代码中的顺序。 

   ```java
   public static String staticField = "静态变量";
   static {
       System.out.println("静态语句块");
   }
   public String field = "实例变量";
   {
       System.out.println("普通语句块");
   }
   ```

    最后才是构造函数的初始化。 

   ```java
   public InitialOrderTest() {
       System.out.println("构造函数");
   }
   ```

   存在继承的情况下，初始化顺序为：

   - 父类（静态变量、静态语句块）
   - 子类（静态变量、静态语句块）
   - 父类（实例变量、普通语句块）
   - 父类（构造函数）
   - 子类（实例变量、普通语句块）
   - 子类（构造函数）

## 五、Object 通用方法

### 概览

```java
public native int hashCode()

public boolean equals(Object obj)

protected native Object clone() throws CloneNotSupportedException

public String toString()

public final native Class<?> getClass()

protected void finalize() throws Throwable {}

public final native void notify()

public final native void notifyAll()

public final native void wait(long timeout) throws InterruptedException

public final void wait(long timeout, int nanos) throws InterruptedException

public final void wait() throws InterruptedException
```

### equals()

1. 等价关系

   两个对象具有等价关系，需要满足以下五个条件：

   Ⅰ 自反性

   ```java
   x.equals(x); // true
   ```

   Ⅱ 对称性

   ```java
   x.equals(y) == y.equals(x); // true
   ```

   Ⅲ 传递性

   ```java
   if (x.equals(y) && y.equals(z))
       x.equals(z); // true;
   ```

   Ⅳ 一致性

   多次调用 equals() 方法结果不变

   ```java
   x.equals(y) == x.equals(y); // true
   ```

   Ⅴ 与 null 的比较

   对任何不是 null 的对象 x 调用 x.equals(null) 结果都为 false

   ```java
   x.equals(null); // false;
   ```

2. 等价与相等

   - 对于基本类型，== 判断两个值是否相等，基本类型没有 equals() 方法。
   - 对于引用类型，== 判断两个变量是否引用同一个对象，而 equals() 判断引用的对象是否等价。

   ```java
   Integer x = new Integer(1);
   Integer y = new Integer(1);
   System.out.println(x.equals(y)); // true
   System.out.println(x == y);      // f
   ```

3. 实现

- 检查是否为同一个对象的引用，如果是直接返回 true；
- 检查是否是同一个类型，如果不是，直接返回 false；
- 将 Object 对象进行转型；
- 判断每个关键域是否相等。

```java
public class EqualExample {

    private int x;
    private int y;
    private int z;

    public EqualExample(int x, int y, int z) {
        this.x = x;
        this.y = y;
        this.z = z;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;

        EqualExample that = (EqualExample) o;

        if (x != that.x) return false;
        if (y != that.y) return false;
        return z == that.z;
    }
}
```

### hashCode()

 hashCode() 返回哈希值，而 equals() 是用来判断两个对象是否等价。等价的两个对象散列值一定相同，但是散列值相同的两个对象不一定等价，这是因为计算哈希值具有随机性，两个值不同的对象可能计算出相同的哈希值。 

 在覆盖 equals() 方法时应当总是覆盖 hashCode() 方法，保证等价的两个对象哈希值也相等。 

 HashSet 和 HashMap 等集合类使用了 hashCode() 方法来计算对象应该存储的位置，因此要将对象添加到这些集合类中，需要让对应的类实现 hashCode() 方法。 

 下面的代码中，新建了两个等价的对象，并将它们添加到 HashSet 中。我们希望将这两个对象当成一样的，只在集合中添加一个对象。但是 EqualExample 没有实现 hashCode() 方法，因此这两个对象的哈希值是不同的，最终导致集合添加了两个等价的对象。 

```java
EqualExample e1 = new EqualExample(1, 1, 1);
EqualExample e2 = new EqualExample(1, 1, 1);
System.out.println(e1.equals(e2)); // true
HashSet<EqualExample> set = new HashSet<>();
set.add(e1);
set.add(e2);
System.out.println(set.size());   // 2
```

 理想的哈希函数应当具有均匀性，即不相等的对象应当均匀分布到所有可能的哈希值上。这就要求了哈希函数要把所有域的值都考虑进来。可以将每个域都当成 R 进制的某一位，然后组成一个 R 进制的整数。 

 R 一般取 31，因为它是一个奇素数，如果是偶数的话，当出现乘法溢出，信息就会丢失，因为与 2 相乘相当于向左移一位，最左边的位丢失。并且一个数与 31 相乘可以转换成移位和减法：`31*x == (x<<5)-x`，编译器会自动进行这个优化。 

```java
@Override
public int hashCode() {
    int result = 17;
    result = 31 * result + x;
    result = 31 * result + y;
    result = 31 * result + z;
    return result;
}
```

### toString()

默认返回 ToStringExample@4554617c 这种形式，其中 @ 后面的数值为散列码的无符号十六进制表示。

```java
public class ToStringExample {

    private int number;

    public ToStringExample(int number) {
        this.number = number;
    }
}
ToStringExample example = new ToStringExample(123);
System.out.println(example.toString());
//ToStringExample@4554617c
```

### clone()

1. cloneable

   clone() 是 Object 的 protected 方法，它不是 public，一个类不显式去重写 clone()，其它类就不能直接去调用该类实例 clone() 方法。

   ```java
   public class CloneExample {
       private int a;
       private int b;
   }
   CloneExample e1 = new CloneExample();
   // CloneExample e2 = e1.clone(); 
   // 'clone()' has protected access in 'java.lang.Object'
   ```

   重写 clone() 得到以下实现：

   ```java
   public class CloneExample {
       private int a;
       private int b;
   
       @Override
       public CloneExample clone() throws CloneNotSupportedException {
           return (CloneExample)super.clone();
       }
   }
   ```

   ```java
   CloneExample e1 = new CloneExample();
   try {
       CloneExample e2 = e1.clone();
   } catch (CloneNotSupportedException e) {
       e.printStackTrace();
   }
   ```

   ```java
   //java.lang.CloneNotSupportedException: CloneExample
   ```

   以上抛出了 CloneNotSupportedException，这是因为 CloneExample 没有实现 Cloneable 接口。

   因该注意的是，clone() 方法并不是 Cloneable() 接口的方法，而是 Object 的一个 protected 方法。Cloneable 接口只是规定，如果一个类没有实现 Cloneable 接口又调用了 clone() 方法，就会抛出 CloneNotSupportedException。 

   ```java
   public class CloneExample implements Cloneable {
       private int a;
       private int b;
   
       @Override
       public Object clone() throws CloneNotSupportedException {
           return super.clone();
       }
   }
   ```

2. 浅拷贝

    拷贝对象和原始对象的引用类型引用同一个对象。 

   ```java
   public class ShallowCloneExample implements Cloneable {
   
       private int[] arr;
   
       public ShallowCloneExample() {
           arr = new int[10];
           for (int i = 0; i < arr.length; i++) {
               arr[i] = i;
           }
       }
   
       public void set(int index, int value) {
           arr[index] = value;
       }
   
       public int get(int index) {
           return arr[index];
       }
   
       @Override
       protected ShallowCloneExample clone() throws CloneNotSupportedException {
           return (ShallowCloneExample) super.clone();
       }
   }
   ```

   ```java
   ShallowCloneExample e1 = new ShallowCloneExample();
   ShallowCloneExample e2 = null;
   try {
       e2 = e1.clone();
   } catch (CloneNotSupportedException e) {
       e.printStackTrace();
   }
   e1.set(2, 222);
   System.out.println(e2.get(2)); // 222
   ```

3. 深拷贝

    拷贝对象和原始对象的引用类型引用不同对象。 

   ~~~java
   public class DeepCloneExample implements Cloneable {
   
       private int[] arr;
   
       public DeepCloneExample() {
           arr = new int[10];
           for (int i = 0; i < arr.length; i++) {
               arr[i] = i;
           }
       }
   
       public void set(int index, int value) {
           arr[index] = value;
       }
   
       public int get(int index) {
           return arr[index];
       }
   
       @Override
       protected DeepCloneExample clone() throws CloneNotSupportedException {
           DeepCloneExample result = (DeepCloneExample) super.clone();
           result.arr = new int[arr.length];
           for (int i = 0; i < arr.length; i++) {
               result.arr[i] = arr[i];
           }
           return result;
       }
   }
   
   ~~~
   
   ~~~java
   DeepCloneExample e1 = new DeepCloneExample();
   DeepCloneExample e2 = null;
   try {
       e2 = e1.clone();
   } catch (CloneNotSupportedException e) {
       e.printStackTrace();
   }
   e1.set(2, 222);
   System.out.println(e2.get(2)); // 2
   ~~~

4. clone() 的代替方案

 使用 clone() 方法来拷贝一个对象即复杂又有风险，它会抛出异常，并且还需要类型转换。Effective Java 书上讲到，最好不要去使用 clone()，可以使用拷贝构造函数或者拷贝工厂来拷贝一个对象。 

```java
public class CloneConstructorExample {

    private int[] arr;

    public CloneConstructorExample() {
        arr = new int[10];
        for (int i = 0; i < arr.length; i++) {
            arr[i] = i;
        }
    }

    public CloneConstructorExample(CloneConstructorExample original) {
        arr = new int[original.arr.length];
        for (int i = 0; i < original.arr.length; i++) {
            arr[i] = original.arr[i];
        }
    }

    public void set(int index, int value) {
        arr[index] = value;
    }

    public int get(int index) {
        return arr[index];
    }
}
```

```java
CloneConstructorExample e1 = new CloneConstructorExample();
CloneConstructorExample e2 = new CloneConstructorExample(e1);
e1.set(2, 222);
System.out.println(e2.get(2)); // 2
```