### **一、线程初始化（Thread.init）**

1. **空间分配**：新线程对象由其父线程（parent线程）分配内存空间
   ```java
// Thread构造方法中的关键代码
public Thread(ThreadGroup group, Runnable target, String name,
              long stackSize) {
    init(group, target, name, stackSize);
}

private void init(ThreadGroup g, Runnable target, String name,
                  long stackSize, AccessControlContext acc,
                  boolean inheritThreadLocals) {
    // 获取当前线程（parent线程）
    Thread parent = currentThread();
    // ... 继承parent的各种属性
}
   ```
2. **属性继承**：子线程继承父线程的多个属性：
    
    - 是否为守护线程（Daemon）
        
    - 优先级（priority）
        
    - 上下文类加载器（contextClassLoader）
        
    - 可继承的ThreadLocal变量
        
3. **唯一标识**：分配一个同步（sync）的唯一ID来标识子线程
    
4. **初始化完成**：此时线程对象已在堆内存中准备就绪，等待运行


---

### **二、线程启动（Thread.start）**

1. **启动调用**：初始化完成后调用`start()`方法启动线程
    
2. **启动机制**：
    
    - 当前线程（父线程）**同步告知Java虚拟机**
        
    - 只要**线程规划器空闲**，应立即启动调用`start()`方法的线程
        
3. **含义理解**：
    
    - `start()`是向JVM发送启动请求
        
    - 实际执行时机由JVM的线程调度器决定