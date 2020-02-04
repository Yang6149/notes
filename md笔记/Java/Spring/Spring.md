# Spring

[怎么回答面试官：你对Spring的理解？](https://www.zhihu.com/question/48427693/answer/723146648)

### IoC

IoC就是所谓的控制反转 ( Inversion of Control ), 就是把原本需要程序员自己创建和维护的一大堆bean统统交由Spring管理。 

 也就是说，Spring将我们从盘根错节的依赖关系中解放了。当前对象如果需要依赖另一个对象，只要打一个@Autowired注解，Spring就会自动帮你安装上。 

![](https://pic3.zhimg.com/80/v2-c0273f7d3641d4e52f0bf4dd6d64fb92_hd.jpg)

以注解开发为例：在class前加上@Component、@Repository、@Service、@Controller

表示的不同但是功能相同，主要区分表示的是哪一(dao层或service层或控制层)，都是把类放到Spring容器里面再相应的变量上面加@Autowired

```java
@Service
public class BlogServiceImpl implements BlogService{
    @Autowired
    private BlogRepository repository;
    .....
}
```

### AOP

 通俗地讲，它一般被用来解决一些系统交叉业务的织入，比如日志啦、事务啥的。 

 做法是将切面代码移动到原始方法的周围： 

![](https://pic1.zhimg.com/80/v2-3ede7765f60cf2bacd36afe2fe874824_hd.jpg)

这里同样是以注解开发为例：

类前面加上`@Aspec`t和`@Component `开启注解扫描

在类里面定义一个方法，可以为空加上`@Pointcut("execution(* com.yang.blog.web.*.*(..))")`之类的，里面参数为作用域

在下面例子中： 比如日志打印，不再是直接硬编码在方法中的零散语句，而是做成一个切面类，通过通知方法去执行切面代码。 

```java
public class LogAspect {

    private Logger logger=LoggerFactory.getLogger(this.getClass());
    @Pointcut("execution(* com.yang.blog.web.*.*(..))")
    public void log(){}
    @Before("log()")
    public void doBefore(JoinPoint joinPoint){
        ......
        String classMethod = joinPoint.getSignature().getDeclaringTypeName()+"."+joinPoint.getSignature().getName();
        Object[] args=joinPoint.getArgs();
        RequestLog requestLog=new RequestLog(url,ip,classMethod,args);
        logger.info("Request: {}",requestLog);
    }
    @After("log()")
    public void doAfter(){
        logger.info("-------------doAfter");
    }
    @AfterReturning(returning = "result",pointcut ="log()")
    public void doAfterReturn(Object result){
        logger.info("-------------doAfterReturn {}",result);
    }
    ......
}
```



在类头部加上@Transactional启用事务，就是AOP生成的代理对象



###  BeanDefinition 

 什么BeanDefinition呢？其实它是bean定义的一个顶级接口: 

![](https://pic4.zhimg.com/80/v2-7ea61ed98aede9409e1ffec392a4f893_hd.jpg)

![](https://pic3.zhimg.com/80/v2-f54704d1f6b9ef1ee47e37e0040a9b16_hd.jpg)

 Class只是描述了一个类有哪些字段、方法，但是无法描述如何实例化这个bean！如果说，Class类描述了一块猪肉，那么BeanDefinition就是描述如何做红烧肉： 

*  单例吗？ 
*  是否需要延迟加载？ 
* 需要调用哪个初始化方法/销毁方法？



* Spring首先会扫描解析指定位置的所有的类得到Resources（可以理解为.Class文件）

  然后依照TypeFilter和@Conditional注解决定是否将这个类解析为BeanDefinition

  稍后再把一个个BeanDefinition取出实例化成Bean

  ![](https://pic4.zhimg.com/80/v2-8c6f9d8294aeffe91a8eeb711fcf83bf_hd.jpg)



## 后置处理器

在Spring中ApplicationContex 作为容器包含的是各个组件

![](https://pic2.zhimg.com/80/v2-819a67daa7540d1a7bc5828b0c8e5dc9_hd.jpg)

上文提到在事务处理上，输出的是代理对象而不是自己写的class创建的实例，这关键就在于后置处理器 。

![](https://pic1.zhimg.com/80/v2-9bd6efe7c86130553896c3744c338778_hd.jpg)

 上面BeanFactory、BeanDefinitionRegistryPostProcessor、BeanPostProcessor都算是后置处理器 

![](https://pic3.zhimg.com/80/v2-c37f32ba1b6562e05772dcad2262880a_hd.jpg)

 BeanFactoryPostProcessor是用来干预BeanFactory创建的，而BeanPostProcessor是用来干预Bean的实例化。 

如果想要在普通Bean中注入ApplicationContext实例是这样的

```
@Autowired
ApplicationContext annotationConfigApplicationContext;
```

还可以让Bean实现 ApplicationContextAware 接口：

![](https://pic3.zhimg.com/80/v2-4a629f6860a9230a64b248d70c0b34c6_hd.jpg)

Bean在实现这个接口后，Spring会遍历所有 BeanPostProcessor  其中就包括处理实现了ApplicationContextAware接口的bean的后置处理器：ApplicationContextAwareProcessor。 

![](https://pic1.zhimg.com/80/v2-b4cc010925170b3641f93c9c1ce57458_hd.jpg)

也就是说，要扩展的类是不确定的，但扩展的流程是确定的，在Bean实例化的某一处，必然要经过很多BeanPostProcessor  

此时实现该接口的意义为：

* 作为后置处理器的判断依据，只有实现了该接口才会去调用
* 提供后置处理器需要调用的方法

![](https://pic1.zhimg.com/v2-dc8000e96551247ddb182edc8f875e1f_r.jpg)