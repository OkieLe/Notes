### 一、ActionBar

#### 1. 简介

ActionBar是Android3.0开始加入的一种界面特性。用来显示应用标识和用户位置，并提供用户操作以及导航功能。

ActionBar可以显示应用图标以及当前Activity的标题；内置了Tab导航功能用于在Fragment间切换，以及下拉列表导航功能。同时Options菜单也被集成在ActionBar中，可以把菜单项直接显示为ActionBar上的图标/&文字，提供更快捷的操作。

#### 2. 使用ActionBar

所有targetSdkVersion或minSdkVersion为11及以上，使用Theme.Holo主题的Activity都默认使用ActionBar。

如果想使用传统的Activity，可以设置Activity主题为Theme.Holo.NoActionBar，或者自定义主题android:windowActionBar设置为false。也可以在代码中调用getActionBar().hide()隐藏，再调用getActionBar().show()恢复。但通过主题移除ActionBar就不能再用代码添加。

隐藏或移除ActionBar会导致Activity重新布局来使用ActionBar占用的空间。所以如果需要经常隐藏和显示ActionBar，可以使用overlay(覆盖)模式，即让ActionBar遮挡部分Activity，这样隐藏和显示就不会导致重新布局。使用overlay模式需要在Activity主题中设置android:windowActionBarOverlay为true。应注意ActionBar不要覆盖有效的内容，ActionBar高度可以用”?android:attr/actionBarSize”(xml)或者getHeight()获取。

#### 3. Options菜单

ActionBar可以给用户比选项菜单更快捷的操作方式。Options菜单的创建和显示没有变化，跟之前版本一样。要显示为action item的菜单项只要再另外添加几个属性就可以了。

action item可以显示为图标和(或)文本，其他Options菜单仍可以通过Menu键调出。没有Menu键的手机会另外显示一个overflow菜单，点击后调出菜单列表。

资源文件中给菜单项添加android:showAsAction属性，可以是以下值或部分值的组合。
- ifRoom：有空间是才显示到ActionBar；
- withText：显示title，如果空间有限制可能设置了也不会显示；
- never：从不显示；
- always：总是显示；
- collapseActionView：API 14增加，点击对应的item，会展开一个自定义布局或者控件界面，布局和控件类分别通过android:actionLayout和android:actionViewClass属性来提供，当然只能二选一。

*注*：系统会先调用Activity的onOptionsItemSelected()，然后调用Fragment的对应函数，所以在Activity有机会优先处理菜单消息。

4.0以后添加的新特性—拆分ActionBar。打开这个特性时，应用运行在窄屏手机会在屏幕底部有一个独立的bar来显示action items，顶部就可以有足够空间显示标题导航等信息。打开这个功能要在AndroidManifest.xml的application或activity标签中加上`uiOptions="splitActionBarWhenNarrow"`

#### 4. 导航功能

##### a) 应用图标导航

应用图标一般显示在ActionBar最左侧，可以使图标行为像action item，点击后返回应用的home activity或者上一层。

在onOptionsItemSelected()中处理android.R.id.home的事件即可达到目的。4.0开始默认图标是不可以点击的，调用setHomeButtonEnabled(true)来打开。

向上跟返回是不同的，向上是在同一个应用中跳转，返回是退回到前一个activity，不管是哪个应用。所以向上在某些情景是更适合的，调用setDisplayHomeAsUpEnabled(true)来打开。同样是对自定义菜单中home事件进行处理。

##### b) Tab页导航

首先布局中要包含一个ViewGroup，其中每个fragment关联一个tab；要给ViewGroup一个id，因为在切换tab的时候需要引用。如果tab内容直接放在activity中，可以直接把fragment放在默认的ViewGroup，引用时id为android.R.id.content。
1. 实现ActionBar.TabListener，用来处理用户事件对应的fragment切换；
2. 添加tab时，创建一个ActionBar.Tab对象，调用setTabListener()设置1中实现的监听器，还有setText()/setIcon()等方法；
3. 通过addTab()添加tab页。

TabListener的回调方法中只有Tab对象和一个FragmentTransaction对象ft，没有你应该切换的fragment的信息，所以你应该自己定义tab和fragment的对应关系，然后使用ft进行fragment事务操作，回调中不能调用ft.commit()，因为系统会自动调用。

Activity中调用ActionBar的setNavigationMode(NAVIGATION_MODE_TABS)方法显示tabs(一般在onCreate()中调用)，还可以调用setDisplayShowTitleEnabled(false)隐藏掉默认的标题栏，一般tab上都有标题/图标。getSelectedNavigationIndex()可以获取到当前tab的索引。

##### c) 下拉列表导航

ActionBar内建了下拉列表导航，也可以叫作过滤。
1. 创建一个SpinnerAdapter提供可选的条目以及列表的布局；
2. 实现ActionBar.OnNavigationListener，来处理用户从列表中选中某条的事件；
3. 调用ActionBar的setNavigationMode(ActionBar.NAVIGATION_MODE_LIST)；
4. setListNavigationCallbacks(spinnerAdapter, navigationListener)。

#### 5. Action Provider

ActionProvider可以用自定义布局替换action item，而且可以控制所有items的行为。ShareActionProvider是ActionProvider的扩展，在ActionBar上显示一个可以分享的目标列表。可以声明一个ShareActionProvider替换传统调起ACTION_SEND的Intent。点击这个action item会显示一个可以处理ACTION_SEND的应用的下拉列表，即时该菜单显示在overflow菜单中。使用这样的Action Provider，就不必处理对应菜单的用户事件。

只需在菜单资源文件中设置android:actionProviderClass值为对应的Provider完整类名，该功能即可开启。虽然Provider会处理菜单事件，但也可以在onOptionsItemSelected()进行拦截，如果没有处理，Provider的onPerformDefaultAction()会被调用。

ShareActionProvider菜单中的应用根据点击次数排序，点击历史存储在一个文件DEFAULT_SHARE_HISTORY_FILE_NAME中，也可以调用setShareHistoryFileName()指定一个xml文件来存储。除了在资源文件声明MenuItem为Provider外，还要为Provider指定Intent，调用getActionProvider()获取ShareActionProvider，再调用setShareIntent()。

可以扩展ActionProvider实现自己的Provider，需要实现以下方法：
1. 构造函数，需要用成员保存传进来的Context；
2. onCreateActionView()，从xml文件inflate自定义布局，设置事件监听等；
3. onPerformDefaultAction()，当用户选中菜单时被调用，但是当你提供子菜单时，要通过onPrepareSubMenu()处理，然后前者就不再会被调用。但如果activity/fragment中处理了onOptionsItemSelected()，该方法不会被调到。

#### 6. 自定义ActionBar风格

```
android:windowActionBarOverlay：前面说过，用来设置overlay模式；
1. action items属性
android:actionButtonStyle：action item按钮风格；
android:actionBarItemBackground：设置action item背景，API 14新增；
android:itemBackground：设置overflow菜单项背景；
android:actionBarDivider：设置分隔符，API 14新增；
android:actionMenuTextColor：action item中文本颜色；
android:actionMenuTextAppearance：action item文本风格；
android:actionBarWidgetTheme：action views主题，API 14新增；
2. 标签导航
android:actionBarTabStyle：tab风格；
android:actionBarTabBarStyle：导航tab下面窄条风格；
android:actionBarTabTextStyle：tab上文本风格；
3. 下拉列表
android:actionDropDownStyle：下拉列表风格，比如背景、文本等。
```
更复杂的风格可以修改Activity主题中的这两个属性：android:actionBarStyle和android:actionBarSplitStyle。

### 二、ViewPager

android.support.v4.view.ViewPager是谷歌官方给我们提供的一个兼容低版本安卓设备的软件包，里面包括了只有在安卓3.0以上版本可以使用的API。ViewPager可以做很多事情，从最简单的导航，到页面菜单等等。它与ListView类似，也需要一个适配器，就是同一个包内的PagerAdapter。

ViewPager的功能就是可以使视图滑动，就像Lanucher界面那样左右滑动。使用需要3个步骤：
1. 在布局文件中添加ViewPager；
```xml
<LinearLayout
    . . .
    <android.support.v4.view.ViewPager
        android:id="@+id/viewpager"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="center" >
    . . .
```
2. 准备要显示的Page，可以是Fragment或者ListView等UI元素，推荐ViewPager和Fragment一起使用，这可能是Google的原意；
3. 实例化PagerAdapter，并与ViewPager关联，一般还会实现一个OnPageChangeListener页面切换监听器，以便页面切换时做相应的处理；
```Java
mViewPager = (ViewPager) findViewById(R.id.viewpager);
mViewPager.setAdapter(new myFragmentPagerAdapter(getFragmentManager()));
mViewPager.setOnPageChangeListener(mPageChangeListener);
```
可以扩展PagerAdapter抽象类，需要实现以下方法：
```Java
instantiateItem(ViewGroup, int)
destroyItem(ViewGroup, int, Object)
getCount()
isViewFromObject(View, Object)
```
也可以使用Android提供的子类FragmentPagerAdapter或FragmentStatePagerAdapter，这时需要实现的方法少一些。
