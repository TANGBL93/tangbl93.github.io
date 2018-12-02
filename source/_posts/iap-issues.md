---
title: 内购丢单问题解决方案
date: 2018-12-02 08:21:32
tags:
- IAP
---

# 内购丢单问题解决方案

> 截止到今天为止,该方案已在线上运行大概4个月左右，目前没有收到用户反馈有丢单问题。

## 之前的处理流程

> 其实之前我在网上查询怎么做内购的时候，发现大部分文章都是介绍的这种方法。

1. 通过服务器创建一个订单，并获取到 trade_no(订单号)，(total_fee订单金额)等数据
2. 请求内购产品信息，并将 trade_no,total_fee等 绑定到 transaction.payment.applicationUsername 上
3. 发起内购支付请求，并监听到内购支付的状态变化
4. 在支付成功状态后将 receipt 缓存起来，并执行 finishTransaction 操作
5. 将 receipt 等数据传给服务器做支付校验，服务器通过请求苹果后台做校验判断是否真实的支付
6. 在服务器校验成功后，将本地缓存清除
7. 如果有丢单情况，会在启动时从缓存中读取 receipt 去执行第4步

实际上，这套方案看似没什么问题。但是还是频繁收到用户反馈说有丢单，然后就有了之后的优化。

从我个人的经验来分析，主要有以下可能性:

1. 后台内购支付处理未做足够的校验(划重点)
2. 缓存未维护好，容易被利用刷钱或者是其他BUG(在这个优化需求出来之前，我司APP疑似被利用该BUG刷了几万块的内购币)
3. 支付中断，比如: 请求支付后退出APP，然后支付成功。等等其他类似的情况
4. 其他原因,等等...

## 优化之后的流程

1. 同上
2. 同上
3. 同上
4. 直接获取 receipt 、trade_no、 transaction_id 值去向服务器校验
5. 在服务器校验成功后，执行 finishTransaction 操作
6. 如果有丢单，或者无法访问服务器导致校验失败,在下次启动时，会重新走进来状态变化的代理方法
7. 如果服务器校验该收据不合法，也执行 finishTransaction 操作，并提示用户

这就是最终优化后的逻辑，这种方法可以保证用户成功支付之后，必然会在服务器校验后结束。

## 优化之后的问题

其实，在优化后，还是有问题的，transaction.payment.applicationUsername 经常获取不到。也就没办法在极端情况下获取到 trade_no 、total_fee 这些参数了。

所幸，还有个迂回的解决方案。在苹果服务器校验收据之后，会返回一个 JSON 数据，内容大致如下:

- environment、environment: 内购环境
- bundle_id、application_version: APP包名、版本
- in_app: 支付成功返回的数据，未支付成功也会返回JSON，但一定没有这个参数。

```
{
  "status": 0,
  "environment": "Production",
  "receipt": {
    "receipt_type": "Production",
    "adam_id": ...,
    "app_item_id": ...,
    "bundle_id": "",
    "application_version": "",
    "download_id": ...,
    "version_external_identifier": ...,
    "receipt_creation_date": "...",
    "receipt_creation_date_ms": "...",
    "receipt_creation_date_pst": "...",
    "request_date": "...",
    "request_date_ms": "...",
    "request_date_pst": "...",
    "original_purchase_date": "...",
    "original_purchase_date_ms": "...",
    "original_purchase_date_pst": "...",
    "original_application_version": "...",
    "in_app": [{
      "quantity": "1",
      "product_id": "<#product_id#>",
      "transaction_id": "...",
      "original_transaction_id": "...",
      "purchase_date": "...",
      "purchase_date_ms": "...",
      "purchase_date_pst": "...",
      "original_purchase_date": "...",
      "original_purchase_date_ms": "...",
      "original_purchase_date_pst": "...",
      "is_trial_period": "false"
    }]
  }
}
```

在后台去拿到这个JSON之后，需要做以下处理:

1. 首先就要去校验`内购环境`, 我司方案是先校验不是 `"Production"` 的话就报错。仅允许内部用户使用沙箱模式(切记:沙箱模式内购不要钱)
2. 然后就校验有木有 `in_app` 参数，有的话就是正常支付，直接往下走。否则就是异常支付，当错误处理，并返回异常信息。
3. 校验通过后，查看时候有收到 `trade_no`、`total_fee` 参数，有就完事大吉，正常处理支付成功逻辑就OK。
4. 如果没有这俩参数，就从 `quantity`(价格等级) 或者 `product_id`(内购商品ID) 去生成 `total_fee`
5. 生成 `total_fee` 之后，`trade_no` 就可以根据 `total_fee`、`original_purchase_date` 以及后台该用户未支付的订单中去匹配最合适的当成支付成功

然后就这样，这套方案就基本没什么问题了。主要就是在后台这块儿需要多处理些漏参数的问题，其他都还好。有丢单现象的同学可以试一试这种方法。

# 推荐阅读

如果你刚开始做内购，对于其中很多东西还不太明白，推荐你看一看贝聊科技的 NewPan 写的这几篇博客。

- [[贝聊科技]贝聊 IAP 实战之满地是坑](https://juejin.im/post/5a3b14f36fb9a045104aa6c8)
- [[贝聊科技]贝聊 IAP 实战之见坑填坑](https://juejin.im/post/5a3b164e6fb9a0450e76466b)
- [[贝聊科技]贝聊 IAP 实战之订单绑定](https://juejin.im/post/5a3b169151882521033469b4)