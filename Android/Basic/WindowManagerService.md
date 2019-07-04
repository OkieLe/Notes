# GUI-WMS

[参考资料](http://www.cnblogs.com/samchen2009/)

## 导言

Android的GUI系统是Android最重要也最复杂的系统之一。它包括以下部分：
1. 窗口和图形系统 - Window and View Manager System.
2. 显示合成系统 - Surface Flinger
3. 用户输入系统 - InputManager System
4. 应用框架系统 - Activity Manager System.

它们之间的关系如下图所示：
![services-relationship](../../_attach/services-relationship.png)

本文主要介绍第一部分的WMS。

## 基本概念

### 1. Window, PhoneWindow 和 Activity

- Activity 是Android 应用的四大组件之一（Activity， Service,  Content Provider,  Broadcast Receiver)， 也是唯一一个与用户直接交互的组件。
- Window 在不同的地方有着不同的含义。在Activity里，Window 是一个抽象类，代表了一个矩形的不可见的容器，里面布局着若干个可视的区域（View）。每个Activity都会有一个Window类成员变量mWindow。而在WindowManagerService里，Window指的是WindowState对象，WindowState与一个 ViewRootImpl里的mWindow对象相对应。所以说，WindowManagerService里管理的Window其实是 Acitivity的ViewRoot。下面提到的Window，如果没有做特殊说明，均指的是WindowManagerService里的 ‘Window' 概念，即一个特定的显示区域。从用户角度来看，Android是个多窗口的操作系统，不同尺寸的窗口区域根据尺寸，位置，z-order及是否透明等参数叠加起来一起并最终呈现给用户。这些窗口既可以是来自一个应用，也可以来自与多个应用，这些窗口既可以显示在一个平面，也可以是不同的平面。总而言之，窗口是有层次的显示区域，每个窗口在底层最终体现为一个个的矩形Buffer, 这些Buffer经过计算合成为一个新的Buffer，最终交付Display系统进行显示。为了辅助最后的窗口管理，Android定义了一些不同的窗口类型：
    - 应用程序窗口 (Application Window): 包括所有应用程序自己创建的窗口，以及在应用起来之前系统负责显示的窗口。
    - 子窗口(Sub Window)：比如应用自定义的对话框，或者输入法窗口，子窗口必须依附于某个应用窗口（设置相同的token)。
    - 系统窗口(System Window): 系统设计的不依附于任何应用的窗口，比如说，状态栏(Status Bar), 导航栏(Navigation Bar), 壁纸(Wallpaper), 来电显示窗口(Phone)，锁屏窗口(KeyGuard), 信息提示窗口(Toast)， 音量调整窗口，鼠标光标等等。
- PhoneWindow 是Activity Window的扩展，是为手机或平板设备专门设计的一个窗口布局方案。

### 2. View, DecorView, ViewGroup, ViewRoot

View 是一个矩形的可见区域。

ViewGroup 是一种特殊的View，它可以包含其他View并以一定的方式进行布局。Android支持的布局有FrameLayout, LinearLayout, RelativeLayout 等。

DecorView 是FrameLayout的子类，FrameLayout 也叫单帧布局，是最简单的一种布局，所有的子View在垂直方向上按照先后顺序依次叠加，如果有重叠部分，后面的View将会把前面的View挡住。我们 经常看到的弹出框，把后面的窗口挡住一部分，就是用的FrameLayout布局。Android的窗口基本上用的都是FrameLayout布局, 所以DecorView也就是一个Activity Window的顶级View, 所有在窗口里显示的View都是它的子View.

ViewRoot 可以认为所有被addView()调用的View是ViewRoot, 因为接口将会生成一个ViewRootImpl 对象，并保存在WindowManagerGlobal的mRoots[] 数组里。一个程序可能有很多了ViewRoot（只要多次调用addView()), 在WindowManagerService端看来，就是多个Window。但在Activity的默认实现里，只有mDecorView通过addView 添加到WindowManagerService里( 见如下代码)
```Java
//frameworks/base/core/java/android/app/Activity.java
void makeVisible() {
    if (!mWindowAdded) {
        ViewManager wm = getWindowManager();
        wm.addView(mDecor, getWindow().getAttributes());
        mWindowAdded = true;
    }
    mDecor.setVisibility(View.VISIBLE);
}
```
一般情况下，我们可以说，一个应用可以有多个Activity，每个 Activity 一个Window(PhoneWindow)， 每个Window 有一个DecorView, 一个ViewRootImpl, 对应在WindowManagerService 里有一个Window(WindowState).

### 3. ViewRootImpl,  WindowManagerImpl,  WindowManagerGlobals

WindowManagerImpl: 实现了WindowManager 和 ViewManager的接口，但大部分是调用WindowManagerGlobals的接口实现的。

WindowManagerGlobal: 一个Singleton对象，对象里维护了三个数组：
- mRoots[]: 存放所有的ViewRootImpl
- mViews[]: 存放所有的ViewRoot
- mParams[]: 存放所有的LayoutParams.

同时，它还维护了两个全局IBinder对象，用于访问WindowManagerService 提供的两套接口：
- IWindowManager:  主要接口是openSession(), 用于在WindowManagerService 内部创建和初始化Session, 并返回IBinder对象。
- IWindowSession:  是Activity Window与WindowManagerService 进行对话的主要接口

ViewRootImpl 在整个Android的GUI系统中占据非常重要的位置，从下图可以看到，ViewRootImpl 与用户输入系统(接收用户按键，触摸屏输入), 窗口系统（复杂窗口的布局，刷新，动画），显示合成系统（包括定时器Choreographer, SurfaceFlinger), 乃至Audio系统（音效输出）等均有密切的关联。

![viewrootimpl](../../_attach/viewroot_impl.png)

三者( ViewRootImpl, WindowManagerImpl, WindowManagerGlobal) 都存在于应用（有Activity)的进程空间里，一个Activity对应一个WindowManagerImpl, 一个DecorView(ViewRoot),以及一个ViewRootImpl (上面说过，实际一个Activity只有一个DecorView），而WindowManagerGlobal是一个全局对象，一个应用永远只有一 个。

注意的是，在某些情况下，一个应用可能会有几个ViewRootImpl对象，比如说ANR是弹出的对话框，或是网页里面一个视频窗口 （SurfaceView), 在WindowManagerService看来，它们也是一个窗口。同时SystemServer的进程空间也有自己的 WindowManagerGlobal和若干个ViewRoot， 因为WindowManagerService 内部也会管理某些系统窗口，如手机顶部的StatusBar, 手机底部的NavigationBar, 以及锁屏窗口，这些窗口不属于某个特定的Activity。

### 4. WindowManager, WindowManagerService 和 WindowManagerPolicy

WindowManager：是一个接口类，定义了一些接口来管理Acitivity里的窗口。WindowManager 是Android应用进程空间里的一个对象，不提供IPC服务。

WindowManagerService：是SystemServer进程里的一个Service，它的主要功能有：
- 窗口的显示刷新。这里的'Window'是ViewRoot, 和上面WindowManager管理的'Window' 是不一样的，前者是实实在在要进行显示的‘窗口’，而后者只是一个View的容器，并不会显示出来。大部分情况下，Android同时只有一个Activity工作，但这并不意思着只有一个Window被显示，Android可能会同时显示来自相同或不同应用的多个Window，比如说，屏幕的上方有一个状态栏，最下方有一个导航栏，有时会弹出一些对话框，背景可能会显示墙纸，在应用启动过程中，会有动画效果，这个时候两个Activity的窗口会有所变形且同时显示出来，这一切都需要 WindowManager来控制何时何地以何种方式将所有的窗口整合在一起显示。
- 预处理用户输入事件，并分发给合适的窗口进行处理。
- 输出显示(Display)管理，包括WifiDisplay。

WindowManagerPolicy 是为WindowManagerService提供UI相关的操作接口，WindowManagerService启动时实例化一个Policy（PhoneWindowManager等），用于实现窗口的堆叠、布局、按键分发以及特殊的窗口类型。

### 5. Token, WindowToken, AppWindowToken, ApplicationToken, appToken

Token在英语中表示标记，信物的意思，在代码中用来标识某个特定的对象。在Android的窗口系统中，有很多的’Token', 它们代表着不同的含义。

- WindowToken: 是在WindowManagerService 中定义的一个基类，顾名思义，它是用来标识某一个窗口。和下面的appWindowToken相比， 它不属于某个特定的Activity,  比如说输入法窗口，状态栏窗口等等。
- AppWindowToken: 顾名思义，它是用来标识app, 跟准确的说法，是用来标识某个具体的Activity.
- ApplicationToken: 指的是ActivityRecord类里的Token子类。AppWindowToken里的appToken也就是它。
- appToken: 和applicationToken是一个意思。

一个WindowToken下面带一个WindowList队列，里面存放着隶属与这个Token的所有窗口。当一个Window 加入WindowManagerService 管理时，必须指定他的Token值，WindowManagerService维护着一个Token与WindowState的键值Hash表。

### 6. Surface, Layer 和 Canvas, SurfaceFlinger, Region， LayerStack

在Android中，Window与Surface一一对应。 如果说Window关心的是层次和布局，是从设计者角度定义的类，Surface则从实现角度出发，是工程师关系和考虑的类。Window的内容是变化 的，Surface需要有空间来记录每个时刻Window的内容。在Android的SurfaceFlinger实现里，通常一个Surface有两块 Buffer, 一块用于绘画，一块用于显示，两个Buffer按照固定的频率进行交换，从而实现Window的动态刷新。

Layer是SurfaceFlinger 进行合成的基本操作单元。Layer在应用请求创建Surface的时候在SurfaceFlinger内部创建，因此一个Surface对应一个 Layer, 但注意，Surface不一定对应于Window，Android中有些Surface并不跟某个Window相关，而是有程序直接创建，比如说 StrictMode，一块红色的背景，用于提示Java代码中的一些异常, 还有SurfaceView, 用于显示有硬件输出的视频内容等。

当多个Layer进行合成的时候，并不是整个Layer的空间都会被完全显示，根据这个Layer最终的显示效果，一个Layer可以被划分成很多的Region, Android SurfaceFlinger 定义了以下一些Region类型：
-  TransparantRegion： 完全透明的区域，在它之下的区域将被显示出来。
-  OpaqueRegion: 完全不透明的区域，是否显示取决于它上面是否有遮挡或是否透明。
-  VisibleRegion: 可见区域，包括完全不透明无遮挡区域或半透明区域。
-  CoveredRegion: 被遮挡区域，在它之上，有不透明或半透明区域。
-  DirtyRegion: 可见部分改变区域，包括新的被遮挡区域，和新的露出区域。

Android 系统支持多种显示设备，比如说，输出到手机屏幕，或者通过WiFi 投射到电视屏幕。Android用Display类来表示这样的设备。不是所有的Layer都会输出到所有的Display, 比如说，我们可以只将Video Layer投射到电视， 而非整个屏幕。LayerStack 就是为此设 计，LayerStack 是一个Display 对象的一个数值， 而类Layer里也有成员变量mLayerStack， 只有两者的mLayerStack 值相同，Layer才会被输出到给该Display设备。所以LayerStack 决定了每个Display设备上可以显示的Layer数目。

SurfaceFlinger的工作内容，就是定期检查所有Layer的参数更新（LayerStack等），计算新的DirtyRegion， 然后将结果推送给底层显示驱动进行显示。这里面有很多的细节，我们将在另外的章节专门研究。

上面描述的几个概念，均是针对于显示这个层面，更多是涉及到中下层模块，应用层并不参与也无需关心。对于应用而言，它关心的是如何将内容画出来。Canvas 是Java层定义的一个类，它对应与Surface上的某个区域并提供了很多的2D绘制函数（借助于底层的Skia或OpenGL)。应用只需通过 LockCanvas() 来获取一个Canvas对象，并调用它的绘画方法，然后 unLockCanvasAndPost()来通知底层将更新内容进行显示。当然，并不是所有应用程序都需要直接操作Canva， 事实上只有少量应用需要直接操作Canvas, Android提供了很多封装好的控件 Widget，应用只需提供素材，如文字，图片，属性等等，这些控件会调用Canvas提供的接口帮用户完成绘制工作。

### 7. SurfaceFlinger, HWComposer, OpenGL 和 Display

SurfaceFlinger 是一个独立的Service， 它接收所有Window的Surface作为输入，根据ZOrder， 透明度，大小，位置等参数，计算出每个Surface在最终合成图像中的位置，然后交由HWComposer或OpenGL生成最终的显示Buffer, 然后显示到特定的显示设备上。

HWComposer 是 Andrid 4.0后推出的新特性，它定义一套HAL层接口，然后各个芯片厂商根据各种硬件特点来实现。它的主要工作是将SurfaceFlinger计算好的Layer的显示参数最终合成到一个显示Buffer上。注意的是，Surface Flinger 并非是HWComposer的唯一输入，有的Surface 不由Android的WindowManager 管理，比如说摄像头的预览输入Buffer， 可以有硬件直接写入，然后作为HWComposer的输入之一与SurfaceFlinger的输出做最后的合成。

OpenGL 是一个2D/3D图形库，需要底层硬件(GPU)和驱动的支持。在Android 4.0后，它取代Skia成为Android 的2D 绘图图形库，大部分的控件均改用它来实现，应用程序也可以直接调用OpenGl函数来实现复杂的图形界面。

Display 是Android 对输出显示设备的一个抽象，传统的Display 设备是手机上的LCD屏，在Andrid 4.1 后，Android 对SurfaceFlinger 进行了大量的改动，从而支持其他外部输入设备，比如HDMI， Wifi Display 等等。Display的输入是根据上面的LayerStack值进行过滤的所有Window的Surface， 输出是和显示设备尺寸相同的Buffer, 这个Buffer 最终送到了硬件的FB设备，或者HDMI设备，或者远处的Wifi Display Sink设备进行显示。输入到输出这条路径上有SurfaceFlinger, OpenGL 和 HWComposer。


## 窗口管理 (View, Canvas, WindowManager)

Android 应用程序创建的大概流程是 ActivityManagerService -> Zygote -> Fork App，然后应用程序在ActivityThread 中进入loop循环等待处理来自AcitivyManagerService的消息。如果一个Android的应用有Acitivity， 那它起来后的第一件事情就是将自己显示出来。

![View WindowManager](../../_attach/window_view_wms.png)

Android 中跟窗口管理相关（不包括显示和按键处理）主要有两个进程，Acitivty所在进程和WndowManagerService 所在进程（SystemServer).  上图中用不同颜色区分这两个进程，黄色的模块运行在Activity的进程里，绿色的模块则在SystemServer内部，本文主要讨论的是WindowManager Service。它们的分工是：Activity进程负责窗口内View的管理，而WindowManagerService 管理来自与不同Acitivity以及系统的窗口。

###1.  Activity显示准备

一个新的应用被fork完后，第一个调用的方法就是 ActivityThread的main()，这个函数主要就是创建一个ActivityThread线程，然后调用loop()开始等待。当收到来自 ActivityManager 的 LAUNCH_ACTIVITY 消息后，Activity开始了他的显示之旅。下图描绘的是Activity在显示前的准备流程。

![](../../_attach/acvitity_surface_create.png)

图分为三部分， 右上角是Acitivity应用的初始化。中间部分是Acitivity 与WindowManagerService的交互准备工作，左下角是window显示的开始。本文主要描述后两部分。

1. Activity内部的准备过程，这里面有一个重要对象，ContextImpl 生成，它是Context类的具体实现，它里面封装了应用程序访问系统资源的一些基本API，比如说连接某一个服务并获取其IBinder，发送Intent， 获取应用程序的信息，访问数据库等等，在应用看来，它就是整个AndroidSDK的入口。ContextImpl 除了实现函数，里面还维护成员变量，其中有一个mDisplay，代表当前应用输出的显示设备，如果应用没有特别指定，一般指向系统的默认显示输出，比如手机的液晶屏。
2. ViewRootImpl 的地位相当与MVC架构中的C，Controller是连接View和Modal的关键，所以需要首先创建它。当addView(view, param)被调用的时候，一个ViewRoot就被创建出来。
3. ViewRootImpl 在构造过程成初始化一些重要的成员变量，包括一个Surface对象（注意这是一个空的Surface对象，没有赋给任何有效的值，后面会通过CopyFromParcel来填充），还有mChoreographer定时器（Singleton对象，每个进程只有一个），与此同时，ViewRootImp通过WindowManagerGlobal创建了一个和WindowManagerService 的通话通道，接下来会利用这条通道做进一步的初始化工作。
4. 还是在addView()里，WindowManagerImpl拿到ViewRoot对象后调用它的setView方法，将view, layout参数交给ViewRootImpl开始接管。在setView()里ViewRootImpl做进一步的初始化工作，包括创建一个InputChannel接收用户按键输入，enable图形硬件加速，请求第一次的Layout等等，这里跟WindowManagerService 有关系的就是加入到WindowManager的窗口管理队列中。这个函数是 addToDisplay()，addToDisplay() 最终会调到WindowManagerService的addWindow() 接口。
5. addWindow() 里首先生成了一个WindowState对象，它是ViewRootImpl 在WindowManagerService端的代表。在它的构造函数里，WindowState 会生成IWindowId.Stub 对象和DeathRecipient对象来分别监听Focus和窗口死亡的信息，根据用户传进来的Window Type计算出窗口的mBaseLayer，mSubLayer和mLastLayer值，分别对应于主窗口，主窗口上弹出的子窗口（如输入法），以及动画时分别对应的ZOrder值，生成一个WindowStateAnimation 负责整个Window的动画，并在内部将windowToken, appWindowToken等关联起来。
6. WindowManagerService 调用openInputChannelPair() 和 RegisterInputChannel(), 创建用于通信的SocketPair，将其传给InputManagerService，用于接下来的用户输入事件对应的响应窗口
7. 最后，WindowManagerService 调用WindowState的attach()，创建了一个Surface Session 并将Surface Session，WindowSession 还有WindowState 三者关联起来
8. WindowManagerService 调用 assignLayersLocked()计算所有Window的Z-Order。
9. addToDisplay() 返回，ViewRootImpl 和 WindowManagerService 内部的准备工作就绪。ActivityThread会发送ACTIVITY_RESUMED消息告诉Activity显示开始。可以是图还没有画，不是吗？对的，此刻Surface还没有真正初始化（前面说过ViewRootImpl只是New了一个空的对象，需要有人往里面填东西）。底层存放绘制结果的Buffer也没有创建，但是最多16ms以后这一切就会开始。

### 2. Choreographer 和 Surface的创建

所有的图像显示输出都是由时钟驱动的，这个驱动信号称为VSYNC。这个名词来源于模拟电视时代，在那个年代，因为带宽的限制，每一帧图像都有分成两次传输，先扫描偶数行（也称偶场）传输，再回到头部扫描奇数行（奇场），扫描之前，发送一个VSYNC同步信号，用于标识这个这是一场的开始。场频，也就是VSYNC 频率决定了帧率（场频/2). 在现在的数字传输中，已经没有了场的概念，但VSYNC这一概念得于保持下来，代表了图像的刷新频率，意味着收到VSYNC信号后，我们必须将新的一帧进行显示。

VSYNC一般由硬件产生，也可以由软件产生（如果够准确的话），Android 中VSYNC来自于HWComposer，接收者就是Choreographer。Choreographer英文意思是编舞者，跳舞很讲究节奏不是吗，必须要踩准点。Choreographer 就是用来帮助Android的动画，输入，还有显示刷新按照固定节奏来完成工作的。看看Chroreographer 和周边的类结构。

![](../../_attach/choreographer_class.png)

Choreographer 是ViewRootImpl 创建的，它拥有一个Receiver，用来接收外部传入的Event，它还有一个Callback Queue，里面存放着若干个CallbackRecord，还有一个FrameHandler，用来handleMessage， 最后它还跟Looper有引用关系。再看看下面这张时序图：

![](../../_attach/choreographer_seq.png)

首先Looper调用loop() 后，线程进入进入睡眠，直到收到一个消息。Looper也支持addFd()方法，这样如果某个fd上发生了IO操作(read/write)，它也会从睡眠中醒来。Choreographer的实现用到了这两种方式，首先他通过某种方式获取到SurfaceFlinger 进程提供的fd，然后将其交给Looper进行监听，只要SurfaceFlinger往这个fd写入VSync事件，looper便会唤醒。Looper唤醒后，会执行onVsync()，这里面只是调用Handler接口 sendMessageAtTime() 往消息队列里又送了一个消息。这个消息最终调到了Handler （实际是FrameHandler)的handleCallback来完成上层安排的工作。为什么要绕这么大个圈？为什么不在onVSync里直接handleCallback()? 毕竟onVSync 和 handleCallback() 都在一个线程里。这是因为MessageQueue 不光接收来自SurfaceFlinger 的VSync 事件，还有来自上层的控制消息。VSync的处理是相当频繁的，如果不将VSync信号送人MessageQueue进行排队，MessageQueue里的事件就有可能得不到及时处理，严重的话会导致溢出。当然了，如果因为VSync信号排队而导致处理延迟，这就是设计的问题了，这也是为什么Android文档里反复强调在Activity的onXXX()里不要做太耗时的工作，因为这些回调函数和Choreographer运行在同一个线程里，这个线程就是所谓的UI线程。

言归正传，继续往前，VSync事件最终在doFrame()里调了三次doCallbacks()来完成不同的功能, 分别处理用户输入事件，动画刷新（动画就是定时更新的图片)，最后执行performTraversals()，这个函数里面主要是检查当前窗口当前状态，比如说是否依然可见，尺寸，方向，布局是否发生改变（可能是由前面的用户输入触发的），分别调用performMeasure()， performLayout, performDraw()完成测量，布局和绘制工作。我们会在后面详细学习这三个函数，这里我们主要看一下第一次进入performTraversals的情况，因为第一次会做些初始化的工作，最重要的一件就是创建Surface对象。

回看显示前的准备流程图，可以看到Surface的创建不是在Activity进程里，而是在WindowManagerService完成的。当一个Activity第一次显示的时候，Android显示切换动画，因此Surface是在动画的准备过程中创建的，具体发生在类WindowStateAnimator的createSurfaced()函数。它最终创建了一个SurfaceControl 对象，WindowManagerService 只能访问SurfaceControl（不能访问Surface），它主要控制Surface的创建，销毁，Z-order，透明度，显示或隐藏，等等。而真正的更新者，View会通过Canvas的接口将内容画到Surface上。那View怎么拿到WMService创建的Surface，答案是下面的代码里，SurfaceControl 被转换成一个Surface对象，然后传回给ViewRoot，前面创建的空的Surface现在有了实质内容。Surface通过这种方式被创建出来，Surface对应的Buffer 也相应的在SurfaceFlinger内部通过HAL层模块（GRAlloc)分配并维护在SurfaceFlinger 内部，Canvas通过dequeueBuffer() 接口拿到Surface的一个Buffer，绘制完成后通过queueBuffer() 还给SurfaceFlinger进行绘制。
```Java
SurfaceControl surfaceControl = winAnimator.createSurfaceLocked();
if (surfaceControl != null) {
    outSurface.copyFrom(surfaceControl);
    if (SHOW_TRANSACTIONS)
        Slog.i(TAG, "  OUT SURFACE " + outSurface + ": copied");
    } else {
        outSurface.release();
    }
    ...
}
```
到这里，我们知道了Activity的三大工作：用户输入响应，动画，和绘制都是由一个定时器驱动的，Surface在Activity第一次启动时由WindowManagerService创建。接下来我们具体看一下View是如何画在Surface Buffer上的。

### 3. View的Measure, Layout 和 Draw

#### Measure

直接从前面提到的performMeasure()函数开始

![](../../_attach/view_measure_seq.png)

因为递归调用，实际的函数调用栈比这里显示的深很多，这个函数会从View的结构树顶（DecorView), 一直遍历到叶节点。中间会经过三个基类：DecorView， ViewGroup 和 View 它们的类结构如下图所示：

![](../../_attach/view_class.png)

所有可见的View(不包括DecorView 和 ViewGroup)都是一个矩形，Measure的目的就是算出这个矩形的尺寸，mMeasuredWidth 和 mMeasuredHeight (注意，这不是最终在屏幕上显示的尺寸），这两个尺寸的计算受其父View的尺寸和类型限制，这些信息存放在 MeasureSpec里。widthMeasureSpec 和 heightMeasureSpec 作为 onMeasure的参数出入，子View根据这两个值计算出自己的尺寸，最终调用 setMeasuredDimension() 更新mMeasuredWidth 和 mMeasuredHeight.

performMeasure() 结束后，所有的View都更新了自己的尺寸，接下来进入performLayout().

performLayout() 的流程和performMeasure() 基本上一样，可以将上面图中的measure() 和 onMeasure()  简单的换成 layout() 和 onLayout()， 也是遍历整课View树，根据之前算出的大小将每个View的位置信息计算出来。这里不做太多描述，我们把重心放到performDraw()，因为这块最复杂，也是最为重要的一块。

> #### Android Graphics Hardware Acceleration

> Android2.3 到 Android 3.0/4.0 的性能的巨大提升，UI界面的滑动效果一下变得顺滑很多，到底是framework的什么改动带来的？

> 如标题所述，最根本的原因就是引入了硬件加速，GPU是专门优化图形绘制的硬件单元，很多GPU都支持OpenGL，一种开放的跨平台的3D绘图API。Android3.0以前，几乎所有的图形绘制都是由Skia完成，Skia是一个向量绘图库，使用CPU来进行运算，所以它的performance是一个问题（当然，Skia也可以用GPU进行加速，有人在研究，但好像GPU对向量绘图的提升不像对Opengl那么明显），所以从Android3.0 开始，Google用hwui取代了Skia，准确的说，是推荐取代，因为Opengl的支持不完全，有少量图形api仍由Skia完成，另外还要考虑到兼容性，硬件加速的功能并不是默认打开，需要程序在AndroidManifests.xml 或代码里控制开关。当然，大部分Canvas的基本操作都通过hwui重写了，hwui下面就是Opengl和后面的GPU，这也是为什么Android 4.0的launcher变得异常流畅的缘故。OK，那我们接下来的重点就是要分析HWUI的实现了。

> 在此之前，简单的介绍一下OpenGL的一些概念，否则很难理解。要想深入理解Opengl，请必读经典的红包书：http://www.glprogramming.com/red/

> Opengl说白了，就是一组图形绘制的API。 这些API都是一些非常基本的命令，通过它，你可以构造出非常复杂的图形和动画，同时，它又是跟硬件细节无关的，所以无需改动就可以运行在不同的硬件平台上（前提是硬件支持所需特性）。OpenGL的输入是最基本几何元素(geometric primitives), 点(points), 线(lines), 多边形(polygons), 以及bitmap和pixel data, 他的输出是一个或两个Framebuffer(真3D立体)。

> *Vertex data*：所有的几何元素（点线面）都可以用顶点(vertics)来描述, 每个点都对应三维空间中的一个坐标(x,y,z)，改变若干点的位置，便可以构造出一个立体的图形。

> 仅仅使用Opengl 和 GPU 取代Skia 就能够大幅提升性能？答案当然不是，性能的优化很大程度上取决于应用，应用必须正确的使用Opengl命令才能发挥其最大效能。Android提供了两种机制来提升性能：
> - 一个就是Display List，就是OpenGL命令的缓存。想象一个复杂物体，需要大量的OpenGL命令来描绘，如果画一次都需要重新调用OpenGL API，并把它转换成Vertex data，显然是很低效的，如果把他们缓存在Display List里，需要重绘的时候，发一个个命令通知OpenGL直接从Display List 读取缓存的Vertex Data，那势必会快很多，如果考虑到Opengl是基于C/S架构，可以支持远程Client，这个提升就更大了。Display也可以缓存BitMap 或 Image, 举个例子，假设要显示一篇文章，里面有很多重复的字符，如果每个字符都去字库读取它的位图，然后告诉Opengl去画，那显然是很慢的。但如果将整个字库放到Display List里，显示字符时候只需要告诉Opengl这个字符的偏移量，OpenGL直接访问Display List，那就高效多了。
> - 另一个叫 Hardware Layer, 即便是使用了DisplayList，对于复杂的图形，仍然需要执行大量的OpenGL命令，如果需要对这一部分进行优化，就需要使用到 HardwareLayer对绘制的图形进行缓存，如果图形不发生任何变化，就不需要执行任何OpenGL命令，而是将之前缓存在GPU内存的Buffer直接与其他View进行合成，从而大大的提高性能。这些存储在GPU内部的Buffer就称为 Hardware Layer。比如说Android的墙纸，一般来说，他是不会发生变化的，因此我们可以将它缓存在Hardware Layer里，这张就不需要每次进行拷贝和重绘，从而大幅提升性能。

> 说白了，优化图形性能的核心在于 1）用硬件来减少CPU的参与，加速图形计算。 2）从软件角度，通过Display List 和 Hardware Layer, 将已经完成的工作尽可能的缓存起来，只做必须要做的事情，尽可能的减少运算量。

> GLES20：就是OpenGL ES 2.0 的API。它的实现一般由GPU的设计厂家提供，可以在设备的/system/lib/egl/ 找到它的so，名字为 libGLES_xxx.so, xxx 就是特定设备的代号，比如说 libGLES_android.so, 是Google提供的软件实现。

> EGL：虽然对于上层应用来说OpenGL接口是跨平台的，但是它的底层(GPU)实现和平台(SoC)是紧密相关的，于是OpenGL组织定义一套接口用来访问平台本地的窗口系统（native platform window system)，这套接口就是EGL，比如说 eglCreateDisplay(), eglCreateSurface(), eglSwapBuffer()等等。

#### Canvas, Renderer, DisplayList 实现

**Canvas**

Canvas是Java层独有的概念，它为View提供了大部分图形绘制的接口。这个类主要用于纯软件的绘制，硬件加速的图形绘制则由DisplayListCanvas取代。

**DisplayListCanvas**

DisplayListCanvas(Java)是Canvas的扩展类，用于记录绘制操作的GL Canvas，对应的native类把Java传入的绘制操作记录到DisplayList。DisplayListCanvas(Java)与DisplayList(native)配合，防止在native使用Bitmap过程中内存被上层释放。

**ThreadedRenderer**

把渲染工作通过RenderProxy转发到RenderThread的硬件渲染工具类，通过访问各种Canvas来控制绘制流程。

#### Draw

![](../../_attach/draw_renderer.png)

绘制流程从performDraw()开始，主要的工作由ThreadedRenderer完成，进入到hwui后实际是由RenderThread执行最后的GL命令，RenderThread后面进行专门讨论。

除了DisplayList、HardwareLayer，Android 还支持Software Layer，和Hardware Layer不同之处在于，它不存在于GPU内部，而是存在CPU的内存里，因此它不经过前面所说的 Hardware Render Pipeline， 而是走Android最初的软件Render pipeline。不管是Hardware Layer 还是 Software Layer， 在draw() 内部均称为Cache，只要有Cache的存在，相对应的View将不用重绘，而是使用已有的Cache。Hardware Layer， Software Layer 和 Display List 是互斥的，同时只能有一种方法生效（当然，Hardware Layer的第一次绘制还是通过Display List 完成），下表总结了它们的差别, 从中可以看到，Hardware Layer 对性能的提升是最大的，唯一的问题是占用GPU的内存（这也是为什么显卡的内存变得越来越大的原因之一），所以一般来说，Hardware Layer使用在那些图片较为复杂，但不经常改变，有动画操作或与其他窗口有合成的场景，比如说WallPaper, Animation 等等。

|                | Enabled if                  | GPU accelerated? | Cached in       | Performance(Google I/O 2001) | Usage                                    |
| -------------- | --------------------------- | ---------------- | --------------- | ---------------------------- | ---------------------------------------- |
| Display List   | Hardware Accelerated = True | Y                | GPU DisplayList | 2.1                          | Complex View                             |
| Hardware Layer | LayerType = HARDWARE        | Y                | GPU Memory      | 0.009                        | Complex View, Color Filter(颜色过滤）, Alpha blending （透明度设置）, etc. |
| Software Layer | LayerType = SOFTWARE        | N                | CPU Memory      | 10.3                         | No Hardware Accelerated, Color filter, Alpha blending, etc. |

### 4. 窗口管理WMS

到此，我们已经了解了一个Acitivty(Window)是如何画出来的，让我们在简要重温一下这个过程：

1. Acitivity创建， ViewRootImpl将窗口注册到WindowManager Service，WindowManager Service 通过SurfaceFlinger 的接口创建了一个Surface Session用于接下来的Surface 管理工作。
2. 由Surface Flinger 传上来的VSYNC事件到来，Choreographer 会运行ViewRootImpl 注册的Callback函数， 这个函数会最终调用 performTraversal 遍历View树里的每个View， 在第一个VSYNC里，WindowManagerService 会创建一个SurafceControll对象，ViewRootImpl 根据Parcel返回的该对象生成了Window对应的Surface对象，通过这个对象，Canvas 可以要求SrufaceFlinger 分配OpenGL绘图用的Buffer。
3. View树里的每个View 会根据需要依次执行 measure()，layout() 和 draw() 操作。Android 引入了硬件加速后，为每个View生成DisplayList，并根据需要在GPU内部生成Hardware Layer，从而充分利用GPU的功能提升图形绘制速度。
4. 当某个View发生变化，它会调用invalidate() 请求重绘，这个函数从当前View 出发，向上遍历找到View Tree中所有Dirty的 View 和 ViewGroup进行重绘。

上面讨论的只是一个窗口的流程，而Android是个多窗口的系统，窗口之间可能会有重叠，窗口切换会有动画产生，窗口的显示和隐藏都有可能会导致资源的分配和释放，这一切需要有一个全局的服务进行统一的管理，这个服务就是我们大名鼎鼎的Window Manager Service (简写 WMS)。

![](../../_attach/wms_class.png)

#### Layout

Layout 是Window Manager Service 重要工作之一，它的流程如下图所示：

![](../../_attach/layout_wms.png)

- 每个View将期望窗口尺寸交给WMS（WindowManagerService).
- WMS 将所有的窗口大小以及当前的Overscan区域传给WPM （WindowPolicy Manager).
- WPM根据用户配置确定每个Window在最终Display输出上的位置以及需要分配的Surface大小。
- 返回这些信息给每个View，他们将在给会的区域空间里绘图。

#### Animation

Animation的原理很简单，就是定时重绘图形。下面的类图中给出了Android跟Animation相关的类。

![](../../_attach/wms_animation_class.png)

- Animation:
>    Animation抽象类，里面最重要的一个接口就是applyTranformation, 它的输入是当前的一个描述进度的浮点数(0.0 ~ 1.0)， 输出是一个Transformation类对象，这个对象里有两个重要的成员变量，mAlpha 和 mMatrix, 前者表示下一个动画点的透明度(用于灰度渐变效果），后者则是一个变形矩阵，通过它可以生成各种各样的变形效果。Android提供了很多Animation的具体实现，比如RotationAnimation, AlphaAnimation 等等，用户也可以实现自己的Animation类，只需要重载applyTransform 这个接口。注意，Animation类只生成绘制动画所需的参数（alpha 或 matrix)，不负责完成绘制工作。完成这个工作的是Animator.

- Animator：

>    控制动画的‘人’， 它通常通过向定时器Choreographer 注册一个Runnable对象来实现定时触发，在回调函数里它要做两件事情：1. 从Animation那里获取新的Transform， 2. 将Transform里的值更新底层参数，为接下来的重绘做准备。动画可以发生在Window上，也可以发生在某个具体的View。前者的动画会通过SurfaceControl直接在某个Surface上进行操作，比如设置Alpha值。后者则通过OpenGL完成（生成我们前面提过的DisplayList).

- WindowStateAnimator, WindowAnimator,  AppWindowAnimator:

>    针对不同对象的Animator. WindowAnimator, 负责整个屏幕的动画，比如说转屏，它提供Runnable实现。WindowStateAnimator, 负责ViewRoot，即某一个窗口的动画。AppWindowAnimator, 负责应用启动和退出时候的动画。这几个Animator都（？）会提供一个函数，stepAnimationLocked(), 它会完成一个动画动作的一系列工作，从计算Transformation到更新Surface的Matrix.

具体来看一下Window的Animation和View的Animation(与Android N有偏差，整体流程类似)：

![](../../_attach/wms_animation.png)

1. WindowManagerService 的 scheduleAnimationLocked() 将windowAnimator的mAnimationRunnable 注册到定时器 Choreographer.
2. 如果应用程序的res/anim/下有xml文件定义animation，在layout过程中，会通过appTransition类的loadAnimation() 函数将XML转换成 Animation_Set 对象，它里面可以包含多个Animation。
3. 当下一个VSYNC事件到来，刚才注册的Callback函数被调用，即WindowAnimator的mAnimationRunnable，里面调用 animateLocked(), 首先，打开一个SurfaceControl的动画会话，animationSession。
4. 首先执行的动画是 appWindowAnimator, 如果刚才loadAnimation() 返回的animation不为空，便会走到Animation的getTransform() 获取动画的参数，这里可能会同时有多个动画存在，通过Transform的compose()函数将它们最终合为一个。
5. 接下来上场的是DisplayContentsAnimator, 它主要用来实现灰度渐变和转屏动画。同样，首先通过stepAnimation() 获取动画变形参数，然后通过SurfaceControl将其更新到SrufaceFlinger内部对应的Layer. 这里首先完成的是转屏的动画
6. 然后就是每个窗口的动画。后面跟着的 perpareSurfaceLocked() 则会更新参数。
7. Wallpaper的动画。
8. 接下来，就是上面提到的DisplayContentsAnimator的第二部分，通过DimLayer实现渐变效果。
9. Surface的控制完成后，关闭对话。然后scheduleAnimationLocked() 规划下一步动画。
10. 接下来的DO_TRAVERSAL会触发relayout，会把所有更新参数的View，或Surface交给OpenGL或HWcomposer进行处理，于是我们就看到了动画效果。

#### 管理窗口

WMS 里面管理着各式各样的窗口, 如下表所示(在WindowManagerService.java 中定义）

|                         | 类型                                       | 用途                      |
| ----------------------- | ---------------------------------------- | ----------------------- |
| mAnimatingAppToken      | ArrayList<AppWindowToken>                | 正在动画中的应用                |
| mExistingAppToken       | ArrayList<AppWindowToken>                | 退出但退出动画还没有完成的应用         |
| mResizingWindows        | ArrayList<WindowState>                   | 尺寸正在改变的窗口，当改变完成后，需要通知应用 |
| mFinishedStarting       | ArrayList<AppWindowToken>                | 已经完成启动的应用               |
| mPendingRemove          | ArrayList<WindowState>                   | 动画结束的窗口                 |
| mLosingFocus            | ArrayList<WindowState>                   | 失去焦点的窗口，等待获得焦点的窗口进行显示   |
| mDestorySurface         | ArrayList<WindowState>                   | 需要释放Surface的窗口          |
| mForceRemoves           | ArrayList<WindowState>                   | 需要强行关闭的窗口，以释放内存         |
| mWaitingForDrawn        | ArrayList<Pair<WindowState, IRemoteCallback>> | 等待绘制的窗口                 |
| mRelayoutWhileAnimating | ArrayList<WindowState>                   | 请求relayout但此时仍然在动画中的窗口  |
| mStrictModeFlash        | StrictModeFlash                          | 一个红色的背景窗口，用于提示可能存在的内存泄露 |
| mCurrentFocus           | WindowState                              | 当前焦点窗口                  |
| mLastFocus              | WindowState                              | 上一焦点窗口                  |
| mInputMethodTarget      | WindowState                              | 输入法窗口下面的窗口              |
| mInputMethodWindow      | WindowState                              | 输入法窗口                   |
| mWallpaperTarget        | WindowState                              | 墙纸窗口                    |
| mLowerWallpaperTarget   | WindowState                              | 墙纸切换动画过程中Z-Order 在下面的窗口 |
| mHigherWallpaperTarget  | WindowState                              | 墙纸切换动画过程中Z-Order 在上面的窗口 |

WindowManager还根据Window的类型进行了分类，根据窗口的类型值来决定Z-Order。不同的窗口类型在不同的硬件产品上有不同的定义，它是实现在WindowManagerPolicy里的windowTypeToLayerLw()，它的ZOrder 顺序越大越在上边。
