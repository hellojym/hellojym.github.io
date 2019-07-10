## 简述   
Activity启动流程和Binder的文章很多，但对于AMS如何“指挥”客户端的细节讲的都不太清楚，本文会着重讲这块。Activity的启动流程虽然调用比较复杂，主要复杂在Activity栈管理等细节方面，本文不去细究ActivityStack和ActivitySupervisor的调用流程，仅仅从AMS(ActivityManagerService简称)和App跨进程角度去理解二者的交互，也加深对Binder的理解，阅读本文前建议了解一下AIDL，对里面的Stub,Proxy等类略有印象。
### Binder简介
如果对binder有一定了解的人会知道，跨进程的C和S端分别通过二者在对方的代理去互相调用对方的方法，对于Binder，初学的人会对里面的概念比较模糊，因为看起来确实有些绕，我在这儿写几点帮助理解。
- Binder是一个类,继承了IBinder，该接口表明了能够跨进程的能力，核心方法就是transact方法，这个东西是Binder驱动和用户空间使用binder的入口
- AIDL里面有几个核心的类，刚开始看需要反复记忆理解一下，否则看别的代码容易混淆。这里总结一下，一个类继承了Stub类，表示这个类是远程服务端，Stub类有个asInterface的静态方法，这个方法用在拿到远程Binder(驱动传过来的)对象时，将该远程binder转化成client端可以使用的对象ServerProxy，即远程server在client端的代理，该代理跟远程server实现了同样的接口（只不过一个是真实现，一个是假实现），调用代理方法时，代理会通过binder驱动（transact方法）远程调用server端的相应方法。简言之，Stub代表server端，Proxy代表server在客户端的代理。
- 以AMS为例，看下图:
![ams.png](https://upload-images.jianshu.io/upload_images/2573909-c245c5ed7a91259c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

AMS继承了Stub类，而Stub类一共实现了三个接口:IActivityManger，IInterface和IBinder，分别对应了三种能力，管理activity、跨进程、asBinder，前两者好理解，那么这里的asBinder能力是干嘛的呢？这里先卖个关子，等下讲启动流程的时候会说明。
##启动流程
我们先从宏观角度理解这个过程，首先，为什么要跨进程呢？我自己在客户端new一个Activity不行吗？显然是不可以的，Android的安全机制以及为了统一管理Activity(比如activity栈)，需要有个大管家去进行所有Activity的管理和控制,而这个管家是运行在一个单独进程的，因此App端如果想发起一个Activity的请求，需要先把“申请”提交给大管家，也就是AMS。AMS处理完这个请求之后，需要再次通过跨进程通知App端，去执行剩下的相应的工作。因此这里的核心就在于两者如何互相调用对方了。好多文章对AMS如何通知App这块讲的不够清楚，甚至忽略，本文着重会说明。
### App端如何调用AMS方法
下面看代码：用户启动一个页面时，会依次调用activity的startActivity-->Instrumentation的executestartActivity-->execStartActivitiesAsUser，这几个调用很容易找到，就简单带过，在最后这个方法里，执行了远程调用，即：
```
 int result = ActivityManager.getService()
                .startActivities(whoThread, who.getBasePackageName(), intents, resolvedTypes,
                        token, options, userId);
```
ActivityManager.getService获取的是什么？看ActivityMangaer.getService()这个代码里面：
```
    public static IActivityManager getService() {
        return IActivityManagerSingleton.get();
    }

    private static final Singleton<IActivityManager> IActivityManagerSingleton =
            new Singleton<IActivityManager>() {
                @Override
                protected IActivityManager create() {
                    final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
                    final IActivityManager am = IActivityManager.Stub.asInterface(b);
                    return am;
                }
            };
```
如果对Binder有所了解，应该很容易知道，这里取得的是AMS在客户端的代理，也就是代码中的最后一行返回的am。因为App要频繁的调用AMS的方法，因此用单例模式缓存在本地了一个AMS的本地代理，从单例的第一次获取可以看到，AMS的Binder是通过ServiceManager.getService()获取到的,那么ServiceMangaer是个什么东西，其实这个就是Android系统统一管理所有远程服务的“大管家”，比如AMS，WMS等系统服务都在这里注册了，客户端想调用任意一个服务，只需要知道名字就可以通过SM获取到相应的Server的Binder。拿到Binder之后便可以通过asInterface静态方法转化成本地代理，从而调用server的方法了。因此第一次获取AMS的Binder的过程实际上是客户端跟ServiceManager的一次跨进程通信。

### AMS如何通知App进程
###### （1）AMS如何获取到App进程的Binder的
从上面的分析知道，App获取AMS的Binder实际上是通过ServiceManager这个大管家间接获取的，那反过来AMS处理完activity的管理任务(栈操作等)之后又如何通知App的呢？
一个App总不可能像AMS那样在ServiceManger中注册吧，而且也没这个必要。那么到底是怎么通知的呢？
答案就是：***App跨进程调用AMS的方法时，还顺便把App进程（这个时候App可以看作是服务端了）的Binder作为参数传给了AMS，AMS拿到这个APP的Binder之后，通过asInterface方法转化成在server端可以使用的代理，然后在需要回调App进程的时候通过这个代理来通知客户端***。其实跟App端逻辑是一致的，只不过C/S调了一下顺序，C变成了S，S变成了C。下面我们从代码里验证：
我们以6.0之前版本的源码为例，新版本改成事务了，有些源码不容易看到，不如直接看老版本的，便于理解。
首先看APP调用startActivity时是如何把App进程的Binder参数传过去的，刚才说了，startActivity实际上调用的是AMS本地代理的startActivity，而AMS本地代理是ActivityMangerProxy，这里AMP是AIDL自动生成的
```
class ActivityManagerProxy implements IActivityManager
{
    public ActivityManagerProxy(IBinder remote)
    {
        mRemote = remote;
    }
    
    public IBinder asBinder()
    {
        return mRemote;
    }
    
    public int startActivity(IApplicationThread caller, Intent intent,
            String resolvedType, Uri[] grantedUriPermissions, int grantedMode,
            IBinder resultTo, String resultWho,
            int requestCode, boolean onlyIfNeeded,
            boolean debug, String profileFile, ParcelFileDescriptor profileFd,
            boolean autoStopProfiler) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(caller != null ? caller.asBinder() : null);
        intent.writeToParcel(data, 0);
        data.writeString(resolvedType);
        data.writeTypedArray(grantedUriPermissions, 0);
        data.writeInt(grantedMode);
        data.writeStrongBinder(resultTo);
        data.writeString(resultWho);
        data.writeInt(requestCode);
        data.writeInt(onlyIfNeeded ? 1 : 0);
        data.writeInt(debug ? 1 : 0);
        data.writeString(profileFile);
        if (profileFd != null) {
            data.writeInt(1);
            profileFd.writeToParcel(data, Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
        } else {
            data.writeInt(0);
        }
        data.writeInt(autoStopProfiler ? 1 : 0);
        mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
        reply.readException();
        int result = reply.readInt();
        reply.recycle();
        data.recycle();
        return result;
    }
```
startActivity方法的第一个参数caller,这个东西是IApplicationThread，这个IApplicationThread就是AMS去通知App做相应处理的接口，它跟IActivityManger配合组成了App和AMS交互的“协议”。那么这个传过来的IApplicationThread的是谁呢，看代码:
```
Instrumentation:
int result = ActivityManager.getService()
                .startActivityAsUser(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, resultWho,
                        requestCode, 0, null, options, user.getIdentifier());


ContextImpl:
        mMainThread.getInstrumentation().execStartActivities(
                getOuterContext(), mMainThread.getApplicationThread(), null,
                (Activity) null, intents, options);

ActivityThread:
    public ApplicationThread getApplicationThread()
    {
        return mAppThread;
    }

```
通过函数调用可以查到，首先是instrumentation类里传入的whoThread，whoThread是ContextImpl传进来的mMainThread.getApplicationThread(),而最后这个是mAppThread，这个东西就是ActivityThread这个类的内部类ApplicationThread,我们看代码：
```
 private class ApplicationThread extends IApplicationThread.Stub {
```
继承自Stub，因此从AIDL语法看出，是一个服务端，对应的客户端是谁呢？当然是AMS了，所以ApplicationThread这个类就是AMS向App进程发消息时的服务端，因此客户端和服务端都是相对的，A要调B的服务，A就是客户端，B就是服务端，反之亦然。
思路回到主线上，上面已经说明了，在客户端调用Binder的时候把ApplicationThread传给了AMS，怎么传的呢？这里回到刚才的ActivityMangerProxy这个类里面来，参数已经传给了startActivity方法，接下来会执行到这一行：
```
data.writeStrongBinder(caller != null ? caller.asBinder() : null);
```
caller.asBinder,这里asBinder方法就涉及到之前遗留的那个问题，服务端为什么要实现IInterface这个接口，就是在这个时候用的，即把server（相对的）转成一个binder，之后binder写入到Parcel里，然后通过transact方法调用底层Binder驱动传给其他进程。可以这样理解，对于Binder驱动来说，它可以看成跨进程的一个“传送带”，从A进程传递给B进程，无所谓谁是服务端谁是客户端，只要你实现了IInterface，就可以放到这个传送带上传送，只要你继承了Binder(实现了IBinder)，就有能力使用这个传送带。还记得获取AMS的Binder的代码吗？是通过ServiceManger.getService(name)获取的，我看不到ServiceMangager的源码，可以大胆猜测，这里客户端获取AMS的Binder也是通过该方法，即：将AMS通过asBinder方法转化成Binder，ServiceManager通过其transact方法传递给客户端。（待考证）。总结下来就是IInterface接口表明了这个类可以转成一个binder从而在binder驱动中跨进程运输，IBinder接口表明了类具有跨进程的能力，即可以通过调用transact方法“使用”Binder驱动。
###### （2）获取到了Binder之后
上面的讨论已经知道，AMS其实在App跨进程调用AMS的时候就把ApplicationThread的Binder传过来了，传过来以后，AMS如果要用，必须得拿到ApplicationThread的代理，怎么拿到的呢？
刚才说了，客户端startActivity，通过AMS的代理发起transact方法，AMS的onTransact会监听到，执行onTransact方法，我们看代码：AMS继承自IActivityManager.Stub，在源码中叫ActivityManagerNative：
```
    public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
            throws RemoteException {
        switch (code) {
        case START_ACTIVITY_TRANSACTION:
        {
            data.enforceInterface(IActivityManager.descriptor);
            IBinder b = data.readStrongBinder();
            IApplicationThread app = ApplicationThreadNative.asInterface(b);
```
可以看到：
IBinder b = data.readStrongBinder();
客户端将binder write到Parcel中，服务端从data中读了出来，然后通过asInterface转换成ApplicationThread的代理ApplicationThreadProxy这个类，然后执行：
```
int result = startActivity(app, intent, resolvedType,
                    grantedUriPermissions, grantedMode, resultTo, resultWho,
                    requestCode, onlyIfNeeded, debug, profileFile, profileFd, autoStopProfiler);
```
即进入到AMS对Activity启动管理流程中了，经过复杂的跳转，最后跑到ActivityStackSupervisor这个类的realStartActivityLocked方法中，里面最终会执行到这行代码：
```
app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                    System.identityHashCode(r), r.info, new Configuration(mService.mConfiguration),
                    r.compat, r.task.voiceInteractor, app.repProcState, r.icicle, r.persistentState,
                    results, newIntents, !andResume, mService.isNextTransitionForward(),
                    profilerInfo);
```
这里的app.thread就是前面ApplicationThread在AMS中的代理，到了这里大家应该理清楚了，接下来通过代理调起App进程的ApplicationThread里的相应方法,即：scheduleLaunchActivity方法，这个方法会发送一个Message给主线程的handler ：H，然后在handleMessage里通过类加载器创建出一个Activity对象，并执行onCreate方法.balabala....

最后用图片总结一下：
![binder.png](https://upload-images.jianshu.io/upload_images/2573909-298fad8065f14ade.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

App和AMS分别通过各自进程中对方的代理跟对方沟通，是不是挺简单的？


