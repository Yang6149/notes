# Java 热部署

Java 类加载器可以动态的去加载类(class字节码)。那么我们可以通过操作类加载来完成一个热部署。

## 1. 一个简单的案例

1. 由于每个类加载器只能加载同一个类一次，那么我们只需要用不同的类加载器对该类进行加载就行了。
2. 写一个循环不停创建新的类加载器来加载目标类，并反射调用方法。
3. 重写loadClass来破坏双亲委派模型(防止都让ApplicationClassLoad加载)
4. 修改目标类并编译
5. 等待循环打印出热部署后的结果。

[代码](https://gist.github.com/Yang6149/1854169a9b6ea392b4acb08d564a4885)

![1593166216691](C:\Users\hasaki\AppData\Roaming\Typora\typora-user-images\1593166216691.png)

