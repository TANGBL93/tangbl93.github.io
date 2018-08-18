---
title: 基于fastlane的实践-自动化打包(下)
date: 2018-08-18 08:56:31
tags:
- fastlane
---

上篇说了一些问题，这篇文章就来看一下怎么打包过程。这里就按照步骤来说一下。

# 需求

按照我们APP现有的需求，先列举一下:

1. 打包的版本号(版本号，编译版本号)必须为偶数(便于优先追踪线上、内测版BUG)
2. 推送到 `fir.im`，且成功后发送一条消息到钉钉
3. 打包结束自动修改版本号、打标签并推送
4. 由于可能会发测试版给外部用户安装，所以需要发送特定版本号给用户
5. 需要支持打混淆分支，并将版本号同步非混淆分支

# 非混淆分支

首先，按照常规步骤先走一遍，来看看是如何做的。

```
desc "打测试包专用"
  lane :fir do
    ...接下来这个步骤的所有命令全部在这里...
  end

```

## 1. 解锁远程机器钥匙串

因为我个人本身还是习惯直接通过 `ssh` 的，所以按照上一篇的加上

```
# unlock_keychain
keychain_pass = sh("cat ~/.keychain_pass_file")
sh "security unlock-keychain -p #{keychain_pass}"
unlock_keychain(path: "~/Library/Keychains/login.keychain", password: "$(cat ~/.keychain_pass_file)")
```

## 2. 拉取最新代码，重置本地修改

毕竟我们打包肯定要用最新代码的吧，所以这一步做了一些事，且保证不会和我们上传的有区别:

- 获取到当前的分支名: 有几次发现某些马甲包无法自动识别分支，导致需要手动操作，其中 `gsub` 表示去除空行、换行符等
- 然后首先重置一次，再拉取服务器最新代码，其中 `--no-edit` 表示自动合并，否则可能拉取之后需要手动输入合并信息
- 最后执行 `pod update` 并再次重置,主要是有时候更新会导致工程文件变化。使用 `--no-repo-update` 纯粹是因为网络不好

```
# git branch
branch_name = sh("git rev-parse --symbolic-full-name --abbrev-ref HEAD").gsub(/\s+/, "")

# git pull
sh "git reset --hard"
sh "git pull origin #{branch_name} --no-edit"

# pod update
sh "pod update --no-repo-update"
sh "git reset --hard"
```

## 3. 修改版本号

这里有点坑，`fastlane` 自带的那个 `increment_build_number` 有点坑，没办法使用。所以只能就手动读取、设置

- 这里做了个兼容，也就是当版本号为x.x.0，或者当前已经为偶数时，会做不同的处理，其他时候为正常+1
- #{ENV["PLISTPATH"]} 表示读取环境变量中 `PLISTPATH` 的值

```
# set version
prev_apk_version = get_info_plist_value(path: "#{ENV["PLISTPATH"]}", key: "CFBundleShortVersionString")
prev_build_version = get_info_plist_value(path: "#{ENV["PLISTPATH"]}", key: "CFBundleVersion")

apk_versions = prev_apk_version.split(pattern=".")

step = 1
if apk_versions[2].to_i == 0
    step = 0
elsif (apk_versions[2].to_i % 2) == 0
    step = 2
end

current_apk_version = "#{apk_versions[0]}.#{apk_versions[1]}.#{apk_versions[2].to_i+step}"
current_build_version = "#{prev_build_version.to_i + step}"
next_apk_version = "#{apk_versions[0]}.#{apk_versions[1]}.#{apk_versions[2].to_i+step+1}"
next_build_version = "#{prev_build_version.to_i + step + 1}"

set_info_plist_value(path: "#{ENV["PLISTPATH"]}", key: "CFBundleShortVersionString", value: current_apk_version)
set_info_plist_value(path: "#{ENV["PLISTPATH"]}", key: "CFBundleVersion", value: current_build_version)
```

## 4. 打包内购包

这个步骤也就是指定了打包目录和打包类型，没什么好说的。

```
# output path
output_name = "#{ENV["SCHEMA"]}_v#{current_apk_version}.ipa"
output_directory = "~/Desktop/ad-hoc/#{ENV["TAGNAME"]}/"
output_path = output_directory + output_name

# gym build
gym(
    scheme: ENV["SCHEMA"], 
    workspace: "ksh3.xcworkspace", 
    include_bitcode: false,
    export_method:"ad-hoc",
    output_name: output_name,
    output_directory:output_directory)
```

## 5. 上传、推送消息

- 上传到 `fir.im` 并从消息中读取到 Release id，这样子可以拼接成指定版本的链接(之前我们都是手动去网站找的，累)
- 如果没有这个功能的话，建议更新一下。或者直接用 `fir publish -s #{ENV["FIRLINK"]} #{output_path}`
- 拼接一条消息推送到钉钉上，方便测试人员下载

```
# fir cli & grep Release id
release_id = sh("fir publish -s #{ENV["FIRLINK"]} #{output_path} | grep -i 'Release id is ' | grep -o -E '[a-z0-9]{16,}'")

# dingtalk
upload_msg = " #{ENV["APPNAME"]}v#{current_apk_version}"
upload_url = "https://fir.im/#{ENV["FIRLINK"]}?release_id=#{release_id}"
dingtalk(access_token:dingtoken, title:upload_msg, message:upload_msg, more_url: upload_url)
```

## 6. 收尾工作

- 重置一次当前的修改，保证当前环境的干净
- 再拉取一次服务器代码。如果在打包时有同事提交了代码，就会导致提交时报错误
- 打标签。这里要注意标签不能重复，否则会报错误
- 再次修改版本号为开发版本号，也就是单数
- 提交，推送，结束

```
# update tags & version
sh "git reset --hard"
sh "git pull origin #{branch_name} --no-edit"

sh "git tag #{ENV['TAGNAME']}_#{current_apk_version}"

set_info_plist_value(path: "#{ENV["PLISTPATH"]}", key: "CFBundleShortVersionString", value: next_apk_version)
set_info_plist_value(path: "#{ENV["PLISTPATH"]}", key: "CFBundleVersion", value: next_build_version)

sh "git add ."
sh "git commit -a -m 'update version'"

sh "git push origin #{branch_name} --tags"
sh "git push origin #{branch_name}"
```

# 混淆分支

由于马甲包为了上架做了些混淆工作，所以会创建一个单独的 `分支名_appstore` 的分支作为混淆分支，这样会便于同步最新代码

主要步骤:
    
    1. 拉取非混淆分支最新代码
    2. 拼接分支名，切换到非混淆分支
    3. 合并非混淆分支代码到非混淆分支(这一步骤需要检查，可能有改动会造成冲突)
    4. 通过正常打包逻辑打混淆包
    5. 打包结束后，切换到正常分支
    6. 遴选到最后更新版本的那条记录到正常分支，然后推送到服务器

```
desc "打混淆包专用"
lane :mix do


# git branch
branch_name = sh("git rev-parse --symbolic-full-name --abbrev-ref HEAD").gsub(/\s+/, "")
# 混淆后的分支名，默认为:分支名_appstore
appstore_branch = "#{branch_name}_appstore"

# 拉服务器最新代码
sh "git reset --hard"
sh "git pull origin #{branch_name} --no-edit"

# 切换到混淆分支打包，之后切回来，遴选更新版本的变更到主分支
sh "git checkout #{appstore_branch}"
sh "git pull origin #{appstore_branch}"
sh "git merge #{branch_name}"
fir
sh "git checkout #{branch_name}"
sh "git cherry-pick #{appstore_branch}"
sh "git push origin #{branch_name}"
end
```

# 完整代码

其实上边已经把我整个打包脚本代码贴出来了，其中也没有什么敏感信息。

## 环境变量文件

> 文件保存到和 `Fastfile` 同级，名称应为 `.env.xxx`。命令记得带上 `fastlane fir --env xxx`

```
// .env.ksh3
SCHEMA=""                       # schema的名字
PLISTPATH=""                    # Info.plist 的位置，路径为相对于程序根目录
FIRLINK=""                      # fir.im 的短链，如 wcfg2018
APPNAME=""                      # APP名，中英文随意
TAGNAME=""                      # git标签名，叫什么随意，不过最好有点意义，可以用拼音或者英文名 
```

## Fastfile

这里我按照上篇的考虑，选择了从远程获取，这样可以始终保持最新版本

```
import_from_git(url: '....git', path: 'Fastfile')
```