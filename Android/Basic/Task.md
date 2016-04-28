在SDK中关于Task（guide/topics/fundamentals.html#acttask），有一个很好的比方，说Task就相当于应用（application）的概念。在开发人员眼中，开发一个Android程序，是做一个个独门独户的组件，但对于一般用户而言，它们感知到的，只是一个运行起来的整体应用，这个整体背后，就是Task。

Task，简单的说，就是一组以栈的模式聚集在一起的Activity组件集合。它们有潜在的前后驱关联，新加入的Activity组件，位于栈顶，并仅有在栈顶的Activity，才会有机会与用户进行交互。而当栈顶的Activity完成使命退出的时候，Task会将其退栈，并让下一个将跑到栈顶的Activity来于用户面对面，直至栈中再无更多Activity，Task结束。

在Android中，每一个Activity的Task模式，都是可以由Activity提供方（通过配置文件…）和Activity使用方（通过Intent中的flag信息…）进行配置和选择。当然，使用方对Activity的控制力，是限定在提供方允许的范畴内进行，提供方明令禁止的模式，使用方是不能够越界使用的。

提供方对组件的配置，是通过配置文件（Manifest）<activity>项来进行的，而调用方，则是通过Intent对象的flag进行抉择的。相对于标准的Task栈的模式，配置的主要方向有两个：一则是破坏已有栈的进出规则，或样式；另一则是开辟新Task栈完成本应在同一Task栈中完成的任务。

对于应用开发人员而言，<activity>中的launchMode属性，是需要经常打交道的。它有四种模式：”standard“, “singleTop“, “singleTask“, “singleInstance“。

- standard模式， 是默认的也是标准的Task模式，在没有其他因素的影响下，使用此模式的Activity，会构造一个Activity的实例，加入到调用者的Task栈中去，对于使用频度一般开销一般什么都一般的Activity而言，standard模式无疑是最合适的，因为它逻辑简单条理清晰，所以是默认的选择。
- singleTop模式，基本上与standard一致，仅在请求的Activity正好位于栈顶时，有所区别。此时，配置成singleTop的Activity，不再会构造新的实例加入到Task栈中，而是将新来的Intent发送到栈顶Activity中，栈顶的Activity可以通过重载onNewIntent来处理新的Intent（当然，也可以无视…）。这个模式，降低了位于栈顶时的一些重复开销，更避免了一些奇异的行为（想象一下，如果在栈顶连续几个都是同样的Activity，再一级级退出的时候，这是怎么样的用户体验…），很适合一些会有更新的列表Activity展示。一个活生生的实例是，在Android默认提供的应用中，浏览器（Browser）的书签Activity（BrowserBookmarkPage），就用的是singleTop。
- singleTop模式，虽然破坏了原有栈的逻辑（复用了栈顶，而没有构造新元素进栈…），但并未开辟专属的Task。而singleTask，和singleInstance，则都采取的另辟Task的蹊径。标志为singleTask的Activity，最多仅有一个实例存在，并且，位于以它为根的Task中。所有对该Activity的请求，都会跳到该Activity的Task中展开进行。singleTask，很象概念中的单件模式，所有的修改都是基于一个实例，这通常用在构造成本很大，但切换成本较小的Activity中。在Android源码提供的应用中，该模式被广泛的采用，最典型的例子，还是浏览器应用的主Activity（名为Browser…），它是展示当前tab，当前页面内容的窗口。它的构造成本大，但页面的切换还是较快的，于singleTask相配，还是挺天作之合的。
- 相比之下，singleInstance显得更为极端一些。在大部分时候singleInstance与singleTask完全一致，唯一的不同在于，singleInstance的Activity，是它所在栈中仅有的一个Activity，如果涉及到的其他Activity，都移交到其他Task中进行。这使得singleInstance的Activity，像一座孤岛，彻底的黑盒，它不关注请求来自何方，也不计较后续由谁执行。在Android默认的各个应用中，很少有这样的Activity，在我个人的工程实践中，曾尝试在有道词典的快速取词Activity中采用过，是因为我觉得快速取词入口足够方便（从notification中点选进入），并且会在各个场合使用，应该做得完全独立。

除了launchMode可以用来调配Task，<activity>的另一属性taskAffinity，也是常常被使用。taskAffinity，是一种物以类聚的思想，它倾向于将taskAffinity属性相同的Activity，扔进同一个Task中。不过，它的约束力，较之launchMode而言，弱了许多。只有当<activity>中的allowTaskReparen ting设置为true，抑或是调用方将Intent的flag添加FLAG_ACTIVITY_NEW_TASK属性时才会生效。如果有机会用到Android的Notification机制就能够知道，每一个由notification进行触发的Activity，都必须是一个设成FLAG_ACTIVITY_NEW_TASK的Intent来调用。这时候，开发者很可能需要妥善配置taskAffinity属性，使得调用起来的Activity，能够找到组织，在同一taskAffinity的Task中进行运行。

####Android下的任务和Activity栈


就像前面提到的，一个activity可以启动另一个，包括那些定义在不同应用程序中的。假设，例如，你想让用户显示一些地方的街道地图。已经有一个activity可以做这个事，所以你的activity所要做的就是将行为对象和需要的信息放在一起，并将它们传递给startActivity()。地图查看器将显示这个地图。当用户按下后退按钮时，你的activity又重新显示在屏幕上了。

对用户来说，这个地图查看器看起来就像是你的应用程序的一部分，即使它定义在另外的应用程序中并运行在那个程序的进程中。Android 通过保持所有的activity在同一个任务中来保持用户体验。简单的的说，任务就是用户所体验到的“应用程序”。它是一组相关的activity，分配到一个栈中。栈中的根activity，是任务的开始——一般来说，它是用户组应用程序加载器中选择的activity。在栈顶的activity正是当前正在运行的——集中处理用户动作的那个。当一个activity启动了另外一个，这个新的activity将压入栈中，它将成为正在运行中的 activity。前一个activity保留在栈中。当用户按下后退按键，当前的这个activity将中栈中弹出，而前面的那个activity恢复成运行中状态。

栈包含了对象，如果一个栈有多于一个相同的Activity的子类的实例打开——比如，多个地图查看器——这个栈分别拥有每个实例的入口。栈中的activity不能重新排列，只能压入和弹出。

任务是一些activity组成的栈，不是清单文件中的类或元素。所以没有办法在独立于它包含的activity的条件下，设置它的值。任务的值作为一个整体设置在根activity中。例如，下一节将讨论“任务的亲和性”；这个值是从根activity亲和性中读取出来的。

一个任务中的所有activity一起作为一个单元。整个任务(整个activity栈)可以移动到前台或者后台.假设，例如，当前的任务有四个 activity在栈中——三个在当前的activity之下。用户按下了HOME键，进入了应用程序加载器，选择了一个新的程序(实际上，是一个新的任务)。当前的任务进入了后台，新任务的根activity显示出来。然后，过了一会，用户退回到主界面，又重新选择了前一个应用程序(前一个任务),栈中有四个activity的那个任务，现在出现在了前台。当用户按下BACK按键，屏幕就不会再显示用户刚刚离开的那个activity，而是删除栈顶的 activity，同任务中的前一个activity将被显示出来。

刚才说明的那些行为，是activity和任务的默认行为。但是有也办法修改它的所有方面。activity和任务的关联，activity在任务中的行为，受控于启动activity的行为对象的标志位和清单文件中的<activity> 元素的属性的互相作用。请求者和相应着都要说明发生了什么。

在这里，主要的行为标志为是：
```Java
FLAG_ACTIVITY_NEW_TASK
FLAG_ACTIVITY_CLEAR_TOP
FLAG_ACTIVITY_RESET_TASK_IF_NEEDED
FLAG_ACTIVITY_SINGLE_TOP
```

主要的<activity> 属性是：
```xml
taskAffinity
launchMode
allowTaskReparenting
clearTaskOnLaunch
alwaysRetainTaskState
finishOnTaskLaunch
```

下面将说明这些标志和属性都有什么用，他们之间怎么互相影响，应该用什么样的方案来控制它们的使用。

####亲和性和新任务

默认情况下，应用程序中的所有activity，都有一个对于其它activity的亲和性—这是一个对于同一个任务中的其他activity的优先权，然后，通过 <activity>元素的 taskAffinity 属性可以可以分别为每一个activity设置亲和性。不同应用程序定义的activity可以共享同一个亲和性，或者同一个应用程序定义的 activity可以指定不同的亲和性。亲和性在两种情况下发挥作用：当行为对象启动了一个包含 FLAG_ACTIVITY_NEW_TASK标志的activity，和当一个activity的allowTaskReparenting 属性设置为“true”。

####FLAG_ACTIVITY_NEW_TASK 标志

正如前面描述的，一个新的activity，默认情况下，被加载进调用startActivity()方法的activity对象所在的那个任务中。它被压入和调用者所在的同一个栈中，但是，如果行为对象在调用startActivity()方法时传递了FLAG_ACTIVITY_NEW_TASK标记，系统将用一个不同的任务来容纳这个新的activity。通常，就像这个标记的名字所代表的。它是一个新任务，但是，它不必非要这样。如果已经存在一个和这个activity亲和性相同的任务，这个activity就会载入到那个任务中，如果不是的话，才会启动新任务。

####allowTaskReparenting 属性

如果activity的allowTaskReparenting 属性设置为“true”，它就能从他启动时所在的任务移动到另一个出现在前台的任务。例如，假设有一个activity可以根据选择的城市包括天气情况，它作为一个旅行应用程序的一部分。它和同一个应用程序中的其他activity有同样的亲和性(默认的亲和性)并且允许重组。你的一个activity开启了天气报告器，所以它属于同一个任务中的这个activity，然而，当旅行应用程序开始运行时，天气报告器将被重新分配并显示到那个任务中。

####启动模式

有4中不同的启动模式可以分配给 <activity> 元素的 launchMode 属性。
- "standard" (默认的模式)
- "singleTop "
- "singleTask"
- "singleInstance"

这些模式主要区别在以下四点：

- 哪个任务存放着activity，用来对行为进行响应。 对“standard ”和“singleTop ”模式来说，这个任务是产生行为(并且调用startActivity() )的那个——除非行为对象包含了 FLAG_ACTIVITY_NEW_TASK 标记。在这种情况下，像前面那节Affinities and new tasks 描述的一样，将会选择一个不同的任务。
- 它们是否可以有多个实例。 "standard "和“singleTop ”类型的activity可以被实例化多次。它们可以属于多个任务，一个特定的任务也可以拥有同一个activity的多个实例。
- 作为比较"singleTask "和"singleInstance "类型的activity只限定有一个实例。因为这些activity是任务的根。这个限制意味着，在设备上不能同时有超过一个任务的实例。
- 是否能有其他的activity在它所在的任务中。"singleInstance " 类型的activity是它所在任务中唯一的activity。如果它启动了其他的activity，不管那个activity的启动模式如何，它都会加载到一个不同的任务中——好像行为对象中的FLAG_ACTIVITY_NEW_TASK 标记。在其他的方面，"singleInstance "和"singleTask "模式是相同的。

其他三种模式运行任务中有多个activity。"singleTask "总是任务中的根activity，但是它可以启动其他的activity并分配到它所在的任务中。"standard "和"singleTop "类型的activity可以出现在任务中的任何地方。

是否启动一个新的实例来处理一个新的行为。 对默认的"standard "模式来说，对于每一个行为都会创建一个新的实例来响应。每个实例只处理一个行为。对于"singleTop "模式，如果一个已经存在的实例位于目标任务activity栈的栈顶，那么他将被重用来处理这个行为。如果它不在栈顶，它将不会被重用，而是为行为创建一个新的实例，并压入栈中。

例如，假设，一个任务的activity栈由根activity A和 B,C,D从上到下按这样的顺序组成，所以这个栈就是A-B-C-D。一个行为指向类型为D的activity。如果D是默认的"standard "加载模式，一个新的实例会被启动，栈现在就是这样A-B-C-D-D。但是，如果D的加载模式是"singleTop "，已经存在的实例会用来处理这个行为(因为它在栈的顶端)并且栈中还应该是A-B-C-D。

在前面提到，"singleTask "和"singleInstance "类型的activity最多只有一个实例，所以他们的实例应该会处理每个新的行为。"singleInstance "类型的activity总是在栈的顶端(因为他是任务中唯一的一个activity)，所以总是能够适当的处理行为。然而，"singleTask "类型的activity也许会有其他的activity在它的上面。如果是这样的话，那就不能处理这个行为，这个行为被丢弃。(即使这个行为被丢弃了，它的到来也会导致那些应该保留不变任务显示到前台来)。

当一个activity被要求处理一个新的行为时，行为对象会通过调用activity的 onNewIntent() 方法传递进来(最初启动activity的行为可以通过调用getIntent() 方法获得)。

注意，当创建一个新的activity实例来处理一个新的行为时，用户总是能够通过按下BACK按键退回到前面的状态(前一个activity)。但是当一个已经存在的activity实例处理一个新的行为时，用户不能通过按下BACK按键退回到前面的状态。

####清理栈

如果用户离开一个任务很长时间。系统将清除除了根activity之外的所有activity。当用户重新回到任务中时，像是用户离开了它，除了只有最初的activity还在。这个理念是这样的，过了一段时间，用户很可能放弃之前所做的事情，回到任务去做一些新的事情。

这只是默认情况，有一些activity的属性可以控制和修改它。

- alwaysRetainTaskState 属性
>如果一个任务的根activity的这个属性设置成了"true",那么刚才提到的那些默认行为就不会发生。这个任务保留所有的activity，甚至经过了很长一段时间。

- clearTaskOnLaunch 属性
>如果任务的根activity的这个属性设置成了”true“，那么只要用户离开了任务并返回，就会清除除了根activity之外的所有activity。换句话说，它和alwaysRetainTaskState正好相反，当用户返回到任务时，总是恢复到最初的状态，不管离开了多长时间。

- finishOnTaskLaunch 属性
>这个属性和clearTaskOnLaunch类似，但是它作用于单个activity，而不是整个任务。它可以导致任何的activity离开，包括根activity。当它设置成"true"的时候，作为任务一部分的activity只对当前会话有效。如果用户离开然后返回到任务中。它将不再出现。

还有其他的方法强制将activity从栈中移除。如果一个行为对象包含了 FLAG_ACTIVITY_CLEAR_TOP 标志，它的目标任务中已经有了一个这样类型的activity实例，所有栈中位于这个实例之上的activity都会被清除，所以这个实例就会出现在栈顶并且对行为进行响应。如果activity被设计成"standard"模式，它也将会从栈中被清除，并且会启动新的实例来处理到来的行为。这是因为当设置成”standard“模式后，对每个新的行为都会创建一个新的实例。

FLAG_ACTIVITY_CLEAR_TOP经常和FLAG_ACTIVITY_NEW_TASK一起使用。当一起使用时，这些标志是定位一个在另一个任务中存在的activity并且将它放在一个可以响应行为的地方的一种方法。

####启动任务

Activity通过将行为过滤器”android .intent.action.MAIN“设置为指定动作和"android .intent.category.LAUNCHER"作为指定类型，来成为任务的入口。(前面关于行为过滤器那一些讨论的例子)。这种类型的过滤器会让activity的图标和标签显示在应用程序加载器上面，可以让用户启动和返回activity。

第二个能力更为重要，用户应该可以在离开一个任务一段时间后返回。因为这样，能够初始化任务的"singleTask"和"singleInstance"模式，只能够用在那些拥有MAIN 和LAUNCHER 过滤器的activity中。想像一下如果没有这两个过滤器会发生什么：一个行为启动了"singleTask"模式的activity，启动了一个新的任务并且用户花了一些时间在这个任务上。然后用户按下了HOME键，这个任务被隐藏到了后台。因为没有在应用程序加载器上显示它，所以就没有办法返回到这个任务。

一个类似的麻烦事 FLAG_ACTIVITY_NEW_TASK 标志。如果这个标志导致activity启动了一个新任务，并且用户按下HOME键离开了它，必须有一些方法将用户引导回它。一些实体(像是通知管理器) 总是在一个外部的任务中启动activity，而不作为它们的一部分，所以他们总是将带有FLAG_ACTIVITY_NEW_TASK 标记的行为对象传递到startActivity() 。如果你有一个可以被外部实体使用这个标签调用的activity，要注意用户应该有办法返回到启动的任务。


对于那些你不想让用户返回到activity的情况，将 <activity>的finishOnTaskLaunch属性设置为”true“，参看前面的清理栈。

####Activity的taskAffinity属性

Activity的归属，也就是Activity应该在哪个Task中，Activity与Task的吸附关系。我们知道，一般情况下在同一个应用中，启动的Activity都在同一个Task中，它们在该Task中度过自己的生命周期，这些Activity是从一而终的好榜样。

那么为什么我们创建的Activity会进入这个Task中？它们会转到其它的Task中吗？如果转到其它的Task中，它们会到什么样的Task中去？

解决这些问题的关键，在于每个Activity的taskAffinity属性。

每个Activity都有taskAffinity属性，这个属性指出了它希望进入的Task。如果一个Activity没有显式的指明该Activity的taskAffinity，那么它的这个属性就等于Application指明的taskAffinity，如果Application也没有指明，那么该taskAffinity的值就等于包名。而Task也有自己的affinity属性，它的值等于它的根Activity的taskAffinity的值。

一开始，创建的Activity都会在创建它的Task中，并且大部分都在这里度过了它的整个生命。然而有一些情况，创建的Activity会被分配其它的Task中去，有的甚至，本来在一个Task中，之后出现了转移。我们首先分析一下android文档给我们介绍的两种情况。


- 第一种情况。如果该Activity的allowTaskReparenting设置为true，它进入后台，当一个和它有相同affinity的Task进入前台时，它会重新宿主，进入到该前台的task中。
我们验证一下这种情况。
```
Application Activity taskAffinity allowTaskReparenting
application1 Activity1 com.winuxxan.affinity true
application2 Activity2 com.winuxxan.affinity false
```
我们创建两个工程，application1和application2，分别含有Activity1和Activity2，它们的taskAffinity相同，Activity1的allowTaskReparenting为true。
>- 首先，我们启动application1,加载Activity1，然后按Home键，使该task（假设为task1）进入后台。然后启动application2，默认加载Activity2。
>- 我们看到了什么现象？没错，本来应该是显示Activity2，但是我们却看到了Activity1。实际上Activity2也被加载了，只是Activity1重新宿主，所以看到了Activity1。

- 第二种情况。如果加载某个Activity的intent，Flag被设置成FLAG_ACTIVITY_NEW_TASK时，它会首先检查是否存在与自己taskAffinity相同的Task，如果存在，那么它会直接宿主到该Task中，如果不存在则重新创建Task。

>来做一个测试。

首先写一个应用，它有两个Activity（Activity1和Activity2），AndroidManifest.xml如下：
```xml
<application android:icon="@drawable/icon" android:label="@string/app_name">
    <activity android:name=".Activity1"
              android:taskAffinity="com.winuxxan.task"
              android:label="@string/app_name">
    </activity>
    <activity android:name=".Activity2">
        <intent-filter>
            <action android:name="android.intent.action.MAIN" />
            <category android:name="android.intent.category.LAUNCHER" />
        </intent-filter>
    </activity>
</application>
```
Activity2的代码如下：
```Java
public class Activity2 extends Activity {
    private static final String TAG = "Activity2";
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.main2);   
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        Intent intent = new Intent(this, Activity1.class);
        intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        startActivity(intent);
        return super.onTouchEvent(event);
    }
}
```
然后，我们再写一个应用MyActivity，它包含一个Activity（MyActivity），AndroidManifest.xml如下：
```xml
<application android:icon="@drawable/icon" android:label="@string/app_name">
    <activity android:name=".MyActivity"
              android:taskAffinity="com.winuxxan.task"
              android:label="@string/app_name">
        <intent-filter>
            <action android:name="android.intent.action.MAIN"/>
            <category android:name="android.intent.category.LAUNCHER"/>
        </intent-filter>
    </activity>
</application>
```

我们首先启动MyActivity，然后按Home键，返回到桌面，然后打开Activity2，点击Activity2，进入Activity1。然后按返回键。

我们发现，我们进入Activity的顺序为Activity2->Activity1，而返回时顺序为Activity1->MyActivity。这就说明了一个问题，Activity1在启动时，重新宿主到了MyActivity所在的Task中去了。


以上是验证了文档中提出的两种TaskAffinity的用法。
我们现在将上一文中的launchMode和本文讲的taskAffinity结合起来。

#####首先是singleTask加载模式与taskAffinity的结合。

还是用上一文中的singleTask的代码，这里就不在列出来了，请读者自己查阅上一文。唯一不同的就是，为MyActivity和Activity1设置成相同的taskAffinity，重新执行上文的测试。

发现测试结果令我们惊讶：从同一应用程序启动singleTask和不同应用程序启动的结果完全与上文讲的相反！

经过思考，就可以把从同一应用程序执行和从不同应用程序执行另种方式同一起来，得到一个结论：
>当一个应用程序加载一个singleTask模式的Activity时，首先该Activity会检查是否存在与它的taskAffinity相同的Task。
1. 如果存在，那么检查是否实例化，如果已经实例化，那么销毁在该Activity以上的Activity并调用onNewIntent。如果没有实例化，那么该Activity实例化并入栈。
2. 如果不存在，那么就重新创建Task，并入栈。

用一个流程来表示：

然后我们来检测singleInstance模式融入taskAffinity时的情况，我们也是用上文中测试singleInstance的例子，在此不列出，读者翻阅前文查阅。唯一不同的是，我们将MyActivity和Activity2设置成相同的taskAffinity。

我们发现测试结果也有一定的出入，就是，当从singleInstance中启动Activity时，并没用重新创建一个Task，而是进入了和它具有相同affinity的MyActivity所在的Task。

>于是，我们也能得到以下结论：
1. 当一个应用程序加载一个singleInstance模式的Activity时，如果该Activity没有被实例化，那么就重新创建一个Task，并入栈，如果已经被实例化，那么就调用该Activity的onNewIntent；
2. singleInstance的Activity所在的Task不允许存在其他Activity，任何从该Activity加载的其它Actiivty（假设为Activity2）都会被放入其它的Task中，如果存在与Activity2相同affinity的Task，则在该Task内创建Activity2。如果不存在，则重新生成新的Task并入栈

整个任务、进程管理的核心实现，尽在ActivityManagerService中。Intent解析，就是这个ActivityManagerService来负责的，其实，它是一个很名不副实的类，因为虽然名为Activity的Manager Service，但它管辖的范围，不只是Activity，还有其他三类组件，和它们所在的进程。

在ActivityManagerService中，有两类数据结构最为醒目，一个是ArrayList，另一个是HashMap。ActivityManagerService有大量的ArrayList，每一个组件，会有多个ArrayList来分状态存放。调度工作，往往就是从一个ArrayList里面拿出来，找个方法调一调，然后扔到另一个ArrayList里面去，当这个组件没对应的ArrayList放着的时候，说明它离死不远了。HashMap，是因为有组件是需要用名字或Intent信息做定位的，比如Content Provider，它的查找，都是依据Uri，有了HashMap，一切都顺理成章了。

ActivityManagerService用一些名曰xxxRecord的数据结构，来表达各个存活的组件。于是就有了，`HistoryRecord`（保存Activity信息的，之所以叫History，是相对Task栈而言的…），`ServiceRecord`，`BroadcastRecord`，`ContentProviderRecord`，`TaskRecord`，`ProcessRecord`，等等。

值得注意的，是`TaskRecord`，我们一直再说，Task栈这样的概念，其实，真实的底层，并不会在TaskRecord中，维系一个Activity的栈。在ActivityManagerService中，各个任务的Activity，都以HistoryRecord的形式，集中存放在一个ArrayList中，每个HistoryRecord，会存放它所在TaskRecord的引用。当有一个Activity，执行完成，从概念上的Task栈中退出，Android是通过从当前HistoryRecord位置往前扫描同一个TaskRecord的HistoryRecord来完成的。这个设计，使得上层很多看上去逻辑很复杂的Task体系，在实现变得很统一而简明，值得称道。

`ProcessRecord`，是整个进程托管实现的核心，它存放有运行在这个进程上，所有组件的信息，根据这些信息，系统有一整套的算法来决议如何处置这个进程，如果对回收算法感兴趣，可以从ActivityManagerService的trimApplications函数入手来看。
