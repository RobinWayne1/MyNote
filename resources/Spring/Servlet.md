# 一、过滤器

https://www.cnblogs.com/xdp-gacl/p/3948353.html

![](E:\Typora\MyNote\resources\Spring\过滤器.png)

一句话就是实现`doFilter(ServletRequest,ServletResponse)`.

# 二、监听器

在ServletContext初始化时调用.

# 三、Servlet

第一次请求来到Servlet则调用`init()`方法,之后的请求将会直接调用`service()`,服务器关闭时调用`destory()`