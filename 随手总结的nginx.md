# nginx
## 反向代理
代理是在服务器和客户端之间假设的一层服务器，代理将接收客户端的请求并将它转发给服务器，然后将服务端的响应转发给客户端。

### 正向代理

> 正向代理，意思是一个位于客户端和原始服务器(origin server)之间的服务器，为了从原始服务器取得内容，客户端向代理发送一个请求并指定目标(原始服务器)，然后代理向原始服务器转交请求并将获得的内容返回给客户端。

正向代理是为我们服务的，即为客户端服务的，客户端可以根据正向代理访问到它本身无法访问到的服务器资源。
正向代理对我们是透明的，对服务端是非透明的，即服务端并不知道自己收到的是来自代理的访问还是来自真实客户端的访问。

翻墙工具其实就是一个正向代理工具

### 反向代理

> 反向代理（Reverse Proxy）方式是指以代理服务器来接受internet上的连接请求，然后将请求转发给内部网络上的服务器，并将从服务器上得到的结果返回给internet上请求连接的客户端，此时代理服务器对外就表现为一个反向代理服务器。

反向代理是为服务端服务的，反向代理可以帮助服务器接收来自客户端的请求，帮助服务器做请求转发，负载均衡等。
反向代理对服务端是透明的，对我们是非透明的，即我们并不知道自己访问的是代理服务器，而服务器知道反向代理在为他服务。

#### 跨域
当前端域名与后端域名不一致的时候导致请求失败。我们可以通过配置Nginx反向代理来解决。

```
location /api {   
    # 请求host传给后端
    proxy_set_header Host $http_host;
    # 请求ip 传给后端
    proxy_set_header X-Real-IP $remote_addr;
    # 请求协议传给后端
    proxy_set_header X-Scheme $scheme;
    # 路径重写
    rewrite  /api/(.*)  /$1  break;
    # 代理服务器
    proxy_pass http://localhost:9000;
}

```


```
    #请求跨域，这里约定代理请求url path是以/apis/开头
    location ^~/apis/ {
        # 这里重写了请求，将正则匹配中的第一个()中$1的path，拼接到真正的请求后面，并用break停止后续匹配
        rewrite ^/apis/(.*)$ /$1 break;
        proxy_pass https://www.kaola.com/;
    }  

```


## 基本配置
### 内置全局变量

```
$args ： #这个变量等于请求行中的参数，同$query_string
$content_length ： 请求头中的Content-length字段。
$content_type ： 请求头中的Content-Type字段。
$document_root ： 当前请求在root指令中指定的值。
$host ： 请求主机头字段，否则为服务器名称。
$http_user_agent ： 客户端agent信息
$http_cookie ： 客户端cookie信息
$limit_rate ： 这个变量可以限制连接速率。
$request_method ： 客户端请求的动作，通常为GET或POST。
$remote_addr ： 客户端的IP地址。
$remote_port ： 客户端的端口。
$remote_user ： 已经经过Auth Basic Module验证的用户名。
$request_filename ： 当前请求的文件路径，由root或alias指令与URI请求生成。
$scheme ： HTTP方法（如http，https）。
$server_protocol ： 请求使用的协议，通常是HTTP/1.0或HTTP/1.1。
$server_addr ： 服务器地址，在完成一次系统调用后可以确定这个值。
$server_name ： 服务器名称。
$server_port ： 请求到达服务器的端口号。
$request_uri ： 包含请求参数的原始URI，不包含主机名，如：”/foo/bar.php?arg=baz”。
$uri ： 不带请求参数的当前URI，$uri不包含主机名，如”/foo/bar.html”。
$document_uri ： 与$uri相同
```

### 配置结构

```
# nginx.conf
# main
events { 

}

http 
{
    upstream {
        
    }
    
    server { 
        location path
        {
            ...
        }
        location path
        {
            ...
        }
    }

    server {
        ...
    }

}

```
- main:nginx的全局配置，对全局生效。
- events:配置影响nginx服务器或与用户的网络连接。
- http：可以嵌套多个server，配置代理，缓存，日志定义等绝大多数功能和第三方模块的配置。
- server：配置虚拟主机的相关参数，一个http中可以有多个server。
- location：配置请求的路由，以及各种页面的处理情况。
- upstream：配置后端服务器具体地址，负载均衡配置不可或缺的部分。

## 动态匹配

```
location = / {
    [ configuration A ]
}

location / {
    [ configuration B ]
}

location /documents/ {
    [ configuration C ]
}

location ^~ /images/ {
    [ configuration D ]
}

location ~* \.(gif|jpg|jpeg)$ {
    [ configuration E ]
}
```


= 表示精确匹配。只有请求的url路径与后面的字符串完全相等时，才会命中（优先级最高）。
^~ 表示如果该符号后面的字符是最佳匹配，采用该规则，不再进行后续的查找。
~ 表示该规则是使用正则定义的，区分大小写。
~* 表示该规则是使用正则定义的，不区分大小写。


### 优先级：
(location =) > (location 完整路径) > (location ^~ 路径) > (location ~,~* 正则顺序) > (location 部分起始路径) > (/)


### 常见配置

```
所以实际使用中，个人觉得至少有三个匹配规则定义，如下：
#直接匹配网站根，通过域名访问网站首页比较频繁，使用这个会加速处理，官网如是说。
#这里是直接转发给后端应用服务器了，也可以是一个静态首页
# 第一个必选规则
location = / {
    proxy_pass http://tomcat:8080/index
}
# 第二个必选规则是处理静态文件请求，这是nginx作为http服务器的强项
# 有两种配置模式，目录匹配或后缀匹配,任选其一或搭配使用
location ^~ /static/ {
    root /webroot/static/;
}
location ~* \.(gif|jpg|jpeg|png|css|js|ico)$ {
    root /webroot/res/;
}
#第三个规则就是通用规则，用来转发动态请求到后端应用服务器
#非静态文件请求就默认是动态请求，自己根据实际把握
#毕竟目前的一些框架的流行，带.php,.jsp后缀的情况很少了
location / {
    proxy_pass http://tomcat:8080/
}
```

#### try_files

```
location / {
    try_files $uri $uri/ /index.php?$query_string;
}
```
当用户请求 http://localhost/example 时，这里的 $uri 就是 /example。 
try_files 会到硬盘里尝试找这个文件。如果存在名为 /$root/example（其中 $root 是项目代码安装目录）的文件，就直接把这个文件的内容发送给用户。 
显然，目录中没有叫 example 的文件。然后就看 $uri/，增加了一个 /，也就是看有没有名为 /$root/example/ 的目录。 
又找不到，就会 fall back 到 try_files 的最后一个选项 /index.php，发起一个内部 “子请求”，也就是相当于 nginx 发起一个 HTTP 请求到 http://localhost/index.php。
#### rewrite
语法rewrite regex replacement [flag];

表明看rewrite和location功能有点像，都能实现跳转，主要区别在于rewrite是在同一域名内更改获取资源的路径，而location是对一类路径做控制访问或反向代理，可以proxy_pass到其他机器。很多情况下rewrite也会写在location里，它们的执行顺序是：

执行server块的rewrite指令
执行location匹配
执行选定的location中的rewrite指令


##### flag标志位
- last : 相当于Apache的[L]标记，表示完成rewrite
- break : 停止执行当前虚拟主机的后续rewrite指令集
- redirect : 返回302临时重定向，地址栏会显示跳转后的地址
- permanent : 返回301永久重定向，地址栏会显示跳转后的地址
因为301和302不能简单的只返回状态码，还必须有重定向的URL，这就是return指令无法返回301,302的原因了。这里 last 和 break 区别有点难以理解：

last一般写在server和if中，而break一般使用在location中
last不终止重写后的url匹配，即新的url会再从server走一遍匹配流程，而break终止重写后的匹配
break和last都能组织继续执行后面的rewrite指令

## Gzip 压缩-提高页面加载速度
### 优点
开启Gzip功能后，Nginx服务器会根据配置的策略对发送的内容, 如css、js、xml、html等静态资源进行压缩, 使得这些内容大小减少，在用户接收到返回内容之前对其进行处理，以压缩后的数据展现给客户; 
### 缺点
会消耗一定的cpu资源。
经过Gzip压缩后页面大小可以变为原来的30%甚至更小，这样，用户浏览页面的时候速度会快得多。
### 使用场景
Gzip 的压缩页面**需要浏览器和服务器双方都支持**，实际上就是服务器端压缩，传到浏览器后浏览器解压并解析

Nginx的Gzip压缩功能虽然好用，但是下面两类文件资源不太建议启用此压缩功能。

1) 图片类型资源 (还有视频文件)

原因：图片如jpg、png文件本身就会有压缩，所以就算开启gzip后，压缩前和压缩后大小没有多大区别，所以开启了反而会白白的浪费资源。（可以试试将一张jpg图片压缩为zip，观察大小并没有多大的变化。虽然zip和gzip算法不一样，但是可以看出压缩图片的价值并不大）
2) 大文件资源
原因：会消耗大量的cpu资源，且不一定有明显的效果。

### 配置

```
server {
    # 开启gzip 压缩
    gzip on;
    # 设置gzip所需的http协议最低版本 （HTTP/1.1, HTTP/1.0）
    gzip_http_version 1.1;
    # 设置压缩级别，压缩级别越高压缩时间越长（1-9）
    gzip_comp_level 4;
    # 设置压缩的最小字节数， 页面Content-Length获取，建议设置1000以上
    gzip_min_length 1000;
    # 设置压缩文件的类型(MIME类型)（默认text/html)(默认不压缩js/css)
    gzip_types text/plain application/javascript text/css;
}

```
#### gzip_http_version

启用 GZip 所需的HTTP 最低版本，
默认值为HTTP/1.1

##### 为什么默认版本为什么是1.1

HTTP/1.1默认支持TCP持久连接，HTTP/1.0 也可以通过显式指定 Connection: keep-alive 来启用持久连接。

对于TCP持久连接上的HTTP 报文，客户端需要一种机制来准确判断结束位置

在 HTTP/1.0中，这种机制只有Content-Length，而在HTTP/1.1中新增的 Transfer-Encoding: chunked 所对应的分块传输机制可以完美解决这类问题。

nginx同样有着配置chunked的属性chunked_transfer_encoding，这个属性是默认开启的。

Nginx在启用了GZip的情况下，不会等文件 GZip 完成再返回响应，而是边压缩边响应，这样可以显著提高 TTFB(Time To First Byte，首字节时间，WEB 性能优化重要指标)。这样唯一的问题是，Nginx 开始返回响应时，它无法知道将要传输的文件最终有多大，也就是**无法给出Content-Length这个响应头部**。

所以，**在HTTP1.0中如果利用Nginx启用了GZip，是无法获得Content-Length的，这导致HTTP1.0中开启持久链接和使用GZip只能二选一**，所以gzip_http_version默认设置为1.1。


## 负载均衡
nginx如何实现负载均衡
- Upstream指定后端服务器地址列表

```
upstream balanceServer {
    server 10.1.22.33:12345;
    server 10.1.22.34:12345;
    server 10.1.22.35:12345;
}
```

- 在server中拦截响应请求，并将请求转发到Upstream中配置的服务器列表。

```
server {
        server_name  fe.server.com;
        listen 80;
        location /api {
            proxy_pass http://balanceServer;
        }
    }
```
- 指定分配策略
##### 轮询
将所有客户端请求轮询分配给服务端

```
upstream balanceServer {
    server 10.1.22.33:12345;
    server 10.1.22.34:12345;
    server 10.1.22.35:12345;
}

```

##### 最小连接数
将请求优先分配给压力较小的服务器

```
upstream balanceServer {
    least_conn;
    server 10.1.22.33:12345;
    server 10.1.22.34:12345;
    server 10.1.22.35:12345;
}

```

##### 最快响应时间
依赖于NGINX Plus，优先分配给响应时间最短的服务器

```
upstream balanceServer {
    least_time header;
    server 10.1.22.33:12345;
    server 10.1.22.34:12345;
    server 10.1.22.35:12345;
}

```

##### 客户端ip绑定
来自同一个ip的请求永远只分配一台服务器，有效解决了动态网页存在的session共享问题
    
```
upstream balanceServer {
    ip_hash;
    server 10.1.22.33:12345;
    server 10.1.22.34:12345;
    server 10.1.22.35:12345;
}
```



##### 备份服务器
当主服务器不可用时，将传递与备份服务器的连接。

```
upstream backend {
    server 127.0.0.1:3000 backup;
    server 127.0.0.1:3001;
}

```
##### 权重

```
upstream backend {
    server 127.0.0.1:3000 weight=2;
    server 127.0.0.1:3001 weight=1;
}

```


## 使用场景
### 解决跨域

### 简单的访问限制

### 适配PC与移动环境

### 合并请求

### 图片处理
## 参考文章
https://juejin.im/post/5bacbd395188255c8d0fd4b2

https://juejin.im/post/5cae9de95188251ae2324ec3


https://juejin.im/post/5c85a64d6fb9a04a0e2e038c#heading-1


https://juejin.im/post/5d81906c518825300a3ec7ca

https://segmentfault.com/a/1190000002797606#articleHeader1
