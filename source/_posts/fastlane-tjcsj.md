---
title: 基于fastlane的实践-添加测试机
date: 2018-08-18 08:58:06
tags:
- fastlane
---

这个呢，其实也不怎么复杂。达到的效果也比较好，可以将减轻工作负担。

## 脚本

由于这个比较独立，不依赖与代码。可以单独创建个空白项目。然后编辑 `Fastfile`

主要有以下几点注意:

1. 如果这个号有多个证书，可以先将 `cert_id: ENV["CERTID"],` 这一行注释掉，然后从打印中找到对应的证书ID再打开
2. 如果仅有一个证书，那么可以将 `cert_id: ENV["CERTID"],` 这一行注释掉
3. 具体的值需要通过环境变量来配置的，请注意哦
4. 这里只配置了AdHoc打包文件，如果需要同时更新，可以简单修改一下 `sigh` 命令那部分
5. `register_devices` 支持读取文件的，平时可以收集整理好，换新马甲时可以直接读取

```
desc "添加测试机"
lane :tjcsj do

    keychain_pass = sh("cat ~/.keychain_pass_file")
    sh "security unlock-keychain -p #{keychain_pass}"
    unlock_keychain(path: "~/Library/Keychains/login.keychain", password: "$(cat ~/.keychain_pass_file)")

    ## 添加新设备
    register_devices(
        devices: {
            "备注名1" => "UDID值1",
            "备注名2" => "UDID值2",
        },
        username: ENV["USERNAME"]
    )
    ## 更新证书文件到本地
    sigh(
        adhoc: true,                                      # 申请Adhoc版本
        force: true,                                      # 强制更新到所有设备的证书文件
        username: ENV["USERNAME"],                        # APPLE账号
        app_identifier: ENV["APPID"],                     # APPID
        cert_id: ENV["CERTID"],                           
        output_path: ENV["PPPATH"],                       # 存储证书文件路径
        provisioning_name: ENV["PPNAME"],                 # 线上证书文件名称
        filename: "#{ENV["PPNAME"]}.mobileprovision"      # 存储证书文件名称
    )
end
```
