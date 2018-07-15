---
title: fix iOS 11 下的 UIImagePickerController 编辑图片时的左下角取消按键难点击的问题
date: 2017-12-15 16:49
tags: 
- iOS 11
- UIImagePickerController
---

# 问题描述

在 iOS 11 下，如果给 `UIImagePickerController` 设置 `allowsEditing = YES`，然后在编辑图片的时候，却发现左下角的那个 `取消` 按键怎么也没法点击到

# 问题原理

在打开编辑模式的时候，实际上是一个 `PUPhotoPickerHostViewController`, 搞不懂为什么左边会出现一条大概 `41` 像素的空白视图。

![snipaste_2017-12-15_19-12-02](https://user-images.githubusercontent.com/13956214/34051861-126cc342-e1fb-11e7-9d12-8ff3e5283fe9.png)

# 解决办法

> [代码出处](https://stackoverflow.com/questions/46762200/ios-11-uiimagepickercontroller-after-selection-cancel-button-bug)

让你的类实现 `UINavigationControllerDelegate` 接口，然后实现这个方法:

```
- (void)navigationController:(UINavigationController *)navigationController didShowViewController:(UIViewController *)viewController animated:(BOOL)animated {
    if ([UIDevice currentDevice].systemVersion.floatValue < 11) {
        return;
    }
    if ([viewController isKindOfClass:NSClassFromString(@"PUPhotoPickerHostViewController")]) {
        [viewController.view.subviews enumerateObjectsUsingBlock:^(__kindof UIView * _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
            if (obj.frame.size.width < 42) {
                [viewController.view sendSubviewToBack:obj];
                *stop = YES;
            }
        }];
    }
}
```