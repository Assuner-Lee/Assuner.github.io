---
layout: post
title: 'Stinger--实践实现特定实例对象的AOP'
subtitle:
date: 2019-3-02
categories: 开源
cover: 'https://user-gold-cdn.xitu.io/2018/1/19/1610c4e56d18d6c9?w=2176&h=468&f=jpeg&s=274940'
tags: runtime libffi hook AOP
---
&ensp;&ensp;&ensp;&ensp; 在 [iOS完整实践: 使用Libffi实现AOP](https://juejin.im/post/5ae28acd6fb9a07ac55fdac0) 一文中，我们介绍了实现AOP的一种方式，通过解析目标方法的签名，使用`ffi_prep_cif	`和`ffi_prep_closure_loc`构造壳函数替换原函数实现，以感知原方法调用时机及捕获参数，最后通过`ffi_call`利用预生成的模板动态调用原实现和block的函数指针。在[Stinger](https://github.com/Assuner-Lee/Stinger)中编写了具体实现。  

&ensp;&ensp;&ensp;&ensp; 但上述方式因为替换了类的方法实现，意味着类的所有实例对象的方法调用都发生了改变。在实际开发中，我们往往有这样的需求，在某个类的特定实例对象的方法里执行aop代码，其他实例对象仍旧执行原方法。实现这类aop的方式有KVO, RACObserve, rac_signalForSelector等。本文介绍了[Stinger](https://github.com/Assuner-Lee/Stinger)的新特性实现细节，可构造与原方法参数相近的block，作为切面代码插入到特定实例对象的方法实现中。使用如下:

```
@implementation ASViewController
- (void)print3:(NSString *)s {
  NSLog(@"---original print4: %@", s);
}
- (void)viewDidLoad {
  [super viewDidLoad];
  // hook for specific instance
  [self st_hookInstanceMethod:@selector(print3:) option:STOptionAfter usingIdentifier:@"hook_print3_after1" withBlock:^(id<StingerParams> params, NSString *s) {
    NSLog(@"---specific instance-self after print3: %@", s);
  }];
}
@end
```

## 实现

&ensp;&ensp;&ensp;&ensp; 一个类可以有很多实例，要对特定实例对象hook方法，免不了有更换方法实现这一步，既然要达到不影响其他实例的方法，这里能想到的是像KVO的实现细节一样，新建一个所属类的子类，然后更改特定实例的isa指针为新建子类，然后对该子类进行单独hook。另外需要设计一种结构，保存hook信息关联到实例对象，执行方法的时候，既执行该对象所属类原有的aop代码，又执行自己的aop代码。  

### Hook class

&ensp;&ensp;&ensp;&ensp; 下边是hook实例对象的代码，首先为实例对象的类新建了子类`ST_##Class`, 并关联到了对象上，避免重复生成，同一个类的多个实例对象可以使用同一个子类作为isa指针指向。  此外，与KVO类似，hook了子类的class方法，以返回原有的类。

### add hook info

&ensp;&ensp;&ensp;&ensp; 在[iOS完整实践: 使用Libffi实现AOP](https://juejin.im/post/5ae28acd6fb9a07ac55fdac0)中，`hookInfoPool`关联到了类上，`hook info`也添加到了该`hookInfoPool`中。hook实例对象的方法时，新建的子类有一个`hookInfoPool`，仅仅是为了存放壳函数，实例对象也有一个`hookInfoPool`，针对实例对象的`hook info`就存于此。这也就意味着，同一个类下的多个实例对象可以共用新建子类的`hookInfoPool`的壳函数，而真正的`hook info`存在对象本身关联的的`hookInfoPool`里。



```
#pragma mark - For specific instance

- (STHookResult)st_hookInstanceMethod:(SEL)sel option:(STOption)option usingIdentifier:(STIdentifier)identifier withBlock:(id)block {
  @synchronized(self) {
    Class stSubClass = getSTSubClass(self);
    if (!stSubClass) return STHookResultOther;
    
    STHookResult hookMethodResult = hookMethod(stSubClass, sel, option, identifier, block);
    if (hookMethodResult != STHookResultSuccuss) return hookMethodResult;
    if (!objc_getAssociatedObject(self, STSubClassKey)) {
      object_setClass(self, stSubClass);
      objc_setAssociatedObject(self, STSubClassKey, stSubClass, OBJC_ASSOCIATION_ASSIGN);
    }
    
    id<STHookInfoPool> instanceHookInfoPool = st_getHookInfoPool(self, sel);
    if (!instanceHookInfoPool) {
      instanceHookInfoPool = [STHookInfoPool poolWithTypeEncoding:nil originalIMP:NULL selector:sel];
      st_setHookInfoPool(self, sel, instanceHookInfoPool);
    }
    
    STHookInfo *instanceHookInfo = [STHookInfo infoWithOption:option withIdentifier:identifier withBlock:block];
    return [instanceHookInfoPool addInfo:instanceHookInfo] ? STHookResultErrorIDExisted : STHookResultSuccuss;
  }
}

NS_INLINE Class getSTSubClass(id object) {
  NSCParameterAssert(object);
  Class stSubClass = objc_getAssociatedObject(object, STSubClassKey);
  if (stSubClass) return stSubClass;
    
  Class isaClass = object_getClass(object);
  NSString *isaClassName = NSStringFromClass(isaClass);
  const char *subclassName = [STClassPrefix stringByAppendingString:isaClassName].UTF8String;
  stSubClass = objc_getClass(subclassName);
  if (!stSubClass) {
    stSubClass = objc_allocateClassPair(isaClass, subclassName, 0);
    NSCAssert(stSubClass, @"Class %s allocate failed!", subclassName);
    if (!stSubClass) return nil;
    
  objc_registerClassPair(stSubClass);
  Class realClass = [object class];
  hookGetClassMessage(stSubClass, realClass);
  hookGetClassMessage(object_getClass(stSubClass), realClass);
}
  return stSubClass;
}

```

### invoke hook info

&ensp;&ensp;&ensp;&ensp; 在当对象isa指向的类的方法被调用时，壳函数被执行，`hookInfoPool`会作为`userData`传进来，这个hookInfoPool时关联在类上的。如果关联的类的类名带有st_class_前缀, 表明这是Hook实例对象新建的子类，那么`hookInfoPool`里的`hook Info`是空的，需要尝试取到预存的真实的类`statedCls`里的hook信息，即针对类的所有实例对象的hook。

&ensp;&ensp;&ensp;&ensp; 在`args`里，我们可以在第一位拿到`self`, 即此时调用方法的实例对象，如果有关联的`hookInfoPool`，表明有针对此对象的Hook。执行切面block代码时，先执行针对的类的`hook info`，再执行针对实例对象的`hook Info`，如果是替换，则优先以实例的为准。

```
#define ffi_call_infos(infos) \
for (id<STHookInfo> info in infos) { \
  id block = info.block; \
  innerArgs[0] = &block; \
  ffi_call(&(hookedClassInfoPool->_blockCif), impForBlock(block), NULL, innerArgs); \
}  \

static void ffi_function(ffi_cif *cif, void *ret, void **args, void *userdata) {
  STHookInfoPool *hookedClassInfoPool = (__bridge STHookInfoPool *)userdata;
  STHookInfoPool *statedClassInfoPool = nil;
  STHookInfoPool *instanceInfoPool = nil;
  SEL sel = hookedClassInfoPool->_sel;
  if ([NSStringFromClass(hookedClassInfoPool->_hookedCls) hasPrefix:STClassPrefix]) {
    statedClassInfoPool = st_getHookInfoPool(hookedClassInfoPool->_statedCls, sel);
  } else {
    statedClassInfoPool = hookedClassInfoPool;
  }
  NSUInteger count = hookedClassInfoPool->_signature.argumentTypes.count;
  void **innerArgs = malloc(count * sizeof(*innerArgs));
  StingerParams *params = [[StingerParams alloc] init];
  void **slf = args[0];
  instanceInfoPool = st_getHookInfoPool((__bridge id)(*slf), sel);
  params.slf = (__bridge id)(*slf);
  params.sel = sel;
  [params addOriginalIMP:hookedClassInfoPool->_originalIMP];
  NSInvocation *originalInvocation = [NSInvocation invocationWithMethodSignature:hookedClassInfoPool->_ns_signature];
  
  for (int i = 0; i < count; i ++) {
    [originalInvocation setArgument:args[i] atIndex:i];
  }
  [params addOriginalInvocation:originalInvocation];
  
  innerArgs[1] = &params;
  memcpy(innerArgs + 2, args + 2, (count - 2) * sizeof(*args));
  
  // before hooks
  ffi_call_infos(statedClassInfoPool->_beforeInfos);
  if (instanceInfoPool) ffi_call_infos(instanceInfoPool->_beforeInfos);
  
  // instead hooks
  if (instanceInfoPool && instanceInfoPool->_insteadInfos.count) {
    id <STHookInfo> info = instanceInfoPool->_insteadInfos[0];
    id block = info.block;
    innerArgs[0] = &block;
    ffi_call(&(hookedClassInfoPool->_blockCif), impForBlock(block), ret, innerArgs);
  } else if (statedClassInfoPool->_insteadInfos.count) {
    id <STHookInfo> info = statedClassInfoPool->_insteadInfos[0];
    id block = info.block;
    innerArgs[0] = &block;
    ffi_call(&(hookedClassInfoPool->_blockCif), impForBlock(block), ret, innerArgs);
  } else {
    // original IMP
    /// if hooked by aspects or jspatch.. which use message-forwarding.
    BOOL isForward = hookedClassInfoPool->_originalIMP == _objc_msgForward
#if !defined(__arm64__)
    || hookedClassInfoPool->_originalIMP == (IMP)_objc_msgForward_stret
#endif
    ;
    if (isForward) {
      [params invokeAndGetOriginalRetValue:ret];
    } else {
      ffi_call(cif, (void (*)(void))hookedClassInfoPool->_originalIMP, ret, args);
    }
  }
  // after hooks
  ffi_call_infos(statedClassInfoPool->_afterInfos);
  if (instanceInfoPool) ffi_call_infos(instanceInfoPool->_afterInfos);
  
  free(innerArgs);
}

```

## 兼容性
>
兼容性：考虑到类或单个对象已经被类似于`rac aspects jspatch`使用`msg_Forward`替换过方法实现。执行原方法实现时，如果函数指针是`msg_Forward`，则不利用`ffi_call`使用模板导入函数指针和参数动态调用以增加效率，需要`invoke Original invocation`走系统发送消息触发消息转发走下边的逻辑。
>

### 谢谢观看，如有错误，请多指正。
文中代码：[https://github.com/Assuner-Lee/Stinger](https://github.com/Assuner-Lee/Stinger)


