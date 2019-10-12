# Gradle插件Hello world实践

### 前言

Gradle是一个很强大的构建工具，掌握Gradle也是安卓开发人员走向高级的一项必备技能，当然，完全了解它需要花费不少功夫。在看本文之前，最好先去了解一下Gradle的一些基本概念，比如Project,Task,Gradle的生命周期等概念，了解了这些，我们就可以尝试编写gradle插件了，gradle插件可以说是gradle十分灵活强大的功能，本文就详细介绍一下插件的HelloWorld项目构建，网上有不少相关文章，但当我实践一遍才发现好多文章写的要么不详细，要么有坑，你照着上面来发现根本构建不成！所以才有了写这篇文章的动机。

### 开始编写

Gradle插件有三种构建方式类型:

|  | 说明 |
| :--- | :--- |
| Build script | 把插件写在 build.gradle 文件中，一般用于简单的逻辑，只在该 build.gradle 文件中可见 |
| buildSrc 项目 | 将插件源代码放在 rootProjectDir/buildSrc/src/main/groovy 中，只对该项目中可见，适用于逻辑较为复杂 |
| 独立项目 | 一个独立的 Groovy 和 Java 项目，可以把这个项目打包成 Jar 文件包，一个 Jar 文件包还可以包含多个插件入口，将文件包发布到托管平台上，供其他人使用。本文将着重介绍此类。 |

一般来说，我们写完插件最后都是要上传到私服或者公共仓库里供大家使用的，所以我们只考虑第三种情况。

#### 步骤1，新建工程，新建插件module：

在AndroidStudio中新建一个Project，我们起名叫“GradlePlugin”，选择默认配置即可。新建完成后，选择file-new-New Module新建一个Module，我们起名就叫plugin吧，类型选择Android Library就行，待会会修改它的build.gradle。

#### 步骤2，删除新module中的无用文件，新建插件文件。

等AndroidStudio构建完成以后我们把新Module中的无用文件夹删掉，包括测试文件，Manifest，res文件夹，然后然后我们将src/main目录下面的所有文件删掉，在main目录下新建两个包，一个叫groovy，一个叫resources，注意名字不要写错，两个文件夹分别存放插件代码和属性文件。接下来：在groovy文件夹下，新建一个带有路径的groovy文件，我们这里新建的名字叫com.bruce.demo.HelloWorld.groovy，这里的包路径和文件名都可以随便起，在HelloWorld.groovy中写入以下内容：

```
package com.bruce.demo

import org.gradle.api.Plugin
import org.gradle.api.Project

class HelloWorld implements Plugin<Project> {

    @Override
    void apply(Project project) {
        println("------------hello world!------------")
    }
}
```

简单的一个helloworld。之后在resources文件夹下新建文件,直接看下图所示：

![](https://s2.ax1x.com/2019/10/12/uO7zO1.png)

注意，META-INFO和gradle-plugins是两层文件夹路径，千万别写错了，大家一定要按照图中对应的结构编写，仔细对照一遍。

我们看com.bruce.hello.properties这个文件，首先是它的名字，这个名字很重要，.properties前面的字符串就是最后我们apply插件的名字，也就是我们 apply plugins 'xxxxxxxx'中的xxxxxx，这里我起的名叫com.bruce.hello,然后我们看这个文件里面的内容，很简单，只有一行：

```
implementation-class=com.bruce.demo.HelloWorld
```

也就是我们前面编写的HelloWorld.groovy的路径+类名，这里注意，后面不要加.groovy后缀，我这里就被网上的博客给坑了。

完成了这一步其实插件已经可以用了，但是为了能共享我们的插件，我们进入第三步，把module上传到maven仓库

#### 步骤三，编写插件module的build.gradle我们直接把新建module自动生成的build.gradle内容全部替换成下面代码：

```
apply plugin: 'groovy'
apply plugin: 'maven'

repositories {
    mavenCentral()
}

dependencies {
    implementation gradleApi()//gradle sdk
    implementation localGroovy()//groovy sdk
    implementation fileTree(dir: 'libs', include: ['*.jar'])
}

uploadArchives {
    repositories {
        mavenDeployer {
            //设置插件的GAV参数
            pom.groupId = 'com.bruce.local'
            pom.version = '1.0.0'
            pom.artifactId = 'hello'
            //文件发布到下面目录
            repository(url: uri('../repo'))
        }
    }
}
```

代码很简单，我们要引入groovy和maven插件，这个插件就是帮助我们上传本地插件module用的，它会自动把我们的插件Module打成jar包上传到配置的仓库中去，我们只需要编写一个上传包的task，即uploadArchives里面的代码。我们看配置的四个属性是什么意思：

pom.groupId，version，artifactId分别表示我们这个包的组名，版本已经名字，其实就像我们引入第三方依赖里面的"com.goole.xxxx:yyyy:2.3.4"类似，就是这个第三方包的唯一标识。这里注意，如果不写artifactId，它会给我门生成一个默认的artifactId，名字就是module的名字。

最后repository，就是仓库的位置，可以是远程的私有仓库或者公开的仓库，也可以是本地文件夹，本文用的是本地文件夹，即放到项目根目录下新建的一个repo的目录中。

接下来，我们执行一下upload这个task，等待一会就上传完毕了，我们看项目里多了一个目录repo,里面有我们生成的插件库。接下来，我们在主项目里就可以使用这个插件了。

#### 在主项目中使用插件

使用插件需要修改项目根目录下的build.gradle和app module下的build.gradle,我们分别来看：

打开根目录下的build.gradle,填入以下内容：

```
buildscript {
    repositories {

        maven{
            url './repo/'
        }
        google()
        jcenter()

    }
    dependencies {
        classpath 'com.android.tools.build:gradle:3.5.1'
        classpath 'com.bruce.local:hello:1.0.0'
    }
}

allprojects {
    repositories {
        google()
        jcenter()

    }
}

task clean(type: Delete) {
    delete rootProject.buildDir
}
```

相比于默认配置，修改了两处：添加maven地址，让其URL指向我们插件的本地路径，也就是代码中的第三行。第二处就是在dependencies中添加我们插件的classpath。classpath这个东西可以理解成加载插件的所依赖需要的资源，（implemetation是项目需要依赖的资源,一个是给构建工程用的，另一个是项目代码用到的，要打到最终apk里的）。

经过上面的配置，主项目已经配置好了插件需要的资源，接下来我们就在app module中引入插件就可以使用啦～很简单，在app的build.gradle中加入：

```
apply plugin 'com.bruce.hello'
```

大功告成～

如果你搞了半天还是不行，请查看项目   [https://github.com/hellojym/GradlePlugin](https://github.com/hellojym/GradlePlugin "项目git")

