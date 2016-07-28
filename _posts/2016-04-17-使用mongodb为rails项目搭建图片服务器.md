---
layout: post
title: 使用 MongoDB 为 Rails 项目搭建图片服务器
---

  在使用 MongoDB 搭建图片服务器之前，我们首先需要准备一个 Rails 项目，demo 代码在 [https://github.com/baya/mongo_image_demo](https://github.com/baya/mongo_image_demo), 与图片服务器的交互将在此 demo 项目中实现。


## 1 安装和配置 nginx-gridfs

我们存取图片的请求首先经过 nginx, 然后再通过 [nginx-gridfs](https://github.com/mdirolf/nginx-gridfs) 转到 mongodb, 所以配置 nginx-gridfs 是很重要的一步。

如果服务器上没有安装 nginx, 我们参考 [https://github.com/mdirolf/nginx-gridfs#installation](https://github.com/mdirolf/nginx-gridfs#installation) 的步骤安装即可，如果已经安装好了 nginx, 但是没有安装 nginx-gridfs 模块，那么我们可以只安装 nginx-gridfs 模块。


## 1.1 安装 nginx 和 nginx-gridfs

为简单起见，我们假设是第一次安装 nginx, 并且是通过源代码安装。

1, 下载 nginx 源码, 并且解压缩

~~~bash
wget http://nginx.org/download/nginx-1.10.0.tar.gz
tar -xvf nginx-1.10.0.tar.gz
~~~

2, 将 nginx-gridfs 克隆到本地

~~~bash
git clone git@github.com:mdirolf/nginx-gridfs.git
cd nginx-gridfs
git checkout v0.8
git submodule init
git submodule update
~~~

3, build

~~~bash
cd /path/to/nginx-source
./configure --add-module=/path/to/nginx-gridfs/source/
make
make install
~~~

## 1.2 配置 nginx-gridfs

这一步其实就是在 nginx 的 conf 文件增加 gridfs 相关的内容。

nginx-gridfs 上的官方配置文档过于简单，不适合生产环境。我们接下来的配置会在 nginx conf 文件里
增加两个 server, 第 1 个 server 将监听 4444 端口，这个 server 将直接和 mongodb 交互，第 2
个 server 将监听 5555 端口，这个 server 上面会添加图片缓存等配置，并且在缓存过期后会将请求转发
到第 1 个 server, 具体配置内容如下:

~~~bash
$ cat nginx.conf

worker_processes  1;

events {
    worker_connections  1024;
}


http {
    default_type  application/octet-stream;
    sendfile        on;

    keepalive_timeout  65;

    include       mime.types;

    proxy_temp_path   /data/nginx_cache/nginx_temp;
    proxy_cache_path  /data/nginx_cache/nginx_cache  levels=1:2   keys_zone=cache_one:4000m inactive=2d max_size=10g;

    server{
        listen       4444;
        location / {
                        gridfs imgdb root_collection=fs field=filename type=string;
                        mongo 127.0.0.1:27017;
          }
    }
     upstream my_server_pool {
        server 127.0.0.1:4444 weight=1;
    }
    server {
        listen       5555;
        location /upload/ {
            proxy_cache cache_one;
            proxy_cache_valid 200 304 2d;
            proxy_cache_valid 301 302 1m;
            proxy_cache_valid any 1s;
            proxy_cache_key $host$uri$is_args$args;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-For $remote_addr;
            proxy_pass http://my_server_pool/;
            add_header X-Cache HIT-LT;
            expires       max;
        }
    }
}

~~~

这样我们就完成了 nginx-gridfs 的配置。

## 2 启动 nginx 和 mongodb, 并测试是否能够正常存取图片


启动 nginx:

~~~bash
/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
~~~


mongodb 的配置文件如下:

~~~bash
port=27017
dbpath=/data/db
logpath=/var/mongodb/log/mongodb.log
logappend=true
fork=true
~~~

启动 mongodb:

~~~bash
mongod -f /path/to/mongodb.conf
~~~

在启动 nginx 和 mongodb 时会出现一些权限和目录不存在的问题，可以使用 `chown` 和 `mkdir -p` 解决。上面的配置步骤都是在 mac os 上操作的。

测试上传图片:

~~~bash
mongofiles --host localhost --port 27017 --db imgdb put ~/my_img.jpg
~~~

