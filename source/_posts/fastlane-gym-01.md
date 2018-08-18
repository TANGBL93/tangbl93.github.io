---
title: 基于fastlane的实践-自动化打包(上)
date: 2018-08-18 08:55:11
tags:
- fastlane
---

> 上篇先说一下打包的周边问题(也可以先跳过)，下篇再聚焦于打包本身。

# 配置机器

由于是在其他机器上打包，所以需要:

1. 将自己电脑和打包机器在同一个局域网下(或者说同一个WIFI)
2. 在 `系统偏好设置` -> `共享` 中打开远程登录、屏幕共享、文件共享三个权限
3. 将马甲包重新命名放在用户目录下，方便单独打包
4. 配置好一份别名，方便直接打包

然后平时打包就是通过 `访达->共享屏幕` 去访问，或者说 `ssh(远程登录)` 链接上去。如果需要远程在家的话，可以考虑挂一个 `TeamViewer`

# 遇到的问题

## 无法读取钥匙串

如果是通过 `ssh方式` 远程登录到其他macOS机器上，直接打包可能由于 `Keychain` 未解锁的问题导致打包失败。

解决办法是在远程机器桌面创建一个登录密码文件,假如当前登录密码为 `12345678`:

```
// 也可以手动创建一份文件，注意文件名需要和后边的匹配
echo 12345678 > ~/.keychain_pass_file
```

然后在脚本最前面，加上:

> 注意:脚本第一行会打印密码出来。如果对这有顾虑的可以研究换一种方法

```
# 解锁远程机器上的钥匙串
keychain_pass = sh("cat ~/.keychain_pass_file")
sh "security unlock-keychain -p #{keychain_pass}"
unlock_keychain(path: "~/Library/Keychains/login.keychain", password: "$(cat ~/.keychain_pass_file)")
```

## Fastfile共用

由于有多个马甲包，且可能由不同人负责的，且打包脚本也有可能更新，如何让他们共用同一份Fastfile呢。

稍微查了下资料后，发现有一个 [`环境变量(env)`](https://docs.fastlane.tools/advanced/#environment-variables) 和 `远程文件` ，完全满足这个需求。

环境变量:
    - 类似于 `xcconfig`,配置键值对，然后在 `Fastfile` 中通过 `ENV["Key"]`来获取不同值
    - 在打包时，执行 `fastlane fir --env weixin` , 其中 `weixin` 必须和 `env` 文件名一致，如: `.env.weixin`
远程文件: 可以导入远程文件，来实现始终为最新版本的打包脚本

## 打包在签名时询问密码

初次打包时，可能会询问密码,这个时候，可以用这一行命令来解决

```
security set-key-partition-list -S apple-tool:,apple: -k "$(cat ~/.keychain_pass_file)" ~/Library/Keychains/login.keychain
```

也可以到钥匙串里，设置证书的访问控制权限来达到目的。

![keychain_access.png](/images/keychain_access.png)
