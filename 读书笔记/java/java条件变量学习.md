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
            Node node = addConditionWaiter();       //取当前线程，并将其放入condition的队列中去
            int savedState = fullyRelease(node);
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {       //判断当前线程是否在待执行队列中，若在，跳出循环，执行下面操作，若不在，则自旋等待
                LockSupport.park(this);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }

            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)    //再次获取锁，继续线程
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }

    

singal源码如下：

     public final void signal() {
            if (!isHeldExclusively())        //判断当前线程是否持有锁，如果没有，则失败，有则继续
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignal(first);          //将condition队列中的第一个线程放入待执行队列中，参与获取锁
        }

    private void doSignal(Node first) {
            do {
                if ( (firstWaiter = first.nextWaiter) == null)           //conditon 队列为空，表示无挂起线程
                    lastWaiter = null;
                first.nextWaiter = null;
            } while (!transferForSignal(first) &&
                     (first = firstWaiter) != null);
        }

    final boolean transferForSignal(Node node) {
       
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;
        Node p = enq(node);                                        //放入待执行队列中
        int ws = p.waitStatus;
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);
        return true;
    }


以上只是简单走读了JDK的关于condition相关代码。

condition用户参见rpc工程的test目录