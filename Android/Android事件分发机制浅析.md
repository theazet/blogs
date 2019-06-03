# Android事件分发机制浅析

### 分发机制基础

Android把一系列的点击时间封装成为了MotionEvent对象：

- ACTION_DOWN：按下
- ACTION_UP：抬起
- ACTION_MOVE：移动
- ACTION_CANCEL：取消

有关事件分发的方法：

- dispatchTouchEvent：所有的事件都需要从此事件开始分发，决定事件是由自己处理还是由子view处理
- onInterceptTouchEvent：ViewGroup独有的事件，默认为false，当返回true时表示拦截事件，不在向子view传递，由自己的处理
- onTouchEvent：处理事件，返回true代表消费事件

伪代码：

```java
public boolean dispatchTouchEvent(MotionEvent ev){
    boolean consume = false;
    if(onInterceptTouchEvent(ev)){ //拦截事件则不向子view传递
        consume=onTouchEvent(ev);  //onTouchEvent返回true则代表已经消费
    }else{					//不拦截则继续向下传递
        consume=child.dispatchTouchEvent(ev);  
    }
    return consume;
}
```

### 分发顺序

由于View是树形结构，那么我们就可以根据结构特点进行有序的分发。

事件分发是从Activity开始的：`Activity->PhoneWindow->DecorView->ViewGroup->View`

如果没有消费事件，那么它会回传：`View->ViewGroup->DecorView->PhoneWindow->Activity`

这就是一种典型的**责任链模式**。

**PhoneWindow和DecorView我们平时没有怎么接触过，它们是什么呢？**

```java
	public boolean dispatchTouchEvent(MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            onUserInteraction();
        }
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        return onTouchEvent(ev);
    }
```

这是Activity中的dispatchTouchEvent方法，也是事件分发的源头，我们发现它调用的是`getWindow().superDispatchTouchEvent(ev)`，我们都知道Window的唯一实现类是PhoneWindow，那么这getWindow()就是PhoneWindow的对象。

```java
    public boolean superDispatchTouchEvent(MotionEvent event) {
        return mDecor.superDispatchTouchEvent(event);
    }

```

这是PhoneWindow的superDispatchTouchEvent方法,PhoneWindow中又调用了Decor的superDispatchTouchEvent方法，DecorView是PhoneWindow的内部类，它继承自FrameLayout，是Activity视图树的根节点视图，是顶级的View。

```java
    public boolean superDispatchTouchEvent(MotionEvent event) {
        return super.dispatchTouchEvent(event);
    }
```

这是DecorView的superDispatchTouchEvent方法，因为它继承自FrameLayout，间接继承自ViewGroup，所以自此事件被传递到ViewGroup，ViewGroup会根据自己的视图树依次向下分发。

### 事件分发相关细节

```java
public boolean dispatchTouchEvent(MotionEvent event) {
    if (mOnTouchListener != null && (mViewFlags & ENABLED_MASK) == ENABLED &&
            mOnTouchListener.onTouch(this, event)) {
        return true;
    }
    return onTouchEvent(event);
}
```

这是在View中dispatchTouchEvent方法的源码，以上源码我们可以知道，如果已经注册了OnTouchListener，且当前View的状态时enabled，onTouch方法返回的是true，那么就直接返回true不再调用onTouchEvent。onClick与OnLongClick事件都是在onTouchEvent()方法中的，所以如果在onTouch方法返回了true且View为enabled，那么onClick与OnLongClick都会无效。

View的onTouchEvent默认会消耗事件，默认返回为true。