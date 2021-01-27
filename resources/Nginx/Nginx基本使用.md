# Nginx基本使用

### 一、反向代理

正向代理服务器位于客户端和服务器之间，为了向服务器获取数据，客户端要向代理服务器发送一个请求，并指定目标服务器，代理服务器将目标服务器返回的数据转交给客户端，这就是正向代理。

在反向代理中，客户端对代理是无感知的，我们只需要将请求发送到反向代理服务器，由反向代理服务器去选择目标服务器获取数据后，在返回给客户端，此时反向代理服务器和目标服务器对外就是一个服务器，暴露的是代理服务器地址，隐藏了真实服务器IP地址。此时就可以在反向代理服务器做一些负载均衡等操作，并且隐藏了真正服务器的IP地址，保证安全性。

#### Ⅰ、Nginx反向代理的配置

```java
server {
    listen 80;
    server_name www.subtitlesearch.com;
    #为了配置主页。因为做了全动静态资源分离，所以后端无法访问静态资源
   location ~ /$ {
    #配置静态资源根路径，具体访问的时候是要在加上location中的模式的，这点和alias区分开
     root D:/SubtitleSearchResource;
    #指定默认主页
     index index.html;
}		
    location / {
        proxy_pass http://127.0.0.1:8090;
        proxy_connect_timeout 600;
	
    }
    #静态资源配置
 location ~ (.jpg|.mp4|.js|.html|.css)$ {
       		 #配置静态资源根路径，具体访问的时候是要在加上location中的模式的，这点和alias区分开
		root D:/SubtitleSearchResource;
		
		expires 3d;
	}
}
```

这是我的项目中的其中一部分配置。虚拟主机的部署通过被包含在`http`模块中的`server`块中实现,一个`server`就代表着一台虚拟主机，其中`listen`表示Nginx监听的端口,`server_name`要填写的是本机的IP地址或者是域名都可以。

而`location`模块编写的则是根据不同的请求路径从而可以进行不同的请求分派,是动静态资源分离的基础。其中有几个主要的参数：

* `location`后编写的是请求的格式,还支持正则表达式匹配,想要开启正则表达式则加一个`~`即可。要注意拦截的请求模式要和`{`之间有空格,不然报错.

* `proxy_pass`则表示的是代理的那个具体服务器的地址
* `proxy_connect_timeout`表示连接超时的时间。
* `root`代表请求实际在Nginx服务器中的路径,实际的访问路径则是将location模式之前的URL替换成root。
* 而`alias`也代表请求实际在Nginx服务器中的路径,但是`alias`是将location模式之前的URL替换成`alias`,并将`location`模式本身删除之后才是其实际路径,也就是实际路径不包含`location`模式本身
* `index`则代表若URL没有访问某个具体的文件时,Nginx所要返回的默认页面,如`/`就没有请求实际文件,则返回`index`所指定的`index.html`

#### Ⅱ、多虚拟主机部署

根据Nginx的配置，可以达到虽然是同一个Nginx服务器，但是可以根据不同的域名、请求的IP或者是端口从而将请求转发给相应的后台服务器从而提供完全不同的服务，这就是虚拟主机的作用。

```java
  server {
        listen       80;
       server_name  www.abc.com;

        location / {
            root   E:/;
            autoindex on;
        }
    }
server {
    listen 80;
    server_name www.subtitlesearch.com;
    #为了配置主页。因为做了全动静态资源分离，所以后端无法访问静态资源
   location ~ /$ {
    #配置静态资源根路径，具体访问的时候是要在加上location中的temp的，这点和alias区分开
     root D:/SubtitleSearchResource;
    #指定默认主页
     index index.html;
	}     
}
```

其中**基于端口的配置**则是简单的在`listen`中配置不同的端口,客户端在访问时指定不同的端口从而获取不同的服务。

而我觉得比较有意思的是**基于域名的虚拟主机的配置**，通过在`server_name`中配置不同的域名,虽然它们都监听着相同同的端口,但是只要客户端请求的域名不相同,最终Nginx也能够将请求分派到不同的后台服务器中。(新版本好像用不了了？？？)

#### Ⅲ、动静态资源分离

其实很简单，只要分别做一个动态资源正则和静态资源正则模式的`location`,动态资源`location`中将动态资源发给具体的后台服务器,而静态资源`location`则访问Nginx服务器中某个具体的文件夹即可.

### 二、负载均衡

负载均衡没时间做具体的测试，就搬网上的例子吧。

```java
http{
#动态服务器组
    upstream load_balancing {
    	ip_hash;
        server localhost:8080;  #tomcat 7.0
        server localhost:8081;  #tomcat 8.0
        server localhost:8082;  #tomcat 8.5
        server localhost:8083;  #tomcat 9.0
    }
    server{
        listen 80;
        server_name 127.0.0.1;
        #其他页面反向代理到tomcat容器
        location ~ .*$ {
            index index.jsp index.html;
            proxy_pass http://load_balancing;
        }
        
    }

}
```

`upstream`放在http模块下,代表负载均衡的配置。`ip_hash`作为其负载均衡的策略,`server`则表示请求分派的各个后台服务器。最后在`location`的`proxy_pass`,将转发的地址改为`upstream`名字就可以实现负载均衡。

#### Ⅰ、轮询

最基本的配置方法，上面的例子就是轮询的方式，它是`upstream`模块默认的负载均衡默认策略。每个请求会按时间顺序**逐一分配**到不同的后端服务器。

`upstream`中的`server`参数:

| 参数         | 作用                                                         |
| ------------ | ------------------------------------------------------------ |
| fail_timeout | 在经历了max_fails次失败后，暂停server服务的时间。max_fails可以和fail_timeout一起使用。 |
| max_fails    | 允许请求失败的次数，默认为1。当超过最大次数时，返回proxy_next_upstream 模块定义的错误。 |
| backup       | 标记该服务器为备用服务器。当主服务器停止时，请求会被发送到它这里。 |
| down         | 标记服务器永久停机了。此时该服务器不参与进负载均衡.          |

#### Ⅱ、权重

权重方式，在轮询策略的基础上指定轮询的几率。

```
#动态服务器组
    upstream dynamic_zuoyu {
        server localhost:8080   weight=2;  #tomcat 7.0
        server localhost:8081;  #tomcat 8.0
        server localhost:8082   backup;  #tomcat 8.5
        server localhost:8083   max_fails=3 fail_timeout=20s;  #tomcat 9.0
    }
```

`weight`默认是1,所以此时tomcat7的访问几率是其他服务器的两倍。此策略比较适合服务器的硬件配置差别比较大的情况。

#### Ⅲ、ip_hash

指定负载均衡器按照基于客户端IP的分配方式，这个方法确保了相同的客户端的请求一直发送到相同的服务器，以保证session会话。**这样每个访客都固定访问一个后端服务器，可以解决session不能跨服务器的问题。**

```
#动态服务器组
    upstream dynamic_zuoyu {
        ip_hash;    #保证每个访客固定访问一个后端服务器
        server localhost:8080   weight=2;  #tomcat 7.0
        server localhost:8081;  #tomcat 8.0
        server localhost:8082;  #tomcat 8.5
        server localhost:8083   max_fails=3 fail_timeout=20s;  #tomcat 9.0
    }
```

- ip_hash不能与backup同时使用。
- 此策略适合有状态服务，比如session。

#### Ⅳ、least_conn

把请求转发给连接数较少的后端服务器。轮询算法是把请求平均的转发给各个后端，使它们的负载大致相同；但是，有些请求占用的时间很长，会导致其所在的后端负载较高。这种情况下，least_conn这种方式就可以达到更好的负载均衡效果。

```
#动态服务器组
    upstream dynamic_zuoyu {
        least_conn;    #把请求转发给连接数较少的后端服务器
        server localhost:8080   weight=2;  #tomcat 7.0
        server localhost:8081;  #tomcat 8.0
        server localhost:8082 backup;  #tomcat 8.5
        server localhost:8083   max_fails=3 fail_timeout=20s;  #tomcat 9.0
    }
```

#### Ⅴ、fair

第三方负载均衡策略。按照服务器端的响应时间来分配请求，响应时间短的优先分配。

```
#动态服务器组
    upstream dynamic_zuoyu {
        server localhost:8080;  #tomcat 7.0
        server localhost:8081;  #tomcat 8.0
        server localhost:8082;  #tomcat 8.5
        server localhost:8083;  #tomcat 9.0
        fair;    #实现响应时间短的优先分配
    }
```

#### Ⅵ、url_hash

按访问url的hash结果来分配请求，使每个url定向到同一个后端服务器，要配合服务器本地缓存来使用。同一个资源多次请求，可能会到达不同的服务器上，导致不必要的多次下载，缓存命中率不高，以及一些资源时间的浪费。而使用url_hash，可以使得同一个url（也就是同一个资源请求）会到达同一台服务器，一旦缓存住了资源，再此收到请求，就可以从缓存中读取。

```
#动态服务器组
    upstream dynamic_zuoyu {
        hash $request_uri;    #实现每个url定向到同一个后端服务器
        server localhost:8080;  #tomcat 7.0
        server localhost:8081;  #tomcat 8.0
        server localhost:8082;  #tomcat 8.5
        server localhost:8083;  #tomcat 9.0
    }
```

