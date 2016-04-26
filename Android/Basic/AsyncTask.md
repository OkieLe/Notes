在有界面的Android应用中，后台异步执行一些事情是常见的场景，这时候从底层开始写起的话，就需要了解比较深层的东西，比如“Android 的消息队列模型”使用的Looper、Handler、Message、MessageQueue。

Android为了降低这个开发难度，提供了AsyncTask。AsyncTask就是一个封装过的后台任务类，顾名思义就是异步任务。

AsyncTask直接继承于Object类，位置为android.os.AsyncTask。要使用AsyncTask工作我们要提供三个泛型参数，并重载几个方法(至少重载一个)。AsyncTask定义了三种泛型类型 Params，Progress和Result。
- Params 启动任务执行的输入参数，比如HTTP请求的URL。
- Progress 后台任务执行的百分比。
- Result 后台执行任务最终返回的结果，比如String。

AsyncTask是抽象类，子类必须实现抽象方法doInBackground(Params... p) ，在此方法中实现任务的执行工作，比如连接网络获取数据等。通常还应该实现onPostExecute(Result r)方法，因为应用程序关心的结果在此方法中返回。最少要重写以下这两个方法：
- doInBackground(Params…) 后台执行，比较耗时的操作都可以放在这里。注意这里不能直接操作UI。此方法在后台线程执行，完成任务的主要工作，通常需要较长的时间。在执行过程中可以调用publicProgress(Progress…)来更新任务的进度。
- onPostExecute(Result) 相当于Handler 处理UI的方式，在这里面可以使用在doInBackground 得到的结果处理操作UI。 此方法在主线程执行，任务执行的结果作为此方法的参数返回

有必要的话你还得重写以下这三个方法，但不是必须的：
- onProgressUpdate(Progress…) 可以使用进度条增加用户体验度。此方法在主线程执行，用于显示任务执行的进度。
- onPreExecute() 这里是最终用户调用excute时的接口，当任务执行之前开始调用此方法，可以在这里显示进度对话框。
- onCancelled() 用户调用取消时，要做的操作。

使用AsyncTask类，以下是几条必须遵守的准则：
- Task的实例必须在UI thread中创建；
- execute方法必须在UI thread中调用；
- 不要手动的调用onPreExecute(), onPostExecute(Result)，doInBackground(Params...), onProgressUpdate(Progress...)这几个方法；
- task只能被执行一次，多次调用时将会出现异常；

AsyncTask的特点是任务在主线程之外运行，而回调方法是在主线程中执行， 这就有效地避免了使用Handler带来的麻烦。阅读AsyncTask的源码可知，AsyncTask是使用java.util.concurrent 框架来管理线程以及任务的执行的，concurrent框架是一个非常成熟，高效的框架，经过了严格的测试。AsyncTask的设计很好的解决了匿名线程存在的问题。

AsyncTask 的执行分为四个步骤，与前面定义的TaskListener类似。每一步都对应一个回调方法，需要注意的是这些方法不应该由应用程序调用，开发者需要做的就是实现这些方法。在任务的执行过程中，这些方法被自动调用。

- onPreExecute() 当任务执行之前开始调用此方法，可以在这里显示进度对话框。
- doInBackground(Params...) 此方法在后台线程执行，完成任务的主要工作，通常需要较长的时间。在执行过程中可以调用publicProgress(Progress...)来更新任务的进度。
- onProgressUpdate(Progress...) 此方法在主线程执行，用于显示任务执行的进度。
- onPostExecute(Result) 此方法在主线程执行，任务执行的结果作为此方法的参数返回。
