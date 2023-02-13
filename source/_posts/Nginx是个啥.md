---
title: Nginx是个啥
tags:
  - Nginx
typora-root-url: ../../themes/butterfly/source
date: 2020-11-01 15:07:46
description: 在我现在的公司Nginx是归前端管的，所以现在对这个的了解不多。但是目前负责的项目从MVC升级Boot的时候遇到了点问题……
cover: /blogImg/反向代理.jpg
categories: Nginx
---

事情的起因是之前在MVC项目升级Boot的时候，有的接口调用发生了如下报错：

```
2019-11-19/16:43:50.672|10.27.0.103|-|-|^_^|[http-nio-8080-exec-3] INFO  org.apache.coyote.http11.Http11Processor 175 - The host [payroll_vipkid_com_cn] is not valid
 Note: further occurrences of request parsing errors will be logged at DEBUG level.
java.lang.IllegalArgumentException: The character [_] is never valid in a domain name.
        at org.apache.tomcat.util.http.parser.HttpParser$DomainParseState.next(HttpParser.java:963)
        at org.apache.tomcat.util.http.parser.HttpParser.readHostDomainName(HttpParser.java:859)
        at org.apache.tomcat.util.http.parser.Host.parse(Host.java:71)
        at org.apache.tomcat.util.http.parser.Host.parse(Host.java:45)
        at org.apache.coyote.AbstractProcessor.parseHost(AbstractProcessor.java:289)
        at org.apache.coyote.http11.Http11Processor.prepareRequest(Http11Processor.java:807)
```

经过一番原因寻找，发现是因为公司的框架升级，所用的spring boot2中的tomcat版本已经升级到了8.5.+，严格限制了host header的格式，不识别下划线，但是公司线上环境会自动增加gslb_开头，联系运维在nginx中location 填加 proxy_set_header Host $host;之后就正常了。那么问题来了，为什么加了这段配置就正常了呢？Nginx又到底是干嘛的？

# Nginx

Nginx是一款轻量级的Web服务器、反向代理服务器，由于它的内存占用少，启动极快，高并发能力强，在互联网项目中广泛应用。

## **特点：**

- 更快：
	- 单次请求会得到更快的响应。
	- 在高并发环境下，Nginx 比其他 WEB 服务器有更快的响应。
- 高扩展性：
	- Nginx 是基于模块化设计，由多个耦合度极低的模块组成，因此具有很高的扩展性。许多高流量的网站都倾向于开发符合自己业务特性的定制模块。
- 高可靠性：
	- Nginx 的可靠性来自于其核心框架代码的优秀设计，模块设计的简单性。另外，官方提供的常用模块都非常稳定，每个 worker 进程相对独立，master 进程在一个 worker 进程出错时可以快速拉起新的 worker 子进程提供服务。
- 低内存消耗：
	- 一般情况下，10000个非活跃的 `HTTP Keep-Alive` 连接在 Nginx 中仅消耗 `2.5MB` 的内存，这是 Nginx 支持高并发连接的基础。
	- 单机支持10万以上的并发连接：**理论上，Nginx 支持的并发连接上限取决于内存，10万远未封顶。**
- 热部署:
	- master 进程与 worker 进程的分离设计，使得 Nginx 能够提供热部署功能，即在 7x24 小时不间断服务的前提下，升级 Nginx 的可执行文件。当然，它也支持不停止服务就更新配置项，更换日志文件等功能。
- 最自由的 BSD 许可协议:
	- 这是 Nginx 可以快速发展的强大动力。BSD 许可协议不只是允许用户免费使用 Nginx ，它还允许用户在自己的项目中直接使用或修改 Nginx 源码，然后发布。

## 反向代理和正向代理

![正向代理和反向代理](/blogImg/正向代理和反向代理.png)

### 正向代理

由于防火墙的原因，我们并不能直接访问谷歌，那么我们可以借助VPN来实现，这就是一个简单的正向代理的例子。这里你能够发现，正向代理“代理”的是客户端，而且客户端是知道目标的，而目标是不知道客户端是通过VPN访问的。

正向代理最大的特点是客户端非常明确要访问的服务器地址；服务器只清楚请求来自哪个代理服务器，而不清楚来自哪个具体的客户端；正向代理模式屏蔽或者隐藏了真实客户端信息

正向代理的用途：

- 访问原来无法访问的资源，如Google
- 可以做缓存，加速访问资源
- 对客户端访问授权，上网进行认证
- 代理可以记录用户访问记录（上网行为管理），对外隐藏用户信息

![正向代理](/blogImg/正向代理.jpg)

### 反向代理

当我们在外网访问百度的时候，其实会进行一个转发，代理到内网去，这就是所谓的反向代理，即反向代理“代理”的是服务器端，而且这一个过程对于客户端而言是透明的，访问者并不知道自己访问的是一个代理。因为客户端不需要任何配置就可以访问。反向代理，"它代理的是服务端，代服务端接收请求"，主要用于服务器集群分布式部署的情况下，反向代理隐藏了服务器的信息。

反向代理的作用：

- 保证内网的安全，通常将反向代理作为公网访问地址，Web服务器是内网
- 负载均衡，通过反向代理服务器来优化网站的负载

![反向代理](/blogImg/反向代理.jpg)

## 负载均衡

我们已经明确了所谓代理服务器的概念，那么接下来，Nginx扮演了反向代理服务器的角色，它是以依据什么样的规则进行请求分发的呢？不用的项目应用场景，分发的规则是否可以控制呢？

这里提到的客户端发送的、Nginx反向代理服务器接收到的请求数量，就是我们说的负载量。

请求数量按照一定的规则进行分发到不同的服务器处理的规则，就是一种均衡规则。

所以，将服务器接收到的请求按照规则分发的过程，称为负载均衡。

负载均衡在实际项目操作过程中，有硬件负载均衡和软件负载均衡两种，硬件负载均衡也称为硬负载，如F5负载均衡，相对造价昂贵成本较高，但是数据的稳定性安全性等等有非常好的保障，如中国移动中国联通这样的公司才会选择硬负载进行操作；更多的公司考虑到成本原因，会选择使用软件负载均衡，软件负载均衡是利用现有的技术结合主机硬件实现的一种消息队列分发机制。

Nginx支持的负载均衡调度算法方式如下：

1. weight轮询（默认，常用，具有HA功效！）：接收到的请求按照权重分配到不同的后端服务器，即使在使用过程中，某一台后端服务器宕机，Nginx会自动将该服务器剔除出队列，请求受理情况不会受到任何影响。 这种方式下，可以给不同的后端服务器设置一个权重值(weight)，用于调整不同的服务器上请求的分配率；权重数据越大，被分配到请求的几率越大；该权重值，主要是针对实际工作环境中不同的后端服务器硬件配置进行调整的。
2. ip_hash（常用）：每个请求按访问ip的hash结果分配，这样每个访客固定访问一个后端服务器，这也在一定程度上解决了集群部署环境下session共享的问题。
3. fair：智能调整调度算法，动态的根据后端服务器的请求处理到响应的时间进行均衡分配，响应时间短处理效率高的服务器分配到请求的概率高，响应时间长处理效率低的服务器分配到的请求少；结合了前两者的优点的一种调度算法。但是需要注意的是Nginx默认不支持fair算法，如果要使用这种调度算法，请安装upstream_fair模块。
4. url_hash：按照访问的url的hash结果分配请求，每个请求的url会指向后端固定的某个服务器，可以在Nginx作为静态服务器的情况下提高缓存效率。同样要注意Nginx默认不支持这种调度算法，要使用的话需要安装Nginx的hash软件包。

## Nginx配置

nginx本身作为一个完成度非常高的负载均衡框架，和很多成熟的开源框架一样，大多数功能都可以通过修改配置文件来完成，使用者只需要简单修改一下nginx配置文件，便可以非常轻松的实现比如反向代理，负载均衡这些常用的功能，同样的，和其他开源框架比如tomcat一样，nginx配置文件也遵循着相应的格式规范，并不能一顿乱配，在讲解如何使用nginx实现反向代理，负载均衡等这些功能的配置前，我们需要先了解一下nginx配置文件的结构。

既然要了解nginx的配置文件，那我总得知道nginx配置文件在哪啊，nginx配置文件默认都放在nginx安装路径下的conf目录，而主配置文件nginx.conf自然也在这里面，我们下面的操作几乎都是对nginx.conf这个配置文件进行修改。

可是，我怎么知道我nginx装哪了？我要是不知道nginx装哪了咋办？

这个，细心的朋友们可能会发现，运行nginx -t命令，下面除了给出nginx配置文件是否OK外，同时也包括了配置文件的路径。诺，就是这个

```shell
[root@izuf61d3ovm5vx1kknakwrz ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

使用vim打开该配置文件，我们一探究竟，不同版本的配置文件可能稍有不同，我的配置文件内容如下：

```
# For more information on configuration, see:
#   * Official English Documentation: http://nginx.org/en/docs/
#   * Official Russian Documentation: http://nginx.org/ru/docs/

user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

# Load dynamic modules. See /usr/share/doc/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    # Load modular configuration files from the /etc/nginx/conf.d directory.
    # See http://nginx.org/en/docs/ngx_core_module.html#include
    # for more information.
    include /etc/nginx/conf.d/*.conf;

    server {
        listen       80 default_server;
        listen       [::]:80 default_server;
        server_name  _;
        root         /usr/share/nginx/html;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        location / {
        }

        error_page 404 /404.html;
            location = /40x.html {
        }

        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }

}
```

下面我们就来详细分析一下nginx.conf这个文件中的内容。

按照功能划分，我们通常将nginx配置文件分为三大块，**全局块，events块，http块**。

### 第一部分：全局块

首先映入眼帘的这一堆：

```
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

# Load dynamic modules. See /usr/share/doc/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;
```

我们称之为**全局块**，知识点呐朋友们，要记住，这里呢，主要会设置一些影响 nginx 服务器整体运行的配置指令，主要包括配 置运行 Nginx 服务器的用户（组）、允许生成的 worker process 数，进程 PID 存放路径、日志存放路径和类型以及配置文件的引入等。

比如  worker_processes auto； 这一行，worker_processes 值越大，我们nginx可支持的并发数量就越多，很多人想这不就爽了吗，我设置成正无穷，无限并发flag达成，秒杀问题轻松解决，这个，受自己服务器硬件限制的，不能乱来。

### 第二部分：events 块：

```
events {
    worker_connections 1024;
}
```

这一堆，就是我们配置文件的第二部分，**events 块**

events 块涉及的指令主要影响 Nginx 服务器与用户的网络连接，常用的设置包括是否开启对多 work process 下的网络连接进行序列化，是否允许同时接收多个网络连接，选取哪种事件驱动模型来处理连接请求，每个 word process 可以同时支持的最大连接数等。

### 第三部分：http块：

```
http {
    server {
        }
}
```

> 注意：
>
> http是一个大块，里面也可以包括很多小块，比如http全局块，server块等。

http 全局块配置的指令包括文件引入、MIME-TYPE 定义、日志自定义、连接超时时间、单链接请求数上限等。

而http块中的server块则相当于一个虚拟主机，一个http块可以拥有多个server块。

server块又包括**全局server**块，和**location**块。

全局server块主要包括了本虚拟机主机的监听配置和本虚拟主机的名称或 IP 配置

location块则用来对虚拟主机名称之外的字符串进行匹配，对特定的请求进行处理。地址定向、数据缓 存和应答控制等功能，还有许多第三方模块的配置也在这里进行。比如，对/usr相关的请求交给8080来处理，/admin则较给8081处理。

### nginx配置反向代理

**反向代理服务器和目标服务器对外就是一个服务器，暴露的是代理服务器地址，隐藏了真实服务器 IP 地址。**

所以接下来我们通过一个小栗子，当我们访问服务器的时候，由于我的服务器没备案，阿里云默认80端口没开，所以这里我们设置对外服务的端口为8888，**当我们访问8888端口的时候，实际上是跳转到8080端口的。**

然后修改我们的配置文件nginx.conf里面的server块，修改之后的内容如下：

```
    server {
        listen       8888 ; ##设置我们nginx监听端口为8888
        server_name  [服务器的ip地址];

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        location / {
            proxy_pass http://127.0.0.1:8080; ##需要代理的服务器地址
            index index.html;
        }

        error_page 404 /404.html;
            location = /40x.html {
        }

        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }
```

然后在浏览器中输入：服务器ip:8888 发现浏览器显示出来了8080端口tomcat的欢迎界面，从而实现了隐藏真实服务器地址这样一个反向代理的要求。

哦？看着好神奇哦，那，我之前经常有看到那种，就是各种/image /video 不同的链接对应的是不同的网站，那也是这么做的咯？

聪明，这里我们再新建一个tomcat容器，端口为8081，同时把在容器中tomcat webapps目录新建一个我们自己的目录，这里叫hello，里面新建一个hello.html文件，内容为

```
<h1>I am Hello<h1>
```

同时我们在端口为8080的tomcat容器中，在webapps新建我们的文件家hi，并新建hi.html文件，内容为

```
<h1>I am Hi<h1>
```

啊，这样的话配置是不是很难啊?

你想多了，敲简单的。

修改我们配置文件中的server快，如下：

```
    server {
        listen       8888 ; ##设置我们nginx监听端口为8888
        server_name  [服务器的ip地址];

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        location /hi/ {
            proxy_pass http://127.0.0.1:8080; ##需要代理的服务器地址
            index index.html;
        }
        
        location /hello/ {
            proxy_pass http://127.0.0.1:8081; ##需要代理的服务器地址
            index index.html;
        }

        error_page 404 /404.html;
            location = /40x.html {
        }

        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }
```

在浏览器中输入：服务器ip:8888/hi/hi.html

浏览器显示  **I am hi** 对应服务器端口为 8080

在浏览器中输入：服务器ip:8888/hello/hello.html

浏览器显示  **I am hello** 对应服务器端口为 8081

从而实现了针对不同url请求分发给不同服务器的功能配置。

少侠，且慢，你是不是忘了什么东西，location /hello/ 是什么意思，只能这么写么？

当然不是。学会location指令匹配路径，随便换姿势

**location指令说明:**

功能：用于匹配URL

语法如下：

1. = ：用于不含正则表达式的 uri 前，要求请求字符串与 uri 严格匹配，如果匹配成功，就停止继续向下搜索并立即处理该请求。
2. ~：用于表示 uri 包含正则表达式，并且区分大小写。
3. ~*：用于表示 uri 包含正则表达式，并且不区分大小写。
4. ^~：用于不含正则表达式的 uri 前，要求 Nginx 服务器找到标识 uri 和请求字符串匹配度最高的 location 后，立即使用此 location 处理请求，而不再使用 location 块中的正则 uri 和请求字符串做匹配。


注意:

> 如果 uri 包含正则表达式，则必须要有 ~ 或者 ~* 标识。

### nginx配置负载均衡：

在nginx中配置负载均衡也是十分容易的，同时还支持了多种负载均衡策略供我们灵活选择。首先依旧是准备两个tomcat服务器，一个端口为8080，一个端口为8081，这里呢，推荐大家用docker部署，太方便了，什么，不会docker，可以移步我的面向后端的docker初级入门教程，真的挺好用，省了很多工作量。

然后修改我们的http块如下:

```
http {

###此处省略一大堆没有改的配置


    ##自定义我们的服务列表
    upstream myserver{
       server 127.0.0.1:8080;
       server 127.0.0.1:8090;
     }


   server {
       listen       8888 ; ##设置我们nginx监听端口为8888
       server_name  [服务器的ip地址];

       # Load configuration files for the default server block.
       include /etc/nginx/default.d/*.conf;

       location / {
           proxy_pass http://myserver; ##叮，核心配置在这里
           proxy_connect_timeout 10; #超时时间，单位秒
       }

       error_page 404 /404.html;
           location = /40x.html {
       }

       error_page 500 502 503 504 /50x.html;
           location = /50x.html {
       }
   }

}
```

之前就有说过，nginx提供了不同的负载均衡策略供我们灵活选择，分别是：

- **轮询(默认方式):**  每个请求按时间顺序逐一分配到不同的后端服务器，如果后端服务器 down 掉，能自动剔除。

	用法：啥也不加，上文实例就是默认的方式，就是默认的

- **权重(weight):**  weight 代表权重,默认为 1,权重越高被分配的客户端越多,权重越大，能力越大，责任越大，处理的请求就越多。

	用法:

	```
	     upstream myserver{
	        server 127.0.0.1:8080 weight =1;
	        server 127.0.0.1:8090 weight =2;
	      }
	```

- **ip_hash**：每个请求按访问 ip 的 hash 结果分配，这样每个访客固定访问一个后端服务器，可以解决 session 的问题。

	用法：

	```
	 upstream myserver{
	       ip_hash;#可与weight配合使用
	       server 127.0.0.1:8080 weight =1;
	       server 127.0.0.1:8090 weight =2;
	     }
	```

### nginx配置动静分离：

> 动静分离就是把很少会发生修改的诸如图像，视频，css样式等静态资源文件放置在单独的服务器上，而动态请求则由另外一台服务器上进行，这样一来，负责动态请求的服务器则可以专注在动态请求的处理上，从而提高了我们程序的运行效率，与此同时，我们也可以针对我们的静态资源服务器做专属的优化，增加我们静态请求的响应速度。

具体的动静分离配置也不是十分的复杂，和负载均衡，反向代理差不多。

为了演示动静分离呢，首先我们需要准备两个文件夹，一个是data文件夹，用来存放我们js，css这些静态资源文件，一个是html文件夹，用来存放我们的html文件。

在html文件夹新建一个html文件，index.html，内容如下

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>我是一个静态界面</title>
</head>
<script type="text/javascript" src="jquery.js"></script>
<body>
<h1>我是一个静态界面</h1>
<div id="test_div"></div>
</body
</html>
```

注意，这里我们并没有将jquery.js 这个文件放在html目录下，而是将它放在了另外一个目录data里面，当服务器接需要请求jquery.js这个文件时，并不会去index.html所在的那个服务器去请求这个文件，而是会直接去我们配置好的服务器或者路径去寻找这个js文件，在本实例中，会去data文件夹下面去找这个jquery.js这个文件。

修改server的配置如下:

```
 server {
        listen      8886 ;
        server_name [你的服务器ip地址];

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        location / {
            root /html/;
            index index.html;
        }

         #拦截静态资源，static里面存放的我们图片什么的静态资源
        location ~ .*\.(gif|jpg|jpeg|bmp|png|ico|js|css)$ {
        root /data/;
       }

        error_page 404 /404.html;
            location = /40x.html {
        }

        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }
```


