# Arouter核心源码和思路详解



### 前言

阅读本文之前，建议读者：

* 对Arouter的使用有一定的了解。
* 对Apt技术有所了解。

Arouter是一款Alibaba出品的优秀的路由框架，本文不对其进行全面的分析，只对其最重要的功能进行源码以及思路分析，至于其拦截器，降级，ioc等功能感兴趣的同学请自行阅读源码，强烈推荐阅读云栖社区的[官方介绍](https://yq.aliyun.com/articles/694111)。

对于一个框架的学习和讲解，我个人喜欢先将其最核心的思路用简单一两句话总结出来：ARouter通过Apt技术，生成保存**路径(路由path)**和**被注解(@Router)的组件类**的映射关系的类，利用这些保存了映射关系的类，Arouter根据用户的请求postcard（明信片）寻找到要跳转的目标地址(class),使用Intent跳转。

原理很简单，可以看出来，该框架的核心是利用apt生成的映射关系，这里要用到Apt技术，读者可以自行搜索了解一下。

### 分析

我们先看最简单的代码的使用：

首先需要在需要跳转的组件添加注解

```java
@Route(path = "/main/homepage")
public class HomeActivity extends BaseActivity {
		onCreate()
		....
}
```

然后在需要跳转的时候调用

```java
Arouter.getInstance().build("main/hello").navigation;
```

这里的路径“main/hello”是用户唯一配置的东西，我们需要通过这个path找到对应的Activity。最简单的思路就是通过APT技术，寻找到所有带有注解@Router的组件，将其注解值path和对应的Activity保存到一个map里，比如像下面这样：

```java
class RouterMap {
	 public Map getAllRoutes {
	 		Map map = new HashMap<String,Class<?>>;
	 		map.put("/main/homepage",HomeActivity.class);
	 		map.put("/main/setting",SettingActivity.class);
	 		map.put("/login/register",LoginRegisterActivity.class);
	 		....
      
      return map;
	 }
}
```

然后在工程代码中将这个map加载到内存中，需要的时候直接get(path)就可以了，这种方案似乎能解决我们的问题。

### 发现问题

上面的思路确实能够实现路由功能，但是这么做会存在一个较大的问题：对于一个大型项目，组件数量会很多，可能会有一两百或者更多，把这么多组件都放到这个Map里，显然会对内存造成很大的压力，因此，Arouter作为一款阿里出品的优秀框架，显然是要解决这个问题的。

这里建议读者自行思考一下，如何解决一次性加载所有映射关系带来的内存损耗问题，我在思考这个问题的时候首先想到的是“懒加载”，但是仅仅懒加载是不够的，因为懒加载后如果还是一次性把所有映射关系加载进来，内存损耗还是一样大的。因此，再深入思考一下，可能还能想出解决一个思路：分段懒加载，思路有了，如何实现呢？这里还是建议大家在阅读下面的内容之前思考一下，或许你能想到一套不同于Arouter的方案来哦。

Arouter采用的方法就是“分组+按需加载”，分组还带来的另一个好处是便于管理，下面我们来看一下实现原理。

### 解决步骤一：分组

首先看如何分组的，Arouter在一层map之外，增加了一层map，我们看WareHouse这个类，里面有两个静态Map：

```java
    static Map<String, Class<? extends IRouteGroup>> groupsIndex = new HashMap<>();
    static Map<String, RouteMeta> routes = new HashMap<>();
```

* groupsIndex 保存了group名字到IRouteGroup类的映射，这一层映射就是Arouter新增的一层映射关系。

* routes  保存了路径path到RouteMeta的映射，其中，RouteMeta是目标地址的包装。这一层映射关系跟我门自己方案里的map是一致的，我们路径跳转也是要用这一层来映射。

这里出现了两个我们不认识的类，IRouteGroup和RouteMeta，后者很简单，就是对跳转目标的封装，我们后续称其为“目标”，其内包含了目标组件的信息，比如Activity的Class。那IRouteGroup是个什么东西？

```java
public interface IRouteGroup {
    /**
     * Fill the atlas with routes in group.
     */
    void loadInto(Map<String, RouteMeta> atlas);
}
```

一个接口，只有一个方法loadInto，都有谁实现了这个接口呢？我拿我手上的一个项目为例，Arouter通过apt生成了下面几个类：

![apt生成文件目录.png](https://upload-images.jianshu.io/upload_images/2573909-05361082591d36bc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通过观察看到，前面三个类都以Arouter\$\$Group开头，我们随便拿一个看看：

```
public class ARouter$$Group$$main implements IRouteGroup {
  @Override
  public void loadInto(Map<String, RouteMeta> atlas) {
    atlas.put("/main/fa/leakscan", RouteMeta.build(RouteType.ACTIVITY, MainFaLeakScanActivity.class, "/main/fa/leakscan", "main", }}, -1, 1));
    atlas.put("/main/login", RouteMeta.build(RouteType.ACTIVITY, LoginActivity.class, "/main/login", "main", null, -1, -2147483648));
    atlas.put("/main/register", RouteMeta.build(RouteType.ACTIVITY, RegPhoneActivity.class, "/main/register", "main", null, -1, -2147483648));
  }
}
```

我们看到，他实现了loadInto方法，在这个方法中，它往这个HashMap中填充了好多数据，填充的是什么呢？填充的是路径path和它对应的目标RouteMeta，也就是我们最终需要的那层映射关系。而且，我们还能观察到：这个类下面所有的路由path都有一个共同点，即全是“/main”开头的，也就是说，这个类加载的映射关系，都是在一个组内的。因此我们总结出：

**Arouter通过apt技术，为每个组生成了一个以Arouter\$\$Group开头的类，这个类负责向atlas这个Hashmap中填充组内路由数据。**

IRouteGroup正如其名字，它就是一个能装载该组路由映射数据的类，其实有点像个工具类，为了方便后续讲解，我们姑且称上面这样一个实现了IRouteGroup的类叫做“组加载器”，本质是一个类。上图中的类是一个组加载器，其他所有以Arouter\$\$Group开头的类都是一个“组加载器”。回到之前的主线，Warehoust中的两个Hashmap，其中groupsIndex这个map中保存的是什么呢？我们通过它的调用找到这一行代码(已简化):

```java
for (String className : routerMap) {
     if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR +SUFFIX_ROOT)) {
        ((IRouteRoot) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.groupsIndex);
			}
}
```

其中 `ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR +SUFFIX_ROOT`  这行代码是几个静态字符串拼起来的，它等于 `com.alibaba.android.arouter.routes.Arouter$$Root` 。另外routerMap是什么呢？它是一个HashSet\<String>：

```
routerMap = ClassUtils.getFileNameByPackageName(mContext, ROUTE_ROOT_PAKCAGE);
```

这一行代码对它进行了初始化，目的是找到 `com.alibaba.android.arouter.routes` 这个包名下所有的类，将其类名保存到routerMap中。因此，上面的代码意思就是将`com.alibaba.android.arouter.routes` 包下所有名字以 `com.alibaba.android.arouter.routes.Arouter$$Root` 开头的类找出来，通过反射实例化并强转成IRouteRoot，然后调用loadInto方法。这里又出来一个新的接口：IRouteRoot，我们看代码：

```java
public interface IRouteRoot {

    /**
     * Load routes to input
     * @param routes input
     */
    void loadInto(Map<String, Class<? extends IRouteGroup>> routes);
}
```



跟IRouteGroup长得还挺像，也是loadInto，我们看它的实现。还是以我的项目为例，在apt生成的文件夹下查找：
![apt生成文件目录.png](https://upload-images.jianshu.io/upload_images/2573909-05361082591d36bc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


最底下一行，有个Arouter\$\$Root\$\$app，它符合前面名字规则，我们进去看看：

```java
public class ARouter$$Root$$app implements IRouteRoot {
  @Override
  public void loadInto(Map<String, Class<? extends IRouteGroup>> routes) {
    routes.put("YDY", ARouter$$Group$$YDY.class);
    routes.put("app", ARouter$$Group$$app.class);
    routes.put("main", ARouter$$Group$$main.class);
    routes.put("payment", ARouter$$Group$$payment.class);
    routes.put("wallet", ARouter$$Group$$wallet.class);
  }
}
```

这个类实现了IRouteRoot，在loadInto方法中，他将组名和组对应的“组加载器”保存到了routes这个map中。也就是说，这个类将所有的“组加载器”给索引了下来，通过任意一个组名，可以找到对应的“组加载器”，我们再回到前面讲的初始化Arouter时候的方法中：

```
((IRouteRoot) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.groupsIndex);
```

理解了吧，这个方法的意义就在于将所有的组路由加载类索引到了groupsIndex这个map中。因此我们就明白了：

**WareHouse中的groupsIndex保存的是组名到“组加载器”的映射关系**

说句题外话：回过头想想前面用到的两个接口：IRouteGroup和IRouteRoot，它们其实是apt生成的类和我们项目中代码之间沟通的桥梁，熟悉AIDL的同学可能会觉得很熟悉，二者其实是异曲同工的，两个系统进行交互的时候都是通过接口来沟通的。当然，在使用apt生成的类时，我们需要用到反射技术。

总结一下Arouter的分组设计：Arouter在原来path到目标的map外，加了一个新的map，该map保存了组名到“组加载器”的映射关系。其中“组加载器”是一个类，可以加载其组内的path到目标的映射关系。 

到此为止，Arouter只是完成了分组工作，但这么做的目的是什么呢？别着急，前面的都只是铺垫，接下来才是这个分组设计发挥作用的地方，我们进入“按需加载”的代码分析：



### 解决步骤二：按需加载

之前说过，Arouter使用的是分组按需加载，分组是为了按需做准备的。我们看Arouter是怎么按需加载的，我们还是从代码的使用入手:

```java
Arouter.getInstance().build("main/hello").navigation;
```

在navigation这个方法中，最终会跳转到这里：

```java
protected Object navigation(final Context context, final Postcard postcard, final int requestCode, final NavigationCallback callback) {
        try {
          //请关注这一行
            LogisticsCenter.completion(postcard);
        } catch (NoRouteFoundException ex) {
            logger.warning(Consts.TAG, ex.getMessage());
				....//简化代码
        }
  			//调用Intent跳转
        return _navigation(context, postcard, requestCode, callback)
```

最后一行的return语句很简单，就是去调用Intent唤起组件了，我们看前面try中的第一行 `LogisticsCenter.completion(postcard)`,我们进到这个函数里：

```java
//从缓存里取路由信息
RouteMeta routeMeta = Warehouse.routes.get(postcard.getPath());
//如果为空，需要加载该组的路由
if (null == routeMeta) {
  Class<? extends IRouteGroup> groupMeta = Warehouse.groupsIndex.get(postcard.getGroup());
  IRouteGroup iGroupInstance = groupMeta.getConstructor().newInstance();
  iGroupInstance.loadInto(Warehouse.routes);
  Warehouse.groupsIndex.remove(postcard.getGroup());
} 
//如果不为空，走后续流程
else {
     postcard.setDestination(routeMeta.getDestination());
  	 ...
}
```

这段代码就是“按需加载”的核心逻辑所在了，我对其进行了简化，分析其逻辑是这样的：

* 首先从Warehouse.routes(前面说了，这里存放的是path到目标的映射)里拿到目标信息，如果找不到，说明这个信息还没加载，实际上，刚开始这个routes里面什么都没有。
* 加载流程：首先从Warehouse.groupsIndex里获取“组加载器”，然后获取的组加载器是一个类，需要通过反射将其实例化，实例化为iGroupInstance，接着调用加载器的加载方法loadInto，将该组的路由映射关系加载到Warehouse.routes中，注意这是，routes中就缓存下来当前group的所有路由映射了，因此这个类加载器其实就没用了，为了节省内存，将其从Warehouse.groupsIndex移除。
* 如果之前加载过，则在Warehouse.routes里面是可以找到路有映射关系的，因此直接将目标信息routeMeta传递给postcard，保存在postcard中，这样postcard就知道了最终要去那个组件了。

到此为止分组按需加载的逻辑就都分析完了，通过这两个步骤，解决了路由映射一次性加载到内存占用内存过大的缺点，这是Arouter这个框架优秀的重要原因之一。当然Arouter还有一些优秀的功能，比如拦截器，依赖注入等，总之，功能全，性能好，使用方便，这些都是Arouter受欢迎的原因，这点值得我们所有开发者去学习。

### 总结

最后结合一张图总结一下Arouter的分组按需加载的逻辑：

![按需分组加载示例.png](https://upload-images.jianshu.io/upload_images/2573909-6866213c6fe91475.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


图中左侧groupsIndex是“组映射”，右侧routes是“路由映射”。Arouter在初始化的时候，通过反射技术，将所有的“组加载器”索引到groupsIndex这个map中，而此时，右侧的routes还是空的。在用户调用navigation()进行跳转的时候，会根据路径提取组名，由组名根据groupsIndex获取到相应组的“组加载器”，由组加载器加载对应组内的路由信息，此时保存全局“路由目标映射的”routes这个map中就保存了刚才组的所有路由映射关系了。同样，当其他组请求时，其他组也会加载组对应的路由映射，这样就实现了整个App运行时，只有用到的组才会加到内存中，没有去过的组就不会加载到内存中，达到了节省内存的目的。




