# Java虚拟类加载机制

### 类加载的时机

1. 遇到new、getstatic、putstatic、invokestatic指令时如果没有初始化过，就会触发初始化。这些指令在Java代码中的体现是：通过new指令新建对象时、调用一个类的静态字段时、调用一个类的静态方法时。
2. 使用java.lang.reflect包方法对类进行反射调用时，如果类没有初始化过，就会触发初始化。
3. 当初始化一个类时，如果该类的父类没有被初始化就会先触发父类的初始化。
4. 当虚拟机启动时，会先初始化主类(含有main方法的类)。
5. 当使用JDK1.7的动态语言支持时，如果一个`java.lang.invoke.MethodHandle`实例最后的解析结果`REF_getStatic,REF_putStatic,REF_invokeStatic`的方法句柄，并且这个方法句柄所对应的类没有进行初始化，则需要先出发初始化。

**子类和父类方法的初始化顺序**

父类的静态成员变量和代码块->子类的静态成员变量和代码块->父类的代码块和普通成员变量->父类的的构造方法->子类的代码块和普通成员变量->子类的的构造方法

### 类加载的过程

![](http://www.theaze.cn/wp-content/uploads/2019/04/6874741.png)

- **加载**

1. 通过全限定名获取类的二进制字节流，来源不做限制，可以从ZIP包、网络、运算结果、其他文件、数据库等途径中获取。
2. 将字节流中的静态存储结构转换为方法区运行时的数据结构。
3. 在内存中生成一个代表这个类的java.lang.Class对象，作为方法区这个类的入口。

- **验证**（重要但不必要，如果确定程序没有问题，以通过-Xverify:none跳过验证阶段）

1. 文件格式验证：确保Class文件的文件流信息符合当前虚拟机的要求。
2. 元数据验证：对字节码描述的信息进行语义分析，确保符合Java的语言规范。
3. 字节码验证：最复杂的一个阶段，通过数据流和控制流的分析，确定程序语义是合法的、符合逻辑的，保证校验类的方法在运行时不会做出危害虚拟机安全的时间。
4. 符号引用验证：确保解析动作能够正常执行。

- **准备**

​       在方法区为类变量分配初始值(一般为0)。

- **解析**

​       在虚拟机将常量池内的符号引用替换为直接引用的过程。

- **初始化**

1. 执行`<clinit>()`方法的过程，该方法是所有类变量赋值动作和静态语句块语句合并而成的。
2. 虚拟机不需要显式的调用父类的`<clinit>()`，它会保证父类的`<clinit>()`先于子类。
3. 虚拟机会保证一个类的`<clinit>()`方法在多线程下正确的加锁与同步，如果多个线程同时去初始化一个类，而这个类的`<clinit>()`中含有耗时很长的操作，那么就会造成多个线程阻塞。

### 类加载器

> “通过一个类的全限定名描述此类的二进制字节流”这个动作要放在Java虚拟机外部实现，实现这个动作的代码模块叫做“类加载器”

每一个类加载器都有一个独立的类名称空间，来自于同一个Class文件的两个类，如果被不同的类加载器加载，那么这两个类必定是不相等的。

#### 启动类加载器

由C++实现，是虚拟机的一部分，负责加载<JAVA_HOME>\lib目录下的类。

#### 扩展类加载器

负责加载<JAVA_HOME>\lib\ext下的内容。开发者可以直接使用扩展类加载器。

#### 应用程序类加载器

开发者可以直接使用应用程序类加载器，如果应用程序中没有自定义过自己的类加载器，一般这就是程序中默认的类加载器。

#### 双亲委派模型

![](http://www.theaze.cn/wp-content/uploads/2019/04/批注-2019-04-29-161930.png)

如果一个类加载器收到了类加载的请求，它不会自己先尝试加载这个类，而是将请求委托给父类加载器去完成，因此，所有的加载请求最终都应传送到顶层的启动类加载器，只有当父加载器反馈无法完成加载请求时(在自己的搜索范围没有搜索到所需要的类)，子加载器才会去加载。

**为什么要实现双亲委派模型？**

保证类是同一个加载器加载的，不会产生混乱(同一个类被不同加载器加载不是同一个类)。

``` java
protected Class<?> loadClass(String name, boolean resolve)
    throws ClassNotFoundException
{
    synchronized (getClassLoadingLock(name)) {
        //检查该类是否已经加载过
        Class c = findLoadedClass(name);
        if (c == null) {
            //如果该类没有加载
            long t0 = System.nanoTime();
            try {
                if (parent != null) {
                    //当父类的加载器不为空，则通过父类的loadClass来加载该类
                    c = parent.loadClass(name, false);
                } else {
                    //当父类的加载器为空，则调用启动类加载器来加载该类
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                //非空父类的类加载器无法找到相应的类，则抛出异常
            }

            if (c == null) {
                //当父类加载器无法加载时，则调用findClass方法来加载该类
                long t1 = System.nanoTime();
                c = findClass(name); //用户可通过覆写该方法，来自定义类加载器

                //用于统计类加载器相关的信息
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            //对类进行link操作
            resolveClass(c);
        }
        return c;
    }
}
```

开发者可重写findClass()方法来实现双亲委派模型，但是重写loadClass()会破坏双亲委派模型。