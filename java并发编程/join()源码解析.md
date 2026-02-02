
# ** join()方法定义**

```java
// Thread类中的join方法
public final void join() throws InterruptedException {
    join(0);
}

public final synchronized void join(long millis) throws InterruptedException {
    long base = System.currentTimeMillis();
    long now = 0;

    if (millis < 0) {
        throw new IllegalArgumentException("timeout value is negative");
    }

    if (millis == 0) {
        // ⚠️ 关键：检查线程是否存活
        while (isAlive()) {
            wait(0);  // 调用Object.wait(0)
        }
    } else {
        while (isAlive()) {
            long delay = millis - now;
            if (delay <= 0) {
                break;
            }
            wait(delay);  // 调用Object.wait(timeout)
            now = System.currentTimeMillis() - base;
        }
    }
}

public final synchronized void join(long millis, int nanos) throws InterruptedException {
    if (millis < 0) {
        throw new IllegalArgumentException("timeout value is negative");
    }
    
    if (nanos < 0 || nanos > 999999) {
        throw new IllegalArgumentException(
            "nanosecond timeout value out of range");
    }
    
    if (nanos >= 500000 || (nanos != 0 && millis == 0)) {
        millis++;
    }
    
    join(millis);
}
```

---
# join()是否释放锁？

### **1. 源码分析**

```java
public final synchronized void join(long millis) throws InterruptedException {
    // ⚠️ 注意：join方法是synchronized的！
    // 它锁的是当前Thread实例（被等待的线程对象）
    
    while (isAlive()) {
        wait(delay);  // ⚠️ 关键：调用wait()会释放锁！
    }
}
```

### **2. 锁的层次分析**

```java
public class JoinLockTest {
    public static void main(String[] args) throws InterruptedException {
        Thread child = new Thread(() -> {
            try {
                System.out.println("子线程开始执行");
                Thread.sleep(2000);
                System.out.println("子线程结束执行");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        
        Thread parent = new Thread(() -> {
            synchronized (child) {  // 主线程不持有child锁
                System.out.println("parent线程获得child锁");
                try {
                    // 这里调用child.join()时，parent线程持有child锁
                    child.join();  // ⚠️ join内部会wait()释放child锁
                    System.out.println("parent线程join完成");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        
        // 另一个线程尝试获取child锁
        Thread other = new Thread(() -> {
            try {
                Thread.sleep(500);  // 确保parent先获取锁
                System.out.println("other线程尝试获取child锁");
                synchronized (child) {
                    System.out.println("other线程获得child锁");
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        
        child.start();
        parent.start();
        other.start();
        
        // 输出：
        // parent线程获得child锁
        // 子线程开始执行
        // other线程尝试获取child锁
        // other线程获得child锁  ⚠️ 证明join期间child锁被释放了！
        // （等待子线程结束）
        // 子线程结束执行
        // parent线程join完成
    }
}
```

### **3. JVM层实现分析**

```cpp
// 当调用join()时，实际上是：
1. 获取当前线程对象（被等待的线程）的锁
2. 检查线程是否存活（isAlive()）
3. 如果存活，调用wait()等待
4. wait()会释放当前线程对象的锁
5. 子线程结束时，JVM会调用notifyAll()唤醒所有等待者
6. join()方法重新获得锁并返回
```
### **4. 关键机制**

```java
// Thread类的exit()方法（线程结束时调用）
private void exit() {
    synchronized (this) {  // ⚠️ 锁住自己（线程对象）
        // ... 清理资源
        notifyAll();  // ⚠️ 关键：唤醒所有在this对象上wait的线程
    }
}
```

**结论：join()会释放锁**（释放当前Thread实例的锁）

---

# join()是否中断敏感？

### **1. 源码分析**

```java
public final synchronized void join(long millis) throws InterruptedException {
    // ⚠️ 方法声明抛出InterruptedException
    
    while (isAlive()) {
        wait(delay);  // ⚠️ wait()是中断敏感的
    }
}
```
### **2. 中断传播机制**

```java
public class JoinInterruptTest {
    public static void main(String[] args) throws InterruptedException {
        Thread child = new Thread(() -> {
            try {
                System.out.println("子线程开始执行，将执行10秒");
                Thread.sleep(10000);  // 长时间运行
                System.out.println("子线程正常结束");
            } catch (InterruptedException e) {
                System.out.println("子线程被中断");
            }
        });
        
        Thread parent = new Thread(() -> {
            try {
                System.out.println("parent线程开始join");
                child.join();  // ⚠️ 等待子线程
                System.out.println("parent线程join完成");
            } catch (InterruptedException e) {
                System.out.println("parent线程join被中断！");
                // 中断parent也会中断child吗？ ⚠️ 不会！
                System.out.println("child线程状态: " + child.getState());
                System.out.println("child是否存活: " + child.isAlive());
            }
        });
        
        child.start();
        parent.start();
        
        // 2秒后中断parent线程
        Thread.sleep(2000);
        System.out.println("主线程中断parent线程");
        parent.interrupt();
        
        parent.join();
        
        // 检查child线程状态
        Thread.sleep(100);
        System.out.println("最终child状态: " + child.getState());
        
        // 输出：
        // 子线程开始执行，将执行10秒
        // parent线程开始join
        // （等待2秒）
        // 主线程中断parent线程
        // parent线程join被中断！
        // child线程状态: TIMED_WAITING
        // child是否存活: true
        // 最终child状态: TIMED_WAITING
        // （子线程继续sleep，8秒后结束）
    }
}
```
### **3. 中断的精确控制**

```java
public class JoinInterruptPropagation {
    public static void main(String[] args) {
        Thread child = new Thread(() -> {
            try {
                Thread.sleep(5000);
                System.out.println("子线程完成");
            } catch (InterruptedException e) {
                System.out.println("子线程被中断");
            }
        });
        
        child.start();
        
        // 创建中断parent的线程
        Thread interrupter = new Thread(() -> {
            try {
                Thread.sleep(1000);
                System.out.println("中断parent线程");
                Thread.currentThread().interrupt();
                
                // 尝试join，但当前线程已中断
                child.join();  // ⚠️ 会立即抛出InterruptedException
            } catch (InterruptedException e) {
                System.out.println("join被中断，但child线程继续运行");
                System.out.println("child状态: " + child.getState());
            }
        });
        
        interrupter.start();
        
        // 输出：
        // （1秒后）
        // 中断parent线程
        // join被中断，但child线程继续运行
        // child状态: TIMED_WAITING
        // （4秒后）
        // 子线程完成
    }
}
```
### **4. JVM层的中断处理**

```cpp
// 当join()等待时被中断：
1. wait()方法检测到中断
2. 抛出InterruptedException
3. join()方法捕获异常并重新抛出
4. 线程的interrupt状态在异常抛出后被清除
5. ⚠️ 重要：被等待的线程（子线程）不受影响
```
**结论：join()是中断敏感的**，但中断只会影响等待的线程，不会影响被等待的线程

---

# join()是否释放CPU？

### **1. 源码分析**

```java
public final synchronized void join(long millis) throws InterruptedException {
    while (isAlive()) {
        wait(delay);  // ⚠️ 调用Object.wait()释放CPU
    }
}
```
### **2. CPU释放机制**

```java
public class JoinCPUTest {
    private static volatile long counter = 0;
    
    public static void main(String[] args) throws InterruptedException {
        // 创建长时间运行的子线程
        Thread child = new Thread(() -> {
            long start = System.currentTimeMillis();
            while (System.currentTimeMillis() - start < 3000) {
                counter++;  // 高CPU占用
            }
            System.out.println("子线程完成，counter=" + counter);
        });
        
        // 创建等待线程
        Thread waiter = new Thread(() -> {
            System.out.println("waiter开始join子线程");
            long startTime = System.currentTimeMillis();
            
            try {
                child.join();  // ⚠️ 等待期间释放CPU
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            
            long endTime = System.currentTimeMillis();
            System.out.println("waiter等待时间: " + (endTime - startTime) + "ms");
        });
        
        // 创建另一个高CPU占用的线程
        Thread busyThread = new Thread(() -> {
            long busyStart = System.currentTimeMillis();
            long localCounter = 0;
            while (System.currentTimeMillis() - busyStart < 2000) {
                localCounter++;  // 高CPU占用
            }
            System.out.println("busyThread执行完成，localCounter=" + localCounter);
        });
        
        child.start();
        waiter.start();
        Thread.sleep(100);  // 确保waiter开始join
        busyThread.start();
        
        waiter.join();
        busyThread.join();
        
        // 输出：
        // waiter开始join子线程
        // （期间busyThread可以获得大量CPU时间）
        // busyThread执行完成，localCounter=很高的值
        // 子线程完成，counter=很高的值
        // waiter等待时间: 3000ms左右
    }
}
```
### **3. 底层系统调用**

```cpp
// join() -> wait() -> 系统调用
调用链：
Thread.join()
  ↓
Object.wait()        // Java层
  ↓
ObjectMonitor::wait() // JVM层
  ↓
ParkEvent::park()    // JVM层
  ↓
pthread_cond_wait()  // Linux系统调用
  ↓
futex()              // 内核系统调用 ⚠️ 释放CPU！
```
### **4. 对比sleep()和wait()的CPU释放**

```java
public class CPUReleaseComparison {
    public static void main(String[] args) throws InterruptedException {
        System.out.println("测试三种方式的CPU释放效果：");
        
        // 1. sleep释放CPU
        new Thread(() -> {
            long start = System.nanoTime();
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            long end = System.nanoTime();
            System.out.println("sleep耗时: " + (end - start)/1000000 + "ms");
        }).start();
        
        // 2. wait释放CPU
        Object lock = new Object();
        new Thread(() -> {
            synchronized (lock) {
                long start = System.nanoTime();
                try {
                    lock.wait(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                long end = System.nanoTime();
                System.out.println("wait耗时: " + (end - start)/1000000 + "ms");
            }
        }).start();
        
        // 3. join释放CPU
        Thread child = new Thread(() -> {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        child.start();
        
        new Thread(() -> {
            long start = System.nanoTime();
            try {
                child.join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            long end = System.nanoTime();
            System.out.println("join耗时: " + (end - start)/1000000 + "ms");
        }).start();
    }
}
```
**结论：join()会释放CPU**，因为它底层调用了wait()