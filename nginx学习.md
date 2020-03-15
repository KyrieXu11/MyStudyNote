# nginx的几个概念

## 正向代理

就比如正常的通过浏览器是无访问谷歌的，所以就要通过代理服务器访问

![正向代理.png](https://i.loli.net/2020/03/12/aehd1mXYwMZcfJp.png)

## 反向代理

反向代理就是用户端是不能看到真正的服务器的端口号的，要使用反向代理将端口号给隐藏起来

![反向代理.png](https://i.loli.net/2020/03/12/Tpt3mB8hu6LjHRo.png)

## 负载均衡

负载均衡就是将一个请求分发到不同的服务器上，让服务器的压力会小一些。

![负载均衡.png](https://i.loli.net/2020/03/12/o1ehEONGPZWldiS.png)

## 动静分离

这里不太好画图了，就是将动态资源和静态资源分开请求，如果动态资源和静态资源都在一个服务器上，那么请求的压力就会更大了，所以将动态资源放在服务器上，静态资源单独请求。

# Docker中配置nginx

## 下载nginx的docker镜像

```shell
docker pull nginx
```

## 启动nginx

```sh
docker run -it --name nginx -p [自定义端口号]:[配置文件中的端口号] -v [自己定义的配置文件]:/etc/nginx/nginx.conf -v [自定义的工作目录]:/home/nginx/www nginx 
```

## 添加配置文件

```json
user nginx;
worker_processes  1;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    server {
        listen       80;
        server_name  localhost;

        # Vue路由模式为history需添加的配置
        location / {
            if (!-e $request_filename) {
                rewrite ^(.*)$ /index.html?s=$1 last;
                break;
            }
            root   /home/nginx/www;
            index  index.html;
        }

        # 获取真实IP以及Websocket需添加的配置
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header REMOTE-HOST $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # 客户端Body大小限制（文件上传大小限制配置）
        client_max_body_size 5m;

        error_page   500 502 503 504 404  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
}
```

上面的配置文件就要放在自定义的配置文件中，然后执行`docker restart nginx`就行了

# 配置Docker的nginx和tomcat

## 启动Tomcat

```shell
docker run -d --name tomcat -p 8081:8080 tomcat
```

当然上面的也可以再加上自定的文件夹映射以便后续的改动，我这里就先不加了，因为访问首页就算访问成功了。

## 配置nginx

有两种配置的方法

### 第一种方法

启动完成tomcat之后，查看服务器的ip地址，使用下面这条命令：

```shell
docker inspect tomcat|grep "IPAddress"
```

然后把ip地址给复制到nginx中的配置文件中，比如查出来的ip是`172.17.0.2`，就在`upstream`块中先自定义一个块名称就比如`tomcat`，再在这个块中添加`172.17.0.2:8080`，然后再在`location`块中配置代理的那个服务器的地址，后面的参数在这里是可以不用的，但是`proxy_pass`是必须要的。

配置文件如下：

```json
user  nginx;
worker_processes  1;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for" "$upstream_addr"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    upstream tomcat {
    	# 多服务器配置，是要有两个server的
        # server 172.17.0.2:8080;
        server 172.17.0.4:8080;
    }

    server {
        listen 80;
        server_name localhost;

        location / {
            proxy_pass http://tomcat;
            proxy_redirect off;
            index index.html index.htm;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Real-Port $remote_port;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        location /static/ {
            alias /usr/share/nginx/html/;
        }
    }
}
```



### 第二种方法

就是将上面配置文件的`upstream`块中的服务ip地址改成本机的ip地址，端口号就是docker中的tomcat映射的端口号，就比如`192.168.0.108:8081`

## 启动nginx

启动命令就是上面的那个

```shell
docker run -it --name nginx -p [自定义端口号]:[配置文件中的端口号] -v [自己定义的配置文件]:/etc/nginx/nginx.conf -v [自定义的工作目录]:/home/nginx/www nginx 
```

