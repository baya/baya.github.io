---
layout: post
title: CheddarGetter集成之旅(一)创建产品和计划
---

公司有一项网络优化服务现在需要对客户收费了，我们决定采用CheddarGetter(CG)作为第三方计费系统。CG是一种周期性计费系统，如果你有一项服务需要每月或者每周即周期性的从用户那里收钱，那么就需要用到这种计费系统，而我们公司的服务需要每个月从用户那里收钱，用它正合适！<br/>
首先要做的是把网络优化服务改造成一种产品，改造过程比较简单，首先给产品取个名字 -> 产品下设计好若干付费计划 -> 每个付费计划定义好收费内容，最后就是把产品信息以及付费计划往CG那儿填一份，当然本地也要保存一份与CG一致的产品和计划信息，我们没有使用数据库去保存这些信息，而只是用了一份yml文件去保存，这样做的主要原因是为了方便，维护一份yml文件比维护几个表轻松的多，最重要的是产品和计划数据一旦定义好一般不会变动，不需要实时去CG同步这些数据。看了下面几张图，你会对整个改造计划有更加清晰的认识。
<div id="cg">
  <p class="step">
    第一步在CG上创建product(你需要在CheddarGetter上注册一个帐号)
    <img src="/images/yottaa/product-step1.png" alt="产品" />
  </p>
  <p class="step">
    在CG上创建一组付费计划 <br/>
    <img id="cg-plans" src="/images/yottaa/cg-price-plans.png" alt="CheddarGetter Price Plans" />
  </p>
  <p class="step">
  在本地保存一份与CG一致的付费计划(这张图没有显示free trial plan，这是因为在我们的服务中不需要客户主动的去注册一个free trial plan，enterprise这个计划有些特别，它在CG那边没有对应的计划，好像是对应的金额特别重大，需要人工操作，这个东西估计是boss激情的产物)，这些数据是直接从本地的yml文件中获取的，这样就相当于一个静态文件，速度快了不少。<br/>
    <img src="/images/yottaa/price-plans.png" alt="计划价格" />
  </p>
  <p class="step">
    Decaa付费计划的收费内容(在CheddarGetter中叫做item)，其他的付费计划与之类似。
    <img src="/images/yottaa/cg-decaa-items.png" alt="CheddarGetter Decaa Items" />
  </p>
</div>

plan和手机的包月套餐非常类似，比如你的手机使用某种套餐，月租费30，送100条短信，80分钟话费，这里月租费相当于plan的价格，100百条短信，80分钟话费就是plan的各个item，如果你发了200条短信，那么除了月租费，你还得出(200-100) = 100条短信的费用。以Decaa plan举例，它的价格是$29，包含两个sites, 100GB的带宽，10万个pv，如果一个客户使用了Decaa计划，那么每个月的固定支出是$29，客户优化了4个sites，那么需要再支付(4-2) x 10 = $20，带宽，pv的计算与site一样。CG提供了<a href='https://cheddargetter.com/developers#set-item-quantity'>Set Item Quantity</a>的api以帮助服务提供商将item的使用量提交给CG，此api的使用详情我会在后面的章节讲述。
