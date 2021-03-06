wait和notify方法在Object类中，类定义如下：

```
public final void wait() throws InterruptedException；

public final native void wait(long timeout) throws InterruptedException;

public final native void notify();

public final native void notifyAll();

```

当在一个对象实例上调用wait线程，当前线程就会等待，并释放当前获取的对象锁。直到有线程调用该对象的notify方法。
示例代码：
```

package com.fuyzh.netty.test.concurrent;

public class TestWaitAndNotify {

    final static Object o = new Object();

    public static class T1 implements Runnable {

        public void run() {
            synchronized (o) {
                try {
                    System.out.println("T1 start..." + System.currentTimeMillis());

                    o.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("T1 end..." + System.currentTimeMillis());

            }
        }
    }

    public static class T2 implements Runnable {

        public void run() {
            synchronized (o) {
                try {
                    System.out.println("T2 start..." + System.currentTimeMillis());
                    Thread.sleep(2000l);
                    o.notify();

                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("T2 end..." + System.currentTimeMillis());

            }
        }
    }

    public static void main(String[] args) {
        Thread t1 = new Thread(new T1());
        Thread t2 = new Thread(new T2());
        t1.start();
        t2.start();
    }
}



```

当线程调用了wait方法后，该线程就进入了该对象的等待队列。当调用notify方法后，会从等待队列中随机取一个线程唤醒。若调用notifyall方法，则将等待队列中的所有线程唤醒。

wait和sleep的区别：调用wait方法，该线程会将对象锁释放。sleep不会。