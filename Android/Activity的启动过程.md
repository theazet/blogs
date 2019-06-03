# Activity的启动过程

由于源码过长，这里只贴出重要的代码

使用启动Activity的方法最终都会调用`startActivityForResult`方法

```java
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
        @Nullable Bundle options) {
    if (mParent == null) {
        Instrumentation.ActivityResult ar =
            mInstrumentation.execStartActivity(
                this, mMainThread.getApplicationThread(), mToken, this,
                intent, requestCode, options);
    } else {
		//交由Parent Activity处理
    }
}
```

上述代码中的mParent是什么呢？它代表的是ActivityGroup，它已经被废弃，推荐采用Fragment来替代ActivityGroup。Activity的启动委托给了`Instrumentation`类来实现

```java
public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        if (mActivityMonitors != null) {
            //Activity的插件化会用到，此处不做讨论
        }
        try {
            //委托给ActivityManagerService(AMS)
            int result = ActivityManager.getService()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
            //检查结果,提示失败原因
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }
```

将任务委托给了`ActivityManagerService`的Binder代理类实现

```java
public static IActivityManager getService() {
    return IActivityManagerSingleton.get();  
}

private static final Singleton<IActivityManager> IActivityManagerSingleton =          
        new Singleton<IActivityManager>() {                                           
            @Override                                                                 
            protected IActivityManager create() {  
                //通过IPC机制直接与 system_server 进程上的AMS通信
                final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
                final IActivityManager am = IActivityManager.Stub.asInterface(b);     
                return am;                                                            
            }                                                                         
        };
```

IPC机制相关知识可以参考我的[android中的IPC机制详解]([https://www.theaze.cn/2019/02/23/android%E4%B8%AD%E7%9A%84ipc%E6%9C%BA%E5%88%B6%E8%AF%A6%E8%A7%A3/](https://www.theaze.cn/2019/02/23/android中的ipc机制详解/))，此时Activity的启动委托给了`ActivityManagerService`。

在AMS中绕一大圈后启动过程最终会到ActivityThread的内部类`ApplicationThread`中，`ApplicationThread`发送一个Handler

收到Handler之后，ActivityThread的`handleLaunchActivity()`来开始执行Activity的启动，在该方法中调用了`performLaunchActivity`方法完成了Activity对象的创建和最终启动。

在`performLaunchActivity`它会做这几件事：

1. 获取Activity的组件信息

```java
        ActivityInfo aInfo = r.activityInfo;
        if (r.packageInfo == null) {
            r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                    Context.CONTEXT_INCLUDE_CODE);
        }

        ComponentName component = r.intent.getComponent();
        if (component == null) {
            component = r.intent.resolveActivity(
                mInitialApplication.getPackageManager());
            r.intent.setComponent(component);
        }

        if (r.activityInfo.targetActivity != null) {
            component = new ComponentName(r.activityInfo.packageName,
                    r.activityInfo.targetActivity);
        }
```

2. 通过`Instrumentation`的newActivity方法使用类加载器创建Activity对象

```java
		Activity activity = null;
        try {
            java.lang.ClassLoader cl = appContext.getClassLoader();
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to instantiate activity " + component
                    + ": " + e.toString(), e);
            }
        }
```

3. 尝试通过LoadAPK的makeApplication方法创建Application,一个进程只有一个Application对象，如果已经创建就不再调用

```java
         Application app = r.packageInfo.makeApplication(false, mInstrumentation);
```

4. 创建ContextImpl对象，并通过Activity的attach方法完成一些重要的初始化，ContextImpl是Context的重要实现类，ContextImpl通过Activity的attach方法和Activity建立联系，attach方法里还会创建window并建立和Activity的联系，这样在window接收到的事件就会传递到Activity了。

```java
		appContext.setOuterContext(activity);
        activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback);
```

5. 调用Activity的onCreate方法，至此Activity就开始了它的生命周期。