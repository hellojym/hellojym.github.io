# LeakCanary原理浅析
LeakCanary是Android内存泄漏的框架，作为一个“面试常见问题”，它一定有值得学习的地方，今天我们就讲一下它。作为一名开发，我觉得给人讲框架或者库的原理，最好先把大概思路给读者讲一下，这样读者后面会按照这个框架往里填内容，理解起来也更容易一些，所以我先把LeakCanary的大致原理放出来：

其思路大致为：监听Activity生命周期-&gt;onDestroy以后延迟5秒判断Activity有没有被回收-&gt;如果没有回收,调用GC，再此判断是否回收，如果还没回收，则内存泄露了，反之，没有泄露。整个框架最核心的问题就是在什么时间点如何判断一个Activity是否被回收了。

下面开始按这个思路分析源码，直接从入口开始：

```
public static RefWatcher install(Application application) {
  return refWatcher(application).listenerServiceClass(DisplayLeakService.class)
      .excludedRefs(AndroidExcludedRefs.createAppDefaults().build())
      .buildAndInstall();
}
```

builder模式构建了一个RefWatcher对象,`listenerServiceClass()`方法绑定了一个后台服务`DisplayLeakService`

这个服务主要用来分析内存泄漏结果并发送通知。你可以继承并重写这个类来进行一些自定义操作，比如上传分析结果等。

我们看最后buildAndInstall\(\)方法：

```
  public RefWatcher buildAndInstall() {
    RefWatcher refWatcher = build();
    if (refWatcher != DISABLED) {
      LeakCanary.enableDisplayLeakActivity(context);
      ActivityRefWatcher.installOnIcsPlus((Application) context, refWatcher);
    }
    return refWatcher;
  }
```

build\(\)方法，这个方法主要是配置一些东西，先大概了解一下，后面用到再说，下面是几个配置项目。

`watchExecutor` : 线程控制器，在 onDestroy\(\)之后并且主线程空闲时执行内存泄漏检测

`debuggerControl`: 判断是否处于调试模式，调试模式中不会进行内存泄漏检测

`gcTrigger`: 用于GC

`watchExecutor`首次检测到可能的内存泄漏，会主动进行GC,GC之后会再检测一次，仍然泄漏的判定为内存泄漏，进行后续操作

`heapDumper`: dump内存泄漏处的heap信息，写入hprof文件

`heapDumpListener`: 解析完hprof文件并通知DisplayLeakService弹出提醒

`excludedRefs`: 排除可以忽略的泄漏路径


**LeakCanary.enableDisplayLeakActivity\(context\)**

这行代码主要是为了开启LeakCanary的应用，显示其图标.

接下来是重点：

**ActivityRefWatcher.installOnIcsPlus\(\(Application\) context, refWatcher\)**

它会进入：

```
    ActivityRefWatcher activityRefWatcher = new ActivityRefWatcher(application, refWatcher);
    activityRefWatcher.watchActivities();
```

接下来：

```
    stopWatchingActivities();
    application.registerActivityLifecycleCallbacks(lifecycleCallbacks);
```

第一行代码是为了确保不会重复绑定，第二行绑定生命周期，之后监听Activity的生命周期。

```
        @Override public void onActivityDestroyed(Activity activity) {
          ActivityRefWatcher.this.onActivityDestroyed(activity);
        }
```

监听到Activity销毁时执行onActivityDestroyed方法,进入看看：

```
public void watch(Object watchedReference, String referenceName) {
    if (this == DISABLED) {
      return;
    }
    checkNotNull(watchedReference, "watchedReference");
    checkNotNull(referenceName, "referenceName");
    final long watchStartNanoTime = System.nanoTime();
    String key = UUID.randomUUID().toString();
    retainedKeys.add(key);
    final KeyedWeakReference reference =
        new KeyedWeakReference(watchedReference, key, referenceName, queue);

    ensureGoneAsync(watchStartNanoTime, reference);
  }
```

整个LeakCanary最核心的思路就在这儿了。

前面几行是这样的，根据Activity生成一个随机Key,并将Key加入到一个Set中，然后讲key，activity传如一个包装的弱引用里。

**这里引出了第一个知识点，弱引用和引用队列ReferenceQueue联合使用时，如果弱引用持有的对象被垃圾回收，Java虚拟机就会把这个弱引用加入到与之关联的引用队列中。即 KeyedWeakReference持有的Activity对象如果被垃圾回收，该对象就会加入到引用队列queue,我们看看RefreceQueue的javadoc：**

```
/**
 * Reference queues, to which registered reference objects are appended by the
 * garbage collector after the appropriate reachability changes are detected.
 *
 * @author   Mark Reinhold
 * @since    1.2
 */
public class ReferenceQueue<T>
```

证实了上面的说法,另外看名字我们就知道，不光弱引用，软和虚引用也可以这样做。

重点是最后一句:ensureGoneAsyc，看字面意思，异步确保消失。这里我们先不看代码，如果要自己设计一套检测方案的话，怎么想？其实很简单，就是在Activiy onDestroy以后，我们等一会，检测一下这个Acitivity有没有被回收,那么问题来了，什么时候检测？怎么检测？这也是本框架的核心和难点。

**LeakCanary是这么做的：onDestroy以后，一旦主线程空闲下来，延时5秒执行一个任务：先判断Activity有没有被回收？如果已经回收了，说明没有内存泄漏，如果还没回收，我们进一步确认，手动触发一下gc，然后再判断有没有回收，如果这次还没回收，说明Activity确实泄漏了，接下来把泄漏的信息展示给开发者就好了。**

思路其实挺清晰的，我们看代码实现：

```
  private void ensureGoneAsync(final long watchStartNanoTime, final KeyedWeakReference reference) {
    watchExecutor.execute(new Retryable() {
      @Override public Retryable.Result run() {
        return ensureGone(reference, watchStartNanoTime);
      }
    });
  }
```

这里watchExecutor是AndroidWatchExecutor,看代码：

```
  @Override public void execute(Retryable retryable) {
    if (Looper.getMainLooper().getThread() == Thread.currentThread()) {
      waitForIdle(retryable, 0);
    } else {
      postWaitForIdle(retryable, 0);
    }
  }
```

主线程和子线程其实一样，都要到主线程中执行，

```
  void waitForIdle(final Retryable retryable, final int failedAttempts) {
    // This needs to be called from the main thread.
    Looper.myQueue().addIdleHandler(new MessageQueue.IdleHandler() {
      @Override public boolean queueIdle() {
        postToBackgroundWithDelay(retryable, failedAttempts);
        return false;
      }
    });
  }
```

**这里有第二个知识点，IdleHandler，这个东西是干嘛的，其实看名字就知道了，就是当主线程空闲的时候，如果设置了这个东西，就会执行它的queueIdle\(\)方法，所以这个方法就是在onDestory以后，一旦主线程空闲了，就会执行，然后我们看它执行了啥：**

```
  private void postToBackgroundWithDelay(final Retryable retryable, final int failedAttempts) {
    long exponentialBackoffFactor = (long) Math.min(Math.pow(2, failedAttempts), maxBackoffFactor);
    long delayMillis = initialDelayMillis * exponentialBackoffFactor;
    backgroundHandler.postDelayed(new Runnable() {
      @Override public void run() {
        Retryable.Result result = retryable.run();
        if (result == RETRY) {
          postWaitForIdle(retryable, failedAttempts + 1);
        }
      }
    }, delayMillis);
  }
}
```

很简单，延时5秒执行retryable的run\(\)，注意，因为这里是backgroundHandler post出来的，所以是下面的run是在子线程执行的。这里的retryable就是前面传过来的:

```
  private void ensureGoneAsync(final long watchStartNanoTime, final KeyedWeakReference reference) {
    watchExecutor.execute(new Retryable() {
      @Override public Retryable.Result run() {
        return ensureGone(reference, watchStartNanoTime);
      }
    });
  }
```

ensureGone\(reference,watchStartNanoTime\),在看它干了啥之前，我们先理一下思路，前面onDestory以后，AndroidWatchExecutor这个东西执行excute方法，这个方法让主线程在空闲的时候发送了一个延时任务，该任务会在5秒延时后在一个子线程执行。理清了思路，我们看看这个任务是怎么执行的。

```
 Retryable.Result ensureGone(final KeyedWeakReference reference, final long watchStartNanoTime) {
    long gcStartNanoTime = System.nanoTime();
    long watchDurationMs = NANOSECONDS.toMillis(gcStartNanoTime - watchStartNanoTime);

    removeWeaklyReachableReferences();

    if (debuggerControl.isDebuggerAttached()) {
      // The debugger can create false leaks.
      return RETRY;
    }
    if (gone(reference)) {
      return DONE;
    }
    gcTrigger.runGc();
    removeWeaklyReachableReferences();
    if (!gone(reference)) {
      long startDumpHeap = System.nanoTime();
      long gcDurationMs = NANOSECONDS.toMillis(startDumpHeap - gcStartNanoTime);

      File heapDumpFile = heapDumper.dumpHeap();
      if (heapDumpFile == RETRY_LATER) {
        // Could not dump the heap.
        return RETRY;
      }
      long heapDumpDurationMs = NANOSECONDS.toMillis(System.nanoTime() - startDumpHeap);
      heapdumpListener.analyze(
          new HeapDump(heapDumpFile, reference.key, reference.name, excludedRefs, watchDurationMs,
              gcDurationMs, heapDumpDurationMs));
    }
    return DONE;
  }
```

前面我们说过思路了，5秒延迟后先看看有没有回收，如果回收了，直接返回，没有发生内存泄漏，如果没有回收，触发GC，gc完成后，在此判断有没有回收，如果还没回收，说明泄漏了，收集泄漏信息，展示给开发者。而上面的代码完全按照这个思路来的。其中，removeWeaklyRechableReferences\(\)和gone\(reference\)这两个方法配合，用来判断对象是否被回收了,看代码：

```
  private void removeWeaklyReachableReferences() {
    // WeakReferences are enqueued as soon as the object to which they point to becomes weakly
    // reachable. This is before finalization or garbage collection has actually happened.
    KeyedWeakReference ref;
    while ((ref = (KeyedWeakReference) queue.poll()) != null) {
      retainedKeys.remove(ref.key);
    }
  }
```

通过知识点1知道：被回收的对象都会放到设置的引用队列queue中,我们从queue中拿出所有的ref，根据他们的key匹配retainedKeys集合中的元素并删除。然后在gone\(\)函数里面判断key是否被移除.

```
  private boolean gone(KeyedWeakReference reference) {
    return !retainedKeys.contains(reference.key);
  }
```

**这个方法挺巧妙的，retainedKeys集合了所有destoryed了的但没有被回收的Activity的key，这个集合可以用来判断一个Activity有没有被回收，但是判断之前需要用removeWeaklyReachableReferences\(\)这个方法更新一下。**

一旦一个Activity检测出泄漏了，就收集泄漏信息然后通过前面配置的DisplayLeakService通知给用户并展示在DisplayLeakActivity中，后面的东西都是UI展示东西，就不是本文的重点了，有兴趣的可以自己查看。

稍微总结一下，我觉得这个框架中用到的一个很重要但冷门技巧就是弱引用的构造方法：传入一个RefrenceQueue，可以记录被垃圾回收的对象引用。说个题外话，一个对象都被回收了，他的弱引用咋办，总不能一直留着吧，（引用本身也是一个强引用对象，不要把引用和引用的对象搞混了，对象可以被回收了，但是它的引用，包括软，弱，虚引用都可以继续存在）。完全不用担心，这个引用在无用之后也会被GC回收的。

以上就是所有内容了，可以看出来LeakCanary其实算是个比较简单的库了～


