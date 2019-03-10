---
layout: post
title: 订单流程重构记
---

我们的订单流程中有一个比较复杂的 checkout 页面, 用户一般会在这个 checkout 页面进行比较多的和订单相关的操作，比如添加或者修改 shipping address 和 billing address,  选择 shippping method, 选择支付方式, 还有使用礼品卡，电子钱包等操作.

![order checkout](/images/order_checkout.png)

首先说下我们遇到的问题, 我们的系统是用 Rails 开发的, 前端用的 js 库是 jQuery
- 问题 1, 为了实现 order checkout 这一个业务, 我们使用了过多的 controller 和 action

	比如说为了实现切换 shpping method 的逻辑，我们使用了单独的 action 去专门处理这一逻辑，前端操作之后，会把新的 shipping method id 和页面上的其它数据拼凑起来发给 action, 然后在 action 里返回新的数据给前端, 前端再用新的数据更新页面，一般会更新shipping fee, order total, 如果有 pickup location 返回还需要更新 pickup location.
	其它的操作比如使用礼品券，电子钱包，切换 payment method, 都建立了相应的 controller 和 action, 这样带来的后果是在 rails 这边我们要维护很多的 controller 和 action, 同时在页面这端我们要维护很多的 url, 比如我们几乎会在每一个可操作的元素上使用 data-url 来绑定对应的 url, 类似下面:
	
```html
	<div class="shippingMethodItem" data-url="/order/shipping_method_checkout">...</div>
	<div class="giftCardItem" data-url="/order/add_gc" data-delete-url="/order/remove_gc">...</div>
```

- 问题 2, 前端页面和 rails 后台程序的交互过程中没有使用标准且一致的数据结构

![checkout flows](/images/checkout_flows.png)

从上图我们可以看到整个的 checkout 逻辑被切割的支离破碎, 这导致无论是 rails 后台程序这边还是前端页面逻辑部分都需要花费很多心力去维护.

重构的目的就是为了解决上面的两个问题,

因此我们建立了一个标准的 checkout payload 模型用于前端和后端程序的交互, 将分散凌乱的 endpoint 规整为一个 checkout endpoint.

在前端页面上，我们埋入一个隐藏的 form 表单,

```erb
<% url    ||= '' %>
<% verb   ||= 'POST' %>
<% target ||= 'order' %>
<% url_for_place_order ||= '' %>
<% request_ajax ||= false %>
<%= form_tag url, id: 'backendCheckoutForm', class: 'hidden', method: verb, "data-url-for-place-order" => url_for_place_order, "data-request-ajax" => request_ajax do %>
  <%= text_field_tag "target", target %>
  <%= text_field_tag "event", event %>
  <%= text_area_tag "checkoutData", entity.to_json %>
<% end %>
```

将 checkout payload 以 json 字符串的形式存储在 name 为 checkoutData 的 textarea 中, 当用户操作 checkout 的时候，会将 textarea 里的 json 字符串取出，转换为 js object, 我们把这个 js object 叫为 entity,

```javascript

function getEntityFromBackendForm() {
    var $ck = $('#checkoutData');

    if(!$ck[0]){
        return {};
    }

    var ckVal = $ck.val();

    if (ckVal) {
        entity = JSON.parse(ckVal);
    } else {
        entity = {};
    }

    return entity;
}

``` 

用户在页面上的操作一旦涉及到数据的改变，我们会更新 entity 对应的数据，然后再将 entity 重新放入到 checkoutData textarea 中,

```javascript
function setEntityToBackendForm(entity) {
    var $ck = $('#checkoutData');

    if(!$ck[0]){
        return;
    }
    var res = JSON.stringify(entity);

    $ck.val(res);
}
```

最后以 form 表单的形式将 entity 提交给 checkout point,

```javascript
function submitBackendForm() {
    var $form = $('#backendCheckoutForm');
    $form.submit();
}
```

在 rails 后台程序这边，只需要一个 checkout endpoint 来处理页面传过来的 entity 即可:

```ruby

def checkout

    if params[:checkoutData].present?
      data = params[:checkoutData]
      data = JSON.parse(data)
      
      checkout_service(data)

    end
    
	...	
  end
```

这样以前分散凌乱的逻辑就抽象为了标准一致的逻辑了，重构后的流程图如下:

![standard checkout](/images/standard_checkout.png)

从上图中我们可以发现, checkout 的业务简化为了: 用户操作 -> 引起 checkout entity 的更新 -> 将 checkout entity 提交给 checkout endpoint .



