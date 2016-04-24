####jQuery 选择器

```
*
eg: $("*") ==> 所有元素
#id
eg: $("#lastname") ==> id="lastname" 的元素
.class
eg: $(".intro") ==> 所有 class="intro" 的元素
element
eg: $("p") ==> 所有 <p> 元素
.class.class
eg: $(".intro.demo") ==> 所有 class="intro" 且 class="demo" 的元素


:first
eg: $("p:first") ==> 第一个 <p> 元素
:last
eg: $("p:last") ==> 最后一个 <p> 元素
:even
eg: $("tr:even") ==> 所有偶数 <tr> 元素
:odd
eg: $("tr:odd") ==> 所有奇数 <tr> 元素


:eq(index)
eg: $("ul li:eq(3)") ==> 列表中的第四个元素（index 从 0 开始）
:gt(no)
eg: $("ul li:gt(3)") ==> 列出 index 大于 3 的元素
:lt(no)
eg: $("ul li:lt(3)") ==> 列出 index 小于 3 的元素
:not(selector)
eg: $("input:not(:empty)") ==> 所有不为空的 input 元素


:header
eg: $(":header") ==> 所有标题元素 <h1> - <h6>
:animated
eg: $(":animated") ==> 所有动画元素


:contains(text)
eg: $(":contains('W3School')") ==> 包含指定字符串的所有元素
:empty
eg: $(":empty") ==> 无子（元素）节点的所有元素
:hidden
eg: $("p:hidden") ==> 所有隐藏的 <p> 元素
:visible
eg: $("table:visible") ==> 所有可见的表格


s1,s2,s3
eg: $("th,td,.intro") ==> 所有带有匹配选择的元素


[attribute]
eg: $("[href]") ==> 所有带有 href 属性的元素
[attribute=value]
eg: $("[href='#']") ==> 所有 href 属性的值等于 "#" 的元素
[attribute!=value]
eg: $("[href!='#']") ==> 所有 href 属性的值不等于 "#" 的元素
[attribute$=value]
eg: $("[href$='.jpg']") ==> 所有 href 属性的值包含以 ".jpg" 结尾的元素


:input
eg: $(":input") ==> 所有 <input> 元素
:text
eg: $(":text") ==> 所有 type="text" 的 <input> 元素
:password
eg: $(":password") ==> 所有 type="password" 的 <input> 元素
:radio
eg: $(":radio") ==> 所有 type="radio" 的 <input> 元素
:checkbox
eg: $(":checkbox") ==> 所有 type="checkbox" 的 <input> 元素
:submit
eg: $(":submit") ==> 所有 type="submit" 的 <input> 元素
:reset
eg: $(":reset") ==> 所有 type="reset" 的 <input> 元素
:button
eg: $(":button") ==> 所有 type="button" 的 <input> 元素
:image
eg: $(":image") ==> 所有 type="image" 的 <input> 元素
:file
eg: $(":file") ==> 所有 type="file" 的 <input> 元素


:enabled
eg: $(":enabled") ==> 所有激活的 input 元素
:disabled
eg: $(":disabled") ==> 所有禁用的 input 元素
:selected
eg: $(":selected") ==> 所有被选取的 input 元素
:checked
eg: $(":checked") ==> 所有被选中的 input 元素
```

来源： [w3school](http://www.w3school.com.cn/jquery/jquery_ref_selectors.asp)


####jQuery 事件方法

事件方法会触发匹配元素的事件，或将函数绑定到所有匹配元素的某个事件。

触发实例：
`$("button#demo").click()`

上面的例子将触发 id="demo" 的 button 元素的 click 事件。

绑定实例：
`$("button#demo").click(function(){$("img").hide()})`

上面的例子会在点击 id="demo" 的按钮时隐藏所有图像。
```
bind(): 向匹配元素附加一个或更多事件处理器
blur(): 触发、或将函数绑定到指定元素的 blur 事件
change(): 触发、或将函数绑定到指定元素的 change 事件
click(): 触发、或将函数绑定到指定元素的 click 事件
dblclick(): 触发、或将函数绑定到指定元素的 double click 事件
delegate(): 向匹配元素的当前或未来的子元素附加一个或多个事件处理器
die(): 移除所有通过 live() 函数添加的事件处理程序。
error(): 触发、或将函数绑定到指定元素的 error 事件
event.isDefaultPrevented(): 返回 event 对象上是否调用了 event.preventDefault()。
event.pageX: 相对于文档左边缘的鼠标位置。
event.pageY: 相对于文档上边缘的鼠标位置。
event.preventDefault(): 阻止事件的默认动作。
event.result: 包含由被指定事件触发的事件处理器返回的最后一个值。
event.target: 触发该事件的 DOM 元素。
event.timeStamp: 该属性返回从 1970 年 1 月 1 日到事件发生时的毫秒数。
event.type: 描述事件的类型。
event.which: 指示按了哪个键或按钮。
focus(): 触发、或将函数绑定到指定元素的 focus 事件
keydown(): 触发、或将函数绑定到指定元素的 key down 事件
keypress(): 触发、或将函数绑定到指定元素的 key press 事件
keyup(): 触发、或将函数绑定到指定元素的 key up 事件
live(): 为当前或未来的匹配元素添加一个或多个事件处理器
load(): 触发、或将函数绑定到指定元素的 load 事件
mousedown(): 触发、或将函数绑定到指定元素的 mouse down 事件
mouseenter(): 触发、或将函数绑定到指定元素的 mouse enter 事件
mouseleave(): 触发、或将函数绑定到指定元素的 mouse leave 事件
mousemove(): 触发、或将函数绑定到指定元素的 mouse move 事件
mouseout(): 触发、或将函数绑定到指定元素的 mouse out 事件
mouseover(): 触发、或将函数绑定到指定元素的 mouse over 事件
mouseup(): 触发、或将函数绑定到指定元素的 mouse up 事件
one(): 向匹配元素添加事件处理器。每个元素只能触发一次该处理器。
ready(): 文档就绪事件（当 HTML 文档就绪可用时）
resize(): 触发、或将函数绑定到指定元素的 resize 事件
scroll(): 触发、或将函数绑定到指定元素的 scroll 事件
select(): 触发、或将函数绑定到指定元素的 select 事件
submit(): 触发、或将函数绑定到指定元素的 submit 事件
toggle(): 绑定两个或多个事件处理器函数，当发生轮流的 click 事件时执行。
trigger(): 所有匹配元素的指定事件
triggerHandler(): 第一个被匹配元素的指定事件
unbind(): 从匹配元素移除一个被添加的事件处理器
undelegate(): 从匹配元素移除一个被添加的事件处理器，现在或将来
unload(): 触发、或将函数绑定到指定元素的 unload 事件
```
来源： [w3school](http://www.w3school.com.cn/jquery/jquery_ref_events.asp)

####jQuery 效果函数

```
animate(): 对被选元素应用“自定义”的动画
clearQueue(): 对被选元素移除所有排队的函数（仍未运行的）
delay(): 对被选元素的所有排队函数（仍未运行）设置延迟
dequeue(): 运行被选元素的下一个排队函数
fadeIn(): 逐渐改变被选元素的不透明度，从隐藏到可见
fadeOut(): 逐渐改变被选元素的不透明度，从可见到隐藏
fadeTo(): 把被选元素逐渐改变至给定的不透明度
hide(): 隐藏被选的元素
queue(): 显示被选元素的排队函数
show(): 显示被选的元素
slideDown(): 通过调整高度来滑动显示被选元素
slideToggle(): 对被选元素进行滑动隐藏和滑动显示的切换
slideUp(): 通过调整高度来滑动隐藏被选元素
stop(): 停止在被选元素上运行动画
toggle(): 对被选元素进行隐藏和显示的切换
```

来源： [w3school](http://www.w3school.com.cn/jquery/jquery_ref_effects.asp)

####jQuery 文档操作方法

这些方法对于 XML 文档和 HTML 文档均是适用的，除了：html()。

```
addClass(): 向匹配的元素添加指定的类名。
after(): 在匹配的元素之后插入内容。
append(): 向匹配元素集合中的每个元素结尾插入由参数指定的内容。
appendTo(): 向目标结尾插入匹配元素集合中的每个元素。
attr(): 设置或返回匹配元素的属性和值。
before(): 在每个匹配的元素之前插入内容。
clone(): 创建匹配元素集合的副本。
detach(): 从 DOM 中移除匹配元素集合。
empty(): 删除匹配的元素集合中所有的子节点。
hasClass(): 检查匹配的元素是否拥有指定的类。
html(): 设置或返回匹配的元素集合中的 HTML 内容。
insertAfter(): 把匹配的元素插入到另一个指定的元素集合的后面。
insertBefore(): 把匹配的元素插入到另一个指定的元素集合的前面。
prepend(): 向匹配元素集合中的每个元素开头插入由参数指定的内容。
prependTo(): 向目标开头插入匹配元素集合中的每个元素。
remove(): 移除所有匹配的元素。
removeAttr(): 从所有匹配的元素中移除指定的属性。
removeClass(): 从所有匹配的元素中删除全部或者指定的类。
replaceAll(): 用匹配的元素替换所有匹配到的元素。
replaceWith(): 用新内容替换匹配的元素。
text(): 设置或返回匹配元素的内容。
toggleClass(): 从匹配的元素中添加或删除一个类。
unwrap(): 移除并替换指定元素的父元素。
val(): 设置或返回匹配元素的值。
wrap(): 把匹配的元素用指定的内容或元素包裹起来。
wrapAll(): 把所有匹配的元素用指定的内容或元素包裹起来。
wrapinner(): 将每一个匹配的元素的子内容用指定的内容或元素包裹起来。
```

来源： [w3school](http://www.w3school.com.cn/jquery/jquery_ref_manipulation.asp)

####Query 属性操作方法

下面列出的这些方法获得或设置元素的 DOM 属性。
这些方法对于 XML 文档和 HTML 文档均是适用的，除了：`html()`。

```
addClass(): 向匹配的元素添加指定的类名。
attr(): 设置或返回匹配元素的属性和值。
hasClass(): 检查匹配的元素是否拥有指定的类。
html(): 设置或返回匹配的元素集合中的 HTML 内容。
removeAttr(): 从所有匹配的元素中移除指定的属性。
removeClass(): 从所有匹配的元素中删除全部或者指定的类。
toggleClass(): 从匹配的元素中添加或删除一个类。
val(): 设置或返回匹配元素的值。
```
注释：jQuery 文档操作参考手册中也列出了以上方法。本参考页的作用是方便用户单独查阅有关属性操作方面的方法。

来源： [w3school](http://www.w3school.com.cn/jquery/jquery_ref_attributes.asp)
