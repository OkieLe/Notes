####什么是Servlet?

- Servlet就是JAVA 类
- Servlet是一个继承HttpServlet类的类
- 这个在服务器端运行，用以处理客户端的请求

####Servlet相关包的介绍

- javax.servlet.* ：存放与HTTP 协议无关的一般性Servlet 类；
- javax.servlet.http.* ：除了继承javax.servlet.* 之外，并且还增加与HTTP协议有关的功能。（注意：大家有必要学习一下HTTP协议，因为WEB开发都会涉及到）

    所有的Servlet 都必须实现javax.servlet.Servlet 接口(Interface)。

    若Servlet程序和HTTP 协议无关，那么必须继承javax.servlet.GenericServlet类；

    若Servlet程序和HTTP 协议有关，那么必须继承javax.servlet.http.HttpServlet 类。

- HttpServlet ：提供了一个抽象类用来创建Http Servlet。

    public void doGet()方法：用来处理客户端发出的 GET 请求

    public void doPost()方法：用来处理 POST请求

    还有几个方法大家自己去查阅API帮助文件

- javax.servlet包的接口：

    ServletConfig接口：在初始化的过程中由Servlet容器使用

    ServletContext接口：定义Servlet用于获取来自其容器的信息的方法

    ServletRequest接口：向服务器请求信息

    ServletResponse接口：响应客户端请求

    Filter接口：

- javax.servlet包的类：

    ServletInputStream类：用于从客户端读取二进制数据

    ServletOutputStream类：用于将二进制数据发送到客户端

- javax.servlet.http包的接口：

    HttpServletRequest接口：提供Http请求信息

    HttpServletResponse接口：提供Http响应

####Servlet生命周期

- Servlet生命周期就是指创建Servlet实例后，存在的时间以及何时销毁的整个过程．

- Servlet生命周期有三个方法

    init()方法：

    service()方法：Dispatches client requests to the protected service method　

    destroy()方法：Called by the servlet container to indicate to a servlet that the servlet is being taken out of service.
- Servlet生命周期的各个阶段

    实例化：Servlet容器创建Servlet实例

    初始化：调用init()方法

    服务：如果有请求，调用service()方法

    销毁：销毁实例前调用destroy()方法

    垃圾收集：销毁实例

####Servlet的基本结构

```Java
package cn.dragon.servlet;
//下面是导入相应的包
import java.io.IOException;
import java.io.PrintWriter;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
/**
* 这是第一个Servlet的例子
* @author cn.dragon
*/
public class ServletDemoFirst extends HttpServlet { 　　
　　//用于处理客户端发送的GET请求 　　
　　public void doGet(HttpServletRequest request, HttpServletResponse response) 　　
　　　　throws ServletException, IOException { 　　
　　　　　response.setContentType("text/html;charset=GB2312");　//这条语句指明了向客户端发送的内容格式和采用的字符编码． 　　
　　　　　PrintWriter out = response.getWriter();　 　　
　　　　　out.println(" 您好！");　//利用PrintWriter对象的方法将数据发送给客户端 　　
　　　　　out.close(); 　　
　　} 　　
　　//用于处理客户端发送的POST请求 　　
　　public void doPost(HttpServletRequest request, HttpServletResponse response) 　　
　　　　throws ServletException, IOException { 　　
　　　　doGet(request, response);　//这条语句的作用是，当客户端发送POST请求时，调用doGet()方法进行处理 　　
　　}
}
```

####Servlet的部署

以下截取部分
```xml
  <servlet>
    <description>任意</description>
    <display-name>任意</display-name>
    <servlet-name>ServletDemoFirst</servlet-name>
    <servlet-class>cn.dragon.servlet.ServletDemoFirst</servlet-class>
  </servlet>

　<servlet-mapping>
    <servlet-name>ServletDemoFirst</servlet-name>
    <url-pattern>/servlet/ServletDemoFirst</url-pattern>
  </servlet-mapping>
```

【注意】
1. 上面的两个`<servlet-name>`必须相同
2. `<servlet-class>`后面指在对应的类上面．　　技巧：你可以直接在你的servlet类中复制过来，这样可以避免出错！
3. `<url-pattern>`必须是/servlet 再加servlet名字.大家现在就这么记.

####Servlet实例演示
```Java
package cn.dragon.servlet;
import java.io.IOException;
import java.io.PrintWriter;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
public class ServletDemoSecond extends HttpServlet {
 　　//初始化
 　　public void init() throws ServletException {
  　　　　System.out.println("我是init()方法！用来进行初始化工作");
 　　}
 　　//处理GET请求
 　　public void doGet(HttpServletRequest request, HttpServletResponse response)
   　　throws ServletException, IOException {
  　　　　System.out.println("我是doGet()方法！用来处理GET请求");
  　　　　response.setContentType("text/html;charset=GB2312");
  　　　　PrintWriter out = response.getWriter();
  　　　　out.println("<HTML>");
  　　　　out.println("<BODY>");
  　　　　out.println("这是Servlet的例子");
  　　　　out.println("</BODY>");
  　　　　out.println("</HTML>");
 　　}
 　　//处理POST请求
 　　public void doPost(HttpServletRequest request, HttpServletResponse response)
   　　throws ServletException, IOException {
  　　　　doGet(request, response);
 　　}
 　　//销毁实例
 　　public void destroy() {
  　　　　super.destroy();
  　　　　System.out.println("我是destroy()方法！用来进行销毁实例的工作");
 　　}
}
```

####web.xml文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app version="2.4"
 　　xmlns="http://java.sun.com/xml/ns/j2ee"
 　　xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
　　 xsi:schemaLocation="http://java.sun.com/xml/ns/j2ee
　　 http://java.sun.com/xml/ns/j2ee/web-app_2_4.xsd">

  <servlet>
    <servlet-name>ServletDemoSecond</servlet-name>
    <servlet-class>cn.dragon.servlet.ServletDemoSecond</servlet-class>
  </servlet>

  <servlet-mapping>
    <servlet-name>ServletDemoSecond</servlet-name>
    <url-pattern>/servlet/ServletDemoSecond</url-pattern>
  </servlet-mapping>

</web-app>
```

来源： [cnblogs](http://www.cnblogs.com/goody9807/archive/2007/06/13/782519.html)
