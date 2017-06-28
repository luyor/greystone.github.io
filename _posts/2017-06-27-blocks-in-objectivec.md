---
title: "Blocks in Objective-C"
date: 2017-06-27 18:16:22
categories: [Objective-C]
tags: [Notes, Coding, Objective-C, Blocks]
---

## 1. Declaration and Definition
  * As a **local variable**:
  ```objc 
  returnType (^blockName)(parameterTypes) = ^returnType(parameters) {...}
  ```
  * As a **method parameter**:
  ```objc
  +/- (void)someMethodThatTakesABlock:(returnType (^nullability)(parameterTypes))blockName;
  ```
  * As an **argument to a method call**:
  ```objc
  [someObject someMethodThatTakesABlock:^returnType(parameters) {...}];
  ```
  * As a **typedef**:
  ```objc
  typedef returnType (^TypeName)(parameterTypes);
  TypeName blockName = ^returnType(parameters) {...};
  ```
  * As a **property**:
  ```objc
  @property (copy, nullability, ...) returnType (^blockName)(parameterTypes);
  ```

## 2. Some Tips
* Normally, a variable is captured when the block is defined. It will be captured as a `const` variable, unless described as `__block`. Meanwhile, the compiler will add a strong reference count to it by 1, which also introduces a retain cycle problem here.

* It’s best practice to use only one block argument to a method. If the method also needs other non-block arguments, the block should come last.

## 3. Avoid Strong Reference Cycles when Capturing self
```objc
__weak __typeof__(self) weakSelf = self;
self.block = ^{
    [weakSelf doSomething];
}
```
However, sometimes, when the `self` is released outside, the method insdie the block will send message to `nil` object. In order to retain the variable during the block, you can try to strongify it:
```objc
__weak __typeof__(self) weakSelf = self;
self.block = ^{
    __strong __typeof__(self) strongSelf = weakSelf;    // __typeof__ is invoked in compile time
    [strongSelf doSomething];
    [strongSelf doMoreThing];
}
```
This `strongSelf` will be released after the block is completed.

## 4. Blocks with Concurrent
### Work with Operation Queue
```objc
NSBlockOperation *operation = [NSBlockOperation blockOperationWithBlock: ^{
    // ...
}];
// ...
NSOperationQueue *mainQueue = [NSOperationQueue mainQueue];
[mainQueue addOperation:operation];  // schedule operation on main queue

NSOperationQueue *queue = [[NSOperationQueue alloc] init];
[queue adOperation:operation];       // scheduel operation on background queue
```

### Work with GCD (Grand Central Dispatch)
```objc
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0); // specify a queue priority
// ...
dispatch_async(queue, ^{
    // ...
});
```

##### References
1. [Programming with Objective-C, Working with Block](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/WorkingwithBlocks/WorkingwithBlocks.html)
2. [Block, weakSelf and strongSelf](https://blog.waterworld.com.hk/post/block-weakself-strongself)
3. [How Do I Declare A Block in Objective-C?](http://fuckingblocksyntax.com)

##### Update Log
* Jun 28, Section 1-4