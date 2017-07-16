java条件变量即Java condition，在多线程时经常用到。常与锁（Lock）一块使用。

`condition`初始化通过Lock的new condition（）方法。

condition常用方法：

1. await()方法会使当前线程等待，并释放锁，当其他线程调用singal（）或singalall（）时，激活当前线程重新获取锁并执行。
2. awaitUninterruptibly() 同await，但不会在等待过程中中断。
3. singal（）唤醒一个等待的线程。singalAll（）会唤醒所有等待线程。


###condition源码分析
啥也不说，撸一波源码：

     public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                LockSupport.park(this);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }
    