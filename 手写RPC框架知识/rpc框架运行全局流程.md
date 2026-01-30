# rpc框架运行流程图
![](assets/rpc框架运行全局流程/file-20260130012643959.png)
# 框架初始化的启动类
```java
/**  
 * RPC 框架应用  
 * 相当于 holder，存放了项目全局用到的变量。双检锁单例模式实现  
 */  
@Slf4j  
public class RpcApplication {  
  
    private static volatile RpcConfig rpcConfig;  
  
    /**  
     * 框架初始化，支持传入自定义配置  
     *  
     * @param newRpcConfig  
     */  
    public static void init(RpcConfig newRpcConfig) {  
        rpcConfig = newRpcConfig;  
        log.info("rpc init, config = {}", newRpcConfig.toString());  
        //注册中心初始化  
        RegistryConfig registryConfig = rpcConfig.getRegistryConfig();  
        Registry registry = RegistryFactory.getInstance(registryConfig.getRegistry());  
        registry.init(registryConfig);  
        log.info("registry init,config = {}",registryConfig);  
        //创建并注册 ShutdownHook ,JVM识别另外开一个线程退出  
        Runtime.getRuntime().addShutdownHook(new Thread(registry::destroy));  
    }  
  
    /**  
     * 初始化  
     */  
    public static void init() {  
        RpcConfig newRpcConfig;  
        try {  
            // 读取配置文件  
            newRpcConfig = ConfigUtils.loadConfig(RpcConfig.class, RpcConstant.DEFAULT_CONFIG_PREFIX);  
        } catch (Exception e) {  
            // 配置加载失败，使用默认值  
            newRpcConfig = new RpcConfig();  
        }  
        init(newRpcConfig);  
    }  
  
    /**  
     * 获取配置  
     *  
     * @return  
     */  
    public static RpcConfig getRpcConfig() {  
        if (rpcConfig == null) {  
            synchronized (RpcApplication.class) {  
                if (rpcConfig == null) {  
                    init();  
                }  
            }  
        }  
        return rpcConfig;  
    }  
}
```
# rpc框架初始化
* 加载配置类
* 