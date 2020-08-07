# 容器

Spring 提供了两种容器类型：

BeanFactory 和 ApplicationContext

### BeanFactory

基础类型IoC容器，默认懒加载。容器初始化速度快，资源少。

### ApplicationContext

基于BeanFactory 实现，支持时间发布、国际化信息支持等。默认容器启动时全部初始化，初始化慢、花费资源多。



BeanDefinition 是用来描述一个Bean的信息的接口

BeanDefinitionRegistry是用来注册(保存)一个bean信息的接口

![1594626857685](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202007/13/155419-436921.png)

# 容器组装

![1594633439006](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202007/13/174401-20545.png)

### 1. 容器启动阶段

加载configuration metadata，

1. 代码方式
2. BeanDefinitionReader 对 configuration metadata进行解析和分析，生成BeanDefinition，然后把这些保存了Bean信息的 BeanDefinition 注册到 BeanDefinitionRegistry。



### 2. Bean 实例化阶段

经过第一阶段，所有的bean信息都通过BeanDefinition的方式注册到了BeanDefinitionRegistry中。当某个请求通过容器的getBean方法明确地请求某个对象，或者因依赖关系需要隐式地调用getBean时，触发实例化阶段。

容器会首先检查该Bean是否已经初始化，如果没有，就根据BeanDefinition所提供地信息实例化被请求对象，并为其注入依赖。

# Bean 的一生

![1594635929486](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202007/13/182530-722534.png)

### 1. Bean的实例化与BeanWrapper

通过反射相应的bean实例，或CGLIB动态字节码动态生成其子类。

返回一个BeanWrapper 对象，作用是对某个Bean 进行包裹，然后对这个Bean进行操作，如设置或获取bean的相应属性值

### 2. 各色的Aware接口

检查该对象是否实现一系列以Aware命名结尾的接口定义。如果有的话就接口中规定的依赖注入给当前对象实例

![1594638754634](C:\Users\hasaki\AppData\Roaming\Typora\typora-user-images\1594638754634.png)

### 3. BeanPostProcessor

包含两个方法`Object postProcessBeforeInitialization(Object bean ,String beanName)`和`Object postProcessAfterInitialization(Object bean ,String beanName)`分别迁至处理和后置处理会执行的方法

ApplicationContext 对应的检测各种Aware，就是在这个前置处理器里进行的。

也是在这里，进行AOP

### 4. InitializingBean 和 init-method

![1594639462211](C:\Users\hasaki\AppData\Roaming\Typora\typora-user-images\1594639462211.png)

调用过前置处理后，会检测对象是否实现了这个接口。如果是，就调用方法。进一步的调整对象状态

但是直接实现这个方法就显得代码入侵性太大，所以代替方法是<bean> 中的 init-method

可以在 beans 标签中统一设置初始化方法

### 5. DisposableBean 与 destory-method

检查 singleton 类型的bean实例，看其是否实现了 DisposableBean 接口，如果是就会为该实例注册一个用于对象销毁的回调，以便在这些singleton类型的对象实例销毁前，执行销毁逻辑。

**其实和4 一样。**

在BeanFactory中，我们需要自己调用destorySingletons()方法，不然之前实现的接口就形同虚设

在ApplicationContext中，AbstratApplicationContext 有registerShutdownHook()方法，底层调用Runtime类的addShutdownHook()方法，在虚拟机退出前会执行。