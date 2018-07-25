---
title: NSAssertionHandler
author: Mattt
translator: Ricky Tan
category: Cocoa
excerpt: "编程结合了人类思考过程中众多学科，从高层次的辩论和语义学——我们用来解释一个系统如何工作的故事——到支撑所有一切的数学和哲学机器。"
---

“如果你一开始没有成功，用一个面向对象的注入点覆写默认的异常处理机制。”如果你是由 `NSAssertionHandler` 养大的话，这就是你小时候会学到的那种建议。

编程结合了人类思考过程中众多学科，从高层次的辩论和语义学——我们用来解释一个系统如何工作的“故事”——到支撑所有一切的数学和哲学机器。

断言是从经典的逻辑学中借用过来的概念。在逻辑学中，断言是对已经证明过的命题的陈述。在编程中，断言是指程序员所做出的关于应用程序在它们所声明的地方的一些假设。

当用于先验条件和后验条件能力范围内时，断言形成了一种[契约](https://zh.wikipedia.org/wiki/%E5%A5%91%E7%BA%A6%E5%BC%8F%E8%AE%BE%E8%AE%A1)，它描述了代码在执行一个方法或函数的开始和结束时的状态的期望。断言也能用于加强运行时的条件，为了当先验条件失败时阻止程序运行。

断言与[单元测试](https://zh.wikipedia.org/wiki/%E5%8D%95%E5%85%83%E6%B5%8B%E8%AF%95)有些类似，它们都定义了代码将会运行的期望结果。与单元测试不同的是，断言存在于程序本身，并且因此被限定在程序的上下文中。因为单元测试是完全独立的，它们完全有能力通过使用像方法桩和模拟对象等工具隔离并且单独测试特定的行为。开发者应当在应该程序中结合使用合理数量的断言和单元测试来测试和定义应用程序行为。

## 基础断言处理

Objective-C 用一个面向对象的途径混合了 C 语言风格的断言宏定义来注入和处理断言失败。即：`NSAssertionHandler`：

> 每个线程拥有它自己的断言处理器，它是 `NSAssertionHandler` 类的实例对象。当被调用时，一个断言处理器打印一条包含方法和类名（或者函数名）的错误信息。然后它抛出一个 `NSInternalInconsistencyException` 异常。

基础类库中[定义](https://gist.github.com/mattt/5031388#file-nsassertionhandler-m-L50-L56)了两套断言宏：

- `NSAssert` / `NSCAssert`
- `NSParameterAssert` / `NSCParameterAssert`

基础类库从语义学上和功能性上使断言处理器的 API 在两个方面区别开来。

第一个区别在于一般断言（`NSAssert`）和参数化断言（`NSParameterAssert`）。方法或函数应当在代码最开始处使用 `NSParameterAssert` / `NSCParameterAssert` 来强制输入的值满足先验条件，这是一条金科玉律；其他情况下使用 `NSAssert` / `NSCAssert`。

第二个区别在于 C 和 Objective-C 的断言：`NSAssert` 应当只用于 Objective-C 环境中（即方法实现中），而 `NSCAssert` 应当只用于 C 环境中（即函数中）。

- 当 `NSAssert` 或 `NSParameterAssert` 的条件不满足时，断言处理器会调用 `-handleFailureInMethod:object:file:lineNumber:description:` 方法。
- 当 `NSCAssert` 或 `NSCParameterAssert` 的条件不满足时，断言处理器会调用 `-handleFailureInFunction:file:lineNumber:description:` 方法。

另外，`NSAssert` / `NSCAssert` 也有一些变体，从 `NSAssert1` 到 `NSAssert5`，它们各自使用不同数量的参数用于 `printf` 风格的格式化字符串。


## 使用 NSAssertionHandler


值得注意的是，从 Xcode 4.2 开始，[发布构建默认关闭了断言](http://stackoverflow.com/questions/6445222/ns-block-assertions-in-objective-c)，它是通过定义 `NS_BLOCK_ASSERTIONS` 宏实现的。也就是说，当编译发布版时，任何调用 `NSAssert` 等的地方都被有效的移除了。


尽管基础类库的断言宏在它们自己的权力下十分有用————虽然只用于开发之中————但是这件趣事不能就此停止。`NSAssertionHandler` 还提供了一套优雅地处理断言失败的方式来保留珍贵的现实世界的使用信息。


> 据说，许多经验丰富的 Objective-C 开发者们告诫不要在生产环境中使用 `NSAssertionHandler`。基础类库中的断言处理是用来在一定安全距离外来理解和感激的。**请小心行事如果你决定在对外发布版的应用中使用它。**

`NSAssertionHandler` 是一个很直接的类，带有两个需要在子类中实现的方法：`-handleFailureInMethod:...` （当 `NSAssert` / `NSParameterAssert` 失败时调用）和 `-handleFailureInFunction:...` （当 `NSCAssert` / `NSCParameterAssert` 失败时调用）。


`LoggingAssertionHandler` 简单地打印出断言失败信息，但是那些信息也可以记录到外部的网络服务中去，用以聚集和分析。


### LoggingAssertionHandler.h

```objc
@interface LoggingAssertionHandler : NSAssertionHandler
@end
```

### LoggingAssertionHandler.m

```objc
@implementation LoggingAssertionHandler

- (void)handleFailureInMethod:(SEL)selector
                       object:(id)object
                         file:(NSString *)fileName
                   lineNumber:(NSInteger)line
                  description:(NSString *)format, ...
{
  NSLog(@"NSAssert Failure: Method %@ for object %@ in %@#%i", NSStringFromSelector(selector), object, fileName, line);
}

- (void)handleFailureInFunction:(NSString *)functionName
                           file:(NSString *)fileName
                     lineNumber:(NSInteger)line
                    description:(NSString *)format, ...
{
  NSLog(@"NSCAssert Failure: Function (%@) in %@#%i", functionName, fileName, line);
}

@end
```

每个线程都可以指定断言处理器。想设置一个 `NSAssertionHandler` 的子类来处理失败的断言，在线程的 `threadDictionary` 对象中设置 `NSAssertionHandlerKey` 字段即可。

大部分情况下，你只需在 `-application:didFinishLaunchingWithOptions:` 中设置当前线程的断言处理器。

### AppDelegate.m

```objc
- (BOOL)application:(UIApplication *)application
didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
  NSAssertionHandler *assertionHandler = [[LoggingAssertionHandler alloc] init];
  [[[NSThread currentThread] threadDictionary] setValue:assertionHandler
                                                 forKey:NSAssertionHandlerKey];
  // ...

  return YES;
}
```

---

`NSAssertionHandler` 是提醒我们作为一个程序员如何通过断言语句清晰地表达我们的期望的一个最佳实践。

但是如果我们进一步深入了解 `NSAssertionHandler`————当然，也深入我们自己内心，还有不少课程需要学习，有关我们的善意和同情心的底线的，有关我们谅解他人能力的，以及从我们自己的错误中恢复的。我们不可能永远都是对的。我们都会犯错。只有接受每个人都是有限的这个事实，我们才能独自成长。诸如此类。
