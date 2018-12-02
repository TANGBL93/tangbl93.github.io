---
title: 基于fastlane的实践-通过Archive生成IPA
date: 2018-12-02 07:48:00
tags:
- fastlane
---

之前说过想做 <code>基于 `AdHoc` 打包的文件，自动生成 `AppStore` 的 `ipa`</code> ,前段时间抽空研究了下,已经测试导出的IPA可以正常上传到App Store。大致步骤如下。

# 1. gym

在打包时，指定 archive_path , 保存 archive。(默认的 archive 路径比较复杂)

```
gym(
    ...
    archive_path:"./MyAppV1.0.xcarchive")
```

# 2. exportOptionsPlist

在转换时需要该文件，该文件的内容其实就是相当于手动导出时选的那些参数、配置。

获取方法其实就是通过 Xcode 手动转换导出一次，然后从导出的文件夹下找到 ExportOptions.plist。

如果不是很懂的话，可以参考这篇文章 [iOS 生成 exportOptionsPlist 文件](https://blog.csdn.net/lovechris00/article/details/79141752)

# 3.xcodebuild

下边就是整个导出IPA的指令，主要修改的参数有三个:

- archivePath: 步骤1中设置的路径
- exportOptionsPlist: 步骤2中导出的配置文件。建议放在同一个目录下
- exportPath: 导出IPA的路径

```
xcodebuild -exportArchive -archivePath ./MyAppV1.0.xcarchive -exportOptionsPlist ./ExportOptions.plist -exportPath ~/Download/MyAppV1.0.ipa
```

成功后，可以进一步与 deliver 整合，自动化转换上传，美滋滋。(这一个我还没做)
