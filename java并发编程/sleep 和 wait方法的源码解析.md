# sleep()方法源码深度解析

### **1. sleep()是否释放锁？**

#### **源码验证**

```java
// Thread.sleep()源码关键点
public static native void sleep(long millis) throws InterruptedException;

// sleep是Thread类的静态方法，与对象锁无关
// 这意味着sleep()本身不涉及任何锁操作
```
#### **JVM层实现**

```cpp
// HotSpot JVM中的sleep实现（关键部分）
JVM_ENTRY(void, JVM_Sleep(JNIEnv* env, jclass threadClass, jlong millis))
  // 1. 获取当前线程
  JavaThread* current_thread = JavaThread::thread_from_jni_environment(env);
  
  // 2. 检查中断状态
  if (Thread::is_interrupted(current_thread, true)) {
    // 抛出中断异常
  }
  
  // 3. 调用操作系统sleep - 注意这里没有释放任何锁！
  if (millis == 0) {
    os::yield();
  } else {
    // ⚠️ 关键：这里直接调用os::sleep，不会释放Java层的锁
    os::sleep(current_thread, millis);
  }
  
  // 4. 再次检查中断
  if (Thread::is_interrupted(current_thread, true)) {
    // 抛出中断异常
  }
JVM_END
```
#### **验证示例**

```java
public class SleepLockTest {
    private static final Object lock = new Object();
    
    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(() -> {
            synchronized (lock) {
                System.out.println("t1获得锁 - 时间: " + System.currentTimeMillis());
                try {
                    // ⚠️ sleep期间不会释放锁！
                    Thread.sleep(3000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("t1释放锁 - 时间: " + System.currentTimeMillis());
            }
        });
        
        Thread t2 = new Thread(() -> {
            System.out.println("t2尝试获取锁 - 时间: " + System.currentTimeMillis());
            synchronized (lock) {
                System.out.println("t2获得锁 - 时间: " + System.currentTimeMillis());
            }
        });
        
        t1.start();
        Thread.sleep(100); // 确保t1先获取锁
        t2.start();
        
        // 输出：
        // t1获得锁 - 时间: 1678888888888
        // t2尝试获取锁 - 时间: 1678888888889
        // （等待3秒）
        // t1释放锁 - 时间: 1678888891888
        // t2获得锁 - 时间: 1678888891888
    }
}
```

**结论：sleep()不释放锁**

---

### **2. sleep()是否中断敏感？**

#### **源码验证**

```java
public static void sleep(long millis) throws InterruptedException {
    // 方法声明中明确抛出InterruptedException
    // 这说明sleep是中断敏感的
}
```
#### **JVM层中断检查**

```cpp
// sleep中的中断检查（两次检查）
void os::sleep(JavaThread* thread, jlong millis) {
  // 第一次检查（在进入sleep前）
  if (Thread::is_interrupted(thread, true)) {
    throw_interruptedException();
    return;
  }
  
  // 进入系统sleep
  int result = pthread_cond_timedwait(...);
  
  // 第二次检查（sleep返回后）
  if (Thread::is_interrupted(thread, true)) {
    throw_interruptedException();
  }
}
```
#### **中断机制实现**

```cpp
// 线程中断的底层实现
void os::interrupt(Thread* thread) {
  OSThread* osthread = thread->osthread();
  if (osthread != NULL) {
    // 设置中断标志
    osthread->set_interrupted(true);
    
    // 唤醒被park/sleep的线程
    ParkEvent * const slp = thread->_SleepEvent;
    if (slp != NULL) {
      slp->unpark();  // ⚠️ 这里唤醒sleep的线程！
    }
    
    // 对于pthread_cond_timedwait的唤醒
    pthread_cond_signal(&_cond);
  }
}
```
#### **验证示例**

```java
public class SleepInterruptTest {
    public static void main(String[] args) throws InterruptedException {
        Thread sleeper = new Thread(() -> {
            System.out.println("线程开始sleep");
            try {
                Thread.sleep(5000);
                System.out.println("sleep正常结束");
            } catch (InterruptedException e) {
                System.out.println("sleep被中断！中断状态: " + 
                    Thread.currentThread().isInterrupted());
                // 中断状态被清除，这里是false
            }
        });
        
        sleeper.start();
        
        // 2秒后中断sleep
        Thread.sleep(2000);
        System.out.println("主线程中断sleeper");
        sleeper.interrupt();  // 这会导致sleep立即抛出InterruptedException
        
        sleeper.join();
        
        // 输出：
        // 线程开始sleep
        // （等待2秒）
        // 主线程中断sleeper
        // sleep被中断！中断状态: false
    }
}
```
**结论：sleep()是中断敏感的**

---

### **3. sleep()是否释放CPU？**

#### **操作系统层实现**
```cpp
// Linux平台sleep实现
int os::sleep(JavaThread* thread, jlong millis) {
  // 转换为纳秒
  struct timespec ts;
  ts.tv_sec = millis / 1000;
  ts.tv_nsec = (millis % 1000) * 1000000;
  
  // 调用nanosleep系统调用
  // ⚠️ 这会立即让出CPU！
  int result = nanosleep(&ts, NULL);
  
  return result;
}

// Windows平台实现
int os::sleep(JavaThread* thread, jlong millis) {
  // 调用Sleep系统调用
  // ⚠️ 这会立即让出CPU！
  DWORD dwMilliseconds = (DWORD)millis;
  Sleep(dwMilliseconds);
  
  return 0;
}
```
#### **sleep(0)的特殊情况**

```java
// sleep(0)的实现
if (millis == 0) {
    // 调用yield而不是sleep
    os::yield();
}

// os::yield的实现
void os::yield() {
  // Linux: sched_yield()
  // Windows: SwitchToThread()
  // ⚠️ 这会让出CPU给其他线程，但可能立即又被调度
  sched_yield();
}
```
#### **验证示例**

```java
public class SleepCPUTest {
    private static volatile long counter = 0;
    
    public static void main(String[] args) throws InterruptedException {
        // 创建两个高CPU占用的线程
        Thread busyThread = new Thread(() -> {
            while (counter < 1000000000) {
                counter++;
            }
        });
        
        Thread sleepingThread = new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                System.out.println("sleepingThread执行 - 时间: " + System.currentTimeMillis());
                try {
                    // ⚠️ sleep期间释放CPU，busyThread可以获得更多CPU时间
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        
        long startTime = System.currentTimeMillis();
        busyThread.start();
        sleepingThread.start();
        
        busyThread.join();
        sleepingThread.join();
        
        long duration = System.currentTimeMillis() - startTime;
        System.out.println("总耗时: " + duration + "ms, counter: " + counter);
    }
}
```
**结论：sleep()会释放CPU**

---

# wait()方法源码深度解析

### **1. wait()是否释放锁？**

#### **源码验证**

```java
// Object.wait()的JNI声明
public final native void wait(long timeout) throws InterruptedException;

// wait()必须在同步块中调用
synchronized (obj) {
    obj.wait();  // 调用wait会释放obj的锁
}
```
#### **JVM层实现 - 关键：释放锁！**
```cpp
// ObjectMonitor::wait的实现（关键部分）
void ObjectMonitor::wait(jlong millis, bool interruptible, TRAPS) {
  // 1. 获取当前线程
  Thread * const Self = THREAD;
  
  // 2. 检查是否持有锁
  if (Self != _owner) {
    // 如果没有持有锁，抛出IllegalMonitorStateException
    THROW_MSG(vmSymbols::java_lang_IllegalMonitorStateException(),
              "current thread not owner");
  }
  
  // 3. ⚠️ 关键步骤：释放锁！
  // exit()方法会释放monitor锁
  exit(true, Self);
  
  // 4. 创建等待节点并加入等待队列
  ObjectWaiter node(Self);
  AddWaiter(&node);
  
  // 5. 进入等待状态
  Self->_ParkEvent->park(millis);
  
  // 6. ⚠️ 被唤醒后重新竞争锁
  // enter()方法会尝试重新获取锁
  int ret = enter(Self);
  
  // 7. 从等待队列移除
  DequeueSpecificWaiter(&node);
}
```
#### **释放锁的具体实现**

```cpp
// ObjectMonitor::exit的实现
void ObjectMonitor::exit(bool not_suspended, TRAPS) {
  // 1. 减少递归计数
  if (_recursions != 0) {
    _recursions--;  // 重入锁减少计数
    return;
  }
  
  // 2. 清除所有者
  _owner = NULL;  // ⚠️ 关键：设置为NULL表示释放锁！
  
  // 3. 如果有等待的线程，唤醒一个
  if (_EntryList != NULL) {
    // 唤醒_EntryList中的线程
    ExitEpilog(Self, not_suspended);
  }
  
  // 4. 内存屏障，确保可见性
  OrderAccess::release();
}
```
#### **验证示例**
```java

public class WaitLockTest {
    private static final Object lock = new Object();
    
    public static void main(String[] args) throws InterruptedException {
        Thread waiter = new Thread(() -> {
            synchronized (lock) {
                System.out.println("waiter获得锁 - 时间: " + System.currentTimeMillis());
                try {
                    // ⚠️ wait()会立即释放锁！
                    lock.wait(3000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("waiter重新获得锁 - 时间: " + System.currentTimeMillis());
            }
        });
        
        Thread worker = new Thread(() -> {
            // 等待100ms确保waiter先获得锁
            try { Thread.sleep(100); } catch (InterruptedException e) {}
            
            System.out.println("worker尝试获取锁 - 时间: " + System.currentTimeMillis());
            synchronized (lock) {
                System.out.println("worker获得锁 - 时间: " + System.currentTimeMillis());
                try {
                    Thread.sleep(1000);  // 模拟工作
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("worker释放锁 - 时间: " + System.currentTimeMillis());
            }
        });
        
        waiter.start();
        worker.start();
        
        // 输出：
        // waiter获得锁 - 时间: 1678888888888
        // worker尝试获取锁 - 时间: 1678888888889
        // worker获得锁 - 时间: 1678888888889  （立即获得，说明锁已被释放）
        // worker释放锁 - 时间: 1678888889889  （1秒后）
        // waiter重新获得锁 - 时间: 1678888889889  （重新获得锁）
    }
}
```
**结论：wait()会释放锁**

---

### **2. wait()是否中断敏感？**

#### **源码验证**

```java
public final void wait() throws InterruptedException {
    wait(0);  // 方法声明抛出InterruptedException
}
```
#### **JVM层中断处理**

```cpp
void ObjectMonitor::wait(jlong millis, bool interruptible, TRAPS) {
  // 1. 进入时检查中断
  if (interruptible && Thread::is_interrupted(Self, true)) {
    THROW(vmSymbols::java_lang_InterruptedException());
  }
  
  // 2. 等待期间可能被中断
  // park()方法支持中断
  int ret = Self->_ParkEvent->park(millis);
  
  // 3. 被唤醒后检查中断
  if (interruptible && Thread::is_interrupted(Self, true)) {
    // 如果是被中断唤醒的，抛出异常
    THROW(vmSymbols::java_lang_InterruptedException());
  }
}
```
#### **中断唤醒的实现**

```cpp
// ParkEvent的unpark方法
void ParkEvent::unpark() {
  // 设置unpark标志
  _counter = 1;
  
  // 唤醒等待的线程
  int status = pthread_cond_signal(&_cond);
  
  // 如果是中断触发的unpark，会设置中断标志
  if (Thread::is_interrupted(thread, false)) {
    thread->set_interrupted(true);
  }
}
```
#### **验证示例**

```java
public class WaitInterruptTest {
    private static final Object lock = new Object();
    
    public static void main(String[] args) throws InterruptedException {
        Thread waiter = new Thread(() -> {
            synchronized (lock) {
                System.out.println("waiter获得锁并开始wait");
                try {
                    lock.wait();  // 无限等待
                    System.out.println("waiter被正常唤醒");
                } catch (InterruptedException e) {
                    System.out.println("waiter被中断唤醒！中断状态: " + 
                        Thread.currentThread().isInterrupted());
                }
            }
        });
        
        waiter.start();
        
        // 等待1秒让waiter进入wait状态
        Thread.sleep(1000);
        
        // 中断waiter
        System.out.println("主线程中断waiter");
        waiter.interrupt();
        
        waiter.join();
        
        // 输出：
        // waiter获得锁并开始wait
        // （等待1秒）
        // 主线程中断waiter
        // waiter被中断唤醒！中断状态: true
    }
}
```
**结论：wait()是中断敏感的**

---

### **3. wait()是否释放CPU？**

#### **JVM层等待实现**

```cpp
void ObjectMonitor::wait(jlong millis, bool interruptible, TRAPS) {
  // 1. 释放锁（前面已分析）
  exit(true, Self);
  
  // 2. 调用park()进入等待状态
  // ⚠️ 关键：park()会让出CPU！
  if (millis == 0) {
    // 无限等待
    Self->_ParkEvent->park();
  } else {
    // 超时等待
    Self->_ParkEvent->park(millis);
  }
  
  // 3. 被唤醒后重新竞争锁
  enter(Self);
}
```
#### **ParkEvent的park实现**

```cpp

void ParkEvent::park(jlong millis) {
  // 1. 准备等待条件
  pthread_mutex_lock(&_mutex);
  
  // 2. 如果已经被unpark，直接返回
  if (_counter > 0) {
    _counter = 0;
    pthread_mutex_unlock(&_mutex);
    return;
  }
  
  // 3. 调用pthread_cond_wait或pthread_cond_timedwait
  // ⚠️ 这些系统调用会让出CPU！
  if (millis <= 0) {
    // 无限等待
    pthread_cond_wait(&_cond, &_mutex);
  } else {
    // 超时等待
    struct timespec ts;
    // 计算超时时间
    pthread_cond_timedwait(&_cond, &_mutex, &ts);
  }
  
  // 4. 唤醒后清理
  _counter = 0;
  pthread_mutex_unlock(&_mutex);
}
```
#### **操作系统层系统调用**

```c

// pthread_cond_wait的底层实现
int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex) {
  // 1. 原子地释放mutex并进入等待
  // 2. 调用futex系统调用让线程进入睡眠
  // ⚠️ 这会立即让出CPU！
  futex_wait(&cond->__data.__futex, ...);
  
  // 3. 被唤醒后重新获取mutex
  pthread_mutex_lock(mutex);
  
  return 0;
}
```
#### **验证示例**

```java
public class WaitCPUTest {
    private static final Object lock = new Object();
    private static volatile long counter = 0;
    
    public static void main(String[] args) throws InterruptedException {
        // 创建等待线程
        Thread waiter = new Thread(() -> {
            synchronized (lock) {
                try {
                    System.out.println("waiter开始wait，释放锁和CPU");
                    lock.wait(2000);  // 等待2秒
                    System.out.println("waiter被唤醒");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        
        // 创建高CPU占用的线程
        Thread busyThread = new Thread(() -> {
            long start = System.currentTimeMillis();
            while (System.currentTimeMillis() - start < 1500) {
                counter++;  // 高CPU占用
            }
            System.out.println("busyThread执行完成，counter=" + counter);
        });
        
        waiter.start();
        Thread.sleep(100);  // 确保waiter先获取锁
        
        long startTime = System.currentTimeMillis();
        busyThread.start();
        
        waiter.join();
        busyThread.join();
        
        System.out.println("总耗时: " + (System.currentTimeMillis() - startTime) + "ms");
        // busyThread在waiter等待期间可以获得几乎100%的CPU
    }
}
```
**结论：wait()会释放CPU**

---

# 三、对比总结表

|特性|sleep()|wait()|
|---|---|---|
|**是否释放锁**|**❌ 不释放**  <br>• 静态方法，与锁无关  <br>• JVM层直接调用os::sleep，不操作monitor  <br>• 持有锁的线程sleep会阻塞其他线程|**✅ 释放锁**  <br>• 必须在同步块中调用  <br>• JVM层调用ObjectMonitor::exit()释放锁  <br>• 释放后其他线程可获得锁|
|**是否中断敏感**|**✅ 敏感**  <br>• 声明抛出InterruptedException  <br>• JVM层两次检查中断状态  <br>• 中断会立即唤醒线程并抛异常  <br>• 中断状态在异常抛出后被清除|**✅ 敏感**  <br>• 声明抛出InterruptedException  <br>• JVM层检查中断并抛出异常  <br>• 中断会唤醒等待的线程  <br>• 中断状态在wait返回后保持|
|**是否释放CPU**|**✅ 释放**  <br>• 调用操作系统sleep/yield系统调用  <br>• 线程进入睡眠状态，让出CPU  <br>• sleep(0)调用yield主动让出CPU  <br>• 时间到后进入就绪队列|**✅ 释放**  <br>• 调用ParkEvent::park()进入等待  <br>• 底层使用pthread_cond_wait/futex  <br>• 线程进入睡眠状态，让出CPU  <br>• 被notify/interrupt后进入就绪队列|
|**唤醒机制**|超时自动唤醒|需要notify()/notifyAll()或中断|
|**恢复后状态**|保持原锁定状态|需要重新竞争锁|
|**性能影响**|系统调用开销|上下文切换+锁竞争开销|

### **源码关键点记忆**：

1. **sleep不碰锁，wait必放锁**：
    
    - sleep是Thread静态方法，与对象锁无关
        
    - wait会调用ObjectMonitor::exit()释放锁
        
2. **两者皆可中断，处理有不同**：
    
    - sleep中断后状态被清除
        
    - wait中断后状态保持
        
3. **都会放CPU，系统调用来**：
    
    - sleep调用nanosleep/Sleep
        
    - wait调用pthread_cond_wait/futex
        
4. **唤醒方式异，使用场景分**：
    
    - sleep自己醒，适合定时
        
    - wait等人叫，适合协调