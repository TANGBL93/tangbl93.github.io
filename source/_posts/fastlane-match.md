---
title: 基于fastlane的实践-管理证书
date: 2018-08-18 08:57:25
tags:
- fastlane
---

其实本来是打算基于 [`match`](https://docs.fastlane.tools/actions/match/) 做证书管理的，最后还是不符合要求。

其实用法非常简单，查查文档就行了。主要的问题在于无法识别当前是否已经安装过这个账号的证书，就会导致重复创建。

## 问题/坑

- 苹果后台限制账号发布证书最多两个，如果包很多就没办法了。但是恰恰我们有些账号下不止一个APP
- 除非是就一个账号下就一个APP，且多人开发，用这个才有意义
- 网上好像有人也在研究这个，可以搜索一下有没有解决方案

## 创建证书

```
# 创建开发与发布证书
lane :setup_certs do
    match(app_identifier:ENV["APPID"], username:ENV["USERNAME"], type: "adhoc", git_url: ENV["MATCHURL"], shallow_clone: true, skip_docs:true, git_branch:ENV["APPNAME"])
    match(app_identifier:ENV["APPID"], username:ENV["USERNAME"], type: "appstore", git_url: ENV["MATCHURL"], shallow_clone: true, skip_docs:true, git_branch:ENV["APPNAME"])
    match(app_identifier:ENV["APPID"], username:ENV["USERNAME"], type: "development", git_url: ENV["MATCHURL"], shallow_clone: true, skip_docs:true, git_branch:ENV["APPNAME"])
end
```

## 使用证书

在打包脚本中加上这一段

```
# 切换到打包证书
match(app_identifier:ENV["APPID"], type: gym_type, git_url: ENV["MATCHURL"], shallow_clone: true, readonly:true, git_branch:ENV["APPNAME"])
```
