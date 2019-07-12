# AQS源码学习 一

### 关键信息
> jdk 1.8
> java.util.concurrent.locks.AbstractQueuedSynchronizer

### 摘要
- state字段
- acquire方法
- release方法

### 锁/竞争资源
AQS直译就是“抽象队列同步器”，有如下功能：
- 原子获取竞争资源
- 原子释放竞争资源
- 排队阻塞等待竞争资源
- 支持条件等待队列

使用 锁/竞争资源 一般的流程如下：
```mermaid
graph LR 
id1
```

```flow
st=>start: 开始
e=>end: 结束
lock=>condition: 获取 锁/竞争资源 成功
block=>condition: 不阻塞等待
failCode=>operation: 其它事务
syncCode=>operation: 同步事务
release=>operation: 释放 锁/竞争资源
wait=>operation: 阻塞等待

st->lock()
lock(yes)->syncCode->release->e
lock(no)->block()
block(no)->lock()
block(yes)->failCode->e
```
### volatile int state
```java
//同步状态字段
private volatile int state;
```
关于[volatile](#)，这里不细说，只指出三个性质：
- 原子性
- 可见性
- 有序性

这三个性质也就是锁需要的三个性质，volatile形容的变量本身就可以用来做原子操作，可以用来实现锁或者可竞争资源
&nbsp;
获取锁和释放锁的过程就需要操作这个state字段，但是单使用一个字段不行
需要配合cas操作来进行 判断+更换 的原子操作
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
cas操作判断当前值是否和expect相等，相等则替换成update
由于操作满足原子性，结果满足可见性，所以当一个线程cas修改成功时，其他线程必然修改失败，此处就有了排它性
### acquire
```java
//阻塞获取资源/锁
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```
此方法阻塞等待获取资源，获取失败时会阻塞等待直至获取成功，流程逻辑大致为：
```flow
st=>start: 开始
e=>end: 结束
tryLock=>condition: 获取 锁/竞争资源 成功
queued=>operation: 排队阻塞等待直至获取成功
isInterrupt=>condition: 是否被中断
interrupt=>operation: 维护线程中断状态

st->tryLock()
tryLock(yes)->e
tryLock(no)->queued->isInterrupt()
isInterrupt(yes)->interrupt->e
isInterrupt(no)->e
```
### release
