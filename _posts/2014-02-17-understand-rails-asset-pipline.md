---
layout: post
title: 理解和运用Rails Asset Pipeline
---

#### Asset Pipeline 本质很简单

刚开始接触Asset Pipeline时，各种概念迎面扑来，比如合并压缩 js 和 css, 支持使用CoffeeScript, Sass 和 ERB 写js, css，给各类资源打上指纹(Fingerprinting)等。
Asset Pipeline使用起来感觉不是特别方便，部署到产品环境时，还需要运行一些rake task，比如 rake assets:precompile RAILS_ENV=production 等。在引入一些第三方的js, css库时也时常会
出现一些恼人的问题，于是有点手足无措了，心里想事情为什么会变的复杂起来呢? 这种复杂有必要吗? 我是否要放弃使用Asset Pipeline呢?

先让我们回忆一下在Rails引入Asset Pipeline这个概念之前，我们是怎么引用js, css, image等资源的，很简单，把你需要引用的js, css, image等资源放在public目录下。

![从public/javascripts目录下引入js文件](/images/Snip20140218_1.png)

不错，上面的方式非常直观和简单，但是当我们的项目越来越大时，我们会引入越来越多的js, css文件，随之而来的是越来越多的http请求，大量的对js, css文件的http请求会降低我们
网站的响应速度，于是我们想合并这些js, css文件到一个或少数几个文件中去，显然上面的方式不能满足我们合并,压缩js, css文件的需求。我们可能使用CoffeScrip写js, 使用Sass写 
css，并且通过 ERB 在 js, css 文件中嵌入 ruby 代码，同样上面的方式也不能满足我们的需求。我们时常听到用户这样的抱怨，你们不是说已经把图片改过来了吗?怎么我现在在你们网站上看到的还是
那张老图片。如果我们能够给资源打上指纹，一旦资源内容发生变化，指纹也相应变化，那么我们就能避免这种浏览器缓存造成的问题。而Asset Pipeline正是为了解决这些问题而出现，并且我想指出的是
Asset Pipeline 和上面的方式几乎一样简单。

我们以引入js文件为例，在app/assets/javascripts目录下有一个application.js的文件，我们可以在这个文件里引入其他的我们需要的js文件，

~~~javascript
//= require jquery
//= require jquery_ujs
//= require app-01
~~~

然后在html文件里通过, 

~~~html
  <%= javascript_include_tag "application.js" %>
~~~

引入application.js文件，这时就能像第一种方式一样，引入jquery, jquery_ujs, app-01等js文件。

![通过application.js这个manifest文件引入js文件](/images/Snip20140219_2.png)

我们可以看到通过manifest文件引入js文件和第一种方式一样简单，此时我们只需要把待引入的js文件(这里的js文件可以是用CoffeScript写的和带erb的,比如app-01.js.coffee.erb)放到app/assets/javascripts目录里既可。
rails还设置了 lib/assets, vendor/assets两个目录来存放资源文件，其中lib/assets存放你自己写的，可以跨应用的资源, vendor/assets存放社区的第三方
的资源。我们应该把js, css, images等资源文件存放在(app|lib|vendor)/assets的子目录下，比如javascripts, stylesheets, images等, 而不是直接放置到(app|lib|vendor)/assets目录下，否则rails会找不到相应的资源文件。比如把app-01.js直接存放到app/assets目录下，那么程序会报找不到app-01的错误。我们可以通过 Rails.application.config.assets.paths 随时查看资源的查找路径。javascripts, stylesheets, images这些名字并没有程序上的约束力，我们可以把js文件放到images下，css文件放到javascripts下等等，这种存放方式不会对程序造成困扰，但是会对人类造成困扰，所以还是要避免这种存放方式。

你也可以自己创建一个manifest文件比如admin.js(这里把admin.js文件存放在app/assets/javascripts目录下)，然后在这个文件里引入你需要的js文件,

~~~javascript
//= require admin-01
//= require admin-02
~~~

~~~html
  <%= javascript_include_tag "admin.js" %>
~~~

~~~html

<html>
 <head>
   <title>Mytestproject</title>
   <script src="/assets/admin-01.js?body=1"></script>
   <script src="/assets/admin-02.js?body=1"></script>
   <script src="/assets/admin.js?body=1"></script>
  </head>
  <body>
  </body>
</html>

~~~

##### 小结

>
  1. 创建manifest文件，在此文件里通过require引入你需要的js或者css文件;
  2. 把你需要引入的js或者css文件存放到(app|lib|vendor)/assets的子目录下;
  

#### 产品环境下预编译资源

在项目下运行 rake assets:precompile RAILS_ENV=production，此命令会将 app/assets/javascript/application.js, app/assets/stylesheets/application.css 以及app/assets子目录下所有的非js, css文件编译到public/assets目录下，并且给这些文件打上指纹标记。

假如 app/assets的目录结构如下:

~~~bash
├── assets
│   ├── images
│   │   └── app-01.js
│   ├── javascripts
│   │   ├── Snip20140219_2.png
│   │   ├── admin-01.js
│   │   ├── admin-02.js
│   │   ├── admin.js
│   │   ├── application.js
│   │   ├── dd.ff
│   │   ├── ff.txt
│   │   ├── home.js.coffee
│   │   ├── jquery.js
│   │   └── jquery_ujs.js
│   ├── stylesheets
│   │   ├── application.css
│   │   └── home.css.scss
│   └── test.png

~~~
那么运行 rake assets:precompile RAILS_ENV=production 命令后会在 public/assets目录下生成如下的文件:

~~~bash

├── assets
│   ├── Snip20140219_2-0ae94acc6afae5867edc57c1aba61382.png
│   ├── application-23160e7b4a32c70be7eb8ce0de474155.js
│   ├── application-23160e7b4a32c70be7eb8ce0de474155.js.gz
│   ├── application-96a552b03ca0e7ebcbfc44b89ca097a6.css
│   ├── application-96a552b03ca0e7ebcbfc44b89ca097a6.css.gz
│   ├── dd-96a552b03ca0e7ebcbfc44b89ca097a6.ff
│   ├── ff-96a552b03ca0e7ebcbfc44b89ca097a6.txt
│   └── manifest-1978e65245608bb85287a21f9c3bf27e.json

~~~

我们注意到admin.js文件没有被编译到public/assets目录下，这会导致我们在产品环境下无法引用到admin.js文件，

~~~html
  <%= javascript_include_tag "admin.js" %>
~~~

`javascript_include_tag "admin.js"` 找不到我们的admin.js文件。


我们需要将admin.js放到我们的资源编译路径中，也就是告诉rails我们需要编译admin.js，下面的代码能够达到这个目的:

~~~ruby
  config.assets.precompile += ['admin.js']
~~~

再运行一次 rake assets:precompile RAILS_ENV=production， 我们将看到public/assets里有相关的admin.js编译后的文件了。

~~~bash
├── admin-65e0fecc1177f0d878e220b2e8e1b665.js
├── admin-65e0fecc1177f0d878e220b2e8e1b665.js.gz
~~~

需要注意的是默认情况下lib/assets, vendor/assets下的各类资源都不会编译到public/assets目录下，如果需要编译，那么可以通过config.assets.precompile来增加
你需要编译的文件。

一旦资源被编译到public/assets目录下，那么我们可以通过配置nginx, apache来访问这些已经打上指纹的静态资源。

~~~html
<%= javascript_include_tag "admin.js" %>
~~~

或者

~~~html
<%= javascript_include_tag "application.js" %>
~~~

会生成类似下面的html代码,

~~~html
<html>
  <head>
    <title>Mytestproject</title>
    <script src="/assets/admin-eafb0c601eac47521bbe90886f560d50.js"></script>
  </head>
</html>
~~~

或者

~~~html
<html>
  <head>
    <title>Mytestproject</title>
    <script src="/assets/application-23160e7b4a32c70be7eb8ce0de474155.js"></script>
  </head>
</html>
~~~

我们来了解下js文件的编译过程，如果admin.js的文件内容如下,

~~~javascript
//= require 'admin-01'
//= require 'admin-02'
~~~

系统首先去查找 admin-01，找到后根据admin-01的扩展名，假如是.coffee.erb，此种情况下会先使用ERB将文件里的ruby代码执行，然后使用CoffeeScript解释器将coffee代码转换为js代码，对admin-02的处理过程与之类似，这样admin.js就是由admin-01和admin-02两部分的
代码合并而成，然后压缩代码，并给代码生成指纹，最后将类似admin-eafb0c601eac47521bbe90886f560d50.js的文件存放到public/assets目录下。这一串长长的字符串eafb0c601eac47521bbe90886f560d50就是编译后的admin.js的指纹，它是唯一的，并且
会随着admin.js内容的改变而改变。

##### 小结

> 资源预编译就是系统把要编译的文件(这些文件通过Rails.config.assets.precompile指定)，编译好(解释，合并，压缩，打指纹等)形成静态资源，然后把这些静态资源挪到public/assets目录下，以供nginx, apache等web服务器访问。

#### 引入第三方库

我们以一个实际的例子来说明引入第三方js库的过程。我做的一个项目有大量的图片上传功能，于是在前端使用了[swfupload](https://code.google.com/p/swfupload/)这个库，这个库借助了flash来实现图片上传的功能。swfupload的目录结构如下，

~~~bash
└── swfupload
    ├── handlers.js
    ├── images
    │   ├── SmallSpyGlassWithTransperancy_17x18.png
    │   ├── error.gif
    │   ├── search3.jpg
    │   ├── toobig.gif
    │   ├── uploadlimit.gif
    │   └── zerobyte.gif
    ├── swfupload.cookies.js
    ├── swfupload.js
    ├── swfupload.proxy.js
    ├── swfupload.queue.js
    ├── swfupload.speed.js
    ├── swfupload.swf
    └── swfupload_fp9.swf
~~~

我们可以简单的把swfupload整个的扔到public目录下，然后通过

~~~html
  <script src="/swfupload/swfupload.js" type="text/javascript"></script>
  <script src="/swfupload/swfupload.proxy.js" type="text/javascript"></script>
  <script src="/swfupload/swfupload.speed.js" type="text/javascript"></script>
  <script src="/swfupload/swfupload.queue.js" type="text/javascript"></script>
  <script src="/swfupload/handlers.js" type="text/javascript"></script>
~~~

来引入我们需要的swfupload js文件，这种方式比较简单直接，但是无疑增加了http请求，并且可能在多个地方需要写这样的代码，这样就增加了维护成本，那我们现在
用asset pipeline来管理我们的swfupload库。swfupload库属于社区的第三方库，我们一般不会直接改动这个库，如果社区有更新，我们可能是整个的更新此库。
按照rails的约定，我们应该把swfupload放到vendor/assets目录下。首先在vendor/assets目录下建立一个叫的uploadimg的文件夹，然后将swfupload放到
uploadimg文件夹下，**注意我们不要把swfupload直接放在vendor/assets目录下**。asset pipeline提供了一种index files技术，可以帮助我们方便的加载库文件，
首先在swfupload目录下建立一个index.js文件, 文件的内容如下,

~~~javascript
//= require swfupload/swfupload
//= require swfupload/swfupload.queue
//= require swfupload/handlers
~~~

也就是说index.js做为一个manifest文件，我们在此文件中引入我们需要加载的js文件，比如swfupload.js, swfupload.queue.js, handlers.js等。

路径app\_root/vendor/assets/uploadimg是系统查找资源的一个路径，所以`require swfupload/swfupload`，会通过app_root/vendor/assets/uploadimg/swfupload/swfupload 找到swfupload.js，`require swfupload/swfupload.queue` 和 `require swfupload/handlers` 执行的效果与之类似。也就是说我们通过
swfupload/index.js来管理 swfupload库里的要加载的js文件。当我们需要引入swfupload库的时候，比如需要在application.js文件中引入swfupload，那么加入`//=require swfupload`即可。

~~~javascript
//= require jquery
//= require jquery_ujs
//= require swfupload
~~~
这样我们就完成了对swfupload这个库的整体引入。要注意的是swfupload依赖swfupload.swf这个flash文件，由于默认情况下rails不会编译vendor/assets下的文件，那么我们需要告诉rails, 你应该帮我们编译swfupload.swf文件。

~~~ruby
config.assets.precompile += ['swfupload/swfupload.swf']
~~~

执行 `rake assets:precompile RAILS_ENV=production`，我们会发现rails帮我们生成了文件public/assets/swfupload/swfupload-6be81fe49801842f4401518d28971175.swf，这样我们就能在产品环境下正常使用swfupload了。

##### 小结

> 通过使用asset pipeline的index files，我们可以轻松地引用复杂的第三方库或者插件。





