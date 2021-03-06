# Gradle基础知识

------

| 版本/状态 | 责任人  |     时间     |  备注  |
| :---: | :--: | :--------: | :--: |
| V1.0  |  官鑫  | 2019/01/27 |  草稿  |

------

[TOC]

------

## 一、前言背景

Gradle是一个基于Apache Ant和Apache Maven概念的项目自动化构建开源工具。它使用一种基于Groovy的特定领域语言(DSL)来声明项目设置，抛弃了基于XML的各种繁琐配置。

目前，Android Studio中默认的构建工具就是gradle，主流的以及我们的项目中使用的构建工具也都是gradle，gradle可以很方便的去构建一个多用途多打包渠道多复杂Android应用。

## 二、Gradle介绍

每次构建（build）至少由一个project构成，一个project 由一到多个task构成。项目结构中的每个build.gradle文件代表一个project，在这编译脚本文件中可以定义一系列的task；task 本质上又是由一组被顺序执行的Action`对象构成，Action其实是一段代码块，类似于Java中的方法。

### Project

1. 每一个 Library 和每一个 App 都是单独的 Project。根据 Gradle 的要求，每一个Project在其根目录下都需要有一个 build.gradle。build.gradle 文件就是该 Project 的编译脚本，类似于 Makefile。

2. Project之间如果出现父子关系，只有根Project才会有setting.gradle配置文件，该配置文件的作用是声明其包含的子项目。

### Task

每个Project是由N个Task组织成的一个“有向无环图”（关于有向无环图的说明可以参考Spark中的解释），task之间的依赖关系决定了它们的执行顺序。

task的来源有三种，如下：

1. Gradle默认自带的task，如：

Dependencies：显示Project的依赖信息

Projects：显示所有Project，包括根Project和子Project

Properties：显示一个Project所包含的所有Property

2. 直接在project中显式创建的task，如：

```groovy
hello << {
    println 'Hello Task'
}
```

3. 以plugin的方式引入的task，如：

通过apply方式引入：apply plugin: 'groovy'

点击AndroidStudio右侧的一个Gradle按钮，会打开一个面板，你可以看到所有的task列表，如下图所示：

 ![gradle的task任务列表](src\gradle的task任务列表.jpeg)

### Property

Project除包含Task外还包含Property，Property默认自带了一些属性，也可以通过ext.XXX的方法来扩展。

### Gradle的工作流程

Gradle的工作流程分为三个部分，每个阶段都可以通过API添加定制的hook去执行一些操作，每次构建的执行本质上执行一系列的Task。某些Task可能依赖其他Task。那些没有依赖的Task总会被最先执行，而且每个Task只会被执行一遍。每次构建的依赖关系是在构建的配置阶段确定的。

如下图所示：

 ![gradle工作流程图](src\gradle工作流程图.jpeg)

1. initialization phase，初始化阶段。

   也是创建Project阶段，构建工具根据每个build.gradle文件创建出一个Project实例。初始化阶段会执行项目根目录下的settings.gradle文件，来分析哪些项目参与构建。

2. configuration phase，配置阶段，解析每个子project的gradle，根据task之间的关系创建有向无环图。

   这个阶段，通过执行构建脚本来为每个project创建并配置Task。配置阶段会去加载所有参与构建的项目的build.gradle文件，会将每个build.gradle文件实例化为一个Gradle的project对象。然后分析project之间的依赖关系，下载依赖文件，分析project下的task之间的依赖关系。

3. execuion phase，执行阶段会将任务链上的所有task全部按依赖顺序执行一遍，下载依赖包，打包发布应用等。

   这是Task真正被执行的阶段，Gradle会根据依赖关系决定哪些Task需要被执行，以及执行的先后顺序。task是Gradle中的最小执行单元，我们所有的构建，编译，打包，debug，test等都是执行了某一个task，一个project可以有多个task，task之间可以互相依赖。例如我有两个task，taskA和taskB，指定taskA依赖taskB，然后执行taskA，这时会先去执行taskB，taskB执行完毕后在执行taskA。

## 三、Android应用下的gradle配置文件

我们用Android Studio开发Android multi-project工程应用时，如果打开AndroidStudio选择Android视图，可以很清楚的看到当前项目下的gradle相关的配置文件，如下图所示。

 ![gradle相关配置文件](src\gradle相关配置文件.png)

### build.gradle（root project）文件

build.gradle(Project:nim_demo)为root project（这里我们称项目的根目录为root project）下的gradle配置文件，该配置文件一般用来做一些全局性的配置。

项目根目录的 build.gradle 文件用来配置针对所有模块的一些属性。它默认包含2个代码块：buildscript{...}和allprojects{...}。前者用于配置构建脚本所用到的代码库和依赖关系，后者用于定义所有模块需要用到的一些公共属性。

相关配置属性说明如下所示：

```groovy
//全局配置构建工具的classpath和远程仓库路径，这里一般配置为gradle的，因为项目在构建过程中需要使用gradle去执行，当然如果你使用到了一些额外的插件，比如注解处理器，也可以放在这里。
buildscript {
    repositories {
        jcenter()
    }

    dependencies {
        classpath 'com.android.tools.build:gradle:2.1.3'
    }
}
//这里是对项目中的所有project进行配置，以Android项目为例，这里的project包括项目下的各种module以及项目根目录root project。
allprojects {
    repositories {
        jcenter()
    }
}
//这里是对该project下的所有子project进行配置，以Android项目为例，一般我们可以把一些公用的操作放在一个gradle配置文件中，然后import到各个子project的gradle中，方便使用
subprojects {
//类似于Java中的import功能，将该文件中定义的方法或变量导入后，其他gradle文件可以直接引用该gradle中的方法或变量
    apply from: 'common.gradle'
}
//定义一些变量，该变量可以被其他gradle使用
ext {
    compileSdkVersion=21
    buildToolsVersion='23.0.2'
    minSdkVersion=9
    targetSdkVersion=19
    versionCode=28
    versionName='3.0.0'
    targetCompatibility=1.7
    sourceCompatibility=1.7
}
```

### setting.gradle文件

settings.gradle(Project Settins)为工程的设置文件。

会在构建的 initialization 阶段被执行，它用于告诉构建系统哪些模块需要包含到构建过程中。对于单模块项目， settings.gradle 文件不是必需的。对于多模块项目，如果没有该文件，构建系统就不能知道该用到哪些模块。

```groovy
include ':uikit'
include ':demo'
```

### local.properties文件

主要是配置SDK和NDK包的位置，如下所示：

```groovy
## This file is automatically generated by Android Studio.
# Do not modify this file -- YOUR CHANGES WILL BE ERASED!
#
# This file must *NOT* be checked into Version Control Systems,
# as it contains information specific to your local configuration.
#
# Location of the SDK. This is only used by Gradle.
# For customization when using a Version Control System, please read the
# header note.
#Wed Dec 19 19:12:58 CST 2018
ndk.dir=E\:\\Users\\admin\\AppData\\Local\\Android\\android-ndk-r12b-windows-x86_64\\android-ndk-r12b
sdk.dir=E\:\\Users\\admin\\AppData\\Local\\Android
```

### gradle-wrapper.properties文件

gradle-wrapper.properties(Gradle Version)为gradle的版本配置信息。

Android Studio中默认会使用 Gradle Wrapper 而不是直接使用Gradle。命令也是使用gradlew而不是gradle。这是因为gradle针对特定的开发环境的构建脚本，新的gradle可能不能兼容旧版的构建环境。为了解决这个问题，使用Gradle Wrapper 来间接使用 gradle。相当于在外边包裹了一个中间层。对开发者来说，直接使用Gradlew 即可，不需要关心 gradle的版本变化。Gradle Wrapper 会负责下载合适的的gradle版本来构建项目。

```groovy
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-2.14.1-all.zip
```

当我们从github下载一个开源项目导入到Android Studio中打开时，如果本地的gradle版本与该配置文件的版本不符就会去联网下载对应版本的gradle。由于gradle有时访问会比较慢，所以建议将网上的项目中该配置文件的版本改成本地已经下载了的版本运行，可以节省时间。有时修改版本信息后可能会出现一些语法错误，这是本地的gradle版本不支持该语法，因此仍然需要重新下载对应的gradle版本。

### gradle.properties

gradle.properties(Project Properties)是与当前项目相关的一个配置文件，主要包括一些当前项目中需要用到的键值对信息。包括java虚拟机的内存参数配置、是否开启并行编译和是否开启守护进程等。

例如：

```groovy
# Project-wide Gradle settings.
# 守护进程
org.gradle.daemon=true
# java虚拟机参数配置
org.gradle.jvmargs=-Xmx2048m -XX:MaxPermSize=512m -XX:+HeapDumpOnOutOfMemoryError -Dfile.encoding=UTF-8
# 并行编译
org.gradle.parallel=true
```

该属性文件会被gradle插件自动加载，所以在project的gradle文件中可以直接使用。你也可以自定义一个属性文件，例如custom.properties，然后在build.gradle文件中通过如下方式读取versionName。

```groovy
def getVersionName() {
    Properties properties = new Properties()
    File file = new File(rootDir.absolutePath + "/custom.properties")
    properties.load(file.newDataInputStream())
    //为了增加可读性，添加return, 根据groovy语法，省略return时自动返回最后一行代码执行结果的类型
    return properties.get("versionName")
}
```

### 主module的build.gradle文件

build.gradle(Module:demo)为主module demo下的gradle配置，该配置只对该module生效。

主module使用的gradle插件是com.android.application，其产物是application应用，对于Android是APK文件。并且不同的插件自带的task不一样，和其他子module build.gradle文件应用的插件com.android.library是不一样的。

例如：

```groovy
//声明该模块最终生成产物为apk
apply plugin: 'com.android.application'

android {
//编译使用的sdk版本
    compileSdkVersion 23
    //使用的编译工具版本，一般与sdk版本对应，比如sdk version为23，则buildToolVersion应选择为23.x.x
    buildToolsVersion rootProject.buildToolsVersion
//所有flavor的默认配置
    defaultConfig {
        minSdkVersion 9
        targetSdkVersion 23
    }
//project的签名配置
    signingConfigs {
        debug { storeFile file("debug.keystore") }

        release {
            storeFile file('release.keystore')
            storePassword 'thisiskeystorepassword'
            keyAlias 'nim_demo'
            keyPassword 'thisiskeypassword'
        }
    }
//project的编译类型
    buildTypes {
        debug {
            signingConfig signingConfigs.debug
        }

        release {
            minifyEnabled true
            zipAlignEnabled true
            proguardFile('proguard.cfg')
            signingConfig signingConfigs.release
        }
    }
    //项目的目录结构
    sourceSets {
        main {
            manifest.srcFile 'AndroidManifest.xml'
            java.srcDirs = ['src']
            aidl.srcDirs = ['src']
            res.srcDirs = ['res']
            assets.srcDirs = ['assets']
            jniLibs.srcDirs = ['libs']

        }

    }
//lint检查的选项
    lintOptions {
        checkReleaseBuilds false
        abortOnError false
    }
    //生成dex的选项
    dexOptions {
        incremental true
        preDexLibraries false
        jumboMode true
        javaMaxHeapSize "4g"
    }
    //打包选项
    packagingOptions {
        exclude 'META-INF/LICENSE.txt'
        exclude 'META-INF/NOTICE.txt'
    }

}
//该project的依赖
dependencies {
//依赖本地libs目录下的jar包
    compile fileTree(dir: 'libs', include: '*.jar')
    //依赖uikit模块
    compile project(path: ':uikit')
}
```

### 子module的build.gradle文件

build.gradle(Module:uikit)为uikit模块下的配置，该配置只对uikit module生效，使用com.android.library插件，其产物为aar文件。

例如：

```groovy
//声明该模块编译产物为aar
apply plugin: 'com.android.library'
//其余配置与dmeo模块类似，这里不再赘述
android {
    useLibrary 'org.apache.http.legacy'

    compileSdkVersion 23
    buildToolsVersion buildToolsVer

    sourceSets {
        main {
            manifest.srcFile 'AndroidManifest.xml'
            java.srcDirs = ['src']
            resources.srcDirs = ['src']
            aidl.srcDirs = ['src']
            renderscript.srcDirs = ['src']
            res.srcDirs = ['res', 'res-ptr']
            assets.srcDirs = ['assets']
            jniLibs.srcDirs = ['libs']
        }
    }
}

dependencies {
    compile 'com.android.support:appcompat-v7:23.3.0'
    compile 'com.android.support:design:23.3.0'
    compile fileTree(dir: 'libs', include: '*.jar')
}
```

## 四、build.gradle文件详解

模块级配置文件 build.gradle 针对每个moudle 的配置，它有3个重要的代码块：plugin，android 和 dependencies。

### plugin

一般在模块级的build.gradle文件中使用的插件是application或者library，如上述代码中的：

主module应用application插件

```groovy
//声明该模块最终生成产物为apk
apply plugin: 'com.android.application'
```

子module应用library插件

```groovy
//声明该模块最终生成产物为aar
apply plugin: 'com.android.library'
```

### Dependencies

**1.依赖远程代码库**

每个库名称包含三个元素：组名:库名称:版本号，如下所示：

```groovy
dependencies {
    compile 'com.android.support:appcompat-v7:25.0.0'
}
```

**2.依赖本地module**

```groovy
dependencies {
    compile project ':localmodule'
}
```

**3.依赖jar包**

（1）把jar包放在libs目录下

（2）在build.gradle中添加配置如下

```groovy
dependencies {
    compile files('libs/jarName.jar')
}
```

**4.依赖 aar 包**

（1）把 aar 包放到 libs 目录下

（2）在 build.gradle 中添加依赖

```groovy
repositories {
    flatDir {
        dirs 'libs'
    }
}
dependencies {
    compile(name:'aarName-release', ext:'aar')
}
```

**5.自定义依赖**

当我们的 aar 包需要被多个 module 依赖时，我们就不能把 aar 包放在单一的 module 中，我们可以在项目的根目录创建一个目录，比如叫 aar 目录，然后把我们的 aar 包放进去，然后在项目的根目录的 build.gradle 的 allprojects 标签下的 repositories 添加 如下配置：

```groovy
allprojects {
    repositories {
        flatDir {
            dirs '../aar'
        }
    }
}
```

然后就可以对需要的module添加依赖了，如下所示：

```groovy
dependencies {
    compile(name:'aarName-release', ext:'aar')
}
```

android配置块中主要包括defaultConfig、signingConfigs、buildTypes、productFlavors、lintOptions等配置，这些都是插件com.android.application中内置的API。下面，我们就来一一介绍一下它们的意义和用法。

### defaultConfig

defaultConfig中主要是配置应用的一些默认属性，比如applicationId、mindSdkVersion、targetSdkVersion、versionCode、versionName以及打包的abi平台版本等，如下所示：

```groovy
defaultConfig {
        applicationId "com.tplink.exsample"
        minSdkVersion 19
        targetSdkVersion 21
        versionCode 101
        versionName "2.1.1"
        resValue("string", "app_name", "应用名称")
        testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner"

        ndk {
            // Specifies the ABI configurations of your native
            // libraries Gradle should build and package with your APK.
            abiFilters 'armeabi'
        }
    }
```

此外还可以动态配置一些额外信息，如下所示：

1.替换AndroidManifest.xml中的占位符

```groovy
android{
    defaultConfig{
        manifestPlaceholders = [appName:"@string/app_name"]
    }
}
```

2.动态设置额外信息

假如想把当前的编译时间、编译的机器、最新的commit版本添加到apk中，动态设置如下：

```groovy
android {
    defaultConfig {
        resValue "string", "build_time", buildTime()
        resValue "string", "build_host", hostName()
    }
}
def buildTime() {
    return new Date().format("yyyy-MM-dd HH:mm:ss")
}
def hostName() {
    return System.getProperty("user.name") + "@" + InetAddress.localHost.hostName
}
```

那么，我们便可以在app中使用getString(R.string.xxx)来使用这些额外信息，如下所示：

```java
getString(R.string.build_time)  // 输出2017-06-12 15:05:05
getString(R.string.build_host)  // 输出zyr@example，这是我的电脑的用户名和PC名
```

3.resConfigs 过滤

**过滤语言**

```groovy
android {
    defaultConfig {
        // 只打包en（英文）、zh-rCN（中文简体）、es（西班牙语）
        resConfigs 'en', 'zh-rCN', 'es'
    }
}
```

 **过滤drawable资源**

```groovy
android {
    defaultConfig {
        // 只打包drawable-*hdpi文件夹里的资源
        resConfigs "hdpi"
    }
}
```

**自定义BuildConfig类静态常量**

通过buildConfigField（type， name， value）来为BuildConfig类添加一个public static final类的常量。如下：

```groovy
android {
    defaultConfig {
        // 添加一个静态常量，sync之后存在与BuildConfig类中
        buildConfigField("boolean", "IS_TEST", "true") // 定义一个bool变量
    }
}
```

这是一个非常有用的配置方式，比如我们可以在buildTypes{}的debug{}和release{}中为IS_TEST配置不同的值，在java代码中通过判断BuildConfig.IS_TEST的值来开启/关闭打印日志的功能。

### signingConfigs

在这里可以配置不同类型的签名，然后在buildTypes中根据不同类型需要使用。如下：

```groovy
android {
    signingConfigs {
        release {
          keyAlias "release#alias"
          keyPassword "release#password"
          storeFile file("release#storefile.keystore")
          storePassword "release#storepassword"
        }
        exsample {
          keyAlias "exsample#alias"
          keyPassword "exsample#password"
          storeFile file("exsample#storefile.keystore")
          storePassword "exsample#storepassword"
        }
    }

    buildTypes {
        release {
            debuggable false
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
            // 使用release版本的签名
            signingConfig signingConfigs.release
        }
        custom {
            debuggable true
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
            // 使用exsample配置的签名
            signingConfig signingConfigs.exsample
        }
    }
}
```

也可以把签名有关的密码、签名等敏感信息统一进行存放，不进行硬编码。写在`gradle.properies`中，可以随意的定义`key-value`形式，gradle.properties文件会被`gradle`自动引入的。

gradle.properties文件可以配置如下：

```groovy
STORE_FILE_PATH ../test_key.jks
KEYSTORE_PASSWORD 123456
KEY_ALIAS abc
KEY_PASSWORD 654321
PACKAGE_NAME_SUFFIX .test
TENCENT_AUTHID aaa0123
```

然后在module的build.gradle文件中可以配置signingConfigs如下：

```groovy
signingConfigs {
    release {
        try {
              storeFile file(STORE_FILE_PATH)
              storePassword STORE_PASSWORD
              keyAlias KEY_ALIAS
              keyPassword KEY_PASSWORD
        }
        catch (exception) {
            throw new InvalidUserDataException("You should define KEYSTORE_PASSWORD and KEY_PASSWORD in gradle.properties.")
        }
    }
}
```

### lintOptions检测

可以配置是否忽略编译器的lint检查，如下：

```groovy
lintOptions {
        // true--所有正式版构建执行规则生成崩溃的lint检查，如果有崩溃问题将停止构建
        checkReleaseBuilds false
        // true--错误发生后停止gradle构建
        abortOnError false
}
```

### compileOptions

在这里可以设置本module的Java编译版本，如下：

```groovy
 android {
    compileOptions {
        //java源文件编码格式
        encoding 'UTF-8'
        //java编译是否使用gradle的增量模式
        incremental true
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
}
```
如果要全局配置Java编译版本的话，可以在allprojects{}中设置如下：

```groovy
allprojects {
    repositories {
        jcenter()
    }
    // 设置所有Java编译任务的Java编译版本为1.8
    tasks.withType(JavaCompile) {
        // java源文件编码格式
        encoding 'UTF-8'
        // java编译是否使用gradle的增量模式
        incremental true
        sourceCompatibility = JavaVersion.VERSION_1_8
        targetCompatibility = JavaVersion.VERSION_1_8
    }
}
```

### aaptOptions

这里可以设置aapt编译资源文件时的一些操作，比如对png进行优化检查、忽略某些资源文件以及设置哪些文件不需要压缩等。

```groovy
// aapt编译资源文件
aaptOptions {
    additionalParameters '--rename-manifest-package', 'com.pf.haha'
    // 对png进行优化检查
    cruncherEnabled true
    // 对res中的资源进行排除
    ignoreAssets '*.jpg'
    // 哪些文件不需要压缩  aapt l -v
    noCompress '*.bat'
}
```

### SourceSets

SourceSet可以定义项目结构，也可以修改项目结构。android插件默认实现了两个SourceSet，main 和 test。每个 SourceSet 都提供了一系列的属性，通过这些属性，可以定义该 SourceSet 所包含的源文件。比如，java.srcDirs，resources.srcDirs 。android插件中定义的其他任务，就根据 main 和 test 的这两个 SourceSet 的定义来寻找产品代码和测试代码等。

```groovy
android {
    sourceSets {
        main {
            // 组件化开发中，经常根据不同的buildTypes使用不同的AndroidManifest.xml文件
            if (isDebug.toBoolean()) {
                manifest.srcFile 'src/debug/AndroidManifest.xml'
            } else {
                manifest.srcFile 'src/release/AndroidManifest.xml'
            }
            // java源文件位置
            java.srcDirs = ['src']
            // resource源文件位置
            resources.srcDirs = ['src']
            // aidl源文件位置
            aidl.srcDirs = ['src']
            // renderscript源文件位置
            renderscript.srcDirs = ['src']
            // assets源文件位置
            assets.srcDirs = ['assets']
            // 根据是否使用了data binding框架来指定源文件位置
            if (!IS_USE_DATABINDING) {
                jniLibs.srcDirs = ['libs']
                // 多加了databinding的资源目录
                res.srcDirs = ['res', 'res-vm']
            } else {
                res.srcDirs = ['res']
            }
        }

        test {
            java.srcDirs = ['test']
        }

        androidTest {
            java.srcDirs = ['androidTest']
        }
    }
}
```

SourceSets实现layout分包

项目比较复杂时，页面布局文件太多，在一个layout包下查找某个layout布局文件比较麻烦，为了解决这个问题，我们可以创建多个 layout 目录，不同模块的布局文件放在不同的 layout 目录中，这样查找起来，就容易很多。

例如，项目中，有两个模块分别为：登录、注册，我们可以按照下列步骤来实现layout分包：

- 第一步：把项目中 layout 文件夹改名字为 layouts
- 第二步：在 layouts 目录下，分别创建 login 、register 目录 。
- 第三步：分别在 login 、register 目录下创建layout 目录。注意这一步是必须的，因为编译器查找的就是layout目录，否则会报错。
- 第四步：把 登录模块布局文件、注册模块布局文件分别放在 第三步创建的对应的 layout 目录下。

目录结构如下图所示：

 ![layout分包](src\layout分包.jpeg)

在SourceSets中实现如下：

```groovy
android {
    sourceSets {
        main {
           // 指定登录和注册模块的布局资源目录
           res.srcDirs 'src/main/res/layouts/login'
           res.srcDirs 'src/main/res/layouts/register'
        }
    }
}
```

### buildTypes

buildTypes{}对应的是 BuildType 类，BuildType类继承自DefaultBuildType类，DefaultBuildType类继承BaseConfigImpl类。其作用是配置编译的版本，默认提供了两种build type，debug和release版本，当然也可以添加自定的build Type。所以可以在这里定制不同build type的属性和方法。

buildTypes的属性如下图示所示：

 ![buildTypes的属性](src\buildTypes的属性.jpeg)

buildTypes的方法如下图所示：

 ![buildTypes的方法](src\buildTypes的方法.jpeg)

部分属性和方法使用示例如下：

```groovy
buildTypes {
    externalNativeBuild {
        // 设置ndkBuild action的mk文件路径
        ndkBuild {
            path 'src/main/jni/Android.mk'
        }
    }
     debug {
         resValue("string", "app_name", "app名称")
         // 开启调试
         debuggable true
         // 设置无混淆
         minifyEnabled false
         proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
         signingConfig signingConfigs.release
     }
     release {
         // 设置不可调试
         debuggable false
         // 开启文件混淆
         minifyEnabled true
         // 设置混淆文件和规则
         proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
         // 使用release版本的签名
         signingConfig signingConfigs.release
     }
     custom {
         debuggable true
         minifyEnabled true
         proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
         // 使用exsample配置的签名
         signingConfig signingConfigs.exsample
     }
     myType {
         // 复制custom type的属性和方法
         initWith custom
         // 覆写debuggabled属性
         debuggabled false
         // 配置其他的属性
         applicationIdSuffix "appIdsuffix"
         versionNameSuffix "myTypeSuffix"
     }
}
```

### productFlavors

假设我们需要在同一个工程中配置生成不同的产品，这些不同的产品名称、渠道、依赖库、代码和资源等都不同，此时我们便可以借助productFlavors来配置差异化构建打包不同产品。

1.不同产品配置不同包名和版本号

```groovy
android {
    productFlavors {
        productA {
            applicationId "com.tplink.product.a"
            versionName "version-a-1.0"
        }

        productB {
            applicationId "com.tplink.product.b"
            versionName "version-b-1.0"
        }
    }
}
```

2.不同产品不同的渠道

productFlavors{}配置如下：

```groovy
android {
    productFlavors {
        productA {
            manifestPlaceholders = [app_appkey:"wandoujia"]
        }

        productB {
            manifestPlaceholders = [app_appkey:"xiaomi"]
        }

        productC {
            manifestPlaceholders = [app_appkey:"qh360"]
        }
    }
}
```

或者，更方便一些可以如下配置：

```groovy
android {
    productFlavors {
        wandoujia {}
        xiaomi {}
        qh360 {}
    }

    productFlavors.all {
        // 设置每个Flavor的AndroidManifest.xml文件中的UMENG_CHANNEL_VALUE占位符的值为flavor的名称
        flavor -> flavor.manifestPlaceholders = {UMENG_CHANNEL_VALUE:name}
    }
}
```

在AndroidManifest.xml文件中配置如下：

```xml
<meta-data
    android:name="APP_APPKEY"
    android:value="${app_appkey}"/>
```

3.不同的产品配置不同的依赖库

```groovy
dependencies {
    // 配置不同flavor的依赖库的格式为applicationVariantName#Compile '包名：group名：依赖库名称：版本'
    productACompile 'io.reactivex.rxjava2:rxjava:2.0.1'
    productATestCompile 'io.reactivex.rxjava2:rxjava:2.0.1'
    productBCompile 'io.reactivex.rxjava2:rxjava:2.0.1'
    debugCompile 'com.squareup.leakcanary:leakcanary-android:1.5.1'
    releaseCompile 'com.squareup.leakcanary:leakcanary-android:1.5.1'
}
```

4.不同产品不同代码和资源

对于不同的产品，可以设置不同的source set，见上文。通常，创建工程后，AndroidStudio默认帮我们创建了应用于所有产品的代码集main，它的对应的目录是src/main，我们也可以创建每个产品特有的代码集src/productA，src/productB这样，**名字**和**产品名字**是对应的。在编译的时候，默认会将这些代码集加入编译，这样就实现了不同产品，不同代码。这种用法也是非常广的，比如两个产品实现同样的接口，但是底层实现不一样。

### applicationVariants

buildTypes和productFlavors两两组合有很多种情况，如果每个组合都需要差异化处理的话，比如每个组合打包的apk文件名称均要求不同的话，通过applicationVariants可以很方便的配置，如下：

```groovy
android {
    // 配置不同版本的apk不同的名字，并按照build type分类存放
    applicationVariants.all { variant ->
        variant.outputs.each { output ->
            output.outputFile = new File(
                    output.outputFile.parent + "/${variant.buildType.name}",
                    "surveillance-${variant.buildType.name}-${variant.versionName}-                                          ${variant.productFlavors[0].name}.apk".toLowerCase())
        }
    }
}
```

再如，删除unaligned apk文件，如下所示：

```groovy
android.applicationVariants.all { variant ->
    variant.outputs.each { output ->
        // 删除unaligned apk
        if (output.zipAlign != null) {
            output.zipAlign.doLast {
                output.zipAlign.inputFile.delete()
            }
        }
    }
}
```

## 五、总结

Gradle是一个特别强大的构建工具，本文仅仅是从脚本配置角度来说明Gradle在构建Android应用时的主要使用说明，由于Gradle是基于Groovy的DSL语言，所以如果想要更深入的掌握Gradle的话，需要熟悉Groovy的语法，闭包等内容。然后，这只是一个初步的文档，后续会继续补充该文档。