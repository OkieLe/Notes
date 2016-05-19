作者：Rivers Huang 著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

来源：[知乎](http://www.zhihu.com/question/22749446/answer/34993287)


[Android开发之如何保证Service不被杀掉（broadcast+system/app）](http://blog.csdn.net/mad1989/article/details/22492519)

当你看完上面这个以后，会发现这等于什么都没说。是的，按目前（正常）的方案，是不能做到360、手机管家、猎豹、LEB这样的自启方案的，至少没那么niubility。

因为你想啊，从原理上看，你的进程都被干掉了，你还有什么Service能用来启动呢，不管你是什么Service，什么子进程，只要你挂靠了相同的packageName，你就必定被干掉。

那么，我不用相同的packageName不就结了，bingo！其实上图我在`adb shell top`上看到的`com.xxx.xxx:xxx`都不是关键的守护进程，真正能起到作用的fork进程，是要用人C++来写的，这样你在系统里面停用也是杀不掉的，我特么的真是佩服中国特色的IT从业者，不得不说牛逼！当然我没有什么内幕资料，我是从一些APP在被卸载后还能弹出对话框，弹出网页让你做调查想到的。

大段大段的代码我就不贴了，最后附上两篇连接

[第一篇 Android : can native code get broadcast intent from android system?](http://stackoverflow.com/questions/21279270/android-can-native-code-get-broadcast-intent-from-android-system/21337119#answer-21337119)
Any Android application can start a process by calling `Runtime.exec()` function.

```Java
Runtime.getRuntime().exec("chmod 755 '/data/data/my.app/files'/native_code");
```
After this line of code gets executed there is another process spawned. This process runs under the same linux user as the application itself.

When a user opens Settings -> Apps -> My App and presses "Force stop" button, main application process gets killed, but the process hosting native program (see above) still runs. I personally believe this is a security issue and I am going to report it back to AOSP.

Such native program can run infinitely and do nothing - just sleeping. But before going to sleep, it registers a termination signal handler which will be called when process is about to be terminated by the system.

```C++
int main(void) {
    signal(SIGTERM, termination_handler);
    while(1) {
        sleep(10);
    }
}

void termination_handler(int sig) {
   // handle termination signal here
}
```

Now you should already know what the last piece is, right? My native termination_handler should be able to launch a browser. I didn't try this in code, but I assume this is possible, because I can do it using adb shell as following
```shell
adb shell am start -a android.intent.action.VIEW -d http://www.google.com
```

Now back to the question about how Dolphin Browser does it. Install the app and launch it at least once. Once started, it registers a native uninstall watcher using the principles described above. To see it, connect to the device and open adb shell. Then call ps to see list of processes. You will see two processes similar to following
```shell
u0_a109   315   ... mobi.mgeek.TunnyBrowser
u0_a109   371   ... /data/data/mobi.mgeek.TunnyBrowser/files/watch_server
```

As you can see it starts a watch_server native program, which is a part of its apk-file. Now open App info page of Dolphin Browser and press "Force Stop". Switch back to terminal and call ps again. You will see there is no mobi.mgeek.TunnyBrowser process anymore, but watch_server still runs.

By the way this approach will only work, if watcher server runs all the time. To make sure it is always up, both apps require "run at startup" permission, where they start their watchers.

Now, when you uninstall the app, Android stops all processes belonging to this application. Watcher receives termination signal and opens browser with predefined URL and then shuts down.

I might look a bit different in some details, but the main concept behind this hack must be as described.

[第二篇 android应用创建子进程的方法探究](http://blog.csdn.net/a332324956/article/details/9114919)