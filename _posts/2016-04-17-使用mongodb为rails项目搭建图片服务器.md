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

这里要注意 `/data/nginx_cache/nginx_temp` 的访问权限问题，如果 nginx 的 error log 中出现类似 `failed (13: Permission denied) while reading upstream` 的错误，可以使用
`chown` 命令将 `/data/nginx_cache/nginx_temp` 的 owner 修改下。

我们分析下 `4444` 端口也就是和 Mongo GridFS 交互的配置的细节:

~~~bash
    server{
        listen       4444;
        location / {
                        gridfs imgdb root_collection=fs field=filename type=string;
                        mongo 127.0.0.1:27017;
          }
    }
~~~


* gridfs：         nginx识别插件的名字;
* imgdb：          数据库名称;
* root_collection: 选择collection，如root_collection=images， mongod就会去找images.files，默认是fs;
* field：          查询字段，支持_id, filename, 可省略, 默认是_id, 在这里我们使用 filename 即图片的名字去查找图片;
* type：           解释field的数据类型，支持objectid, int, string, 可省略, 默认是int, 如果我们 field 用的是 filename, 那么此处需要设置为 string;
* user:           用户名, 可省略;
* pass:           密码, 可省略;
* mongo：          mongodb 的 host 和 port;



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
mongofiles --host localhost --port 27017 --db imgdb --local=/path/to/dog.jpg put my_dog.jpg
~~~

通过浏览器可以访问已经上传的图片:

![my_img.jpg](/images/Snip20160730_22.png)

## 3 与 Rails 项目集成

### 3.1 demo 项目简介

在写这个 demo 时，我原本打算参照我所在公司的项目的代码来完成图片上传的功能，但是由于公司使用的 rails 版本比较低，相关配套的 gem 在适配比较新的 rails 版本(这个 demo 用的 rails 版本是 4.2.6)时出现了很多问题, 并且相关的配置过于繁琐复杂，我决定只使用 mongodb 的官方 ruby driver 来实现图片上传的功能。

公司项目中为了实现图片上传使用了如下的 gem:

```
# Gemfile
gem 'carrierwave'
gem 'carrierwave-mongoid'
gem 'mongoid'
```

只使用 mongodb 的官方 ruby driver: mongo,

```
# Gemfile
gem 'mongo', '~> 2.2'
```

在此 demo 中, 图片上传的业务流程可以简单到用一句话说清楚:

> 应用为图片生成唯一的名称，然后将图片提交给图片服务器，如果图片服务器保存图片成功，则将图片名称存储到
数据库以供应用将来从图片服务器拿图片。

### 3.2 demo 设计

当然在实际的应用中还涉及到图片的压缩，裁减，变换，加水印等业务逻辑, 但是这些业务逻辑应该是和图片上传隔离的不应该耦合在一起，从这种角度考虑，我比较讨厌 `carrierwave` 之类的图片上传库把图片的上传，裁减，变换等工作都放到一个 Uploader 里去实现。

作为一名程序员，将自己的想法转化成直观的图形有两点好处:

1. 方便自己写代码

2. 方便别人写代码

所以我们仍然对图片上传这一看似简单的功能画一张图:

![app-client-server](/images/Snip20160806_25.png)

从上面的图中我们可以分4个步骤实现图片上传的功能:

1. 应用方面，我们需要生成 image params, 比如 image name, image 实体等参数提交给图片服务器客户端;

2. 图片服务器客户端方面, 我们需要写服务器的配置信息, 并连接服务器;

3. 图片服务器客户端方面, 我们需要将图片参数提交给服务器；

4. 应用方面, 我们需要处理图片服务器的响应, 如果响应成功我们需要将图片的名称等参数记录下来;

### 3.3 demo 实现


## 参考

* [Mongodb GridFS图片文件存储解决方案: http://chwshuang.iteye.com/blog/2065974](http://chwshuang.iteye.com/blog/2065974)

* [Mongo Ruby Driver: https://github.com/mongodb/mongo-ruby-driver](https://github.com/mongodb/mongo-ruby-driver)
