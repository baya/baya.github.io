---
layout: post
title: CheddarGetter集成之旅(三)客户掏钱了！
---


付费的客户分为两种:从试用期过来的和无试用期，直接付费。<br/>
这两种客户的付费流程在形式上是一样的，但是却有着本质的区别。第一种客户是从免费的计划升级到付费的计划，是update的操作；第二种用户是从无到有订购一个付费计划，是create的操作，但是他们的入口都是一样的：
<div id="cg">
  <p class="sign-step">
    首先，在下面的表单中选中一种合适的付费计划，假设客户选中了<strong>decaa</strong>计划。
    <img src="/images/yottaa/price-plans.png" alt="计划价格" />
  </p>
  <p class="sign-step">
    引导客户到达下面的确认页面，确认页面有两个作用，第一是让客户确认眼前的这个计划是不是自己所需要的；第二就是分流的作用，前面说的两种客户通过确认页面会被引到不同类型的<a href="#hpp">hpp</a>页面上去。
    <img src="/images/yottaa/confirm-plan.png" alt="确认计划" />
  </p>
  <p class="sign-step">
    直接付费的客户的hpp页面。
    <img src="/images/yottaa/hpp-create.png" alt="hpp of creating" />
    可以对<strong>https://yottaa.chargevault.com/<span class="red">create</span></strong>发起GET或者POST请求并加入适当的参数，就可以跳转到此hpp页面。
  </p>
  <p class="sign-step">
    <img src="/images/yottaa/hpp-update.png" alt="hpp of creating" />
    可以对<strong>https://yottaa.chargevault.com/<span class="red">update</span></strong>发起GET或者POST请求并加入适当的参数，就可以跳转到此hpp页面。
   从客户的角度来看，这两种hpp页面没有什么区别，要填的数据都是一样的。
  </p>

  <p class="sign-step">
    客户终于填好了各种数据，点击提交，CG告诉客户订购计划成功，cool!客户终于可以舒畅地享受我们提供的优质服务了。
    看起来一切都很顺利，但问题是我们怎么知道客户已经向CG成功提交信用卡信息了(还有其他信息，比如名字，email，但是最关键的信息是信用卡)？
    当然我们不能指望客户给我们打个电话或者发封邮件告诉我们，她已经提供信用卡了。最有能力干这件事情的是CG，可是CG却没有提供这种通知功能，这很像“有关部门”的作风。
    更准确的说CG提供了一种很弱的通知功能，弱到我们不敢使用。我们可以在CG里设置一个return url，当客户在hpp页面(create或者update)成功提交信用卡信息后，CG会把客户引到一个成功页面，然后如果客户比较有耐心，等待几秒钟(这个等待时间可以设置)就会跳转到这个return url，这个通知功能弱暴了，如果客户没有什么耐心，直接关了页面，或者浏览器崩溃了，那我们就等着客户投诉吧！没办法，我们只能写个后台进程每两分钟就去请求一下CG，获取客户的信息从而判断客户是否已经订购了某种付费计划，所以说CG不给人方便，那咱就让他的服务器不消停。
 </p>
</div>

<div id="hpp">
  <h2>什么是HPP?</h2>
  <p>
    HPP是Hosted Payment Pages的简称。简单的说，hpp是CG提供给你的一个专有页面，供你create，upgrade&update和cancel CG用户的。为什么需要这么一个页面呢？
    举一个简单但是很重要的例子，如果一个客户升级到一个付费计划，那么CG应该知道这个客户的信用卡信息，才能从客户那里扣钱，怎么把该客户的信用卡提供给CG呢？理想情况是让客户首先把信用卡信息提交到我们这，然后我们调用CG的api把客户的信用卡信息告诉CG，这样客户的整个升级过程都在我们的掌控之中，但是这里最大的问题是我们首先要取得一种叫做PCI的资质才能收集客户的信用卡信息，而PCI资质的审核相当严格，比较难获取，而通过hpp页面，把客户的信用卡信息直接提交给CG，就可以避免PCI审核的麻烦了。
   hpp是可以配置的，你可以设置它的url，样式等。 HPP详细的文档在<a href="http://support.cheddargetter.com/kb/hosted-payment-pages/hosted-payment-pages-setup-guide"><strong>Hosted Payment Pages Setup Guide</strong></a>
   <img src="/images/yottaa/hpp-config.png" alt="hpp配置" />
  </p>
</div>
