---
title: iOS 快速判断系统版本号
date: 2017-12-15 17:10
tags:
---

> 有些需要兼容一些旧版本，但是也需要使用一些新系统版本的接口，这时候就需要判断设备的版本号

## 最低支持版本在 iOS 8 以上

可以通过 `NSProcessInfo` 来判断

```objc
// 比如匹配 10.3.3
[[NSProcessInfo processInfo] operatingSystemVersion].majorVersion == 10
[[NSProcessInfo processInfo] operatingSystemVersion].minorVersion == 3
[[NSProcessInfo processInfo] operatingSystemVersion].majorVersion == 3

// 这是 `NSOperatingSystemVersion` 的结构体定义
typedef struct {
    NSInteger majorVersion;
    NSInteger minorVersion;
    NSInteger patchVersion;
} NSOperatingSystemVersion;
```

## 最低支持版本在 iOS 8 以下

### 基于 `UIDevice` 获取的系统版本的浮点值判断

基本只能判断主版本，次版本判断失误很高，比如 `10.12 居然 小于 10.3.3`，最后一位补丁版本直接丢失了. 所以也就适合简单比较下主版本的时候使用

```objc
[[UIDevice currentDevice].systemVersion floatValue] >= 10
```

### 基于 `UIDevice` 的预处理宏，准确率可以

```objc
/*
 *  System Versioning Preprocessor Macros
 */ 

#define SYSTEM_VERSION_EQUAL_TO(v)                  ([[[UIDevice currentDevice] systemVersion] compare:v options:NSNumericSearch] == NSOrderedSame)
#define SYSTEM_VERSION_GREATER_THAN(v)              ([[[UIDevice currentDevice] systemVersion] compare:v options:NSNumericSearch] == NSOrderedDescending)
#define SYSTEM_VERSION_GREATER_THAN_OR_EQUAL_TO(v)  ([[[UIDevice currentDevice] systemVersion] compare:v options:NSNumericSearch] != NSOrderedAscending)
#define SYSTEM_VERSION_LESS_THAN(v)                 ([[[UIDevice currentDevice] systemVersion] compare:v options:NSNumericSearch] == NSOrderedAscending)
#define SYSTEM_VERSION_LESS_THAN_OR_EQUAL_TO(v)     ([[[UIDevice currentDevice] systemVersion] compare:v options:NSNumericSearch] != NSOrderedDescending)
```

### 基于 `NSFoundationVersionNumber` 值来判断版本

> 当前我使用 Xcode 8.3 打印出来的值为 `1349.55`
> 而此时提供的宏最大的为 `#define NSFoundationVersionNumber_iOS_9_x_Max 1299`
> 个人感觉不太适合用来判断当前版本的细微比较


```objc
#ifndef iOS8
#define iOS8 (NSFoundationVersionNumber >= NSFoundationVersionNumber_iOS_8_0)
#endif
```

## available 

### Swift 

直接用 `#available` 按需填系统及版本号，需要在括号内最后一个条件后带上 `,*`


// 仅判断 iOS 系统版本
```swift
if #available(iOS 10.0, *) {
            
}
        
// 同时判断其他系统版本
if #available(iOS 10.0, macOS 10.11,tvOS 1.0,watchOS 1.0, *) {
            
}
```

### ObjC

```objc
if (@available(iOS 11.0, *)) {
    
}

```

这部分具体的可以参考：

1. http://swift.gg/2016/04/13/swift-qa-2016-04-13/
2. http://yulingtianxia.com/blog/2017/07/17/What-s-New-in-LLVM-2017/