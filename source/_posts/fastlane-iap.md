---
title: 基于fastlane的实践-新建内购项目
date: 2018-08-18 08:58:46
tags:
- fastlane
---

如果APP有内购功能的话，就需要在后台添加内购产品了。如果添加过的话，那么一定会觉得很麻烦。这里用脚本来实现。

## 注意点

1. 不允许有同名内购项目的存在，删除的内购ID之后没法再使用。
2. 这个脚本本身依赖于 `spaceship` , 而 `spaceship` 属于 `fastlane` 的底层驱动逻辑。具体的可以去看看源代码
3. 这个脚本属于 `ruby` 脚本，所以可以当成是 `ruby` 代码执行。
4. 本脚本仅创建消耗品内购产品(CONSUMABLE),其他类型的可以参考实现

## 脚本内容

有以下几点注意:

1. 用户名和密码需要在命令行输入，这个是基于 `fastlane` 的核心实现的
2. `screenshot` 表示备注截图，这里使用的同一张图片，请注意
3. 在创建时的 `versions` 字段中填写国际化信息，此处仅有中文
4. `review_notes` 表示审核备注信息
5. `pricing_intervals` 表示价格信息，这里仅固定价格，根据需要调整
6. `iap_id` 表示内购商品ID固定前缀，按具体情况来做
7. 商品名长度必须大于10，否则会失败。请注意

```
require "spaceship"

# 参数

## 用户名和密码
username = "在此处填入APPLE账号"
password = "在此处填入APPLE密码"          

## APPID
appid = "在此处填入应用ID" 
## 屏幕快照路径
screenshot = "在此处填入屏幕快照路径"

# 登录账号
Spaceship::Tunes.login(username, password)
# 获取指定APP
app = Spaceship::Tunes::Application.find(appid)

# 创建新的内购项目

## 内购商品ID格式，可按需求来实现
iap_id = "#{appid}.offical."

## 消耗型项目
## 6元=36个钻石
## 18元=120钻石
## 30元=206个钻石
## 98元=678个钻石
## 268元=1868个钻石
## 548元=3828个钻石

## reference 长度必须大于10,tier表示价格等级
iap_items = [
    {"id" => "07", "reference" => "6元 = 36个钻石", "tier" => "1"},
    {"id" => "08", "reference" => "18元 = 120钻石", "tier" => "3"},
    {"id" => "09", "reference" => "30元 = 206个钻石", "tier" => "7"},
    {"id" => "10", "reference" => "98元 = 678个钻石", "tier" => "15"},
    {"id" => "11", "reference" => "268元 = 1868个钻石", "tier" => "42"},
    {"id" => "12", "reference" => "548元 = 3828个钻石", "tier" => "57"},
]

puts "-------开始创建内购项目-------"

iap_items.each do |item|
    puts "正在创建的内购项目:#{item["reference"]}"
    ## 实际的创建命令
    app.in_app_purchases.create!(
        ## 内购类型
        type: Spaceship::Tunes::IAPType::CONSUMABLE, 
        ## 国际化描述信息
        versions: {
            "zh-Hans" => {
                name: item["reference"].gsub(/\s+/, ""),
                description: item["reference"]
            }
        },
        cleared_for_sale: true,
        reference_name: item["reference"].gsub(/\s+/, ""),
        product_id: "#{iap_id}#{item["id"]}",
        review_notes: nil,
        review_screenshot: screenshot,
        ## 价格信息
        pricing_intervals: [
            {
                country: "WW",
                begin_date: nil,
                end_date: nil,
                tier: item["tier"].to_i
            }
        ] 
    )
end

puts "-------创建内购项目结束-------"
```
