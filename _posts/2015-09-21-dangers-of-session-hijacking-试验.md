---
layout: post
title: Dangers of Session Hijacking 试验
---

试验的原始创意来自于 http://railscasts.com/episodes/356-dangers-of-session-hijacking


## 试验前的准备

在试验之前先搭建试验环境,

~~~bash
$ git clone git@github.com:baya/hijacking-session-demo.git
$ cd hijacking-session-demo 
$ bundle install
$ bundle exe rake db:create
$ bundle exe rake db:migrate
~~~

然后使用 `tcpdump` 监听网络通信,

~~~bash
$ sudo tcpdump -i lo0 -A
~~~

## 无保护的 Rails 应用

我们签出 insecure-v-1.0 tag, 并启动 rails 服务,

~~~bash
$ git checkout -b session-hijacking insecure-v-1.0
$ bundle exe rails s
~~~

注册一个用户, login 是 'hi-jack', password 是 '123123', 然后用 hi-jack 登录系统，登录成功后，我们使用浏览器访问 http://localhost:3000/, 

![welcome](/images/Snip20150926_15.png)

这时我们可以通过 `tcpdump` 得到浏览器访问 http://localhost:3000/ 时的请求头数据,

~~~bash
Host: localhost:3000
Connection: keep-alive
Pragma: no-cache
Cache-Control: no-cache
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_8_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/45.0.2454.99 Safari/537.36
Accept: */*
Referer: http://localhost:3000/
Accept-Encoding: gzip, deflate, sdch
Accept-Language: zh-CN,zh;q=0.8,en;q=0.6,zh-TW;q=0.4,ja;q=0.2
Cookie: _PingPong_session=BAh7B0kiD3Nlc3Npb25faWQGOgZFVEkiJTU3NGVkOTM4ZWMyODFjNjQwNDM4
~~~

注意 Cookie 值很长，这里没有给出完整的值。

我们首先使用 curl 不带 cookie 访问 http://localhost:3000/,

~~~bash
$ curl http://localhost:3000
~~~

得到的响应如下:

~~~html
<div class="content">
  <a href="http://localhost:3000/login">Please Login</a>
</div>
~~~

这说明登录失败了。

然后我们将通过 `tcpdump` 监听到的 cookie 带给 curl 去访问 http://localhost:3000/,

~~~bash
$ curl http://localhost:3000 -H 'Cookie: _PingPong_session=BAh7B0kiD3Nlc3Npb25faWQGOgZFVEkiJTU3NGVkOTM4ZWMyODFjNjQwNDM4'
~~~

得到响应如下:

~~~html
<div class="content">
  <h2>Welcome: hi-jack </h2>
  <a rel="nofollow" data-method="delete" href="http://localhost:3000/logout">Logout</a>
</div>
~~~

这说明登录成功了。

注意此处的 cookie 使用你本地电脑实际监听到的 cookie。

## 为 Rails 应用增加保护措施

### 强制使用 SSL

切换到强制使用 SSL 的代码版本,

~~~bash
$ git checkout -b force_use_ssl secure-force-ssl
$ bundle exe rails s
~~~

使用 `git diff` 命令比较两个tag: insecure-v-1.0 和 secure-force-ssl， 

~~~bash
$ git diff insecure-v-1.0 secure-force-ssl
~~~

可以知道 secure-force-ssl 增加了代码,

~~~ruby
# config/application.rb

+ config.force_ssl = true

~~~

此时使用浏览器访问 http://localhost:3000/, 会自动跳转到 https://localhost:3000/,

注意 HTTP 协议由 http 变为了 https。

![force-use-ssl](/images/Snip20150926_17.png)


### 使用 cookies.signed

切换到使用 signed cookies 的代码,

~~~bash
$ git co -b use_signed_cookie insecure-signed-cookie
~~~

看下代码改动,

~~~bash
$ git diff insecure-v-1.0 insecure-signed-cookie
~~~

~~~ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
   helper_method :current_user
 
   def current_user
-    @current_user ||= User.find(session[:user_id]) if session[:user_id]
+    if cookies.signed[:secure_user_id] == "secure#{session[:user_id]}"
+      @current_user ||= User.find(session[:user_id]) if session[:user_id]
+    end
   end
   
 end

# app/controllers/sessions_controller.rb
class SessionsController < ApplicationController
     user = User.find_by(login: params[:login])
     if user && user.authenticate(params[:password])
       session[:user_id] = user.id
+      cookies.signed[:secure_user_id] = "secure#{user.id}"
       redirect_to root_url
     else
       flash[:notice] = 'Login Failed'
~~~


cookies.signed 只能防止 cookies 不被篡改，但是不能防止 sessions 被劫持，同样我们通过

`tcpdump` 可以监听到请求的 cookie, 然后将此 cookie 发送给服务器就能冒充真实的用户了,

试验过程如下,

~~~bash
$ curl http://localhost:3000 -H 'Cookie: _hijacking-session-demo_session=Ync0ajZLVUY1NnlWdjNhYjFiblQ2OThKdjBzNDRsVHFGRUczeUcrUlhnL1l4Zyt0aCtUdjNIQ25ScUxFNitaR29qdGJ4d0FMbVZzeUxRYW55OFZGbVJRNHFjenJ6Mk1OY2d3UENLalVESmRuL1RqdnN4QmZsbmo2MU9BcFp2TUd4YnFCRzBHN29PTEZCSTR6VnB6dm03WkNzNjlVTWU2Q0VHV1pBR1FzMUxZPS0tczdham1MelZ4S3M4OVAvbm9UcHB3Zz09--116fcd0cb66032ba40911112e61d6667c94e2654; secure_user_id=InNlY3VyZTIi--15ed677ddb2e2dd6d2ca0614c4f2960e6d4bc7f1'
~~~

服务器响应,

~~~html
<div class="content">
  <h2>Welcome: hi-jack </h2>
  <a rel="nofollow" data-method="delete" href="http://localhost:3000/logout">Logout</a>
</div>
~~~

说明登录成功。


## 小结

每一个负责任的网站应该强制使用 SSL。



