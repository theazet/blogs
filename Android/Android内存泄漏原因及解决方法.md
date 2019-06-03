# Android内存泄漏

> **什么是内存泄漏?**
> Java语言不需要手动释放申请的资源，资源的回收由系统自动进行，Java回收机制可以看我的文章[JAVA内存回收机制详解]([https://www.theaze.cn/2019/04/04/java%E5%86%85%E5%AD%98%E5%9B%9E%E6%94%B6%E6%9C%BA%E5%88%B6%E8%AF%A6%E8%A7%A3/](https://www.theaze.cn/2019/04/04/java内存回收机制详解/)),而如果在使用完毕后却无法释放，一直占据着内存空间，这就是所谓的内存泄漏。随着程序的不断运行，可用的内存不断减少(每一个进程都有其内存限制，一般是十几到几十M)，那么程序就会变得不稳定，直至出现OOM错误。

## 会导致内存泄漏的操作

### 一、static Activity

如果静态的Activity变量引用了当前Activity的实例，且这个静态Activity变量没有被清空，那么在应用的整个生命周期中，这个静态Activity变量都会引用着当前Activity，当前Activity就算结束了也不会被回收，这样就会发生内存泄漏。

```java
public class MyActivity extends AppCompatActivity {
	static Activity activity;
    void getReference(){
        activity=this;
    }
}
```

### 二、static Views

 和上边的static Activity的原理一样，强行延长了View的生命周期，无法释放资源。

```java
public class MyActivity extends AppCompatActivity {
	static View view;
    private Button button
    void getReference(){
        view=button;
    }
}
```

### 三、not-static Inner Classes

如果一个Activity中拥有一个非静态内部类或匿名类，且持有一个静态引用，这样就会发生内存泄漏。而如果在Activity退出之后，内部类中如果仍有方法在执行，那么仍可能发生内存泄漏。它们产生内存泄漏的原因是内部类和匿名类都会持有一个外部类的隐性引用，当Activity退出后，隐性调用依旧存在，无法回收。

```java
public class MyActivity extends AppCompatActivity {
	static InnerClass object;
    class InnerClass{}
    void getReference(){
        object=new InnerClass();
    }
}
```

```java
public class MyActivity extends AppCompatActivity {
    class InnerClass{
        public void test(){
    		new Thread(new Runnable() {
                @Override
                public void run() {
                    for (;;) {
                        System.out.println("1s");
                        try {
                            Thread.sleep(1000);
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                }
            }).start();
        } 
    }
    void getReference(){
        new InnerClass().test();
    }
}
```

### 四、注册系统服务

通过Context调用getSystemService获取系统服务，如果Context对象需要在一个Service内部事件发生时随时收到通知，则需要把自己作为一个监听器注册进去，这样服务就会持有一个Activity，如果忘记了在Activity被销毁前注销这个监听器，这样就导致内存泄漏。

```java
void registerListener() {
       SensorManager sensorManager = (SensorManager) getSystemService(SENSOR_SERVICE);
       Sensor sensor = sensorManager.getDefaultSensor(Sensor.TYPE_ALL);
       sensorManager.registerListener(this, sensor, SensorManager.SENSOR_DELAY_FASTEST);
}
```

### 五、无限循环属性动画

如果定义了无限循环的属性动画，如果没有cancel()掉，那么动画会依旧运行，发生内存泄漏。解决的方法是在onDestroy方法中调用animator.cancel()来停止动画。

```java
animator.setRepeatCount(ValueAnimator.INFINITE);
```

