---
title: "Concurrency in Objective-C"
date: 2017-06-28 15:56:29
categories: [Objective-C]
tags: [Notes, Coding, Objective-C, Concurrency]
---

## 1. Mechanism
* NSOperation & NSOperationQueue (Cocoa-based)
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

## 3. Grand Central Dispatch
### Advantages
* Straightforward programming interface than threads
* Offer automatic and holistic thread pool management
* Provide the speed of tuned assembly
* Much more memory efficient
* Not trap to the kernel under load
* Asynchronous dispatching of tasks to a dispatch queue cannot deadlock the queue
* Scale gracefully under contention
* Serial dispatch queue offer a more efficinet alternative to locks and other synchronization primitives

### Queue-Related Technologies
* Dispatch groups
    * Be able to monitor a set of block objects for completion
* Dispatch semaphores
    * Similar than the traditional semaphore, but more efficient
* Dispatch sources
    * Monitor events such as process notifications, signals, and descriptor events among others

### Create & Manage Dispatch Queue
* Get Dispatch Queue

```objc
dispatch_queue_t concurrentQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
// The second argument is reserved for future expansion. Just always pass 0 for this argument.

dispatch_queue_t serialQueue = dispatch_get_main_queue();
// Main queue is a special serial queue.
```

* Create Dispatch Queue

```objc
dispatch_queue_t concurrentQueue = \
        dispatch_queue_create("com.inverse.domain.queue1", DISPATCH_QUEUE_CONCURRENT);
// Concurrent

dispatch_queue_t serialQueue = dispatch_queue_create("com.inverse.domain.queue2", NULL);
// Serial: NULL / DISPATCH_QUEUE_SERIAL
```

* Other dispatch queue
    * `dispatch_get_current_queue`

* Memory Management
    * `dispatch_retain`
    * `dispatch_release`

* Customize Queue
* Set up a queue clean up function for serial queue

```objc
void finalizerFunction(void *context) {
    SomeObject *theData = (SomeObject *)context;
    selfCleanUpFunction(theData);
    free(theData);
}

dispatch_queue_t createCustomizedQueue() {
    SomeObject *data = (SomeObject *) malloc(sizeof(SomeObject));
    initializeFunction(data);

    dispatch_queue_t serialQueue = dispatch_queue_create("com.example.serial", NULL);
    dispatch_set_context(serialQueue, data);
    dispatch_set_finalizer_f(serialQueue, &finalizerFunction);

    return serialQueue;
}
```

### Add Task to a Queue
* Single Task

```objc
dispatch_async(someQueue, ^{
    // this will not block current thread
});

dispatch_sync(someQueue, ^{
    // this will block current thread
    // current thread will wait for this block to complete
});
```

* Do not call the `dispatch_sync` function from a task that is executing on the same queue that you pass to your function call. Doing so will deadlock the queue.

* Completion Block

```objc
void async(dispatch_queue_t queue) {
    dispatch_retain(queue);     // retain the queue to make sure it does not disappear

    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFUALT, 0), ^{
        // main work here
        dispatch_async(queue, ^{ completionBlock(); });

        dispatch_release(queue);
    });
}
```

* Loop Iteration

```objc
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);

dispatch_apply(count, queue, ^(size_t i){
    // iteration work
    printf("%u\n", i);
});
```

* Performing Tasks on Main Thread
    * `dispatch_main`

### Suspending & Resuming
* `dispatch_suspend`
* `dispatch_resume`

### Dispatch Semaphores
```objc
#define RESOURCE_COUNT 5

dispatch_semaphore_t fd_sema = dispatch_semaphore_create(RESOURCE_COUNT);

dispatch_semaphore_wait(fd_sema, DISPATCH_TIME_FOREVER);
// use finite resource
dispatch_semaphore_signal(fd_sema);
```

### Waiting For a Group of Task

```objc
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_group_t group = dispatch_group_create();
 
// Add a task to the group
dispatch_group_async(group, queue, ^{
   // Some asynchronous work
});

dispatch_group_wait(group, DISPATCH_TIME_FOREVER);
// dispatch_group_notify(group, queue, ^{
//     // Some asynchronous work
// });
dispatch_release(group);
```

### Dispatch Source
* Timer
* R/W Data from/to a Descriptor
* Monitoring a File System Object
* Monitoring Signals
* Monitoring a Process
* Canceling
* Suspending & Resuming


##### Reference
1. [Concurrency Programming Guide](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008091-CH1-SW1)
2. [Grand Central Dispatch Basic (CHN)](http://www.dreamingwish.com/article/gcdgrand-central-dispatch-jiao-cheng.html)

##### Update Log
* Jun 28, Section 1-2
* Jun 29, Section 3