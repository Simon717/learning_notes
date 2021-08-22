## 什么是API-网关

职责：客户需要通过不同的api请求调用不同的后端服务，api网关去统一管理这些api请求，作为客户端和后端服务之间的桥梁。

![img](https://upload-images.jianshu.io/upload_images/14814543-ff52ad16128922f6.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

## API网关涵盖的基本功能

- 统一API入口
- 隔离后端
- 认证鉴权流控
- 负载均衡
- 安全控制

## kong的概念

Kong是一个在Nginx中运行的Lua应用程序，并且通过lua-nginx模块实现。OpenResty 也不是 Nginx的分支，而是一组扩展其功能的模块。

Kong是一个可扩展的开源API网关，运作在RESTfull API之前，提供统一的入口，并且通过插件的形式进行扩展，插件提供了平台核心功能意外的功能和服务，例如鉴权、流控等等。

#### kong的特点：

RESTFUL API通过请求kong的admin端口（8001)进行配置，类似于Nginx的配置文件

#### kong的基本功能

kong的基本功能==Nginx，代理转发，通过一个例子体会kong的基本功能以及kong和Nginx的关系

例：一个典型的 Nginx 配置

```json
upstream helloUpstream {
    server localhost:3000 weight=100;
}

server {
    listen 80;
    location /hello {
        proxy_pass http://helloUpstream;
    }
}
```
上述配置文件作用：在当前配置下，访问本机的`localhost:80/hello`，将会跳转到`localhost:3000`

##### 上述Nginx配置可以通过kong实现

对应的 Kong 配置：

```bash
# 配置 upstream
curl -X POST http://localhost:8001/upstreams --data "name=helloUpstream"
# 配置 target
curl -X POST http://localhost:8001/upstreams/hello/targets --data "target=localhost:3000" --data "weight=100"
# 配置 service
curl -X POST http://localhost:8001/services --data "name=hello" --data "host=helloUpstream"
# 配置 route
curl -X POST http://localhost:8001/routes --data "paths[]=/hello" --data "service.id=8695cc65-16c1-43b1-95a1-5d30d0a50409"
```

这一切都是动态的，无需手动 reload nginx.conf

注：

* localhost:8001 是kong的admin端口，用于修改kong配置

当前配置项的效果：访问`localhost:8000/hello`会跳转到`localhost:3000`

#### kong 核心概念

上面的配置引出了Kong 最最核心的四个对象`upstream，target，service，route`

* **route** 监听主机8000端口的请求，匹配请求，如果匹配成功，将跳转到对应的`service`上
* **service** 抽象层面的后端服务，有两种情况
  * service指向一个物理服务(ip:port/path)
  * service指向一个`upstream`对象，upstream对应到多个物理服务，实现负载均衡
* **upstream**  后端服务抽象
* **target** 代表一个物理服务，（ip:host)


 **他们的对应关系如下：**

- upstream 和 target ：1 对 n
- service 和 upstream ：1 对 1 或 1 对 0 （service 也可以直接指向具体的 target，相当于不做负载均衡）
- service 和 route：1 对 n

#### 补充nginx知识

>  例子：通过**Nginx代理请求，访问后端javaweb服务**

　　a.我在tomcat下部署了一个`javaweb服务`，tomcat安装的服务器IP为：`192.168.37.136`，部署的项目在tomcat下的访问地址为：http://192.168.37.136:8080/lywh/

　　b.我在IP为`192.168.37.133`的服务器下面安装成功了`Nginx`。

　　c.那怎么样**将tomcat下部署的网站使用Nginx代理呢**？，修改Nginx的配置文件，修改命令：vim /usr/local/nginx/conf/nginx.conf

```json
http {
     #配置tomcat的IP地址和访问端口
     upstream gw {
         server 192.168.37.136:8080 weight=1;
     }

     server {
         listen       80;
         server_name  localhost;

         location / {
             root   html;
             index  index.html index.htm;
         }
         
         #Nginx代理配置
         location /lywh {
             proxy_pass http://gw/lywh;
         }
         location /sapi {
             proxy_pass http://gw/shopappapi;
         }
         location /cas{
             proxy_pass http://gw/cas-server-webapp-4.0.0/login;
         }
         location /doc{
             proxy_pass http://gw/docs;
         }
}
}
```

![img](https://images2015.cnblogs.com/blog/359161/201602/359161-20160204161915585-1501529986.png)

### kong实战：

通过简单的两个小练习，掌握kong的基本使用

##### 练习1：配置service+route，实现API代理

https://www.cnblogs.com/sunhongleibibi/p/12024386.html

1. 使用Admin API添加**Service**

    ```bash
    $ curl -i -X POST \
      --url http://localhost:8001/services/ \
      --data 'name=example-service-2' \
      --data 'url=http://mockbin.org'
    ```
    指定url，链接到一个具体的后端服务

2. 为Service添加**Route**

    ```bash
    $ curl -i -X POST \
      --url http://localhost:8001/services/example-service-2/routes \
      --data 'name=test2-api-proxy' \
      --data 'hosts[]=test2.example.com' \
      --data 'paths[]=/request' \
      --data 'strip_path=false'
    ```
    从一个request的URL链接到kong的service

3. 验证Admin API 代理结果：

    ```bash
    $ curl -i -X GET \
      --url http://localhost:8000/request \
      --header 'Host: test2.example.com'
    ```
> curl命令中，-x 用于指定方法请求 （GET/POST/PUT/DELETE/HEAD 等）



##### 练习2：配置upsteam实现负载均衡

`route`根据`paths`转发给相应的`service`，`service`根据`host（upstream的name）`转发给 `upstream``upstream`负载均衡至`targets`，这就是kong的负载均衡执行流程。

1. 配置upstream

    创建**upstream**

    ```bash
    $ curl -X POST localhost:8001/upstreams \
    --data "name=app.com"
    ```

    为upstream配置**target**

    在同一个upsteam对象下挂上两个target

    ```bash
    $ curl -X POST localhost:8001/upstreams/app.com/targets \
    --data "target=myhost1:8881" \
    --data "weight=100"

    $ curl -X POST localhost:8001/upstreams/app.com/targets \
    --data "target=myhost2:8882" \
    --data "weight=100"
    ```

    等同于创建了如下配置：

    ```bash
    upstream upstream.api {
        server myhost1:8881 weight=100;
        server myhost2:8882 weight=100;
    }
    ```

2. 配置service

    ```bash
    $ curl -X POST localhost:8001/services \
    --data "name=my-app-service" \
    --data "host=app.com"
    ```

3. 配置route([more](https://docs.konghq.com/1.0.x/admin-api/#add-route))

    ```bash
    $ curl -X POST localhost:8001/services/a9b8a3e9-826b-47fa-ae78-0fcf111662a1/routes \
    --data "name=test-app-route" \
    --data "hosts[]=test.app.com" \
    --data 'strip_path=false'
    ```

    或者

    ```bash
    $ curl -X POST localhost:8001/routes \
    --data "name=test-app-route" \
    --data "hosts[]=test.app.com" \
    --data "service.id=a9b8a3e9-826b-47fa-ae78-0fcf111662a1" \
    --data 'strip_path=false'
    ```

**浏览器测试**
通过`Shift+F5 或 Ctrl+Shift+R`，不使用缓存进行请求测试

![img](https://mrgao.oss-cn-beijing.aliyuncs.com/md/kong/19-12-13/kong_20191213104236.png?x-oss-process=style/watermark)



### kong配置总结

kong的配置通常是向后连接，即定义**service**时指定**upsteam**，定义**upstream**时需要指定**targets**，定义**route**时需要指定**service**

这个规律也不是绝对的，也可以先定义upstream，然后在定义target时指定upstream。

### kong的workflow

![img](https://mrgao.oss-cn-beijing.aliyuncs.com/md/kong/19-12-12/kong_20191212164923.png?x-oss-process=style/watermark)

### kong工作端口

#### **kong工作端口**

* **8000  http**
* 8000 http
* 8443 https
* **8001 admin**
* 8444 

默认情况下，Kong会监听以下端口：

- `:8000`Kong用来监听来自客户端的传入HTTP流量，并将其转发到上游服务
- `:8443`Kong用来监听传入的HTTPS流量。此端口具有和`:8000`端口类似的行为，但它仅用于HTTPS流量。可以通过配置文件禁用此端口。
- `:8001`[Admin API](https://docs.konghq.com/1.1.x/admin-api)用于配置Kong监听。
- `:8444`Admin API监听HTTPS流量。





资源：

https://www.jianshu.com/p/a68e45bcadb6