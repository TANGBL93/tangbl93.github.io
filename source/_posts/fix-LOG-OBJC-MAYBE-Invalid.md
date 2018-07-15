---
title: Implicit declaration of function 'LOG_OBJC_MAYBE' is invalid in C99
date: 2018-04-17 13:10
tags:
- LOG_OBJC_MAYBE
- CocoaLumberjack
---

## 问题来源

当在 Podfile 中开启  `use_frameworks!` 后遇到

## 解决办法

1. 将 `HTTPLogging.h` 文件中的 `#import "DDLog.h"` 替换为  `#import <CocoaLumberjack/CocoaLumberjack.h>`
2. 在 `HTTPLogging.h` 文件中添加以下宏定义: 

```
#define HTTP_LOG_OBJC_MAYBE(async, lvl, flg, ctx, frmt, ...) \
do{ if(HTTP_LOG_ASYNC_ENABLED) LOG_MAYBE(async, lvl, flg, ctx, nil, sel_getName(_cmd), frmt, ##__VA_ARGS__); } while(0)

#define HTTP_LOG_C_MAYBE(async, lvl, flg, ctx, frmt, ...) \
do{ if(HTTP_LOG_ASYNC_ENABLED) LOG_MAYBE(async, lvl, flg, ctx, nil, __FUNCTION__, frmt, ##__VA_ARGS__); } while(0)
```
3. 将 `HTTPLogging.h` 文件中所有的 `LOG_OBJC_MAYBE` 替换为 `HTTP_LOG_OBJC_MAYBE` , `LOG_C_MAYBE` 替换为 `HTTP_LOG_C_MAYBE`

## 参考链接

> https://stackoverflow.com/questions/40084786/cocoahttpserver2-3-implicit-declaration-of-function-log-objc-maybe-is-invalid
