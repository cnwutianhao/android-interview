# 为什么在子线程中创建 Handler 会抛异常？

Android 的 UI 框架设计为单线程模型，即所有的 UI 操作必须在主线程（也叫 UI 线程）中执行。为了方便在后台线程中执行耗时操作后更新 UI，Android 提供了 Handler 和 Looper 机制来协助线程之间的通信。

`Handler` 是用于处理线程间通信的一种机制，它允许你发送和处理 `Message` 和 `Runnable` 对象与一个线程的 `MessageQueue` 关联。每个 `Handler` 实例都会绑定到创建它的线程的 `Looper` 上。`Looper` 是用来循环取出 `MessageQueue` 中的消息，并分发给对应 `Handler` 处理的。

## 主线程与子线程的区别

在 Android 应用程序启动时，系统会为主线程（UI 线程）准备好 `Looper` 和 `MessageQueue`。因此，在主线程创建 `Handler` 时不会有问题，因为它已经有了一个可用的 `Looper`。

```java
/frameworks/base/core/java/android/app/ActivityThread.java

public final class ActivityThread {

    public static void main(String[] args) {
        ...

        // 设置主线程的 Looper
        Looper.prepareMainLooper();

        ...
    }
}
```

```java
/frameworks/base/core/java/android/os/Looper.java

public final class Looper {

    public static void prepareMainLooper() {
        ...

        synchronized (Looper.class) {
            ...

            sMainLooper = myLooper();
        }
    }
}
```

然而，对于子线程，默认情况下是不会创建 `Looper` 的，如果尝试在一个子线程中创建一个 `Handler` 而该线程没有初始化 `Looper`，将会抛出一个异常。

```
java.lang.RuntimeException: Can't create handler inside thread that has not called Looper.prepare()
```

## 异常原因分析

在 `Handler` 的构造函数中，获取当前线程的 `Looper`，用来将任务或消息发送到它所关联的 `Looper` 的消息队列中，并能处理发送过来的消息。如果没有 `Looper`，`Handler` 就没有消息队列可以发送或处理消息。

```java
/frameworks/base/core/java/android/os/Handler.java

public class Handler {

    public Handler(@Nullable Callback callback, boolean async) {
        ...

        // 获取当前线程的 Looper
        mLooper = Looper.myLooper();
        if (mLooper == null) {
            // 如果 Looper 为空，则抛出异常
            throw new RuntimeException(
                "Can't create handler inside thread " + Thread.currentThread()
                        + " that has not called Looper.prepare()");
        }
        // 获取与 Looper 关联的消息队列
        mQueue = mLooper.mQueue;
        
        ...
    }
}
```

```java
/frameworks/base/core/java/android/os/Looper.java

public final class Looper {

    public static @Nullable Looper myLooper() {
        // 获取当前线程的 Looper，如果当前线程没有调用 prepare()，则返回 null
        return sThreadLocal.get();
    }
}
```

子线程默认不会创建 `Looper` 对象，当在子线程中尝试创建 `Handler` 之前，如果没有先调用 `Looper.prepare()` 初始化 `Looper`，就会由于找不到绑定的 `Looper`，`Handler` 是无法知道将消息或者 `Runnable` 发送到哪里的，因此会抛出 `RuntimeException`。

`Looper.prepare()` 方法的作用是为当前线程初始化一个 `Looper` 对象。

```java
/frameworks/base/core/java/android/os/Looper.java

public final class Looper {

    private static void prepare(boolean quitAllowed) {
        // 如果当前线程已经有 Looper，则抛出异常
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        // 初始化一个 Looper 并设置给当前线程
        sThreadLocal.set(new Looper(quitAllowed));
    }
}
```