---
layout: post
title: 线程间通信
author: 细雪
header-style: text
lang: en
published: true
categories:  多线程
tags:
  - 多线程
---

# 线程间通信常见问题
1. 三个线程分别打印 A，B，C，要求这三个线程一起运行，打印 n 次，输出形如“ABCABCABC…”
2. 两个线程交替打印 0~100 的奇偶数
3. 通过 N 个线程顺序循环打印从 0 至 100
4. 多线程按顺序调用，A->B->C，AA 打印 5 次，BB 打印10 次，CC 打印 15 次，重复 10 次
5. 用两个线程，一个输出字母，一个输出数字，交替输出 1A2B3C4D…26Z

## Lock解法 第一题
``` java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class PrintABCUsingLock {

    private int times; // 控制打印次数
    private int state;   // 当前状态值：保证三个线程之间交替打印
    private Lock lock = new ReentrantLock();

    public PrintABCUsingLock(int times) {
        this.times = times;
    }

    private void printLetter(String name, int targetNum) {

        for (int i = 0; i  < times;){
             lock.lock();
             if (state % 3 == targetNum) {
                 state++;
                 i++;
                 System.out.print(name);
            }
            lock.unlock();
        }
}

    public static void main(String[] args) {

        //顺序打印10次
        PrintABCUsingLock loopThread = new PrintABCUsingLock(10);

        new Thread(() -> {
            loopThread.printLetter("A", 0);
        }, "A").start();

        new Thread(() -> {
            loopThread.printLetter("B", 1);
        }, "B").start();
        
        new Thread(() -> {
            loopThread.printLetter("C", 2);
        }, "C").start();
    }

}
```

## wait/notify解法 第一题
``` java
public class PrintABCUsingWaitNotify {

    private int state;
    private int times;
    private static final Object LOCK = new Object();

    public PrintABCUsingWaitNotify(int times) {
        this.times = times;
    }

    private void printLetter(String name, int targetState) {
        for (int i = 0; i < times; i++)
            synchronized (LOCK) {
                while (state % 3 != targetState) {
                try {
                    LOCK.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            state++;
            System.out.print(name);
            LOCK.notifyAll();
        }
    }

    public static void main(String[] args) {

        PrintABCUsingWaitNotify printABC = new PrintABCUsingWaitNotify(10);
        new Thread(() -> {
            printABC.printLetter("A", 0);
        }, "A").start();
        new Thread(() -> {
            printABC.printLetter("B", 1);
        }, "B").start();
        new Thread(() -> {
            printABC.printLetter("C", 2);
        }, "C").start();
    }

}
```

## wait/notify解法 第二题
``` java
package cn.wideth.util.thread;

public class OddEvenPrinter {

    private Object monitor = new Object();
    private final int limit;
    private volatile int count;

    OddEvenPrinter(int initCount, int times) {
        this.count = initCount;
        this.limit = times;
    }

    private void print() {

        synchronized (monitor) {
            while (count < limit){
            try {
                System.out.println(String.format("线程[%s]打印数字:%d", Thread.currentThread().getName(), ++count));
                monitor.notifyAll();
                monitor.wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        //防止有子线程被阻塞未被唤醒，导致主线程不退出
        monitor.notifyAll();
      }
   }

    public static void main(String[] args) {

        OddEvenPrinter printer = new OddEvenPrinter(0, 10);
        new Thread(printer::print, "odd").start();
        new Thread(printer::print, "even").start();
    }
}
```

## wait/notify解法 第五题
``` java 
package cn.wideth.util.thread;

public class NumAndLetterPrinter {

    private static char c = 'A';
    private static int i = 0;
    static final Object lock = new Object();

    public static void main(String[] args) {
        new Thread(() -> printer(), "numThread").start();
        new Thread(() -> printer(), "letterThread").start();
    }

    private static void printer() {
        synchronized (lock) {
            for (int i = 0; i < 26; i++) {
                if (Thread.currentThread().getName() == "numThread") {
                    //打印数字1-26
                    System.out.print((i + 1));
                    // 唤醒其他在等待的线程
                    lock.notifyAll();
                    try {
                        // 让当前线程释放锁资源，进入wait状态
                        lock.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                } else if (Thread.currentThread().getName() == "letterThread") {
                    // 打印字母A-Z
                    System.out.print((char) ('A' + i));
                    // 唤醒其他在等待的线程
                    lock.notifyAll();
                    try {
                        // 让当前线程释放锁资源，进入wait状态
                        lock.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
            lock.notifyAll();
        }
    }
}
```

## Condition解法 第一题
``` java
package cn.wideth.util.thread;

import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class PrintABCUsingLockCondition {

    private int times;
    private int state;
    private static Lock lock = new ReentrantLock();
    private static Condition c1 = lock.newCondition();
    private static Condition c2 = lock.newCondition();
    private static Condition c3 = lock.newCondition();

    public PrintABCUsingLockCondition(int times) {
        this.times = times;
    }

    public static void main(String[] args) {
        PrintABCUsingLockCondition print = new PrintABCUsingLockCondition(10);
        new Thread(() -> {
            print.printLetter("A", 0, c1, c2);
        }, "A").start();
        new Thread(() -> {
            print.printLetter("B", 1, c2, c3);
        }, "B").start();
        new Thread(() -> {
            print.printLetter("C", 2, c3, c1);
        }, "C").start();
    }

    private void printLetter(String name, int targetState, Condition current, Condition next) {
        for (int i = 0; i < times;){
            lock.lock();
            try {
               while (state % 3 != targetState) {
                  current.await();
               }
            state++;
            i++;
            System.out.print(name);
            next.signal();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
      }
   }
}
```

## Semaphore解法 第一题
``` java
import java.util.concurrent.Semaphore;

public class PrintABCUsingSemaphore {

    public static void main(String[] args) {
        // 初始化许可数为1，A线程可以先执行
        Semaphore semaphoreA = new Semaphore(1);
        // 初始化许可数为0，B线程阻塞
        Semaphore semaphoreB = new Semaphore(0);
        // 初始化许可数为0，C线程阻塞
        Semaphore semaphoreC = new Semaphore(0);

        new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                try {
                    // A线程获得许可，同时semaphoreA的许可数减为0,进入下一次循环时
                    // A线程会阻塞，知道其他线程执行semaphoreA.release();
                    semaphoreA.acquire();
                    // 打印当前线程名称
                    System.out.print(Thread.currentThread().getName());
                    // semaphoreB许可数加1
                    semaphoreB.release();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }, "A").start();

        new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                try {
                    semaphoreB.acquire();
                    System.out.print(Thread.currentThread().getName());
                    semaphoreC.release();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }, "B").start();

        new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                try {
                    semaphoreC.acquire();
                    System.out.print(Thread.currentThread().getName());
                    semaphoreA.release();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }, "C").start();
    }
}
```

## Semaphore解法 第三题
``` java
public class LoopPrinter {

    //线程个数
    private final static int THREAD_COUNT = 3;
    private static int result = 0;
    //最大数字
    private static int maxNum = 10;

    public static void main(String[] args) throws InterruptedException {

        final Semaphore[] semaphores = new Semaphore[THREAD_COUNT];
        for (int i = 0; i  < THREAD_COUNT; i++){           //非公平信号量，每个信号量初始计数都为1
            semaphores[i] = new Semaphore(1);
            if (i != THREAD_COUNT - 1) {
               //  System.out.println(i+"==="+semaphores[i].getQueueLength());
               //获取一个许可前线程将一直阻塞, for 循环之后只有 syncObjects[2] 没有被阻塞
                 semaphores[i].acquire();
            }
        }

        for (int i = 0; i  < THREAD_COUNT; i++){          // 初次执行，上一个信号量是 syncObjects[2]
            final Semaphore lastSemphore = i == 0 ? semaphores[THREAD_COUNT - 1] : semaphores[i - 1];
            final Semaphore currentSemphore = semaphores[i];
            final int index = i;
            new Thread(() -> {
            try {
             while (true) {
                // 初次执行，让第一个 for 循环没有阻塞的 syncObjects[2] 先获得令牌阻塞了
                lastSemphore.acquire();
                System.out.println("thread" + index + ": " + result++);
                if (result > maxNum) {
                    System.exit(0);
                }
                // 释放当前的信号量，syncObjects[0] 信号量此时为 1，下次 for 循环中上一个信号量即为syncObjects[0]
                currentSemphore.release();
             }
         } catch (Exception e) {
            e.printStackTrace();
        }
        }).start();
       }
    }
}
```

## LockSupport解法 第一题
``` java
import java.util.concurrent.locks.LockSupport;

public class PrintABCUsingLockSupport {

    private static Thread threadA, threadB, threadC;

    public static void main(String[] args) {
        threadA = new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                // 打印当前线程名称
                System.out.print(Thread.currentThread().getName());
                // 唤醒下一个线程
                LockSupport.unpark(threadB);
                // 当前线程阻塞
                LockSupport.park();
            }
        }, "A");
        threadB = new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                // 先阻塞等待被唤醒
                LockSupport.park();
                System.out.print(Thread.currentThread().getName());
                // 唤醒下一个线程
                LockSupport.unpark(threadC);
            }
        }, "B");
        threadC = new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                // 先阻塞等待被唤醒
                LockSupport.park();
                System.out.print(Thread.currentThread().getName());
                // 唤醒下一个线程
                LockSupport.unpark(threadA);
            }
        }, "C");
        threadA.start();
        threadB.start();
        threadC.start();
    }
}
```
## LockSupport解法 第五题
``` java
import java.util.concurrent.locks.LockSupport;

public class NumAndLetterPrinterByLockSupport {

    private static Thread numThread, letterThread;

    public static void main(String[] args) {

        letterThread = new Thread(() -> {
            for (int i = 0; i < 26; i++) {
                System.out.print((char) ('A' + i));
                LockSupport.unpark(numThread);
                LockSupport.park();
            }
        }, "letterThread");

        numThread = new Thread(() -> {
            for (int i = 1; i <= 26; i++) {
                System.out.print(i);
                LockSupport.park();
                LockSupport.unpark(letterThread);
            }
        }, "numThread");
        numThread.start();
        letterThread.start();
    }
}
```