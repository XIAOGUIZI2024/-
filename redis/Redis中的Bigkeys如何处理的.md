
1. **BigKey 定义**
    
    并非 key 名称过长，而是**String 类型 value 体积过大**，或**Hash/List/Set/ZSet 等集合元素数量过多**；结合 Redis 单线程模型，极易引发性能问题。
    
2. **BigKey 核心危害**
    

- 阻塞 Redis 主线程，引发业务接口超时、服务雪崩；
- 占用大量内网带宽，影响整体服务响应；
- Redis 集群出现内存倾斜，节点负载不均衡；
- 加重主从同步、RDB/AOF 持久化延迟；
- 扰乱内存淘汰策略，容易触发缓存雪崩。

3. **BigKey 检测方式**

- 快速扫描：使用 `redis-cli --bigkeys` 一键排查全局大 key；
- 精准查询：通过 `MEMORY USAGE key` 查看单个 key 内存占用；
- 全面排查：利用 `SCAN` 游标遍历所有 key，结合 HLEN、LLEN 等命令统计集合元素；
- 线上常态化：依靠监控平台配置内存、key 大小告警，定时巡检。

4. **优雅删除 BigKey 方案**

- Redis4.0 及以上：优先使用 `UNLINK` 异步删除，替代同步阻塞的 `DEL`；
- 集合类型大 key：通过 `HSCAN/SSCAN/ZSCAN` 游标**分批少量删除**，规避一次性操作阻塞；
- 大 List：使用 `LTRIM` 截断保留少量数据，渐进清理；
- 低版本 Redis：无异步删除能力，必须分批 + 间隔删除，禁止直接删除；
- 通用规范：低峰期操作，杜绝全量查询命令（HGETALL、SMEMBERS 等），从业务层面拆分大 key，提前规避。