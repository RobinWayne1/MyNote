# Blog 11 Tomcat架构

### 一、前言
一般而言，对于一个复杂的系统，直接扎进去看源码会是很难受的，会浪费大量的时间和脑细胞，却得不到理想的效果。

这个时候，策略很重要，应该明白，越是复杂的东西，越会有良好的逻辑和层次，否则开发者自己估计过段时间怕也搞不清了。

tomcat是一个很大的系统，有复杂的结构，想要了解它，就应该顺着开发者设计之初的思路来，先了解整体的结构，对整体有了一定的掌控后，再逐个分析，了解感兴趣的细节。

### 二、Tomcat顶层结构
先上一张图：

![](https://img-blog.csdnimg.cn/20181029160418284.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3cxOTkyd2lzaGVz,size_27,color_FFFFFF,t_70)

上图大概展示了tomcat的结构，下面先简单介绍各个模块：

> Server：服务器的意思，代表整个tomcat服务器，一个tomcat只有一个Server；
> Service：Server中的一个逻辑功能层， 一个Server可以包含多个Service；
> Connector：称作连接器，是Service的核心组件之一，一个Service可以有多个Connector，主要是连接客户端请求；
> Container：Service的另一个核心组件，按照层级有Engine，Host，Context，Wrapper四种，一个Service只有一个Engine，其主要作用是执行业务逻辑；
> Jasper：JSP引擎；
> Session：会话管理；
> …

上面简单列了tomcat的模块结构，下面结合配置文件更加具体一点来分析，当然更多是集中在Connector和Container两个组件上，毕竟这是两个核心组件，后续的内容也会更多集中在这两个组件上面。

先将conf/server.xml配置文件内容贴出供参考（注释部分没有贴出）：



```xml
<?xml version="1.0" encoding="UTF-8"?>
<Server port="8005" shutdown="SHUTDOWN">
    <Listener className="org.apache.catalina.startup.VersionLoggerListener"/>
    <Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="on"/>
    <Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener"/>
    <Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener"/>
    <Listener className="org.apache.catalina.core.ThreadLocalLeakPreventionListener"/>
<GlobalNamingResources>
    <Resource name="UserDatabase" auth="Container"
              type="org.apache.catalina.UserDatabase"
              description="User database that can be updated and saved"
              factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
              pathname="conf/tomcat-users.xml"/>
</GlobalNamingResources>

<Service name="Catalina">
    <Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443"/>
    <Connector port="8009" protocol="AJP/1.3" redirectPort="8443"/>
    <Engine name="Catalina" defaultHost="localhost">
        <Realm className="org.apache.catalina.realm.LockOutRealm">
            <Realm className="org.apache.catalina.realm.UserDatabaseRealm"
                   resourceName="UserDatabase"/>
        </Realm>

        <Host name="localhost" appBase="webapps"
              unpackWARs="true" autoDeploy="true">
            <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
                   prefix="localhost_access_log" suffix=".txt"
                   pattern="%h %l %u %t &quot;%r&quot; %s %b"/>
        </Host>
    </Engine>
</Service>
</Server>
```


### 三、Server
Server是Tomcat最顶层的容器，代表着整个服务器，**即一个Tomcat只有一个Server，Server中包含至少一个Service组件，用于提供具体服务**。这个在配置文件中也得到很好的体现（port=“8005” shutdown="SHUTDOWN"是在8005端口监听到"SHUTDOWN"命令，服务器就会停止）。

Tomcat中其标准实现是：org.apache.catalina.core.StandardServer类，其继承结构类图如下：

![](https://img-blog.csdnimg.cn/20181029160430581.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3cxOTkyd2lzaGVz,size_27,color_FFFFFF,t_70)

StandardServer实现Server很好理解，tomcat为所有的组件都提供了生命周期管理，继承LifecycleMBeanBase则跟tomcat中的生命周期机制有关，后续文章会有介绍。

### 四、Service
可以想象，一个Server服务器，它最基本的功能肯定是：

> 接收客户端的请求，然后解析请求，完成相应的业务逻辑，然后把处理后的结果返回给客户端，一般会提供两个节本方法，一个start打开服务Socket连接，监听服务端口，一个stop停止服务释放网络资源。
>

这时的服务器就是一个Server类：

![](https://img-blog.csdnimg.cn/20181029160443697.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3cxOTkyd2lzaGVz,size_27,color_FFFFFF,t_70)

如何实现这个简单的服务器，看过《深入剖析tomcat》的应都知道，这部分代码之前也敲过，在github上（https://github.com/w1992wishes/tomcat-work），其实就是在一个端口上监听Socket请求，然后解析请求，返回处理结果。

**但如果将请求监听和请求处理放在一起，扩展性会变差，毕竟网络协议不止HTTP一种，如果想适配多种网络协议，请求处理又相同，这时就无能为力了**，tomcat的设计大师不会采取这种做法，而是将请求监听和请求处理分开为两个模块，分别是Connector和Container，Connector负责处理请求监听，Container负责处理请求处理。

但显然tomcat可以有多个Connector，同时Container也可以有多个。那这就存在一个问题，哪个Connector对应哪个Container，提供复杂的映射吗？相信看过server.xml文件的人已经知道了tomcat是怎么处理的了。

没错，Service就是这样来的。在conf/server.xml文件中，可以看到Service组件包含了Connector组件和Engine组件（前面有提过，Engine就是一种容器），==即Service相当于Connector和Engine组件的包装器，**将一个或者多个Connector和一个Engine建立关联关系。**==在默认的配置文件中，定义了一个叫Catalina 的服务，它将HTTP/1.1和AJP/1.3这两个Connector与一个名为Catalina 的Engine关联起来。

一个Server可以包含多个Service（它们相互独立，只是公用一个JVM及类库），一个Service负责维护多个Connector和一个Container。

其标准实现是StandardService，UML类图如下：

![](https://img-blog.csdnimg.cn/20181029160454106.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3cxOTkyd2lzaGVz,size_27,color_FFFFFF,t_70)

这时tomcat就是这样了：

![](https://img-blog.csdnimg.cn/20181029160504744.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3cxOTkyd2lzaGVz,size_27,color_FFFFFF,t_70)

### 五、Connector
前面介绍过Connector是连接器，用于接受请求并将请求封装成Request和Response，然后交给Container进行处理，Container处理完之后在交给Connector返回给客户端。

一个Connector会监听一个独立的端口来处理来自客户端的请求。server.xml默认配置了两个Connector：

<Connector port="8080" protocol="HTTP/1.1" connectionTimeout="20000" redirectPort="8443"/>，它监听端口8080,这个端口值可以修改，connectionTimeout定义了连接超时时间，单位是毫秒，redirectPort 定义了ssl的重定向接口，根据上述配置，Connector会将ssl请求转发到8443端口。
<Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />, AJP表示Apache Jserv Protocol，它将处理Tomcat和Apache http服务器之间的交互，此连接器用于处理我们将Tomcat和Apache http服务器结合使用的情况，如在同一台物理Server上部署一个Apache http服务器和多台Tomcat服务器，通过Apache服务器来处理静态资源以及负载均衡时，针对不同的Tomcat实例需要AJP监听不同的端口。
Connector在tomcat中的设计比较复杂，先大致列上一个图：

![](https://img-blog.csdnimg.cn/20181029160517904.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3cxOTkyd2lzaGVz,size_27,color_FFFFFF,t_70)

这里先简单介绍Connector，后续会详细分析。

==**Connector使用ProtocolHandler来处理请求的，不同的ProtocolHandler代表不同的连接类型，**==比如：Http11Protocol使用的是普通Socket来连接的（tomcat9已经删除了这个类，不再采用BIO的方式），**Http11NioProtocol使用的是NioSocket来连接的**。

其中ProtocolHandler由包含了三个部件：Endpoint、Processor、Adapter。

==**Endpoint用来处理底层Socket的网络连接，Processor用于将Endpoint接收到的Socket封装成Request（这个Request和ServletRequest无关），Adapter充当适配器，用于将Request转换为ServletRequest交给Container进行具体的处理。**==
**==Endpoint由于是处理底层的Socket网络连接，因此Endpoint是用来实现TCP/IP协议的，而Processor用来实现HTTP协议的，Adapter将请求适配到Servlet容器进行具体的处理。==**
Endpoint的抽象实现AbstractEndpoint里面定义的Acceptor和AsyncTimeout两个内部类和一个Handler接口。**Acceptor用于监听请求，AsyncTimeout用于检查异步Request的超时，Handler用于处理接收到的Socket，在内部调用Processor进行处理。**
先不用太纠结，了解Connector是做什么的就可以，后续再深入。

### 六、Container
tomcat的container层次如下：

![](https://img-blog.csdnimg.cn/20181029160528405.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3cxOTkyd2lzaGVz,size_27,color_FFFFFF,t_70)

#### 5.1、Engine
一个Service中有多个Connector和一个Engine，Engine表示整个Servlet引擎，一个Engine下面可以包含一个或者多个Host，即一个Tomcat实例可以配置多个虚拟主机，默认的情况下 conf/server.xml 配置文件中<Engine name="Catalina" defaultHost="localhost"> 定义了一个名为Catalina的Engine。

Engine的UML类图如下：

![](https://img-blog.csdnimg.cn/20181029160538363.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3cxOTkyd2lzaGVz,size_27,color_FFFFFF,t_70)

ContainerBase和LifecycleBase都是抽象出来的公共层。

#### 5.2、Host
Host，代表一个站点，也可以叫虚拟主机，一个Host可以配置多个Context，在server.xml文件中的默认配置为<Host name="localhost" appBase="webapps" unpackWARs="true" autoDeploy="true">, 其中appBase=webapps， 也就是<CATALINA_HOME>\webapps目录，unpackingWARS=true 属性指定在appBase指定的目录中的war包都自动的解压，autoDeploy=true 属性指定对加入到appBase目录的war包进行自动的部署。

一个Engine包含多个Host的设计，使得一个服务器实例可以承担多个域名的服务，是很灵活的设计。

其标准实现继承图如下：

![](https://img-blog.csdnimg.cn/20181029160550682.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3cxOTkyd2lzaGVz,size_27,color_FFFFFF,t_70)

#### 5.3、Context
Context，代表一个应用程序，就是日常开发中的web程序，或者一个WEB-INF目录以及下面的web.xml文件，换句话说每一个运行的webapp最终都是以Context的形式存在，每个Context都有一个根路径和请求路径；与Host的区别是Context代表一个应用，如，默认配置下webapps下的每个目录都是一个应用，其中ROOT目录中存放主应用，其他目录存放别的子应用，而整个webapps是一个站点。

在Tomcat中通常采用如下方式创建一个Context：

在<CATALINA_HOME>\webapps 目录中创建一个目录dirname，此时将自动创建一个context，默认context的访问url为http://host:port/dirname，也可以通过在ContextRoot\META-INF 中创建一个context.xml文件，其中包含如下内容来指定应用的访问路径：
在server.xml文件中增加context 元素，如下：<Context path="/urlpath" docBase="/test/xxx" reloadable=true />
这样就可以通过http://host:port/urlpath访问上面配置的应用。

可以打开tomcat目录对照一下：

![](https://img-blog.csdnimg.cn/20181029160600314.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3cxOTkyd2lzaGVz,size_27,color_FFFFFF,t_70)

其标准实现类图如下：

![](https://img-blog.csdnimg.cn/20181029160610324.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3cxOTkyd2lzaGVz,size_27,color_FFFFFF,t_70)

#### 5.4、Wrapper
**一个Context可以包含多个Servlet处理不同请求，当然现在的SpringMVC，struts框架的出现导致程序中不再是大量的Servlet，但其实本质是没变的，都是由Servlet来处理或者当作入口。**

在tomcat中Servlet被称为wrapper，其标准类图如下：

![](https://img-blog.csdnimg.cn/20181029160630561.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3cxOTkyd2lzaGVz,size_27,color_FFFFFF,t_70)

那么为什么要用Wrapper来表示Servlet？这和tomcat的处理机制有关，为了更加灵活，便于扩展，tomcat是用管道（pipeline）和阀(valve)的形式来处理请求，所以将Servlet丢给Wrapper。

Pipeline-Valve是责任链模式，**责任链模式是指在一个请求处理的过程中有很多处理者依次对请求进行处理，每个处理者负责做自己相应的处理，处理完之后将处理后的请求返回，再让下一个处理着继续处理。**

![img](https://mmbiz.qpic.cn/mmbiz_png/UtWdDgynLdZlz44rysyrVpFqrW1Yc0eV05rnYJouVYYVYRA1HecPAQq665tNOc8ScURxoGHuFcADkk8SwVhcBA/640?tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

但是！Pipeline-Valve使用的责任链模式和普通的责任链模式有些不同！区别主要有以下两点：

（1）每个Pipeline都有特定的Valve，而且是在管道的最后一个执行，这个Valve叫做BaseValve，BaseValve是不可删除的；

（2）**在上层容器的管道的BaseValve中会调用下层容器的管道**。

我们知道Container包含四个子容器，而这四个子容器对应的Base Valve分别在：StandardEngineValve、StandardHostValve、StandardContextValve、StandardWrapperValve。

Pipeline的处理流程图如下（图D）：

![img](https://mmbiz.qpic.cn/mmbiz_png/UtWdDgynLdbS6ribibqjHPC195DmZX0NBvfyQ2Dk9X4pQiakBQ7XYtDjQwUZpXqRibYzjRmHVRecm4epZW0d7ibITvg/640?tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

（1）Connector在接收到请求后会首先调用最顶层容器的Pipeline来处理，这里的最顶层容器的Pipeline就是EnginePipeline（Engine的管道）；

（2）在Engine的管道中依次会执行EngineValve1、EngineValve2等等，最后会执行StandardEngineValve，在StandardEngineValve中会调用Host管道，然后再依次执行Host的HostValve1、HostValve2等，最后在执行StandardHostValve，然后再依次调用Context的管道和Wrapper的管道，最后执行到StandardWrapperValve。

（3）==**当执行到StandardWrapperValve的时候，会在StandardWrapperValve中创建FilterChain，并调用其doFilter方法来处理请求，这个FilterChain包含着我们配置的与请求相匹配的Filter和Servlet，其doFilter方法会依次调用所有的Filter的doFilter方法和Servlet的service方法，这样请求就得到了处理！**==

（4）当所有的Pipeline-Valve都执行完之后，并且处理完了具体的请求，这个时候就可以将返回的结果交给Connector了，Connector在通过Socket的方式将结果返回给客户端。

那么现在tomcat就是这样的：

![](https://img-blog.csdnimg.cn/20181029160643151.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3cxOTkyd2lzaGVz,size_27,color_FFFFFF,t_70)



