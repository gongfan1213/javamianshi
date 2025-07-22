# 社群真题1：元空间会产生内存溢出么？在什么情况下会产生内存溢出？
具体问题：元空间会产生内存溢出么？在什么情况下会产生内存溢出？。

java8  及以后的版本使用Metaspace来代替永久代，Metaspace是方法区在HotSpot中的实现，它与永久代最大区别在于，

Metaspace并不在虚拟机内存中而是使用本地内存也就是在JDK8中,classe

metadata(the virtual machines internal presentation of Java  class),被存储在叫做Metaspace的
native memory.

永久代（java 8 后被元空间Metaspace取代了）存放了以下信息：

- 虚拟机加载的类的信息
- 常量池
- 静态变量
- 即时编译后的代码

### 出现问题原因
错误的主要原因, 是加载到内存中的 class 数量太多或者体积太大。

### 解决办法
增加 Metaspace 的大小
```
-XX:MaxMetaspaceSize=521m
```
### 代码演示
模拟Metaspace空间溢出，我们不断生成类往元空间灌，类占据的空间是会超过Metaspace指定的空间大小的
查看元空间大小

```
java -XX:+PrintFlagsInitial
```

<img width="606" height="149" alt="image" src="https://github.com/user-attachments/assets/f50cd038-0463-4eb1-bdf4-082e7450f0b8" />
