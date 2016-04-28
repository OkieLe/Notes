### HTML Basic Document

```html
<html>
<head>
<title>Document name goes here</title>
</head>
<body>
Visible text goes here
</body>
</html>
```

### Text Elements

```html
<p>This is a paragraph</p>
<br> (line break)
<hr> (horizontal rule)
<pre>This text is preformatted</pre>
```

### Logical Styles

```html
<em>This text is emphasized</em>
<strong>This text is strong</strong>
<code>This is some computer code</code>
```

### Physical Styles

```html
<b>This text is bold</b>
<i>This text is italic</i>
```

### Links, Anchors, and Image Elements

```html
<a href="http://www.example.com/">This is a Link</a>
<a href="http://www.example.com/"><img src="URL" alt="Alternate Text"></a>
<a href="mailto:webmaster@example.com">Send e-mail</a>A named anchor:
<a name="tips">Useful Tips Section</a>
<a href="#tips">Jump to the Useful Tips Section</a>
```

### Unordered list

```html
<ul>
<li>First item</li>
<li>Next item</li>
</ul>
```

### Ordered list

```html
<ol>
<li>First item</li>
<li>Next item</li>
</ol>
```

### Definition list

```html
<dl>
<dt>First term</dt>
<dd>Definition</dd>
<dt>Next term</dt>
<dd>Definition</dd>
</dl>
```

### Tables

```html
<table border="1">
<tr>
  <th>someheader</th>
  <th>someheader</th>
</tr>
<tr>
  <td>sometext</td>
  <td>sometext</td>
</tr>
</table>
```

### Frames

```html
<frameset cols="25%,75%">
  <frame src="page1.htm">
  <frame src="page2.htm">
</frameset>
```

### Forms

```html
<form action="http://www.example.com/test.asp" method="post/get">
<input type="text" name="lastname" value="Nixon" size="30" maxlength="50">
<input type="password">
<input type="checkbox" checked="checked">
<input type="radio" checked="checked">
<input type="submit">
<input type="reset">
<input type="hidden">
<select>
<option>Apples
<option selected>Bananas
<option>Cherries
</select>
<textarea name="Comment" rows="60" cols="20"></textarea>
</form>
```

### Entities

```html
&lt; is the same as <
&gt; is the same as >
&#169; is the same as ©
```

### Other Elements

```html
<!-- This is a comment -->
<blockquote>
Text quoted from some source.
</blockquote>
<address>
Address 1<br>
Address 2<br>
City<br>
</address>
```

来源： [w3school](http://www.w3school.com.cn/html/html_quick.asp)

**XHTML 是作为 XML 被重新设计的 HTML。与 HTML 相比最重要的区别：**

### 文档结构

```
XHTML DOCTYPE 是强制性的
<html> 中的 XML namespace 属性是强制性的
<html>、<head>、<title> 以及 <body> 也是强制性的
```

### 元素语法

```
XHTML 元素必须正确嵌套
XHTML 元素必须始终关闭
XHTML 元素必须小写
XHTML 文档必须有一个根元素
```

### 属性语法

```
XHTML 属性必须使用小写
XHTML 属性值必须用引号包围
XHTML 属性最小化也是禁止的
```

来源： [w3school](http://www.w3school.com.cn/html/html_xhtml.asp)
