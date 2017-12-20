---
layout: post
title: '简单好用可任意定制的iOS Popover气泡'
subtitle:
date: 2017-10-25
categories: 开源
cover: 'http://upload-images.jianshu.io/upload_images/4133010-a454fc662202a857.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240'
tags: 开源组件
---
# 先看效果
![popover](http://upload-images.jianshu.io/upload_images/4133010-0a2ab3afb2c8ed1e.gif?imageMogr2/auto-orient/strip)
[swift: https://github.com/corin8823/Popover](https://github.com/corin8823/Popover)
[OC: https://github.com/Assuner-Lee/PopoverObjC](https://github.com/Assuner-Lee/PopoverObjC)
## 使用示例
```
pod 'PopoverObjC'
```
```
#import "ASViewController.h"
#import <PopoverObjC/ASPopover.h>

@interface ASViewController ()

@property (weak, nonatomic) IBOutlet UIButton *btn;
@property (nonatomic, strong) ASPopover *btnPopover;
@property (nonatomic, strong) ASPopover *itemPopover;

@end

@implementation ASViewController

- (void)viewDidLoad {
[super viewDidLoad];
[self.btn addTarget:self action:@selector(clickBtn:) forControlEvents:UIControlEventTouchUpInside];

self.navigationItem.rightBarButtonItem = [[UIBarButtonItem alloc] initWithTitle:@"item" style:UIBarButtonItemStylePlain target:self action:@selector(clickItem:)];
}

- (void)didReceiveMemoryWarning {
}
```
#### 初始化Popover
```
- (ASPopover *)btnPopover {
if (!_btnPopover) {
ASPopoverOption *option = [[ASPopoverOption alloc] init];
option.popoverType = ASPopoverTypeUp;
option.autoAjustDirection = NO;
option.arrowSize = CGSizeMake(9, 6);
option.blackOverlayColor = [UIColor clearColor];
option.popoverColor = [UIColor lightGrayColor];
option.dismissOnBlackOverlayTap = YES;
option.animationIn = 0.5;
//...

_btnPopover = [[ASPopover alloc] initWithOption:option];
}
return _btnPopover;
}


- (ASPopover *)itemPopover {
if (!_itemPopover) {
ASPopoverOption *option = [[ASPopoverOption alloc] init];
option.autoAjustDirection = NO;
option.arrowSize = CGSizeMake(10, 6);
option.blackOverlayColor = [UIColor clearColor];
option.sideEdge = 7;
option.dismissOnBlackOverlayTap = YES;
option.popoverColor = [[UIColor blackColor] colorWithAlphaComponent:0.7];
option.autoAjustDirection = YES;
option.animationIn = 0.4;
option.springDamping = 0.5;
option.initialSpringVelocity = 1;
option.overlayBlur = [UIBlurEffect effectWithStyle:UIBlurEffectStyleLight];
//...

_itemPopover = [[ASPopover alloc] initWithOption:option];
}
return _itemPopover;
}
```
popover的属性可在option里设置。
#### 弹出气泡
```
- (void)clickBtn:(id)sender {
UIView *view = [[UIView alloc] initWithFrame:CGRectMake(0, 0, [UIScreen mainScreen].bounds.size.width - 50, 40)];
[self.btnPopover show:view fromView:self.btn];  // in delegate window
}

- (void)clickItem:(id)sender {
UIView *view = [[UIView alloc] initWithFrame:CGRectMake(0, 0, 100, 200)];
UIView *itemView = [self.navigationItem.rightBarButtonItem valueForKey:@"view"]; // you should use custom view in item;
if (itemView) {
//    [self.itemPopover show:view fromView:itemView];
CGPoint originPoint = [self.itemPopover originArrowPointWithView:view fromView:itemView];
originPoint.y += 5;
[self.itemPopover show:view atPoint:originPoint];
}
}
@end
```
可在某一个视图或某一个point上弹出内容view
#### Popover interface
```
#import <UIKit/UIKit.h>
#import "ASPopoverOption.h"

typedef void (^ASPopoverBlock)(void);

@interface ASPopover : UIView

@property (nonatomic, copy) ASPopoverBlock willShowHandler;
@property (nonatomic, copy) ASPopoverBlock willDismissHandler;
@property (nonatomic, copy) ASPopoverBlock didShowHandler;
@property (nonatomic, copy) ASPopoverBlock didDismissHandler;

@property (nonatomic, strong) ASPopoverOption *option;

- (instancetype)initWithOption:(ASPopoverOption *)option;
- (void)dismiss;

- (void)show:(UIView *)contentView fromView:(UIView *)fromView;
- (void)show:(UIView *)contentView fromView:(UIView *)fromView inView:(UIView *)inView;
- (void)show:(UIView *)contentView atPoint:(CGPoint)point;
- (void)show:(UIView *)contentView atPoint:(CGPoint)point inView:(UIView *)inView;

- (CGPoint)originArrowPointWithView:(UIView *)contentView fromView:(UIView *)fromView;
- (CGPoint)arrowPointWithView:(UIView *)contentView fromView:(UIView *)fromView inView:(UIView *)inView popoverType:(ASPopoverType)type;
@end
```
contentView: 要显示的内容；
fromView: 气泡从某一个视图上show;
inview: 气泡绘制在某一个视图上，一般为delegate window；
atPoint: 气泡从某一点上show; 可先获取originPoint, 偏移；
#### PopoverOption Interface
```
typedef NS_ENUM(NSInteger, ASPopoverType) {
ASPopoverTypeUp = 0,
ASPopoverTypeDown,
};

@interface ASPopoverOption : NSObject

@property (nonatomic, assign) CGSize arrowSize;
@property (nonatomic, assign) NSTimeInterval animationIn; // if 0, no animation
@property (nonatomic, assign) NSTimeInterval animationOut;
@property (nonatomic, assign) CGFloat cornerRadius;
@property (nonatomic, assign) CGFloat sideEdge;
@property (nonatomic, strong) UIColor *blackOverlayColor;
@property (nonatomic, strong) UIBlurEffect *overlayBlur;
@property (nonatomic, strong) UIColor *popoverColor;
@property (nonatomic, assign) BOOL dismissOnBlackOverlayTap;
@property (nonatomic, assign) BOOL showBlackOverlay;
@property (nonatomic, assign) CGFloat springDamping;
@property (nonatomic, assign) CGFloat initialSpringVelocity;
@property (nonatomic, assign) ASPopoverType popoverType;
@property (nonatomic, assign) BOOL highlightFromView;
@property (nonatomic, assign) CGFloat highlightCornerRadius;
@property (nonatomic, assign) BOOL autoAjustDirection; // down preferred, effect just in view not at point

@end
```
## 多谢观看
