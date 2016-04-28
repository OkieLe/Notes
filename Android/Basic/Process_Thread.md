####Process

在Android的底层，进程构造了底部的一个运行池，不仅仅是Task中的各个Activity组件，其他三大组件Service、Content Provider、Broadcast Receiver，都是寄宿在底层某个进程中进行运转。在这里，进程更像一个资源池（概念形如线程池，上层要用的时候取一个出来就好，而不关注具体取了哪一个），只是为了承载各个组件的运行，而各个组件直接的逻辑关系，它们并不关心。但我们可以想象，为了保证整体性，在默认情况下，Android肯定倾向于将同一Task、同一应用的各个组件扔进同一个进程内，但是当然，出于效率考虑，Android也是允许开发者进行配置。

在Android中，整体的<application>（将影响其中各个组件…）和底下各个组件，都可以设置<process>属性，相同<process>属性的组件将扔到同一个进程中运行。最常见的使用场景，是通过配置<application>的process属性，将不同的相关应用，塞进一个进程，使得它们可以同生共死。还有就是将经常和某个Service组件进行通信的组件，放入同一个进程，因为与Service通信是个密集操作，走的是RPC，开销不小，通过配置，可以变成进程内的直接引用，消耗颇小。

除了通过<process>属性，不同的组件还有一些特殊的配置项，以Content Provider为例（通过<provider>项进行配置）。<provider>项有一个mutiprocess的属性，默认值为false，这意味着Content Provider，仅会在提供该组件的应用所在进程构造一个实例，第三方想使用就需要经由RPC传输数据。这种模式，对于构造开销大，数据传输开销小的场合是非常适用的，并且可能提高缓存的效果。但是，如果是数据传输很大，抑或是希望在此提高传输的效率，就需要将mutiprocess设置成true，这样，Content Provider就会在每一个调用它的进程中构造一个实例，避免进程通信的开销。

Android中各个进程的生死，和运行在其中的各个组件有着密切的联系，进程们依照其上组件的特点，被排入一个优先级体系，在需要回收时，从低优先级到高优先级回收。Android进程共分为五类优先级，分别是：Foreground Process, Visible Process, Service Process, Background Process, Empty Process。顾名思义不难看出，这说明，越和用户操作紧密相连的，越是正与用户交互的，优先级越高，越难被回收。

####Thread


在Android核心的调度层面，是不屑于考量线程的，它关注的只有进程，每一个组件的构造和处理，都是在进程的主线程上做的，这样可以保证逻辑的足够简单。多线程，往往都是开发人员需要做的。

Android的线程，也是通过派生Java的Thread对象，实现Run方法来实现的。但当用户需要跑一个具有消息循环的线程的时候，Android有更好的支持，来自于Handler和Looper。Handler做的是消息的传送和分发，派生其handleMessage函数，可以处理各种收到的消息，和win开发无异。Looper的任务，则是构造循环，等候退出或其他消息的来临。在Looper的SDK页面，有一个消息循环线程实现的标准范例，当然，更为标准的方式也许是构造一个HandlerThread线程，将它的Looper传递给Handler。

在Android中，Content Provider的使用，往往和线程挂钩，谁让它和数据相关呢。在前面提到过，Content Provider为了保持更多的灵活性，本身只提供了同步调用的接口，而由于异步对Content Provider进行增删改查是一个常做操作，Android通过AsyncQueryHandler对象，提供了异步接口。这是一个Handler的子类，开发人员可以调用startXXX方法发起操作，通过派生onXXXComplete方法，等待执行完毕后的回调，从而完成整个异步调用的流程，十分的简约明了。
