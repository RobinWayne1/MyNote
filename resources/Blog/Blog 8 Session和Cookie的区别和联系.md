# Blog 8 Session和Cookie的区别和联系

### 一、Session的概念
Session 是存放在服务器端的，类似于Session结构来存放用户数据，当浏览器 第一次发送请求时，服务器自动生成了一个Session和一个Session ID用来唯一标识这个Session，并将其通过响应发送到浏览器。当浏览器第二次发送请求，会将前一次服务器响应中的Session ID放在请求中一并发送到服务器上，服务器从请求中提取出Session ID，并和保存的所有Session ID进行对比，找到这个用户对应的Session。

一般情况下，**服务器会在一定时间内（默认30分钟）保存这个 Session，过了时间限制，就会销毁这个Session。**在销毁之前，程序员可以将用户的一些数据以Key和Value的形式暂时存放在这个 Session中。当然，也有使用数据库将这个Session序列化后保存起来的，这样的好处是没了时间的限制，坏处是随着时间的增加，这个数据 库会急速膨胀，特别是访问量增加的时候。一般还是采取前一种方式，以减轻服务器压力。

### 二、Session的客户端实现形式（即Session ID的保存方法）
一般浏览器提供了两种方式来保存，还有一种是程序员使用html隐藏域的方式自定义实现：

[1] ==使用Cookie来保存==，这是最常见的方法，本文“记住我的登录状态”功能的实现正式基于这种方式的。服务器通过设置Cookie的方式将Session ID发送到浏览器。如果我们不设置这个过期时间，那么这个Cookie将不存放在硬盘上，当浏览器关闭的时候，Cookie就消失了，这个Session ID就丢失了。如果我们设置这个时间为若干天之后，那么这个Cookie会保存在客户端硬盘中，即使浏览器关闭，这个值仍然存在，下次访问相应网站时，同 样会发送到服务器上。

[2] ==使用URL附加信息的方式==，也就是像我们经常看到JSP网站会有aaa.jsp?JSESSIONID=*一样的。这种方式和第一种方式里面不设置Cookie过期时间是一样的。

> URL地址重写是对客户端不支持Cookie的解决方案。**URL地址重写的原理是将该用户Session的id信息重写到URL地址中。服务器能够解析重写后的URL获取Session的id。这样即使客户端不支持Cookie，也可以使用Session来记录用户状态。**HttpServletResponse类提供了encodeURL(Stringurl)实现URL地址重写，例如：
>
> 
>
> ```html
> <td>
> <a href="<%=response.encodeURL("index.jsp?c=1&wd=Java") %>"> 
> Homepage</a>
> </td>
> ```
>
> 
>
> ==**该方法会自动判断客户端是否支持Cookie。如果客户端支持Cookie，会将URL原封不动地输出来。如果客户端不支持Cookie，则会将用户Session的id重写到URL中。**==重写后的输出可能是这样的：
>
> 
>
> ```html
> <td>
> <ahref="index.jsp;jsessionid=0CCD096E7F8D97B0BE608AFDC3E1931E?c=
> 1&wd=Java">Homepage</a>
> </td>
> ```
>
> 
>
> 即在文件名的后面，在URL参数的前面添加了字符串“;jsessionid=XXX”。其中XXX为Session的id。分析一下可以知道，增添的jsessionid字符串既不会影响请求的文件名，也不会影响提交的地址栏参数。用户单击这个链接的时候会把Session的id通过URL提交到服务器上，服务器通过解析URL地址获得Session的id。
>
> 如果是页面重定向（Redirection），URL地址重写可以这样写：
>
> 
>
> ```jsp
> <%
> if(“administrator”.equals(userName))
> 
> {
> 
>    response.sendRedirect(response.encodeRedirectURL(“administrator.jsp”));
> 
>     return;
> 
> }
> %>
> ```
>
> 
>
> 效果跟response.encodeURL(String url)是一样的：如果客户端支持Cookie，生成原URL地址，如果不支持Cookie，传回重写后的带有jsessionid字符串的地址。
>
> 对于WAP程序，由于大部分的手机浏览器都不支持Cookie，WAP程序都会采用URL地址重写来跟踪用户会话。比如用友集团的移动商街等。
>
> 
>
> 注意**：==TOMCAT判断客户端浏览器是否支持Cookie的依据是请求中是否含有Cookie。==**尽管客户端可能会支持Cookie，但是由于第一次请求时不会携带任何Cookie（因为并无任何Cookie可以携带），URL地址重写后的地址中仍然会带有jsessionid。当第二次访问时服务器已经在浏览器中写入Cookie了，因此URL地址重写后的地址中就不会带有jsessionid了。
>




[3] ==第三种方式是在页面表单里面增加隐藏域==，这种方式实际上和第二种方式一样，**只不过前者通过GET方式发送数据，后者使用POST方式发送数据。但是明显后者比较麻烦。**

cookie与session的区别：

cookie数据保存在客户端，session数据保存在服务器端。

简 单的说，当你登录一个网站的时候，如果web服务器端使用的是session,那么所有的数据都保存在服务器上面，客户端每次请求服务器的时候会发送 当前会话的sessionid，服务器根据当前sessionid判断相应的用户数据标志，以确定用户是否登录，或具有某种权限。由于数据是存储在服务器 上面，所以你不能伪造，但是如果你能够获取某个登录用户的sessionid，用特殊的浏览器伪造该用户的请求也是能够成功的。sessionid是服务 器和客户端链接时候随机分配的，一般来说是不会有重复，但如果有大量的并发请求，也不是没有重复的可能性，我曾经就遇到过一次。登录某个网站，开始显示的 是自己的信息，等一段时间超时了，一刷新，居然显示了别人的信息。

如果浏览器使用的是 cookie，那么所有的数据都保存在浏览器端，比如你登录以后，服务器设置了 cookie用户名(username),那么，当你再次请求服务器的时候，浏览器会将username一块发送给服务器，这些变量有一定的特殊标记。服 务器会解释为 cookie变量。所以只要不关闭浏览器，那么 cookie变量便一直是有效的，所以能够保证长时间不掉线。如果你能够截获某个用户的 cookie变量，然后伪造一个数据包发送过去，那么服务器还是认为你是合法的。所以，使用 cookie被攻击的可能性比较大。如果设置了的有效时间，那么它会将 cookie保存在客户端的硬盘上，下次再访问该网站的时候，浏览器先检查有没有 cookie，如果有的话，就读取该 cookie，然后发送给服务器。如果你在机器上面保存了某个论坛 cookie，有效期是一年，如果有人入侵你的机器，将你的 cookie拷走，然后放在他的浏览器的目录下面，那么他登录该网站的时候就是用你的的身份登录的。所以 cookie是可以伪造的。当然，伪造的时候需要主意，直接copy cookie文件到 cookie目录，浏览器是不认的，他有一个index.dat文件，存储了 cookie文件的建立时间，以及是否有修改，所以你必须先要有该网站的 cookie文件，并且要从保证时间上骗过浏览器，曾经在学校的vbb论坛上面做过试验，copy别人的 cookie登录，冒用了别人的名义发帖子，完全没有问题。

Session是由应用服务器维持的一个服务器端的存储空间，==用户在连接服务器时，会由服务器生成一个唯一的SessionID==,用该SessionID 为标识符来存取服务器端的Session存储空间。而SessionID这一数据则是保存到客户端，用Cookie保存的，用户提交页面时，会将这一 SessionID提交到服务器端，来存取Session数据。这一过程，是不用开发人员干预的。所以一旦客户端禁用Cookie，那么Session也会失效。

服务器也可以通过URL重写的方式来传递SessionID的值，因此不是完全依赖Cookie。如果客户端Cookie禁用，则服务器可以自动通过重写URL的方式来保存Session的值，并且这个过程对程序员透明。

可以试一下，即使不写Cookie，在使用request.getCookies();取出的Cookie数组的长度也是1，而这个Cookie的名字就是JSESSIONID，还有一个很长的二进制的字符串，是SessionID的值。

### 三:Cookie机制

在程序中，会话跟踪是很重要的事情。==**理论上，一个用户的所有请求操作都应该属于同一个会话，而另一个用户的所有请求操作则应该属于另一个会话，二者不能混淆。**==例如，用户A在超市购买的任何商品都应该放在A的购物车内，不论是用户A什么时间购买的，这都是属于同一个会话的，不能放入用户B或用户C的购物车内，这不属于同一个会话。

而Web应用程序是使用HTTP协议传输数据的。HTTP协议是无状态的协议。一旦数据交换完毕，客户端与服务器端的连接就会关闭，再次交换数据需要建立新的连接。这就意味着服务器无法从连接上跟踪会话。即用户A购买了一件商品放入购物车内，当再次购买商品时服务器已经无法判断该购买行为是属于用户A的会话还是用户B的会话了。要跟踪该会话，必须引入一种机制。

Cookie就是这样的一种机制。它可以弥补HTTP协议无状态的不足。在Session出现之前，基本上所有的网站都采用Cookie来跟踪会话。

##### (一) 什么是Cookie

Cookie实际上是一小段的文本信息。客户端请求服务器，如果服务器需要记录该用户状态，就使用response向客户端浏览器颁发一个Cookie。客户端浏览器会把Cookie保存起来。==***当浏览器再请求该网站时，浏览器把请求的网址连同该Cookie一同提交给服务器。***==服务器检查该Cookie，以此来辨认用户状态。服务器还可以根据需要修改Cookie的内容。

注意：Cookie功能需要浏览器的支持。

如果浏览器不支持Cookie（如大部分手机中的浏览器）或者把Cookie禁用了，Cookie功能就会失效。

不同的浏览器采用不同的方式保存Cookie。

IE浏览器会在“C:\Documents and Settings\你的用户名\Cookies”文件夹下以文本文件形式保存，一个文本文件保存一个Cookie。

##### ==(二) Cookie的不可跨域名性==

很多网站都会使用Cookie。例如，Google会向客户端颁发Cookie，Baidu也会向客户端颁发Cookie。那浏览器访问Google会不会也携带上Baidu颁发的Cookie呢？或者Google能不能修改Baidu颁发的Cookie呢？

答案是否定的。Cookie具有不可跨域名性。**根据Cookie规范，浏览器访问Google只会携带Google的Cookie，而不会携带Baidu的Cookie。Google也只能操作Google的Cookie，而不能操作Baidu的Cookie。**

Cookie在客户端是由浏览器来管理的。浏览器能够保证Google只会操作Google的Cookie而不会操作Baidu的Cookie，从而保证用户的隐私安全。浏览器判断一个网站是否能操作另一个网站Cookie的依据是域名。Google与Baidu的域名不一样，因此Google不能操作Baidu的Cookie。

需要注意的是，虽然网站images.google.com与网站www.google.com同属于Google，但是域名不一样，二者同样不能互相操作彼此的Cookie。



注意：用户登录网站www.google.com之后会发现访问images.google.com时登录信息仍然有效，而普通的Cookie是做不到的。这是因为Google做了特殊处理。本章后面也会对Cookie做类似的处理。



##### (三) 设置Cookie的所有属性

除了name与value之外，Cookie还具有其他几个常用的属性。每个属性对应一个getter方法与一个setter方法。Cookie类的所有属性如表1.1所示。

表1.1  Cookie常用属性

| 属  性  名        | 描    述                                                     |
| ----------------- | ------------------------------------------------------------ |
| String name       | 该Cookie的名称。Cookie一旦创建，名称便不可更改               |
| Object value      | 该Cookie的值。如果值为Unicode字符，需要为字符编码。如果值为二进制数据，则需要使用BASE64编码 |
| int maxAge        | 该Cookie失效的时间，单位秒。如果为正数，则该Cookie在maxAge秒之后失效。如果为负数，该Cookie为临时Cookie，关闭浏览器即失效，浏览器也不会以任何形式保存该Cookie。如果为0，表示删除该Cookie。默认为–1 |
| boolean secure    | 该Cookie是否仅被使用安全协议传输。安全协议。安全协议有HTTPS，SSL等，在网络上传输数据之前先将数据加密。默认为false |
| ==String path==   | 该Cookie的使用路径。如果设置为“/sessionWeb/”，则只有contextPath为“/sessionWeb”的程序可以访问该Cookie。如果设置为“/”，则本域名下contextPath都可以访问该Cookie。注意最后一个字符必须为“/” |
| ==String domain== | 可以访问该Cookie的域名。如果设置为“.google.com”，则所有以“google.com”结尾的域名都可以访问该Cookie。注意第一个字符必须为“.” |
| String comment    | 该Cookie的用处说明。浏览器显示Cookie信息的时候显示该说明     |
| int version       | 该Cookie使用的版本号。0表示遵循Netscape的Cookie规范，1表示遵循W3C的RFC 2109规范 |

##### (四)  Cookie的有效期

Cookie的maxAge决定着Cookie的有效期，单位为秒（Second）。Cookie中通过getMaxAge()方法与setMaxAge(int maxAge)方法来读写maxAge属性。

如果maxAge属性为正数，则表示该Cookie会在maxAge秒之后自动失效。浏览器会将maxAge为正数的Cookie持久化，即写到对应的Cookie文件中。无论客户关闭了浏览器还是电脑，只要还在maxAge秒之前，登录网站时该Cookie仍然有效。下面代码中的Cookie信息将永远有效。



Cookie cookie = new Cookie("username","helloweenvsfei");   // 新建Cookie

cookie.setMaxAge(Integer.MAX_VALUE);           // 设置生命周期为MAX_VALUE

response.addCookie(cookie);                    // 输出到客户端



如果maxAge为负数，则表示该Cookie仅在本浏览器窗口以及本窗口打开的子窗口内有效，关闭窗口后该Cookie即失效。maxAge为负数的Cookie，为临时性Cookie，不会被持久化，不会被写到Cookie文件中。Cookie信息保存在浏览器内存中，因此关闭浏览器该Cookie就消失了。Cookie默认的maxAge值为–1。

如果maxAge为0，则表示删除该Cookie。Cookie机制没有提供删除Cookie的方法，因此通过设置该Cookie即时失效实现删除Cookie的效果。失效的Cookie会被浏览器从Cookie文件或者内存中删除，



例如：

Cookie cookie = new Cookie("username","helloweenvsfei");   // 新建Cookie

cookie.setMaxAge(0);                          // 设置生命周期为0，不能为负数

response.addCookie(cookie);                    // 必须执行这一句



response对象提供的Cookie操作方法只有一个添加操作add(Cookie cookie)。

要想修改Cookie只能使用一个同名的Cookie来覆盖原来的Cookie，达到修改的目的。删除时只需要把maxAge修改为0即可。



注意：从客户端读取Cookie时，包括maxAge在内的其他属性都是不可读的，也不会被提交。浏览器提交Cookie时只会提交name与value属性。maxAge属性只被浏览器用来判断Cookie是否过期。



##### ==(五)  Cookie的域名==

Cookie是不可跨域名的。域名www.google.com颁发的Cookie不会被提交到域名www.baidu.com去。这是由Cookie的隐私安全机制决定的。隐私安全机制能够禁止网站非法获取其他网站的Cookie。

正常情况下，同一个一级域名下的两个二级域名如www.helloweenvsfei.com和images.helloweenvsfei.com也不能交互使用Cookie，因为二者的域名并不严格相同。如果想所有helloweenvsfei.com名下的二级域名都可以使用该Cookie，需要设置Cookie的domain参数，例如：

Cookie cookie = new Cookie("time","20080808"); // 新建Cookie

cookie.setDomain(".helloweenvsfei.com");           // 设置域名

cookie.setPath("/");                              // 设置路径

cookie.setMaxAge(Integer.MAX_VALUE);               // 设置有效期

response.addCookie(cookie);                       // 输出到客户端



读者可以修改本机C:\WINDOWS\system32\drivers\etc下的hosts文件来配置多个临时域名，然后使用setCookie.jsp程序来设置跨域名Cookie验证domain属性。

注意：domain参数必须以点(".")开始。另外，name相同但domain不同的两个Cookie是两个不同的Cookie。如果想要两个域名完全不同的网站共有Cookie，可以生成两个Cookie，domain属性分别为两个域名，输出到客户端。



##### (六) Cookie的路径

domain属性决定运行访问Cookie的域名，而path属性决定允许访问Cookie的路径（ContextPath）。例如，如果只允许/sessionWeb/下的程序使用Cookie，可以这么写：

Cookie cookie = new Cookie("time","20080808");     // 新建Cookie

cookie.setPath("/session/");                          // 设置路径

response.addCookie(cookie);                           // 输出到客户端

设置为“/”时允许所有路径使用Cookie。path属性需要使用符号“/”结尾。name相同但domain相同的两个Cookie也是两个不同的Cookie。



注意：页面只能获取它属于的Path的Cookie。例如/session/test/a.jsp不能获取到路径为/session/abc/的Cookie。使用时一定要注意。



##### (七)  Cookie的安全属性

HTTP协议不仅是无状态的，而且是不安全的。使用HTTP协议的数据不经过任何加密就直接在网络上传播，有被截获的可能。使用HTTP协议传输很机密的内容是一种隐患。如果不希望Cookie在HTTP等非安全协议中传输，可以设置Cookie的secure属性为true。浏览器只会在HTTPS和SSL等安全协议中传输此类Cookie。下面的代码设置secure属性为true：



Cookie cookie = new Cookie("time", "20080808"); // 新建Cookie

cookie.setSecure(true);                           // 设置安全属性

response.addCookie(cookie);                        // 输出到客户端



提示：secure属性并不能对Cookie内容加密，因而不能保证绝对的安全性。如果需要高安全性，需要在程序中对Cookie内容加密、解密，以防泄密。


### 四：Session与Cookie区别和联系：
Cookies是属于Session对象的一种。但有不同，Cookies不会占服务器资源，是存在客服端内存或者一个cookie的文本文件中；而“Session”则会占用服务器资源。所以，尽量不要使用Session，而使用Cookies。但是我们一般认为cookie是不可靠的，session是可靠地，但是目前很多著名的站点也都以来cookie。**有时候为了解决禁用cookie后的页面处理，通常采用url重写技术，调用session中大量有用的方法从session中获取数据后置入页面。**

Cookies与Session的应用场景： 
Cookies的安全性能一直是倍受争议的。虽然Cookies是保存在本机上的，但是其信息的完全可见性且易于本地编辑性，往往可以引起很多的安全问题。所以Cookies到底该不该用，到底该怎样用，就有了一个需要给定的底线。

先来看看，网站的敏感数据有哪些。

登陆验证信息。一般采用Session(“Logon”)＝true or false的形式。 
用户的各种私人信息，比如姓名等，某种情况下，需要保存在Session里 
需要在页面间传递的内容信息，比如调查工作需要分好几步。每一步的信息都保存在Session里，最后在统一更新到数据库。

当然还会有很多，这里列举一些比较典型的 
假如，一个人孤僻到不想碰Session，因为他认为，如果用户万一不小心关闭了浏览器，那么之前保存的数据就全部丢失了。所以，他出于好意，决定把这些用Session的地方，都改成用Cookies来存储，这完全是可行的，且基本操作和用Session一模一样。那么，下面就针对以上的3个典型例子，做一个分析 
很显然，只要某个有意非法入侵者，知道该网站验证登陆信息的Session变量是什么，那么他就可以事先编辑好该Cookies，放入到Cookies目录中，这样就可以顺利通过验证了。这是不是很可怕？ 
Cookies完全是可见的，即使程序员设定了Cookies的生存周期（比如只在用户会话有效期内有效），它也是不安全的。假设，用户忘了关浏览器 或者一个恶意者硬性把用户给打晕，那用户的损失将是巨大的。 
这点如上点一样，很容易被它人窃取重要的私人信息。但，其还有一个问题所在是，可能这些数据信息量太大，而使得Cookies的文件大小剧增。这可不是用户希望所看到的。

显然，Cookies并不是那么一块好啃的小甜饼。但，Cookies的存在，当然有其原因。它给予程序员更多发挥编程才能的空间。所以，使用Cookies改有个底线。这个底线一般来说，遵循以下原则。 
不要保存私人信息。 
任何重要数据，最好通过加密形式来保存数据（最简单的可以用URLEncode，当然也可以用完善的可逆加密方式，遗憾的是，最好不要用md5来加密）。 
是否保存登陆信息，需有用户自行选择。 
长于10K的数据，不要用到Cookies。 
也不要用Cookies来玩点让客户惊喜的小游戏。


