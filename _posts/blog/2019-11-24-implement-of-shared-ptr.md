### shared_ptr实现原理

#### 引用计数
通过计算引用实际对象的个数，当计数为0时，表示当前对象已经没有指针指向它，可以析构。


#### shared_ptr原理示意图
![简易表示](/images/implement-shared-ptr/simple-shared_ptr.png)

上述示意图为简单表示，所有的shared_ptr共用一个控制块。当增加或减少一个shared_ptr，相应控制块的引用计算也会相应的增加或者减少，当引用计数减少至0时，所管理的控制块和实际对象都将销毁。

#### shared_ptr具体实现（TODO）
标准库里实现的比较精妙，后续将着重分析源码，帮助理解。

