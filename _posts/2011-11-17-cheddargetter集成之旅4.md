---
layout: post
title: CheddarGetter集成之旅(四)紧张的8号
---

每个月的8号是我们向客户收钱的日子，哈哈这时候钱才会真正地流向我们公司的帐号。到了这一步时，我可以对整个的CheddarGetter集成之旅做一个小小的总结:

1. 客户A试用我们的产品
2. 客户A订购一个付费计划
3. 在每个月的3号向CG push上个月A的消耗量
4. CG在8号根据我们 push的数据和订购的计划计算出上个月A的费用，并将这些钱转到我们的帐号里
5. 检测A是否已经付款并决定是否继续为其提供优化服务

1,2两步在前面的三篇博客中已经介绍过了，先讲第4步。假设客户A是在10月19号成功订阅了我们的一个付费计划DECAA，默认情况下CG会将收费日期设定为下个月的19日即11月19日，以后的收费日期以此类推。所谓的默认情况就是我们不做任何干涉。为了把收费日期统一为每个月的8号，我们要做的就是在客户订阅付费计划成功时，调用CG的api，主动将收费日期设定为11月8号，这样聪明的CG就会明白我们是想在每个月的8号收这个客户的钱，设定收费日期只需一次，不需要每次提前去设定。
改变收费日期其实是对subscription的一个update操作，我们需要利用CG的<a href='https://cheddargetter.com/developers#update-subscription'><strong>Update a Subscription Only</strong></a> api
~~~ruby
  bill_date = Chronic.parse('8th day next month').strftime('%Y-%m-%d')
  cg_client.edit_subscription({:code => "account_code", :changeBillDate => bill_date)
~~~
一旦设定好了收费日期为8号，CG就会在每个月的8号主动帮我们从客户那儿收钱，现在可以讲第3步了。客户A在10月19号至10月31号之间激活了3个site，使用了400GB的bandwidth，那么超出的费用是(3-2)*20 + (400 - 200)* 1 = $220，当然这个不需要我们来计算，我们只需要把这些使用数据推送给CG就行了。

推送客户用量是通过CG的<a href='https://cheddargetter.com/developers#set-item-quantity'><strong>Set Item Quantity</strong></a> api来进行的。使用cheddargetter_client_ruby这个gem时，推送这一过程会变得更加直接和简单。
假设客户A的code是'customer_A', site, bandwidth的item code分别是'site_item_code', 'bw_item_code'

~~~ruby
cg_client.set_item_quantity(:code => 'customer_A',
                            :item_code => 'site_item_code',
							:quantity => 3)
							
cg_client.set_item_quantity(:code => 'customer_A',
                           :item_code => 'bw_item_code',
						   :quantity => 400)
~~~

10月19号-10月31号之间只有12天，所以套餐费$29不能全扣，应该给用户一些返利，CG提供了两种计算方法，我们选择了最简单的

    (29 / 30) * 18 = $17.4

$17.4是应该退还给客户A的钱。CG对这种情况提供了详细的文档支持[Pricing Plan Changes (Upgrade/Downgrade)](http://support.cheddargetter.com/kb/pricing-plans/pricing-plan-changes-upgradedowngrade)，我现在提到的这种情况比较特殊，就是客户A没有计划的改变，只是第一个月的使用天数不满30天，但是和文档所说的情况本质上是一样的，那就是**多退少补**。
现在说重点，怎么把这$17.4退还给客户？准确的说不能叫**退还**，因为此时我们还没有收到客户的钱，我们应该在收钱之前把该收的钱都计算准确。
为了实现这个目标，我们需要使用CG的[Add a Custom Charge/Credit](https://cheddargetter.com/developers#add-charge) api，这个api初看是用来加钱的，但是把钱设置成一个负数，就可以起到"退钱"的作用了。

~~~ruby
cg_client.add_charge(:code => 'customer_A',
                     :chargeCode => 'credit',
					 :quantity => 18,
					 :eachAmount => -29/30)
~~~

上面的代码很好理解，quantity就是客户A没有多少天没有使用我们的计划，eachAmount是每天的费用，设置成负数就可以达到替客户减钱的效果了。到目前为止，CG对我们来说还是一个黑盒子，虽然我们能够调用它提供的api来帮我们收钱，但是CG是怎么收钱的呢？现在我要说说CG最重要的概念**Invoice**，中文的意思就是**发票**，在我看来CG就是通过这些一张张的发票驱动起来的。
任何时候，CG都会给客户提供一张[Current Invoice](/images/yottaa/cg-invoice.png)，CG就是通过这张发票来向客户收钱的，我们前面做的修改bill date，
推送客户用量，以及多退少补等操作都最终会在上面这张发票上得到体现。为了证实我的说法，我做一个试验，就是给用户返利（多退少补）。
下面是全部的代码：

~~~ruby
require 'cheddargetter_client_ruby'

# 下面的一些帐号信息做了保密
CGEmail       = 'xxxxx'
CGProductCode = 'optimizer'
CGPassword    = 'xxxxxx'
CustomerCode  = 'xxxxxxxx'

cg_client = CheddarGetter::Client.new(
                               :product_code => CGProductCode,
                               :username     => CGEmail,
                               :password     => CGPassword
                               )

# chargeCode是我们自己定义的，可以为任意字符串，长度限制在36个字符内
# eachAmout确保精确到小数点后两位，也就是精确到分
result = cg_client.add_charge(:code       => CustomerCode,
                              :chargeCode => 'credit',
                              :quantity   => 18,
                              :eachAmount => "%.2f" % (-129.0 / 30)
                               )
~~~

我们看到不需要指定对哪张发票进行操作，CG会自动帮我们把所有的操作指向当前的发票即**Current Invoice**。

![Current Invoice](/images/yottaa/cg-invoice-credit.png)

我们看到current invoice里面多了一个计费item叫做Credit，当然这个item计费的金额是负数，其实就是向客户返利。
于是我们如果想控制CG的收费行为，要做的事情很简单就是操作current invoice，当然CG的核心业务就是自动且循环的生成这些发票（如果没有人为打断的话），
这样我们的收费就能按照我们的预期目标持续不断的进行下去，所以在我看来invoice是CG里最重要的一个东西，如果一个开发者理解了invoice，那么CG的集成之旅对他来说就会变的很平淡无奇了:)。

这次旅行的最后一步就是检查钱是否已经到帐了，如果到帐了，就继续为客户提供优化服务，否则就好言相劝，让客户交钱，如果客户意志坚决不肯交钱了，那么就停掉客户的服务。
所有的invoice又分成两种类型，一种是one-time，另外一种是subscription。
one-time就是一次性的，打个简单的比方，one-time相当于罚单，即开即收钱，并且这次交了钱就罚完了，不会每个月每个月的让你交钱，subscription就相当于税条，这次交了钱，下个月得继续交，没完没了。

~~~ruby
 # 所有的发票信息都包含在客户信息里面
 result = cg_client.get_customer(:code => CustomerCode)
 # 获取客户的current invoice
 current_invoice = result.customer_invoice

 # 检查客户9月份的invoice是否已经支付
 # 10月份的invoice的billing date应该是11月8号

 # 首先获取客户所有的invoices
 # 然后通过billing date日期遍历获取客户10月份的invoice
 # 注意客户在每个月只有一张subscription类型的invoice
 # 但是one-time类型的invoice可能无也可能有多张
 # 因为所有的one-time invoice都是即开即收钱，所以one-time invoice
 # 都是paid，因此我们只需检查subscription invoice是否已经paid

 date = Time.utc(2011, 11)
 invoice = result.custome_invoices.detect {|invoice|
   invoice[:billingDatetime].strftime("%Y-%m") == date.strftime("%Y-%m") \
   and invoice[:type] == 'subscription'
 }
 
 # 若'paidTransactionId'不为空，则invoice已经支付，否则就是未支付
 if invoice
   invoice[:paidTransactionId].nil? ? 'unpaid' : 'paid'
 end
~~~

在上面的代码中，我们不能通过检查客户的current invoice来判断客户是否已经付款，因为一旦CG从客户那儿扣款成功，就会立即生成一张新的current invoice，
也就是说current invoice永远都是unpaid。我在上面的代码注释中给出了找到正确的invoice方法。
现在我们已经知道怎么判断客户是否已经支付，剩下的路程无论是继续为客户提供稳定优质的服务，还是停止给客户服务都已经与CheddarGetter无关了，
当然还会有些意外的情况会发生，某个客户可能不想使用我们的服务了，她会在中途停止我们的服务或者她想改变目前使用的计划，这些我将在下一章谈谈。
