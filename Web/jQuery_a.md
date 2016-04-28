[w3school](http://www.w3school.com.cn/jquery/index.asp)

jQuery 库可以通过一行简单的标记被添加到网页中。

#### jQuery 库 - 特性

jQuery 是一个 JavaScript 函数库。
jQuery 库包含以下特性：
- HTML 元素选取
- HTML 元素操作
- CSS 操作
- HTML 事件函数
- JavaScript 特效和动画
- HTML DOM 遍历和修改
- AJAX
- Utilities

#### 向您的页面添加 jQuery 库

jQuery 库位于一个 JavaScript 文件中，其中包含了所有的 jQuery 函数。

可以通过下面的标记把 jQuery 添加到网页中：
```html
<head>
<script type="text/javascript" src="jquery.js"></script>
</head>
```
请注意，`<script>` 标签应该位于页面的 `<head>` 部分。HTML5中不必使用 `type="text/javascript"`。

#### 下载 jQuery

共有两个版本的 jQuery 可供下载：一份是精简过的，另一份是未压缩的（供调试或阅读）。

这两个版本都可从 jQuery.com 下载。

#### 库的替代

Google 和 Microsoft 对 jQuery 的支持都很好。

如果您不愿意在自己的计算机上存放 jQuery 库，那么可以从 Google 或 Microsoft 加载 CDN jQuery 核心文件。

#### 使用 Google 的 CDN

```html
<head>
<script type="text/javascript" src="http://ajax.googleapis.com/ajax/libs
/jquery/1.4.0/jquery.min.js"></script>
</head>
```

#### 使用 Microsoft 的 CDN

```html
<head>
<script type="text/javascript" src="http://ajax.microsoft.com/ajax/jquery
/jquery-1.4.min.js"></script>
</head>
```

#### jQuery 语法

jQuery 语法是为 HTML 元素的选取编制的，可以对元素执行某些操作。

基础语法是：`$(selector).action()`

美元符号定义 jQuery

选择符（selector）“查询”和“查找” HTML 元素

jQuery 的 `action()` 执行对元素的操作

##### 示例

```
$(this).hide() - 隐藏当前元素
$("p").hide() - 隐藏所有段落
$(".test").hide() - 隐藏所有 class="test" 的所有元素
$("#test").hide() - 隐藏所有 id="test" 的元素
```

#### 文档就绪函数

您也许已经注意到在我们的实例中的所有 jQuery 函数位于一个 document ready 函数中：
```JavaScript
$(document).ready(function(){
--- jQuery functions go here ----
});
```
这是为了防止文档在完全加载（就绪）之前运行 jQuery 代码。

jQuery 使用 $ 符号作为 jQuery 的简介方式。

某些其他 JavaScript 库中的函数（比如 Prototype）同样使用 $ 符号。jQuery 使用名为 noConflict() 的方法来解决该问题。

`var jq=jQuery.noConflict()`，帮助您使用自己的名称（比如 jq）来代替 $ 符号。

由于 jQuery 是为处理 HTML 事件而特别设计的，那么当您遵循以下原则时，您的代码会更恰当且更易维护：
```
把所有 jQuery 代码置于事件处理函数中
把所有事件处理函数置于文档就绪事件处理器中
把 jQuery 代码置于单独的 .js 文件中
如果存在名称冲突，则重命名 jQuery 库
```

#### AJAX

AJAX 是与服务器交换数据的艺术，它在不重载全部页面的情况下，实现了对部分网页的更新。

AJAX = 异步 JavaScript 和 XML（Asynchronous JavaScript and XML）。简短地说，在不重载整个网页的情况下，AJAX 通过后台加载数据，并在网页上进行显示。jQuery 提供多个与 AJAX 有关的方法。

通过 jQuery AJAX 方法，您能够使用 HTTP Get 和 HTTP Post 从远程服务器上请求文本、HTML、XML 或 JSON - 同时您能够把这些外部数据直接载入网页的被选元素中。

#### jQuery load() 方法

jQuery load() 方法是简单但强大的 AJAX 方法。
load() 方法从服务器加载数据，并把返回的数据放入被选元素中。

##### 语法：

`$(selector).load(URL,data,callback);`
- 必需的 URL 参数规定您希望加载的 URL。
- 可选的 data 参数规定与请求一同发送的查询字符串键/值对集合。
- 可选的 callback 参数是 load() 方法完成后所执行的函数名称。

这是示例文件（"demo_test.txt"）的内容：
```html
<h2>jQuery and AJAX is FUN!!!</h2>
<p id="p1">This is some text in a paragraph.</p>
```
下面的例子会把文件 "demo_test.txt" 的内容加载到指定的 `<div>` 元素中：
```JavaScript
$("#div1").load("demo_test.txt");
```

也可以把 jQuery 选择器添加到 URL 参数。
下面的例子把 "demo_test.txt" 文件中 id="p1" 的元素的内容，加载到指定的 `<div>` 元素中：
```JavaScript
$("#div1").load("demo_test.txt #p1");
```
可选的 callback 参数规定当 load() 方法完成后所要允许的回调函数。回调函数可以设置不同的参数：
- responseTxt - 包含调用成功时的结果内容
- statusTXT - 包含调用的状态
- xhr - 包含 XMLHttpRequest 对象

下面的例子会在 load() 方法完成后显示一个提示框。如果 load() 方法已成功，则显示“外部内容加载成功！”，而如果失败，则显示错误消息：
```JavaScript
$("button").click(function(){
  $("#div1").load("demo_test.txt",function(responseTxt,statusTxt,xhr){
    if(statusTxt=="success")
      alert("外部内容加载成功！");
    if(statusTxt=="error")
      alert("Error: "+xhr.status+": "+xhr.statusText);
  });
});
```

来源： [w3school](http://www.w3school.com.cn/jquery/jquery_ajax_load.asp)

#### HTTP 请求：GET vs. POST

jQuery get() 和 post() 方法用于通过 HTTP GET 或 POST 请求从服务器请求数据。

两种在客户端和服务器端进行请求-响应的常用方法是：GET 和 POST。

- GET - 从指定的资源请求数据
- POST - 向指定的资源提交要处理的数据

GET 基本上用于从服务器获得（取回）数据。注释：GET 方法可能返回缓存数据。

POST 也可用于从服务器获取数据。不过，POST 方法不会缓存数据，并且常用于连同请求一起发送数据。

##### jQuery $.get() 方法

`$.get()` 方法通过 HTTP GET 请求从服务器上请求数据。语法：

`$.get(URL,callback);`
>必需的 URL 参数规定您希望请求的 URL。
可选的 callback 参数是请求成功后所执行的函数名。

下面的例子使用 `$.get()` 方法从服务器上的一个文件中取回数据：

```JavaScript
$("button").click(function(){
  $.get("demo_test.asp",function(data,status){
    alert("Data: " + data + "\nStatus: " + status);
  });
});
```

$.get() 的第一个参数是我们希望请求的 URL（"demo_test.asp"）。
第二个参数是回调函数。第一个回调参数存有被请求页面的内容，第二个回调参数存有请求的状态。

提示：这个 ASP 文件 ("demo_test.asp") 类似这样：
```asp
<%
response.write("This is some text from an external ASP file.")
%>
```

##### jQuery $.post() 方法

$.post() 方法通过 HTTP POST 请求从服务器上请求数据。
语法：

`$.post(URL,data,callback);`
>必需的 URL 参数规定您希望请求的 URL。
可选的 data 参数规定连同请求发送的数据。
可选的 callback 参数是请求成功后所执行的函数名。

下面的例子使用 `$.post()` 连同请求一起发送数据：

```JavaScript
$("button").click(function(){
  $.post("demo_test_post.asp",
  {
    name:"Donald Duck",
    city:"Duckburg"
  },
  function(data,status){
    alert("Data: " + data + "\nStatus: " + status);
  });
});
```

$.post() 的第一个参数是我们希望请求的 URL ("demo_test_post.asp")。
然后我们连同请求（name 和 city）一起发送数据。

"demo_test_post.asp" 中的 ASP 脚本读取这些参数，对它们进行处理，然后返回结果。

第三个参数是回调函数。第一个回调参数存有被请求页面的内容，而第二个参数存有请求的状态。

提示：这个 ASP 文件 ("demo_test_post.asp") 类似这样：
```asp
<%
dim fname,city
fname=Request.Form("name")
city=Request.Form("city")
Response.Write("Dear " & fname & ". ")
Response.Write("Hope you live well in " & city & ".")
%>
```

来源： [w3school](http://www.w3school.com.cn/jquery/jquery_ajax_get_post.asp)
