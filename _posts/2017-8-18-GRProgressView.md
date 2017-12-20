---
layout: post
title: '更易用的progressView--GRProgressView'
subtitle:
date: 2017-8-18
categories: 开源
cover: 'http://upload-images.jianshu.io/upload_images/4133010-27b670deaa04bdad.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240'
tags: 开源组件
---
# 效果
![演示.gif](http://upload-images.jianshu.io/upload_images/4133010-5880d22ca35ebbe4.gif?imageMogr2/auto-orient/strip)

# 接口
##### 与UIProgressView接口基本一致，增加了更丰富的动画接口，可自由设置frame
```
#import <UIKit/UIKit.h>

@interface GRProgressView : UIImageView

@property (nonatomic, assign) float progress;                        // 0.0 .. 1.0, default is 0.0. values outside are pinned.
@property (nonatomic, strong, nullable) UIColor* progressTintColor;
@property (nonatomic, strong, nullable) UIColor* trackTintColor;
@property (nonatomic, strong, nullable) UIImage* progressImage;
@property (nonatomic, strong, nullable) UIImage* trackImage;
- (void)setProgress:(float)progress animateWithDuration:(double)duration delay:(double)delay
- (void)setProgress:(float)progress animateWithDuration:(double)duration;
- (void)setProgress:(float)progress animated:(BOOL)animated;

@end
```
[git地址:https://github.com/Assuner-Lee/GRProgressView.git 附demo](https://github.com/Assuner-Lee/GRProgressView.git)
#### 水一篇 谢谢观看
