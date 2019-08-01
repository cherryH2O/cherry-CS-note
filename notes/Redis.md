### 一、初识redis

- redis特性

    - 速度快，读写性能可以达到10万/s，快的原因有
        1. redis所有数据都存放在内存中。
        2. redis是**C语言**实现的，一般来说，C语言实现的程序“距离”操作系统更近，执行速度更快。
        3. redis使用了**单线程**架构，预防了多线程可能产生的竞争问题。
        4. 集性能和优雅于一身的开源代码。
        
    - 基于键值对的数据结构服务器 
        1. 字符串 string
        2. 哈希 hash
        3. 列表 list
        4. 集合 set
        5. 有序集合 sort set
        
    - 持久化：RDB、AOF，从内存到磁盘
    
    - 主从复制
    
    - 高可用和分布式


    单线程为什么还能这么快？
    第一：纯内存访问，redis将所有数据存在内存中，内存的响应时长大约为100纳秒，这是redis达到每秒万级别访问的重要基础；
    第二：非阻塞I/O，redis使用epoll作为I/O多路复用技术的实现，再加上redis自身的事件处理模型将epoll中的连接、读写、关闭都转为事件，不在网络I/O浪费过多时间。
    第三：单线程避免了线程切换和竞态产生的消耗。

        
### 二、数据结构

- 全局命令：

    1. 查看所有键 [ 慎用-时间复杂度O(n) ] ：会将全局所有key全部输出。
    ```html
    keys *
    ```
    2. 键key总数，时间复杂度是O(1)，不会遍历所有key，直接获取redis内置的key总数变量。
    ```html
    dbsize  
    ```
    3. 检查key是否存在：存在返回1，不存在返回0。
    ```html
    exists key
    ```
    4. 删除键：无论什么数据结构类型，都可以。返回成功删除key的个数。
    ```html
    del key[key...]
    ```
    5. 键过期时间设置：
    ```html
    expire key seconds  
    ttl key
    ```    
    ttl 命令会返回键的剩余过期时间，有3种返回值：
    - 大于等于0的整数——键剩余的过期时间。
    - -1——键没设置过期时间。
    - -2——键不存在。
    6. 键的数据结构类型
    ```html
    type key
    ```  

- 数据结构和内部编码：

> [What Redis dadta structures look like](https://redislabs.com/ebook/part-1-getting-started/chapter-1-getting-to-know-redis/1-2-what-redis-data-structures-look-like/)

1. 字符串 string: Redis 的字符串叫着「SDS」，也就是 Simple Dynamic String 。它的结构是一个带长度信息的字节数组，最大不能超过512M。
    ```html
    struct SDS<T> {
    T capacity; // 数组容量（分配）
    T len; // 数组长度（实际）
    byte flags; // 特殊标识位，不理睬它
    e byte[] content; // 数组内容
    }
    ```
    - 使用泛型T，不直接使用int，是为了对内存做极致的优化，适当的时候也可用byte、short.
    - Redis 的字符串有三种存储方式，在长度特别短时，使用 emb 形式存储 (embeded)，当长度超过 44 时，使用 raw 形式存储。
        1. int：8个字节的长整型
            ```html
            > set key 8653
            OK
            > object encoding key
            "int"
            ``` 
        2. embstr：小于等于39个字节的字符串
        3. raw：大于等于39个字节的字符串
    - 扩容策略：字符串在长度小于 1M 之前，扩容空间采用**加倍**策略，也就是保留 100% 的冗余空间。当长度超过 1M 之后，为了避免加倍后的冗余空间过大而导致浪费，每次扩容只
           会多分配 1M 大小的冗余空间。