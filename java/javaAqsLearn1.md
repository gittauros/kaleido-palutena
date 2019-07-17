# AQS源码学习 一

### 关键信息
> jdk 1.8<br/>
> java.util.concurrent.locks.AbstractQueuedSynchronizer

### 摘要
- [state字段](#state字段)
- [acquire方法](#acquire)
- [release方法](#release)

### 锁/竞争资源
AQS直译就是“抽象队列同步器”，有如下功能：
- 原子获取竞争资源
- 原子释放竞争资源
- 排队阻塞等待竞争资源
- 支持条件等待队列

使用 锁/竞争资源 一般的流程如下：

```mermaid
graph LR
st(开始)
e(结束)
lock{获取 锁/竞争资源}
block{失败循环等待}
failCode[其它事务]
syncCode[同步事务]
release[释放 锁/竞争资源]
wait[阻塞等待]

st --> lock
lock -->|成功| syncCode
syncCode --> release
release --> e
lock -->|失败| block
block -->|是| wait
wait --> lock
block -->|否| failCode
failCode --> e
```

### state字段
```java
//同步状态字段
private volatile int state;
```
关于[volatile](#)，这里不细说，只指出三个性质：
- 原子性
- 可见性
- 有序性

这三个性质也就是锁需要的三个性质，volatile形容的变量本身就可以用来做原子操作，可以用来实现锁或者可竞争资源<br/>
获取锁和释放锁的过程就需要操作这个state字段，但是单使用一个字段不行<br/>
需要配合cas操作来进行  **判断+更换** 的原子操作
```java
//state的getter
protected final int getState() {
    return state;
}
//state的setter
protected final void setState(int newState) {
    state = newState;
}
//state的cas原子修改操作
protected final boolean compareAndSetState(int expect, int update) {
    // See below for intrinsics setup to support this
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```
cas操作判断当前值是否和expect相等，相等则替换成update<br/>
由于操作满足原子性，结果满足可见性，所以当一个线程cas修改成功时，其他线程必然修改失败，此处就有了排它性
### acquire
```java
//排它获取资源/锁
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```
此方法阻塞等待获取资源，获取失败时会阻塞等待直至获取成功，流程逻辑大致为：

```mermaid
graph LR
st(开始)
e(结束)
tryLock{获取 锁/竞争资源}
isInterrupt{当前线程是否被中断}
queued[排队阻塞等待直至获取成功]
interrupt[维护线程中断状态]

st --> tryLock
tryLock -->|成功| e
tryLock -->|失败| queued
queued --> isInterrupt
isInterrupt -->|是| interrupt
isInterrupt -->|否| e
interrupt --> e
```

*tryAcquire(arg)* 是个模板方法，交由子类实现具体逻辑，方法意为尝试获取资源或锁，只做尝试，不进入队列阻塞等待<br/>
参数arg这里并没展现出具体意义，根据不同的实现会有不同的用处，例如：
- ReentrantLock的实现为增加重入次数
- ReentrantReadWriteLock的实现为增加写锁重入次数

*acquireQueued(addWaiter(Node.EXCLUSIVE), arg)* 这一步将当前线程包装成 **排它节点** 加入等待队列并阻塞当前线程，被唤醒后会再次调用tryAcquire，直至成功为止

*selfInterrupt()* 用来维护中断状态

### acquireShared
有排它获取就有共享获取，也就是 *acquireShared(arg)*
```java
//共享获取资源/锁
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```
这里的逻辑貌似简单了些，其实内部逻辑和acquire方法是一样的，也是失败了会加入等待队列阻塞等待直至成功，成功后会判断维护线程中断状态，逻辑被包含在了doAcquireShared方法中

*tryAcquireShared(arg)* 同样是个模板方法，交由子类实现具体逻辑，意义稍不一样，为以共享方式尝试获取，可以按共享锁来理解<br/>
它的返回值为负数表示失败（实际大多为-1），返回值大于等于0时意义也不尽相同，例如：
- Semaphore的实现为剩余信号量数量
- ReentrantReadWriteLock的实现为直接返回1表示获取读锁成功

实际上可以理解为返回值为**获取后的共享资源余量**

*doAcquireShared(arg)* 会把当前线程包装成 **共享节点** 加入等待队列并阻塞当前线程，被唤醒后会再次调用tryAcquireShared，直至成功为止

### release
```java
//释放锁/资源
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```
此方法尝试释放资源，成功则唤醒等待队列头结点，流程逻辑大致为：

```mermaid
graph LR
st(开始)
e(结束)
tryRelease{释放 锁/竞争资源}
head{头结点存在<br/>且等待状态有意义}
unpark[唤醒头结点]
rTrue[返回成功]
rFalse[返回失败]

st --> tryRelease
tryRelease -->|成功| head
tryRelease -->|失败| rFalse
head -->|是| unpark
head -->|否| rTrue
unpark --> rTrue
rFalse --> e
rTrue --> e
```

*tryRelease(arg)* 与之前相同，模板方法，子类实现，释放成功返回true，举一些返回值例子：
- ReentrantLock的实现为释放后锁是否完全被释放，也就是重入数为0
- ReentrantReadWriteLock的实现为写锁的重入数是否为0（写锁完全释放）

所以ReentrantLock和ReentrantReadWriteLock是当 **排它锁完全释放** 时才会唤醒等待队列去竞争锁，避免了一些无效竞争
> 虽然release方法会把tryRelease的值返回出去，但是ReentrantLock和ReentrantReadWriteLock的unlock方法都不关心release的返回值

*unparkSuccessor* 会清空取消等待的节点并唤醒等待队列的第一个有效等待节点的线程

### releaseShared
```java
//释放共享模式锁/资源
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```
释放共享模式资源，如读锁、信号量<br/>
与release方法整体逻辑也差不多，其中的头结点状态判断和线程唤醒也都是被包含在了doReleaseShared方法中

*tryReleaseShared(arg)* 又双叒是个模板方法，子类实现，案例附上：
- Semaphore的实现为剩余信号量释放数量
- ReentrantReadWriteLock的实现为释放一次读锁，其中arg参数未被使用，固定是释放一次读锁

此方法返回值意义也不尽相同：
- Semaphore的实现为释放信号量成功，除非数量溢出，否则一定为true
- ReentrantReadWriteLock的实现为是否全部读锁都释放完毕（包括每个读锁的重入数也都完全释放）

*doReleaseShared* 会判断是否有节点需要唤醒，是的话会清空等待队列的取消等待节点，唤醒第一个有效节点开始以共享模式竞争

### 总结
本篇基本介绍了AQS提供的 **锁/竞争资源** 统一抽象竞争流程：
- 竞争
- 阻塞等待
- 释放

通过acquire和acquireShared方法来竞争获取，release和releaseShared方法来释放资源<br/>
子类通过实现tryXXX方法，**结合原子操作修改state字段** 则可以实现不同的效果，例如：
- 信号量
- 可重入锁
- 可重入读写锁

> state字段的原子操作，基本都是在子类实现的tryXXX方法中操作的，不在AQS中<br/>
> 关于等待队列结构、acquireQueued、doAcquireShared、unparkSuccessor本篇不细说
