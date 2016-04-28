Gradle是一种依赖管理工具，基于Groovy语言，面向Java应用为主，它抛弃了基于XML的各种繁琐配置，取而代之的是一种基于Groovy的内部领域特定（DSL）语言。Gradle是一个基于JVM的构建工具，它提供了：
- 像Ant一样，通用灵活的构建工具
- 可以切换的，基于约定的构建框架
- 强大的多工程构建支持
- 基于Apache Ivy的强大的依赖管理
- 支持maven, Ivy仓库
- 支持传递性依赖管理，而不需要远程仓库或者是pom.xml和ivy.xml配置文件。
- 对Ant的任务做了很好的集成
- 基于Groovy，build脚本使用Groovy编写
- 有广泛的领域模型支持构建

学习Gradle前，需要有一个Groovy语言的基础，以免被Groovy的语法困扰，反而忽略了Gradle的知识。

### 1 Projects和tasks

先明确两个概念，projects和tasks，它们是Gradle中的两个重要概念。
任何一个Gradle构建，都是由一个或多个projects组成的。Project就是你想要用Gradle做什么，比如构建一个jar包，构建一个web应用。Project也不单指构建操作，部署你的应用或搭建一个环境，也可以是一个project。
一个project由多个task组成。每个task代表了构建过程当中的一个原子性操作，比如编译，打包，生成javadoc，发布等等这些操作。

### 2 编写第一个构建脚本

新建一个文件build.gradle，然后添加以下代码：
```gradle
task hello {
    doLast {
        println 'Hello, Gradle!'
    }
}
```
这是本系列文章里的第一个构建脚本，它定义了一个叫hello的task，task的内容是在最后打印出“Hello, Gradle!”。
我们输入命令gradle hello来执行它：
```shell
msdx@msdx-ubuntu:~/tmp$ gradle hello
:hello
Hello, Gradle!

BUILD SUCCESSFUL
```
Gradle是领域驱动设计的构建工具，在它的实现当中，Project接口对应上面的project概念，Task接口对应上面的task概 念，实际上除此之外还有一个重要的领域对象，即Action，对应的是task里面具体的某一个操作。一个project由多个task组成，一个 task也是由多个action组成。
当执行gradle hello的时候，Gradle就会去调用这个hello task来执行给定操作(Action)。这个操作其实就是一个用Groovy代码写的闭包，代码中的task是Project类里的一个方法，通过调用 这里的task方法创建了一个Task对象，并在对象的doLast方法中传入println 'Hello, Gradle!'这个闭包。这个闭包就是一个Action。
Task是Gradle里定义的一个接口，表示上述概念中的task。它定义了一系列的诸如doLast, doFirst等抽象方法，具体可以看gradle api里org.gradle.api.Task的文档。

在上面执行了gradle hello后，除了输出“Hello, Gradle!”之外，我们发现像“:hello”这样的其他内容。这其实是Gradle打印出来的日志，如果不想输出这些内容，可以在gradle后面 加上参数 -q。即：gradle -q hello。

### 3 快速定义任务

上面的代码，还有一种更简洁的写法，如下：
```gradle
task hello << {
    println 'Hello, Gradle!'
}
```

执行这个脚本，打印出来的是一样的。也就是我们把像doLast这样的代码，直接简化为<<这个符号了。这其实是Gradle利用了 Groovy的操作符重载的特性，把左位移操作符实现为将action加到task的最后，相当于调用doLast方法。看Gradle的api文档里对 doLast()和leftShift()这两个方法的介绍，可知它们的作用是一样的，所以在这里，<<左移操作符即doLast的简写方 式。

### 4 代码即脚本

Gradle脚本是采用Groovy编写的，所以也像Groovy一样，以脚本方式来执行代码，如下面例子：
```gradle
task upper << {
    String someString = 'mY_nAmE'
    println "Original: " + someString
    println "Upper case: " + someString.toUpperCase()
}
```

执行结果如下，它将定义的字符串转为大写：
```shell
msdx@msdx-ubuntu:~/tmp$ gradle -q upper
Original: mY_nAmE
Upper case: MY_NAME
```

这也就是说，我们在写Gradle脚本的时候，可以像写Groovy代码一样。而Groovy是基于Java的，兼容Java语法，所以Java的朋友们，是不是忽然发现Gradle脚本很好上手了呢？

### 5 任务依赖

我们可以通过以下方式创建依赖：
```gradle
task hello << {
    print 'Hello, '
}
task intro(dependsOn: hello) << {
    println "Gradle!"
}
```

定义一个任务hello，输出“Hello, ”，然后定义一个任务intro，并依赖hello，输出“Gradle!”。结果是打印出“Hello, Gradle!”，如下：
```shell
msdx@msdx-ubuntu:~/tmp$ gradle -q intro
Hello, Gradle!
```

另外，被依赖的task不必放在前面声明，在后面也是可以的，这一点在后面将会用到。

### 6 动态任务

借助于强大的Groovy，我们还可以动态地创建任务。如下代码：
```gradle
4.times { counter ->
    task "task$counter" << {
        println "I'm task number $counter"
    }
}
```

我们定义了4个task，分别是task0, task1, task2, task3。我们来执行task1，如下：
```shell
msdx@msdx-ubuntu:~/tmp$ gradle -q task1
I'm task number 1
```

另外，gradle tasks命令可以查看我们定义的task，从这里我们也可以看到定义的task，如下：
```
...
Other tasks
-----------
task0
task1
task2
task3
...
```
注意，如果任务还未定义，不能使用短标记法（见本篇后续内容）来运行任务。

### 7 任务操纵

在Gradle当中，任务创建之后可以通过API进行访问，这是Gradle与Ant的不同之处。

### #7.1 增加依赖

还是以上面的例子，但是我们添加一行代码，如下：
```gradle
4.times { counter ->
    task "task$counter" << {
        println "I'm task number $counter"
    }
}
task1.dependsOn task0, task3
```

然后还是执行 gradle -q task1，看看结果：
```shell
msdx@msdx-ubuntu:~/tmp$ gradle -q task1
I'm task number 0
I'm task number 3
I'm task number 1
```

它先执行了task0和task3，因为task1依赖于这两个。

### #7.2 增加任务行为

如下代码：
```gradle
task hello << {
    println 'Hello, Gradle!'
}
hello.doFirst {
    println 'I am first.'
}
hello.doLast {
    println 'I am last.'
}
hello << {
    println 'I am the the last'
}
```

执行后的输出：
```shell
msdx@msdx-ubuntu:~/tmp$ gradle -q hello
I am first.
Hello, Gradle!
I am last.
I am the the last
```

### 8 短标记法

如果你对groovy有一定了解，那你也许会注意到，每个task都是一个构建脚本的属性，所以可以通过“$”这种短标记法来访问任务。如下：
```gradle
task hello << {
    println 'Hello, Gradle!'
}
hello.doLast {
    println "Greetings from the $hello.name task."
}
```

执行结果：
```shell
msdx@msdx-ubuntu:~/tmp$ gradle -q hello
Hello, Gradle!
Greetings from the hello task.
```

注意，通过这种方法访问的任务一定是要已经定义的。

### 9 增加自定义属性

```gradle
task myTask {
    ext.myProperty = "myValue"
}

task printTaskProperties << {
    println myTask.myProperty
}
```

输出结果：
```shell
msdx@msdx-ubuntu:~/tmp$ gradle -q printTaskProperties
myValue
```

### 10. 调用Ant任务

比如利用AntBuilder执行ant.loadfiile。
```gradle
task loadfile << {
    def files = file('config').listFiles().sort()
    files.each { File file ->
        if (file.isFile()) {
            ant.loadfile(srcFile: file, property: file.name)
            println " *** $file.name ***"
            println "${ant.properties[file.name]}"
        }
    }
}
```

执行结果：
```shell
msdx@msdx-ubuntu:~/tmp$ gradle -q loadfile
 *** db.config ***
db=mysql
username=root
password=123456

 *** user.config ***
username=admin
writeable=true
```

### 11 方法抽取

在上面的脚本中，我们可以把部分代码抽取出来，如下：
```gradle
task loadfile << {
    fileList('config').each { File file ->
        ant.loadfile(srcFile: file, property: file.name)
        println " *** $file.name ***"
        println "${ant.properties[file.name]}"
    }
}
File[] fileList(String dir) {
    file(dir).listFiles({file -> file.isFile() } as FileFilter).sort()
}
```

执行结果一样。

### 12. 定义默认任务

```gradle
defaultTasks 'clean', 'run'

task clean << {
    println 'Default Cleaning!'
}

task run << {
    println 'Default Running!'
}

task other << {
    println "I'm not a default task!"
}
```

执行结果：
```shell
msdx@msdx-ubuntu:~/tmp$ gradle -q
Default Cleaning!
Default Running!
```

### 13 DAG配置

Gradle使用DAG（Directed acyclic graph，有向非循环图）来决定任务执行的顺序。通过这一特性，我们可以实现依赖任务做不同输出。
如下代码：
```gradle
task distribution << {
    println "We build the zip with version=$version"
}

task release(dependsOn: 'distribution') << {
    println 'We release now'
}

gradle.taskGraph.whenReady {taskGraph ->
    if (taskGraph.hasTask(release)) {
        version = '1.0'
    } else {
        version = '1.0-SNAPSHOT'
    }
}
```

执行结果如下：
```shell`
msdx@msdx-ubuntu:~/tmp$ gradle -q distribution
We build the zip with version=1.0-SNAPSHOT
msdx@msdx-ubuntu:~/tmp$ gradle -q release
We build the zip with version=1.0
We release now
msdx@msdx-ubuntu:~/tmp$
```
在上面的脚本代码中，whenReady会在release任务执行之前影响它，即使这个任务不是主要的任务（即不是通过命令行传入参数来调用）。

本文原创，参考自Gradle官方文档，可看作是阅读该文档的笔记。
原文出处：[CSDN博客](http://blog.csdn.net/maosidiaoxian/article/details/40340571)
