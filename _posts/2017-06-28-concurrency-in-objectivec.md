---
title: "Concurrency in Objective-C"
date: 2017-06-28 15:56:29
categories: [Objective-C]
tags: [Notes, Coding, Objective-C, Concurrency]
---

## 1. Mechanism of Concurrency Control in Objective-C
* NSOperation & NSOperationQueue (Cocoa-based)
    * Create a operation and execute it / add it into a queue (run automatically)
* Grand Central Dispatch (Low-level)
    * Dispatch Queue
    * Dispatch Source (listen on Mach / kqueue event)
* OpenCL
* Threads

## 2. NSOperation & NSOperationQueue
### Advantages
* Graph-based dependencies
* Optional completion block
* Execution state observation
* Priority
* Canceling semantics

### Concurrent Versus Non-concurrent Operations
* You can run a opertion by calling `start` method, however, this will not guarantee that the operation runs concurrently. Check the status by calling `isConcurrent` method (default return `NO`).
* Additional codes are required to configure a operation for concurrent execution, such as spawning a separate thread, calling an asynchronous system function...

### NSInvocationOperation
* Taking existing methods as operations
* Especially helpful when you do refactor 
```objc
NSInvocationOperation *op = [[NSInvocationOperation alloc] initWithTarget:self selector:@selector(someMethod) object:nil];
[op start];
```

### NSBlockOperation
* If more than 1 block is added into NSBlockOperation, then those blocks will run concurrently
```objc
NSBlockOperation *op = [NSBlockOperation blockOperationWithBlock:^{
    // block 1
}];
[op addExecutionBlock:^{
    // block 2
}];
[op start];
```

### NSOperationQueue
* NSOperationQueue is a **FIFO** structure, but take **priority** and **dependency** into account
* NSOperationQueue will run operations concurrently (depends on how many resource the device has, and use as much as possible)
* When a operation is added into a queue, it will be automatically executed
```objc
NSOperationQueue *queue = [[NSOperationQueue alloc] init];
[queue addOperation:anOperation];
[queue addOpertions:anArrayOfOperations waitUntilFinished:NO];
[queue addOperationWithBlock:^{
    // directly add a block into queue
}]
```
* `[queue setMaxConcurrentOperationCount:(NSInteger)]`


### Customize Operation Object
* Inherit NSOperation (abstruct) and Override
    * `initilization`
    * `main`
* Response to Cancellation Events
    * Call the object's `isCancelled` method (lightweight enough) periodically and return immediately if it ever return `YES`
* Configure Operations for Concurrent Execution
    * Required Override
        * `start`
        * `isExecuting` & `isFinished`
        * `isConcurrent`, override this and return `YES`
* KVO Compliance
    * `isCancelled`, `isConcurrent`, `isExecuting`, `isFinished`, `isReady`
    * `dependencies`, `queuePriority`
    * `completionBlock`
* See more details on [Apple's Guide](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationObjects/OperationObjects.html#//apple_ref/doc/uid/TP40008091-CH101-SW16)

### Customize Execution Behavior
* Configure Interoperation Dependencies
    * `[operation addDependency:anotherOperation];`
* Change an Operation's Execution Priority
    * `[queue setQueuePriority:(NSOperationQueuePriority)]`
* Change the Underlying Thread Priority
    * `NSQualityOfService`
* Set Up a Completion Block
    * `[operation setCompletionBlock:^void(void){}]`

### Advanced Topic
* Memory Managements
* Error Handling & Exception