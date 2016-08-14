---
layout: post
title: iOS 代理为啥要用weak修饰?
---

转载自[汉斯哈哈哈: iOS 万能跳转界面方法](http://www.jianshu.com/p/8b3a9155468d)

---

## 在开发项目中，会有这样的需求： ##

* 推送：根据服务端推送过来的数据规则，跳转到对应的控制器
* feeds列表：不同类似的cell，可能跳转不同的控制器（嘘！产品经理是这样要求：我也不确定会跳转哪个界面哦，可能是这个又可能是那个，能给我做灵活吗？根据后台返回规则任意跳转？）

## 实现 ##

利用runtime动态生成对象、属性、方法这特性，我们可以先跟服务端商量好，定义跳转规则，比如要跳转到A控制器，需要传属性id、type，那么服务端返回字典给我，里面有控制器名，两个属性名跟属性值，客户端就可以根据控制器名生成对象，再用kvc给对象赋值，这样就搞定了.

比如：根据推送规则跳转对应界面HSFeedsViewController

HSFeedsViewController.h：
{% highlight objective-c %}
@interface HSFeedsViewController : UIViewController

// 注：根据下面的两个属性，可以从服务器获取对应的频道列表数据

/** 频道ID */
@property (nonatomic, copy) NSString *ID;

/** 频道type */
@property (nonatomic, copy) NSString *type;

@end
{% endhighlight %}

AppDelegate.m：
{% highlight objective-c %}
// 这个规则肯定事先跟服务端沟通好，跳转对应的界面需要对应的参数
NSDictionary *userInfo = @{
     @"class": @"HSFeedsViewController",
     @"property": @{
          @"ID": @"123",
          @"type": @"12"
    }
};
{% endhighlight %}

* 接收推送消息
{% highlight c %}
- (void)application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)userInfo
{
    [self push:userInfo];
}
{% endhighlight %}

* 跳转界面
{% highlight c %}
- (void)push:(NSDictionary *)params
{
    // 类名
    NSString *class =[NSString stringWithFormat:@"%@", params[@"class"]];
    const char *className = [class cStringUsingEncoding:NSASCIIStringEncoding];

    // 从一个字串返回一个类
    Class newClass = objc_getClass(className);
    if (!newClass)
    {
        // 创建一个类
        Class superClass = [NSObject class];
        newClass = objc_allocateClassPair(superClass, className, 0);
        // 注册你创建的这个类
        objc_registerClassPair(newClass);
    }
    // 创建对象
    id instance = [[newClass alloc] init];

    // 对该对象赋值属性
    NSDictionary * propertys = params[@"property"];
    [propertys enumerateKeysAndObjectsUsingBlock:^(id key, id obj, BOOL *stop) {
        // 检测这个对象是否存在该属性
        if ([self checkIsExistPropertyWithInstance:instance verifyPropertyName:key]) {
            // 利用kvc赋值
            [instance setValue:obj forKey:key];
        }
    }];

    // 获取导航控制器
    UITabBarController *tabVC = (UITabBarController *)self.window.rootViewController;
    UINavigationController *pushClassStance = (UINavigationController *)tabVC.viewControllers[tabVC.selectedIndex];
    // 跳转到对应的控制器
    [pushClassStance pushViewController:instance animated:YES];
}
{% endhighlight %}

* 检测对象是否存在该属性
{% highlight c %}
- (BOOL)checkIsExistPropertyWithInstance:(id)instance verifyPropertyName:(NSString *)verifyPropertyName
{
    unsigned int outCount, i;

    // 获取对象里的属性列表
    objc_property_t * properties = class_copyPropertyList([instance
                                                           class], &outCount);

    for (i = 0; i < outCount; i++) {
        objc_property_t property =properties[i];
        //  属性名转成字符串
        NSString *propertyName = [[NSString alloc] initWithCString:property_getName(property) encoding:NSUTF8StringEncoding];
        // 判断该属性是否存在
        if ([propertyName isEqualToString:verifyPropertyName]) {
            free(properties);
            return YES;
        }
    }
    free(properties);

    return NO;
}
{% endhighlight %}

具体使用和代码: https://github.com/HHuiHao/Universal-Jump-ViewController