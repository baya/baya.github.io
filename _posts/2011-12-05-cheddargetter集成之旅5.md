---
layout: post
title:  CheddarGetter集成之旅(五)案例分析
---

SeriousCookie是一家在线销售美味饼干的站点，顾客们可以在SeriousCookie上浏览到五颜六色看起来很好吃的饼干，一旦看到喜欢的饼干，顾客就可以在线上下定单，支付然后等着饼干从天而降了。这么多饼干，顾客们当然无法一下子决定该买哪些饼干，于是她们点开一个又一个链接，希望能够找到最好的饼干，麻烦来了，SeriousCookie上面有大量的图片也可能是其它的原因，总之顾客们发现网页有时候很难打开，顾客的耐心总是有限的，于是一些顾客离开了，所以SeriousCookie的经营者们在确认自己的饼干绝对美味后，决定想办法提高自己网站的访问速度。他们觉得自己做饼干是一流的，但是一想到怎么提高网站访问速度就很茫然了。我们公司的网站优化服务正是为这样的客户开发的，首先他们的规模不是很大，没有专门的技术团队去做站点优化，甚至都没有专门的技术团队，但是他们有稳定的收益，站点的访问速度正在制约着他们业务的发展，他们需要简单高效，傻瓜式的解决方案。<br />

SeriousCookie首先在我们公司的站点[Yottaa]('https://www.yottaa.com')上注册了一个用户，然后激活了他们的站点，此时他们就自动进入了我们优化服务中的免费试用计划。一切顺利，SeriousCookie在试用期内(9月12号)升级到了decaa计划，我们最小的付费计划，其中含有$29的月租费，100K pageviews, 100GB bandwidth,可以激活两个site，不包含SSL证书。10天后(9月22号)客户升级到gigaa计划，gigaa计划含有$399的月租费，500K pageviews，500GB bandwidth，可以激活8个site，同时包含2份SSL证书。到了9月30号时，SeriousCookie消费了300k pageviews, 200GB bandwidth，使用了1份SSL证书。在10月2,3号的时候，我们向CG推送SeriousCookie的usgae，告诉CG，SeriousCookie消费了300K pageviews, 200GB bandwidth，使用了1份SSL证书。

~~~ruby

# push usage of pageviews
cg_client.set_item_quantity({:code => 'serious_cookie_code',
                             :item_code => 'pv_item_code',
                             :quantity => 300})
# push usage of bandwidth
cg_client.set_item_quantity({:code => 'serious_cookie_code',
                             :item_code => 'bandwidth_item_code',
                             :quantity => 200})
# push usage of site
cg_client.set_item_quantity({:code => 'serious_cookie_code',
                             :item_code => 'site_item_code',
                             :quantity => 1})
# push usge of ssl certs
cg_client.set_item_quantity({:code => 'serious_cookie_code',
                             :item_code => 'ssl_item_code',
                             :quantity => 1})

~~~

到了10月8号，CG帮我们从SeriousCookie收取gigaa计划的$399月租费，因为pageviews, bandwidth和SSL证书的使用量都没有超出gigaa计划的容量，
所以这些项目不会收取客户的费用，在此案例中我们使用了一种比较简单的计费策略，就是完全按照客户当前所处的计划来进行收费，至于客户的历史计划对此次的收费没有影响。
这样做的好处是程序简单直接，无论客户在一个月中多少次改变计划，都不会使我们的计算变得复杂，客户最后的计划才是我们需要关心的。
不足之处是客户可能会有抱怨，她可能会觉得9月份我只用了8天gigaa计划，却重重地扣了$399，我们会清楚地告诉客户我们的收费机制，
如果客户觉得中途升级会有损失的话，可以耐心的等到下个月的月初升级。目前为止这个客户觉得我们的服务很棒，访问速度的提升使客户的体验更好，
同时给这位客户带来了更多的流量和收入，所以他们觉得为优化掏的钱是值得的。

最后给出一张图说明我们billing系统的客户的整个状态迁移过程，
![billing flow](/images/yottaa/billing_flow.jpg )





