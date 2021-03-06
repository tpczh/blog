#  Android性能优化

| tag     | author     | date       | history      |
| ------- | ---------- | ---------- | ------------ |
| Android | caizhenghe | 2018-03-18 | create doc   |
| Android | caizhenghe | 2018-03-20 | complete doc |

[TOC]

## 内存泄漏优化

> 内存泄漏的优化分为两方面，一方面是在开发过程中避免写出有内存泄漏的代码，另一方面是借助工具找出并解决潜在的内存泄漏。

### 泄漏场景

#### 静态成员

静态变量生命周期比Activity长，如果一个静态变量持有Activity的引用，会导致Activity销毁后依然无法被释放，从而引发内存泄漏。

```java
// MainActivity.java
private static Context sContext = this;
private static View sView = new View(this);
```

#### 单例模式

类似于静态变量，Activity被一个单例模式的对象持有，一直无法被释放。

```java
public class Single {
    public interface DataListner {
        void onDataChanged();
    }
    // Single内部维护着一个监听器列表，每个Listener都是一个接口
    private List<DataListner> mListeners = new ArrayList<DataListner>();
    
    public void registerListener(DataListner listener) {
        // Activity实现接口并注册到Single中。
        if (!mListeners.contains(listener))
        	mListeners.add(listener);
    }
    
    public void unRegisterListener(DataListner listener) {
        // 反注册接口
        mListeners.remove(listener);
    }
    
    private static class SingleHolder {
        // 实现lazy-loading单例模式
        public static final Single INSTANCE = new Single();
    }
    
    public static Single getInstance() {
        return SingleHolder.INSTANCE;
    }
    
}
```

若Activity注册监听事件之后没有及时的反注册，就会被单例Single对象一直持有，最终导致泄漏。

- 定义SingleHolder的目的是实现单例模式的Lazy-loading
- 非静态内部类无法定义静态成员（final static修饰的基础数据类型除外），详情查看[内部类](../java/2018-03-19-java内部类.md)

#### 属性动画

属性动画中有一种无限循环的动画，若没有在Activity的onDestroy中停止动画，该动画会一直播放下去（尽管没有在UI上显示出来），且动画会持有对应的View，又因为View会持有它所依附的Activity对象，最终就导致内存泄漏。

```java
ObjectAnimator animator = ObjAnimation.ofFloat(mButton, "rotation", 0, 360).setDuration(2000);
animator.setRepeatCount(ValueAnimator.INFINITE);
animator.start();
// animator.cancel();
```

> VIew持有Activity对象的原因：初始化View时需要传递一个Context参数，通常都会传入Activity的上下文对象，View会接收并持有这个上下文对象。

#### Handler（非静态内部类）

非静态内部类对象会隐式持有外围类对象的引用，当Handler发送了一条消息后，Message会持有Handler的引用(message.target)，而MessageQueue会持有Message的引用，Looper会持有MessageQueue的引用，ThreadLocal会持有Looper的引用。ThreadLocal是一个静态变量，生命周期与工程一致，也就是说，当Message没有被消耗时，Activity将一直无法被释放（即使该Activity已经被销毁），最终造成内存泄漏。引用链如下所示：

> ThreadLocal->Looper->MessageQueue->Message->Handler->Activity

解决思路：

- 在Activity的生命周期onPause中清空Handler的消息队列。之所以选择onPause是因为onPause具有实时性，而onStop和onDestroy会有一定延时，依然存在内存泄漏的可能。如果考虑到不希望在onPause中执行耗时操作，也可以在onStop和onDestroy中清空消息队列。
- 将Handler定义为静态内部类，不持有外围类对象的引用。

### 分析工具

#### LeakCanary

- 加入引用：

  ```groovy
   dependencies {
     debugCompile 'com.squareup.leakcanary:leakcanary-android:1.3'
     releaseCompile 'com.squareup.leakcanary:leakcanary-android-no-op:1.3'
   }
  ```

  ​

- 注册：

  ```Java
  public class ExampleApplication extends Application {

    public static RefWatcher getRefWatcher(Context context) {
      ExampleApplication application = (ExampleApplication) context.getApplicationContext();
      return application.refWatcher;
    }

    private RefWatcher refWatcher;

    @Override public void onCreate() {
      super.onCreate();
      refWatcher = LeakCanary.install(this);
    }
  }
  ```


- 使用：

  ```java
  public abstract class BaseFragment extends Fragment {
  
    @Override public void onDestroy() {
      super.onDestroy();
      RefWatcher refWatcher = ExampleApplication.getRefWatcher(getActivity());
      refWatcher.watch(this);
    }
  }
  ```


优点：使用简便，调用堆栈简单明了。

缺点：调用堆栈可能出错，只能检测Activity和Fragment的内存泄露。

#### profiler

1. 点击force gc后点击Dump java heap
2. 在Heap Dump界面选择Arrange by package，按包名排序，方便查找自己工程的类。
3. 可以查看有内存中有哪些对象

优点：使用简便。

缺点：无法查看对象的引用链，跟踪具体问题

#### MAT

1. 在Profiler的基础上下载hprof文件
2. 进入sdk->platform-tools文件夹，利用命令行工具执行hprof-conv命令，转换hprof文件：hprof-conv -z test.hprof test_mat.hprof
3. 下载MAT：<http://www.eclipse.org/mat/downloads.php> 
4. 使用MAT打开转换后的hprof文件，点击Histogram，输入我们想要查找的类，右键选择“Merge shortest path to GC Roots”，选择“exclude all ... reference”，就会显示出该对象完整的引用链。

优点：可以查看对象的引用链，内容具体，方便追踪问题。

缺点：使用复杂。

## 布局优化

> 减少无效的布局嵌套，在嵌套层级相同的情况下：FrameLayout > LinearLayout > RelativeLayout；

原因：RelativeLayout中的子View会执行两次onMeasure；LinearLayout若指定了weight，也会执行两次onMeasure；FrameLayout只会执行一次。

### merge标签

通常会与include标签搭配使用，在Adapter或者自定义ViewGroup中使用也比较频繁。下面是使用include属性的注意点：

- includ标签仅支持layout_开头的属性，比如不支持background属性
- 如果include中指定了layout开头的属性，则要求layout_width和layout_height必须存在
- 如果include标签和被包含的根布局同时指定了id属性，则以include中的id为准。

如果被包含的根布局是多余的，可以采用merge标签减少嵌套层级

### ViewStub

继承自View，宽/高均为0，意义在于按需加载所需的布局文件。比如网络异常时的界面，没必要在整个界面初始化的时候加载进来。示例代码如下：

```Xml
    <ViewStub
        android:id="@+id/stub_id"
        android:inflatedId="@+id/layout_id"
        android:layout="@layout/layout_import"
        android:layout_width="match_parent"
        android:layout_height="wrap_content" />
```

> 其中id时ViewStub的id，inflatedId是layout_import布局的根元素的id。

使用方式有如下两种：

```Java
((ViewStub)findViewById(R.id.stub_id)).setVisibilitu(View.VISIBLE);
View importView = ((ViewStub)findViewById(R.id.stub_id)).inflate();
```

ViewStub有以下几个局限性：

- 当ViewStub通过setVisibility或者inflate加载后，它就会被内部布局替换，也就是说它只能被加载一次，后续就不可再操作ViewStub
- ViewStub不支持merge标签

## 绘制优化

避免在View的onDraw中执行大量操作。主要有以下两方面：

- 不要在onDraw中创建局部对象，这样不仅会占用内存还容易引起内存抖动。
- 不要在onDraw中执行耗时操作，避免做大量的循环操作，容易抢占CPU的时间片。根据Android官方给出的性能优化的标准，View的绘制帧率保证60fps最佳，这就要求每帧的绘制时间不超过16ms（1000/60）

## 响应速度优化

避免在主线程做耗时操作，这主要会体现在Activity的启动速度上。通常Activity如果5秒内无法响应屏幕触摸事件或者键盘输入事件就会出现ANR，Broadcast如果10秒内还没有执行完操作，或者Service20秒未响应就会出现ANR。

### ANR日志分析

当一个进程发生ANR后，会在	/data/anr目录下生成一个文件traces.txt。通过分析这个文件来定位ANR的原因。

导出traces文件：

```
adb pull /data/anr/traces.txt 电脑本机路径
```

### ANR分析工具

#### WatcherDog

设置一个定时睡眠的子线程，每隔固定时间往主线程丢一个标志符，下次醒来后若标志符被主线程修改则说明没有发生ANR。

缺点：可能会遗漏ANR问题

#### AndroidPerformanceMonitor

通过Looper的setLogging方法在执行任务前和任务后分别打印自定义的日志，判断是哪个任务发生了耗时操作。

缺点：部分情况无法捕获ANR异常，比如在onTouchEvent中执行耗时操作。（猜测此时还未将触摸事件作为消息分发给主线程的Looper，因此没有捕获ANR）

## APP启动速度优化

APP启动分为两种：冷启动和热启动。

- 冷启动：当应用启动时，后台没有该应用的进程，系统会重新创建一个进程并分配给该应用（此时会重新创建Application）
- 热启动：已经给应用分配了进程，但是点击Home、返回键回到桌面或者进入其它程序，再重新打开该应用

### 白屏问题

> AS2.0之后的Instant Run带来的问题，通常在release版本的程序中不会出现该问题

解决方案在style文件的AppTheme标签中添加两个属性，代码如下：

```xml
<item name="android:windowIsTranslucent">true</item> 
<item name="android:windowNoTitle">true</item>
```

### 测量启动时间

主要有两种方式：

- 在Activity的onCreate中调用reportFullyDrawn方法：

  ```java
  try{
      reportFullyDrawn();
  }catch(SecurityException e){
  }
  ```

- 使用adb命令生成APP启动视频，在视频中会显示每一帧画面的时间戳，分别记录**APP图标高亮**这一帧与**UI绘制出内容**这一帧的时间戳，并计算差值，adb命令如下：

  ```Shell
  $ adb shell screenrecord --bugreport /sdcard/launch.mp4
  $ adb pull /sdcard/launch.mp4
  ```

### 优化冷启动时间

> 冷启动时间是指当用户点击APP图标到系统调用Activity.onCreate之间的时间段，在这段时间内，系统会先加载app主题样式中的windowBackground作为app的预览元素，然后再去真正加载activity的layout布局

可以通过一些小技巧在视觉上加快加载速度，核心思想就是自定义主题样式中的windowBackground图片，流程如下：

1. 为启动的Activity自定义一个Theme

   ```Xml
   <style name="AppTheme.Launcher">
       <item name="android:windowBackground">@drawable/window_background_statusbar_toolbar_tab</item>
   </style>
   ```

2. 将新的Theme应用到设置到AndroidManifest.xml中

   ```xml
   <activity
       android:name=".MainActivity"
       android:theme="@style/AppTheme.Launcher">
     
       <intent-filter>
           <action android:name="android.intent.action.MAIN" />
           <category android:name="android.intent.category.LAUNCHER" />
       </intent-filter>
   </activity>
   ```

3. 由于给MainActivity设置了一个新的Theme，这样做会覆盖原来的Theme，所以在MainActivity中需要设置回原来的Theme

   ```java
   public class MainActivity extends AppCompatActivity {
    
       @Override
       protected void onCreate(Bundle savedInstanceState) {
    
           // Make sure this line comes before calling super.onCreate().
           setTheme(R.style.AppTheme);
    
           super.onCreate(savedInstanceState);
       }
   }
   ```

   ​

## ListView和Bitmap优化

### ListView加载图片卡顿

从三方面解决：

- 避免在主线程中加载图片

- 降低ListView加载图片的频率。可以设置标识位，在滑动列表时不进行加载图片的操作，在IDLE状态下再通知列表加载图片（notify）

- 启动硬件加速：在Manifest中设置Activity属性：

  ```Xml
  android:hardwareAccelerate="true"
  ```

  ​

### Bitmap OOM

加载Bitmap之前设置采样率压缩图片，避免OOM，详情查看[图片加载](2018-03-02-Android图片加载.md)

## 线程优化

推荐使用线程池ThreadPoolExecutor来创建和管理线程，详情查看[Android线程](2018-03-20-Android线程和线程池.md)