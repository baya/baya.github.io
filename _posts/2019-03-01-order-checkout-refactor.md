---
layout: post
title: 订单流程重构记
---

我们的订单流程中有一个比较复杂的 checkout 页面, 用户一般会在这个 checkout 页面进行比较多的和订单相关的操作，比如添加或者修改 shipping address 和 billing address,  选择 shippping method, 选择支付方式, 还有使用礼品卡，电子钱包等操作.

![order checkout](/images/order_checkout.png)

首先说下我们遇到的问题, 我们的系统是用 Rails 开发的, 前端用的 js 库是 jQuery
- 问题 1, 为了实现 order checkout 这一个业务, 我们使用了过多的 controller 和 action

	比如说为了实现切换 shpping method 的逻辑，我们使用了单独的 action 去专门处理这一逻辑，前端操作之后，会把新的 shipping method id 和页面上的其它数据拼凑起来发给 action, 然后在 action 里返回新的数据给前端, 前端再用新的数据更新页面，比如更新shipping fee, order total, 如果有 pickup location 返回还需要更新 pickup location.
	其它的操作比如使用礼品券，电子钱包，切换 payment method, 都建立了相应的 controller 和 action, 这样带来的后果是在 rails 这边我们要维护很多的 controller 和 action, 在页面这端我们要维护许多的 url, 比如我们几乎会在每一个可操作的元素上使用 data-url 绑定对应的 url, 类似下面:
	
```html
	<div class="shippingMethodItem" data-url="/order/shipping_method_checkout">...</div>
	<div class="giftCardItem" data-url="/order/add_gc" data-delete-url="/order/remove_gc">...</div>
```

- 问题 2, 前端页面和 rails 后台程序的交互过程中没有使用标准且一致的数据结构
