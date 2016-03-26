---
layout: post
title: CheddarGetter集成之旅(二)免费试用
---

一般来说，用户掏钱之前都需要一个试用过程，用的爽的话给钱，不爽就和你拜拜，所以我们会创建一个free trial plan，客户首先会处于free trial plan，当然也会有客户跳过试用，直接掏钱使用我们的付费计划。free trial plan和其他的付费计划最根本的区别是客户注册free trial plan不需要提供信用卡信息。在我们的应用中，我们会提供一个"优化站点"的链接，客户通过这个链接就能进入我们预设的free trial plan中，整个过程中，客户不需要填写任何信息，我们会通过程序生成CG所需要的注册信息，然后通过调用CG(CheddarGetter)中的<a href="https://cheddargetter.com/developers#add-customer">create a new customer</a> api为客户在CG注册一个帐号，这样能快捷地把客户纳入到我们的计费系统中。<br/>
<a href="https://cheddargetter.com/developers#add-customer"><strong>Create a New Customer</strong></a>这个api需要的参数非常多，初看让人头大，但是如果只是创建一个free trial plan的customer，那么就只需要_code_, _firstName_, _lastName_, _email_和_subscription[planCode]_这5个参数。_code_就是你用来标识customer的一个记号，将来如果你需要对customer进行增删改查都需要利用这个code，这个记号必须是唯一的，_firsName_, _lastName_被限制在40个字符以内, 总之不要起的太长了，_email_必须是合法的email地址，_subscription[planCode]_表示你将要订购的plan的code，简单的说就是你需要订购哪种计划，而这些计划正是由这些plan code来标识。在<a href="/images/yottaa/cg-price-plans.png">CG Plans</a>的图中，你可以找到free trial plan的plan code是_OPTIMIZER_FREE_TRIAL_，并且这个code是可以改变的。

~~~ruby

cg_client = CheddarGetter::Client.new(:product_code => "product code",
                                      :username     => "your email for CG",
                                      :password     => "youre password for CG"
                  )
cg_client.new_customer(:code         => "a uniq code",
                       :firstName    => "fist name",
                       :lastName     => "last name",
                       :email        => "example@email.com",
                       :subscription => {:plan_code => "a free plan code"}
      )
end

~~~

上面的代码可以在CG中创建一个免费的customer，其中的CheddarGetter::Client来自于官方支持的gem
[cheddargetter_client_ruby](https://github.com/expectedbehavior/cheddargetter_client_ruby)，它对CG的各api作了良好的封装。_product_code_可以在CG的管理后台中找到。
<div class="descImg">
  <img src="/images/yottaa/product-info.png" />
</div>

一旦客户处于试用期间，新的，更多的工作就要开始了！第一要确保试用期的客户能够享受到我们优质的网络优化服务，客户的试用期有多长的时间，一周，半个月或者一个月？各个客户需要的试用期限可能各不同，试用期快结束时，应该发送一些邮件礼貌的提醒客户该付费了，试用期结束后，如果客户没有升级到一个付费的计划，那么需要停掉客户的试用计划，如果客户升级到了某一个付费计划，那么客户的billing数据也要做相应的调整。在我们本地系统中有一个Account model(省略其存储逻辑，我们实际用的是MongoMapper)，专门处理这一系列的业务。

~~~ruby
class Account
  key :plan_code, String
  key :action, String
  key :expire_at, Time
end
~~~

一个处于试用期的客户对应的account的数据结构如下，

~~~ruby
account.plan_code   #=> 'OPTIMIZER_FREE_TRIAL'
account.action      #=> 'optimizing'
account.expire_at   #=>  Sat Nov 12 22:18:21 +0800 2011
~~~

我们会启用一些后台进程，每天轮询数据库，检查account，如果临近过期比如离试用期结束只有三天了(通过检查expire_at这个字段)，
并且客户没有升级到一个付费的计划，那么我们会发送提醒邮件，如果已经过期并且客户没有升级到付费计划，那么应该停止客户的优化服务，
并且将account.action设置为'expired'，如果客户已经升级到一个付费计划，比如升级到decaa plan，那么我们会将
account.plan_code同步为'OPTIMIZER_DECAA'。如果客户想延长试用期限，只需要延长expire_at即可。是否为客户提供优化服务的
唯一依据就是检查account.action是否为'optimizing'。那么怎么判断客户升级到了付费计划呢?这个需要调用CG的[Get a Single Customer](https://cheddargetter.com/developers#single-customer) api来获取客户在CG的信息。

~~~ruby
# 此customer_code就是我们在CG创建一个新客户时，用来标识客户的那个记号
result = cg_client.get_customer(:code => "customer_code")
if result.valid?
  result.customer_subscription[:plan][:code]  #=> "plan code"
  result.customer_subscription[:plan][:plan]  #=> "plan name"
  result.customer_plan[:code]                 #=> "plan code"
  result.customer_plan[:name]                 #=> "plan name"
end
~~~

客户在试用期间觉得我们提供的优化服务确实能够提升他们网站的访问速度，增加了他们的收入，于是他们决定给我们掏钱了!
下一章讲的就是<a href="/2011/11/13/cheddargetter%E9%9B%86%E6%88%90%E4%B9%8B%E6%97%853.html">客户掏钱了!</a>
