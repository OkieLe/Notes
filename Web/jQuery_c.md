#### jQuery CSS 操作函数

下面列出的这些方法设置或返回元素的 CSS 相关属性。
```
css(): 设置或返回匹配元素的样式属性。
height(): 设置或返回匹配元素的高度。
offset(): 返回第一个匹配元素相对于文档的位置。
offsetParent(): 返回最近的定位祖先元素。
position(): 返回第一个匹配元素相对于父元素的位置。
scrollLeft(): 设置或返回匹配元素相对滚动条左侧的偏移。
scrollTop(): 设置或返回匹配元素相对滚动条顶部的偏移。
width(): 设置或返回匹配元素的宽度。
```

来源： [w2school](http://www.w3school.com.cn/jquery/jquery_ref_css.asp)

#### jQuery Ajax 操作函数

jQuery 库拥有完整的 Ajax 兼容套件。其中的函数和方法允许我们在不刷新浏览器的情况下从服务器加载数据。
```
jQuery.ajax(): 执行异步 HTTP (Ajax) 请求。
.ajaxComplete(): 当 Ajax 请求完成时注册要调用的处理程序。这是一个 Ajax 事件。
.ajaxError(): 当 Ajax 请求完成且出现错误时注册要调用的处理程序。这是一个 Ajax 事件。
.ajaxSend(): 在 Ajax 请求发送之前显示一条消息。
jQuery.ajaxSetup(): 设置将来的 Ajax 请求的默认值。
.ajaxStart(): 当首个 Ajax 请求完成开始时注册要调用的处理程序。这是一个 Ajax 事件。
.ajaxStop(): 当所有 Ajax 请求完成时注册要调用的处理程序。这是一个 Ajax 事件。
.ajaxSuccess(): 当 Ajax 请求成功完成时显示一条消息。
jQuery.get(): 使用 HTTP GET 请求从服务器加载数据。
jQuery.getJSON(): 使用 HTTP GET 请求从服务器加载 JSON 编码数据。
jQuery.getScript(): 使用 HTTP GET 请求从服务器加载 JavaScript 文件，然后执行该文件。
.load(): 从服务器加载数据，然后把返回到 HTML 放入匹配元素。
jQuery.param(): 创建数组或对象的序列化表示，适合在 URL 查询字符串或 Ajax 请求中使用。
jQuery.post(): 使用 HTTP POST 请求从服务器加载数据。
.serialize(): 将表单内容序列化为字符串。
.serializeArray(): 序列化表单元素，返回 JSON 数据结构数据。
```

[Ajax 教程](http://www.w3school.com.cn/ajax/index.asp)

来源： [w2school](http://www.w3school.com.cn/jquery/jquery_ref_ajax.asp)

#### jQuery 遍历函数

jQuery 遍历函数包括了用于筛选、查找和串联元素的方法。
```
.add(): 将元素添加到匹配元素的集合中。
.andSelf(): 把堆栈中之前的元素集添加到当前集合中。
.children(): 获得匹配元素集合中每个元素的所有子元素。
.closest(): 从元素本身开始，逐级向上级元素匹配，并返回最先匹配的祖先元素。
.contents(): 获得匹配元素集合中每个元素的子元素，包括文本和注释节点。
.each(): 对 jQuery 对象进行迭代，为每个匹配元素执行函数。
.end(): 结束当前链中最近的一次筛选操作，并将匹配元素集合返回到前一次的状态。
.eq(): 将匹配元素集合缩减为位于指定索引的新元素。
.filter(): 将匹配元素集合缩减为匹配选择器或匹配函数返回值的新元素。
.find(): 获得当前匹配元素集合中每个元素的后代，由选择器进行筛选。
.first(): 将匹配元素集合缩减为集合中的第一个元素。
.has(): 将匹配元素集合缩减为包含特定元素的后代的集合。
.is(): 根据选择器检查当前匹配元素集合，如果存在至少一个匹配元素，则返回 true。
.last(): 将匹配元素集合缩减为集合中的最后一个元素。
.map(): 把当前匹配集合中的每个元素传递给函数，产生包含返回值的新 jQuery 对象。
.next(): 获得匹配元素集合中每个元素紧邻的同辈元素。
.nextAll(): 获得匹配元素集合中每个元素之后的所有同辈元素，由选择器进行筛选（可选）。
.nextUntil(): 获得每个元素之后所有的同辈元素，直到遇到匹配选择器的元素为止。
.not(): 从匹配元素集合中删除元素。
.offsetParent(): 获得用于定位的第一个父元素。
.parent(): 获得当前匹配元素集合中每个元素的父元素，由选择器筛选（可选）。
.parents(): 获得当前匹配元素集合中每个元素的祖先元素，由选择器筛选（可选）。
.parentsUntil(): 获得当前匹配元素集合中每个元素的祖先元素，直到遇到匹配选择器的元素为止。
.prev(): 获得匹配元素集合中每个元素紧邻的前一个同辈元素，由选择器筛选（可选）。
.prevAll(): 获得匹配元素集合中每个元素之前的所有同辈元素，由选择器进行筛选（可选）。
.prevUntil(): 获得每个元素之前所有的同辈元素，直到遇到匹配选择器的元素为止。
.siblings(): 获得匹配元素集合中所有元素的同辈元素，由选择器筛选（可选）。
.slice(): 将匹配元素集合缩减为指定范围的子集。
```

来源： [w2school](http://www.w3school.com.cn/jquery/jquery_ref_traversing.asp)

#### jQuery 数据操作函数

这些方法允许我们将指定的 DOM 元素与任意数据相关联。
```
.clearQueue(): 从队列中删除所有未运行的项目。
.data(): 存储与匹配元素相关的任意数据。
jQuery.data(): 存储与指定元素相关的任意数据。
.dequeue(): 从队列最前端移除一个队列函数，并执行它。
jQuery.dequeue(): 从队列最前端移除一个队列函数，并执行它。
jQuery.hasData(): 存储与匹配元素相关的任意数据。
.queue(): 显示或操作匹配元素所执行函数的队列。
jQuery.queue(): 显示或操作匹配元素所执行函数的队列。
.removeData(): 移除之前存放的数据。
jQuery.removeData(): 移除之前存放的数据。
```

来源： [w2school](http://www.w3school.com.cn/jquery/jquery_ref_data.asp)

#### jQuery DOM 元素方法

```
.get(): 获得由选择器指定的 DOM 元素。
.index(): 返回指定元素相对于其他指定元素的 index 位置。
.size(): 返回被 jQuery 选择器匹配的元素的数量。
.toArray(): 以数组的形式返回 jQuery 选择器匹配的元素。
```

来源：[w2school](http://www.w3school.com.cn/jquery/jquery_ref_dom_element_methods.asp)

#### jQuery 核心函数

```
jQuery(): 接受一个字符串，其中包含了用于匹配元素集合的 CSS 选择器。
jQuery.noConflict(): 运行这个函数将变量 $ 的控制权让渡给第一个实现它的那个库。
```

来源：[w2school](http://www.w3school.com.cn/jquery/jquery_ref_core.asp)

#### jQuery 属性

下面列出的这些方法设置或返回元素的 CSS 相关属性。
```
context: 在版本 1.10 中被弃用。包含传递给 jQuery() 的原始上下文。
jquery: 包含 jQuery 版本号。
jQuery.fx.interval: 改变以毫秒计的动画速率。
jQuery.fx.off: 全局禁用/启用所有动画。
jQuery.support: 表示不同浏览器特性或漏洞的属性集合（用于 jQuery 内部使用）。
length: 包含 jQuery 对象中的元素数目。
```

来源： [w2school](http://www.w3school.com.cn/jquery/jquery_ref_prop.asp)
