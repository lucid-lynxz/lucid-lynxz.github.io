# LifeCycle解析

> > 之前在 [掘金](https://juejin.im/post/6844903838474993677) 发布过, 重新整理在这里

> 之前用过一些android架构组件,但也仅限于api调用,知其然也该知其所以然,所以尝试了解下其源码实现,后续会出livedata,viewmodel的;
>
> 本文基于: macOS 10.13/AS 3.3.1/support-v7 28.0.0/lifecycle 1.1.1

[demo项目](https://github.com/lucid-lynxz/BlogSamples/tree/master/LifeCycleDemo)

## 使用方法简介

以前拆分业务逻辑到独立的 `presenter` 中时,需要重写 `Activity`/`Fragment` 各生命周期,然后告知 `presenter`, 写起来麻烦, 有没有比较简单的方式能把这些"脏活"给处理掉呢?

我们看看 `Lifecycle` 架构组件的写法:
```kotlin
// MainActiviy.kt
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        // 注册一个Observer即可,不需要重写各个生命周期方法
        lifecycle.addObserver(MainActObserver())
    }
}

// MainActObserver.kt
class MainActObserver : LifecycleObserver {
    @OnLifecycleEvent(Lifecycle.Event.ON_CREATE)
    fun onCreate() {
        Logger.d("MainActObserver $this onCreate")
    }
    
    @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
    fun onResume() {
        Logger.d("MainActObserver onResume")
    }
    // 其他生命周期回调,此处省略
}
```

可以看到就简单一句 `lifecycle.addObserver(MainActObserver())` 就完成了 `Activity` 各生命周期的监听;
> P.S. 由于 Android Studio 创建项目时默认导入了 support 的 appcompat-v7 包,已经把 Lifecycle 相关代码导入进来了, 因此我们可以直接使用,不需要额外添加依赖;

以上明显是观察者模式: 
1. `Lifecycle` 作为被观察对象,持有生命周期状态信息;
2. `Activity/Fragment` 是 `Lifecycle` 的宿主, 抽象为: `LifecycleOwner`, 会将自身生命周期的变化告知 `lifecycle`, 后者更新当前状态值(`State`);
3. 原 `presenter` 可当作观察者,记为: `LifecycleObserver`;

![lifecycle-observer-mode](https://user-gold-cdn.xitu.io/2019/3/3/1694375994eb5ba7?w=912&h=448&f=png&s=39669)


## Lifecycle 类解析

`Lifecycle` 类需要对各生命周期事件作出反应, 然后保存当前宿主的状态,最后通知各 `Observer`:
我们看下源码:
```java
// Lifecycle.java
public abstract class Lifecycle {
    @MainThread
    public abstract void addObserver(@NonNull LifecycleObserver observer);

    @MainThread
    public abstract void removeObserver(@NonNull LifecycleObserver observer);

    @MainThread
    @NonNull
    public abstract State getCurrentState();

    // 生命周期事件
    public enum Event {
        ON_CREATE, ON_START, ON_RESUME, ON_PAUSE, ON_STOP, ON_DESTROY, ON_ANY
    }

    // 生命周期状态
    public enum State {
        // 以下均已 Activity 为例,介绍各状态值,具体请看源码注释
        DESTROYED, // 在 onDestroy() 前即被标记为 DESTROYED 状态
        INITIALIZED, // 对象创建后但尚未收到 onCreate() 通知之前
        CREATED,// onCreate() 之后, onStop() 之前
        STARTED, // onStart() 之后 ,onPause() 之前
        RESUMED; // onResume() 之后
        public boolean isAtLeast(@NonNull State state) {
            return compareTo(state) >= 0;
        }
    }
}
```

可以发现 `Lifecycle` 只是个抽象类,而且没有对生命周期的变化做出响应的方法, 因此应该有个实现类对生命周期事件作出处理;
我们通过 AS 提供 `Type Hierarchy` 功能(`ctrl+h`)得知 `LifecycleRegistry` 类,并得到如下类图:

![lifecycle](https://user-gold-cdn.xitu.io/2019/3/3/169438b857e2addf?w=1640&h=644&f=png&s=120181)




