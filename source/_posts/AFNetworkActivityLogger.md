---
title: AFNetworkActivityLogger 源码解读
date: 2018-07-22 13:17:23
tags:
- AFNetworking
- Logger
- 源码解读
---

# 介绍

> AFNetworkActivityLogger -> 依赖于 `AFNetworking` 的官方日志组件

## 作用

通过了解这个源码，可以方便定制网络请求日志监控模块。

## 原理

这个日志组件本身的实现原理是通过 `AFNetworking` 的通知来实现的。

## 类文件总览

将源码下载下来，打开发现一共就三个类/接口：

- `AFNetworkActivityLogger`         管理类
- `AFNetworkActivityConsoleLogger`  内置的控制台日志实现
- `AFNetworkActivityLoggerProtocol` 日志接口/代理

# 源代码

## 上手使用

按照 `README` 上说的，仅需要在启动时加上：

```
[[AFNetworkActivityLogger sharedLogger] startLogging];
```

## `AFNetworkActivityLogger` 类

基本上，作为一个普通使用者而言，这个类一般是整个组件中唯一需要接触到的。

### loggers

- 对外暴露出当前组件中注册过的日志记录器(logger)
- 内部维护一个单独的可变集合对象，用于做增删操作

### sharedLogger / init

在初始化时：

- 初始化一个可变集合对象，用于保存注册过的日志记录器
- 会初始化一个 `AFNetworkActivityConsoleLogger` 日志记录器，并通过 `addLogger:` 方法注册到组件中

### startLogging / stopLogging

在 `startLogging` 时，监听 `AFNetworkingTaskDidResumeNotification` 、 `AFNetworkingTaskDidCompleteNotification` 通知事件，并分别绑定到 `networkRequestDidStart:` 以及 `networkRequestDidFinish:` 方法上

在 `stopLogging` 时，取消监听事件，此外，在 `dealloc` 方法也会调用该方法

### setLogLevel:

遍历组件中注册过的日志记录器，将其 `level` 修改成指定值

### addLogger: / removeLogger:

在 `loggers` 中添加 / 移除传入的日志记录器

### networkRequestDidStart:

在记录请求开始的方法中，主要有以下几个步骤:

1. 从通知对象中取出 `task`、`request`
2. 检测 `request` 是否存在，不存在就退出，存在就继续执行
3. 将当前时间保存到 `task` 上，用于在请求结束时计算耗时时间
4. 遍历已注册的日志记录器
5. 检测当前遍历到的日志记录器是否支持过滤，如支持且符合过滤条件，则中断执行
6. 调用日志记录器的 `URLSessionTaskDidStart:` 方法，记录请求开始事件

```
static void * AFNetworkRequestStartDate = &AFNetworkRequestStartDate;

- (void)networkRequestDidStart:(NSNotification *)notification {
    // #1
    NSURLSessionTask *task = [notification object];
    NSURLRequest *request = task.originalRequest;

    // #2
    if (!request) {
        return;
    }

    // #3
    objc_setAssociatedObject(notification.object, AFNetworkRequestStartDate, [NSDate date], OBJC_ASSOCIATION_RETAIN_NONATOMIC);

    // #4
    for (id <AFNetworkActivityLoggerProtocol> logger in self.loggers) {

        // #5
        if (request && logger.filterPredicate && [logger.filterPredicate evaluateWithObject:request]) {
            return;
        }

        // #6
        [logger URLSessionTaskDidStart:task];
    }
}
```

### networkRequestDidFinish:

在记录请求结束的方法中，主要有以下几个步骤:

1. 从通知对象中取出 `task`、`request` 、`response` , 以及尝试获取 `error`
2. 如果 `request` 、`response` 均为空，则中断执行
3. 从通知对象的 `userInfo` 取出服务器返回的数据
4. 从 `task` 中取出请求开始时保存的时间，并计算出请求耗时
5. 遍历，同上
6. 检查过滤，同上
7. 调用日志记录器的 `URLSessionTaskDidFinish:withResponseObject:inElapsedTime:withError:` 方法，记录请求结束事件

```java
- (void)networkRequestDidFinish:(NSNotification *)notification {
    // #1
    NSURLSessionTask *task = [notification object];
    NSURLRequest *request = task.originalRequest;
    NSURLResponse *response = task.response;
    NSError *error = AFNetworkErrorFromNotification(notification);

    // #2
    if (!request && !response) {
        return;
    }

    // #3
    id responseObject = nil;
    if (notification.userInfo) {
        responseObject = notification.userInfo[AFNetworkingTaskDidCompleteSerializedResponseKey];
    }

    // #4
    NSTimeInterval elapsedTime = [[NSDate date] timeIntervalSinceDate:objc_getAssociatedObject(notification.object, AFNetworkRequestStartDate)];

    // #5
    for (id <AFNetworkActivityLoggerProtocol> logger in self.loggers) {

        // #6
        if (request && logger.filterPredicate && [logger.filterPredicate evaluateWithObject:request]) {
            return;
        }

        // #7
        [logger URLSessionTaskDidFinish:task withResponseObject:responseObject inElapsedTime:elapsedTime withError:error];
    }
}
```

## `AFNetworkActivityLoggerProtocol` 代理

### 日志级别 AFHTTPRequestLoggerLevel

```
typedef NS_ENUM(NSUInteger, AFHTTPRequestLoggerLevel) {
    AFLoggerLevelOff,
    AFLoggerLevelDebug,
    AFLoggerLevelInfo,
    AFLoggerLevelError
};
```

### filterPredicate 、 level

通过断言或者日志级别来过滤某些日志信息

### 日志记录器方法

```
/// 记录请求开始
- (void)URLSessionTaskDidStart:(NSURLSessionTask *)task;
/// 记录请求结束
- (void)URLSessionTaskDidFinish:(NSURLSessionTask *)task withResponseObject:(id)responseObject inElapsedTime:(NSTimeInterval)elapsedTime withError:(NSError *)error;
```

## `AFNetworkActivityConsoleLogger` 控制台日志记录器

这个类作为内置的日志记录器，大致实现了应有的功能，可以从中学习如何定制日志记录器

### level

在初始化方法中，设置为 `AFLoggerLevelInfo`

### filterPredicate

未实现过滤器

### URLSessionTaskDidStart:

```
- (void)URLSessionTaskDidStart:(NSURLSessionTask *)task {
    NSURLRequest *request = task.originalRequest;

    NSString *body = nil;
    if ([request HTTPBody]) {
        body = [[NSString alloc] initWithData:[request HTTPBody] encoding:NSUTF8StringEncoding];
    }

    switch (self.level) {
        case AFLoggerLevelDebug:
            NSLog(@"%@ '%@': %@ %@", [request HTTPMethod], [[request URL] absoluteString], [request allHTTPHeaderFields], body);
            break;
        case AFLoggerLevelInfo:
            NSLog(@"%@ '%@'", [request HTTPMethod], [[request URL] absoluteString]);
            break;
        default:
            break;
    }
}
```

### URLSessionTaskDidFinish:withResponseObject:inElapsedTime:withError:

```
- (void)URLSessionTaskDidFinish:(NSURLSessionTask *)task withResponseObject:(id)responseObject inElapsedTime:(NSTimeInterval )elapsedTime withError:(NSError *)error {
    NSUInteger responseStatusCode = 0;
    NSDictionary *responseHeaderFields = nil;
    if ([task.response isKindOfClass:[NSHTTPURLResponse class]]) {
        responseStatusCode = (NSUInteger)[(NSHTTPURLResponse *)task.response statusCode];
        responseHeaderFields = [(NSHTTPURLResponse *)task.response allHeaderFields];
    }

    if (error) {
        switch (self.level) {
            case AFLoggerLevelDebug:
            case AFLoggerLevelInfo:
            case AFLoggerLevelError:
                NSLog(@"[Error] %@ '%@' (%ld) [%.04f s]: %@", [task.originalRequest HTTPMethod], [[task.response URL] absoluteString], (long)responseStatusCode, elapsedTime, error);
            default:
                break;
        }
    } else {
        switch (self.level) {
            case AFLoggerLevelDebug:
                NSLog(@"%ld '%@' [%.04f s]: %@ %@", (long)responseStatusCode, [[task.response URL] absoluteString], elapsedTime, responseHeaderFields, responseObject);
                break;
            case AFLoggerLevelInfo:
                NSLog(@"%ld '%@' [%.04f s]", (long)responseStatusCode, [[task.response URL] absoluteString], elapsedTime);
                break;
            default:
                break;
        }
    }
}
```



















