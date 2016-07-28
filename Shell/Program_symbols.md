# SHELL编程之特殊符号

> 版权声明： 本文章内容在非商业使用前提下可无需授权任意转载、发布。
> 转载、发布请务必注明作者和其微博、微信公众号地址，以便读者询问问题和甄误反馈，共同进步。
> 微博ID：**orroz**
> 微信公众号：**Linux系统技术**

#### 前言

本文是shell编程系列的第四篇，集中介绍了bash编程可能涉及到的特殊符号的使用。学会本文内容可以帮助你写出天书一样的bash脚本，并且顺便解决以下问题：

1. 输入输出重定向是什么原理？
2.  `exec 3<> /tmp/filename`是什么鬼？
3. 你玩过bash的关联数组吗？
4. 如何不用if判断变量是否被定义？
5. 脚本中字符串替换和删除操作不用sed怎么做？
6. ” “和’ ‘有什么不同？
7. 正则表达式和bash通配符是一回事么？

这里需要额外注意的是，相同的符号出现在不同的上下文中可能会有不同的含义。我们会在后续的讲解中突出它们的区别。

#### 重定向(REDIRECTION)

重定向也叫输入输出重定向。我们先通过基本的使用对这个概念有个感性认识。

##### 输入重定向

大家应该都用过cat命令，可以输出一个文件的内容。如：`cat /etc/passwd`。如果不给cat任何参数，那么cat将从键盘（标准输入）读取用户的输入，直接将内容显示到屏幕上，就像这样：
```
[zorro@zorrozou-pc0 bash]$ cat
hello
hello
I am zorro!
I am zorro!
```
可以通过输入重定向让cat命令从别的地方读取输入，显示到当前屏幕上。最简单的方式是输入重定向一个文件，不过这不够“神奇”，我们让cat从别的终端读取输入试试。我当前使用桌面的终端terminal开了多个bash，使用ps命令可以看到这些终端所占用的输入文件是哪个：
```
[zorro@zorrozou-pc0 bash]$ ps ax|grep bash
4632 pts/0 Ss 0:00 -bash
5087 pts/2 S+ 0:00 man bash
5897 pts/1 Ss 0:00 -bash
5911 pts/2 Ss 0:00 -bash
9071 pts/4 Ss 0:00 -bash
11667 pts/3 Ss+ 0:00 -bash
16309 pts/4 S+ 0:00 grep --color=auto bash
19465 pts/2 S 0:00 sudo bash
19466 pts/2 S 0:00 bash
```
通过第二列可以看到，不同的bash所在的终端文件是哪个，这里的`pts/3`就意味着这个文件放在`/dev/pts/3`。我们来试一下，在`pts/2`对应的bash中输入：
```
[zorro@zorrozou-pc0 bash]$ cat < /dev/pts/3
```
然后切换到`pts/3`所在的bash上敲入字符串，在`pts/2`的bash中能看见相关字符：
```
[zorro@zorrozou-pc0 bash]$ cat < /dev/pts/3
safsdfsfsfadsdsasdfsafadsadfd
```
这只是个输入重定向的例子，一般我们也可以直接`cat < /etc/passwd`，表示让cat命令不是从默认输入读取，而是从`/etc/passwd`读取，这就是输入重定向，使用”<“。

##### 输出重定向

绝大多数命令都有输出，用来显示给人看，所以输出基本都显示在屏幕（终端）上。有时候我们不想看到，就可以把输出重定向到别的地方：
```
[zorro@zorrozou-pc0 bash]$ ls /
bin boot cgroup data dev etc home lib lib64 lost+found mnt opt proc root run sbin srv sys tmp usr var
[zorro@zorrozou-pc0 bash]$ ls / > /tmp/out
[zorro@zorrozou-pc0 bash]$ cat /tmp/out
bin boot cgroup data dev ......
```
使用一个”>”，将原本显示在屏幕上的内容给输出到了`/tmp/out`文件中。这个功能就是输出重定向。

##### 报错重定向

命令执行都会遇到错误，一般也都是给人看的，所以默认还是显示在屏幕上。这些输出使用”>”是不能进行重定向的：
```
[zorro@zorrozou-pc0 bash]$ ls /1234 > /tmp/err
ls: cannot access '/1234': No such file or directory
```
可以看到，报错还是显示在了屏幕上。如果想要重定向这样的内容，可以使用”2>”：
```
[zorro@zorrozou-pc0 bash]$ ls /1234 2> /tmp/err
[zorro@zorrozou-pc0 bash]$ cat /tmp/err
ls: cannot access '/1234': No such file or directory
```
以上就是常见的输入输出重定向。在进行其它技巧讲解之前，我们有必要理解一下重定向的本质，所以要先从文件描述符说起。
