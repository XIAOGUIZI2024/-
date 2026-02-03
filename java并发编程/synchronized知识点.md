# synchronized 的三种使用方式

## **同步实例方法**

作用于当前实例，进入方法前需要获得**当前实例的锁**
```java
public class Counter {
    private int count = 0;
    
    // 同步实例方法
    public synchronized void increment() {
        count++;
    }
    
    public synchronized int getCount() {
        return count;
    }
}
```
## **同步静态方法**

作用于当前类，进入方法前需要获得**当前类的Class对象锁**
```java
public class StaticCounter {
    private static int count = 0;
    
    // 同步静态方法
    public static synchronized void increment() {
        count++;
    }
    
    public static synchronized int getCount() {
        return count;
    }
}
```

##  **同步代码块**

更细粒度的控制，可以指定锁对象
```java
public class FineGrainedLock {
    private int count1 = 0;
    private int count2 = 0;
    private final Object lock1 = new Object();
    private final Object lock2 = new Object();
    
    public void increment1() {
        // 只同步必要的代码
        synchronized(lock1) {
            count1++;
        }
    }
    
    public void increment2() {
        synchronized(lock2) {
            count2++;
        }
    }
}