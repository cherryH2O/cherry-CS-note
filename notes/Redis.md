<!-- GFM-TOC -->
* [一、初识redis](#一、初识redis)
    * [redis特性](#redis特性)
* [二、数据结构](#二、数据结构)
    * [STRING](#STRING)
    * [LIST](#LIST)
    * [HASH](#HASH)
    * [SET](#SET)
    * [ZSET](#ZSET)
* [三、典型使用场景](#三、典型使用场景)
    * [Bitmaps(位图)](#Bitmaps(位图))
    * [HyperLogLog](#HyperLogLog)
    * [分布式锁](#分布式锁)
    * [延时队列](#延时队列)
    * [Bloom Filter(布隆过滤器)](#Bloom Filter(布隆过滤器))
* [四、事务](#四、事务)
<!-- GFM-TOC -->

### 一、初识redis

#### redis特性

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
    1. 纯内存访问，redis将所有数据存在内存中，内存的响应时长大约为100纳秒，这是redis达到每秒万级别访问的重要基础；
    2. 非阻塞I/O，redis使用epoll作为I/O多路复用技术的实现，再加上redis自身的事件处理模型将epoll中的连接、读写、关闭都转为事件，不在网络I/O浪费过多时间。
    3. 单线程避免了线程切换和竞态产生的消耗。

        
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

#### STRING

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
        1. **int**：8个字节的长整型
            ```html
            > set key 8653
            OK
            > object encoding key
            "int"
            ``` 
        2. **embstr**：小于等于39个字节的字符串
        3. **raw**：大于等于39个字节的字符串
    - 扩容策略：字符串在长度小于 1M 之前，扩容空间采用**加倍**策略，也就是保留 100% 的冗余空间。当长度超过 1M 之后，为了避免加倍后的冗余空间过大而导致浪费，每次扩容只
           会多分配 1M 大小的冗余空间。

    - **使用场景：**
        1. 缓存功能
        2. 计数
        3. 共享session
        4. 限速 [ SET KEY VALUE EX 60 NX ]
#### LIST

2. 列表 list：相当于 Java 语言里面的 LinkedList，注意它是**链表**而不是数组。
    - 这意味着 list 的插入和删除操作非常快，时间复杂度为 O(1)，但是索引定位很慢，时间复杂度为 O(n)，这点让人非常意外。
    - list 常用来做**异步队列**使用。将需要延后处理的任务结构体序列化成字符串塞进 Redis 的列表，另一个线程从这个列表中轮询数据进行处理
    - 早期版本存储 list 列表数据结构使用的是**压缩列表 ziplist** 和普通的**双向链表 linkedlist**，也就是元素少时用 ziplist，元素多时用 linkedlist。
    
    ```html
     // 链表的节点
     struct e listNode<T> {
     listNode* prev;
     listNode* next;
     T value;
     }
     // 链表
     struct list {
     listNode *head;
     listNode *tail;
     long length;
     }
    ```
    - 考虑到链表的附加空间相对太高，prev 和 next 指针就要占去 16 个字节 (64bit 系统的指针是 8 个字节)，另外每个节点的内存都是单独分配，会加剧内存的碎片
      化，影响内存管理效率。后续版本对列表数据结构进行了改造，使用 **quicklist** 代替了 ziplist 和 linkedlist。
    - redis 3.2 版本提供了 quicklist 内部编码， 简单说是以一个 ziplist 为节点的 linklist， 结合了两者的优势。
    - quicklist 内部默认单个 ziplist 长度为 8k 字节，超出了这个字节数，就会新起一个 ziplist
    
    ```html
    struct ziplist {
    ...
    }
    struct ziplist_compressed {
    int32 size;
    byte[] compressed_data;
    }
    struct quicklistNode {
    quicklistNode* prev;
    quicklistNode* next;
    ziplist* zl; // 指向压缩列表
    int32 size; // ziplist 的字节总数
    int16 count; // ziplist 中的元素数量
    int2 encoding; // 存储形式 2bit，原生字节数组还是 LZF 压缩存储
    ...
    }
    struct quicklist {
    quicklistNode* head;
    quicklistNode* tail;
    long count; // 元素总数
    int nodes; // ziplist 节点的个数
    int compressDepth; // LZF 算法压缩深度
    ...
    }
    ```
    - 压缩深度：quicklist默认压缩深度为0（不压缩）
    - 为了支持快速的 push/pop 操作，quicklist 的首尾两个 ziplist 不压缩，此时深度就是 1。如果深度为 2，就表示 quicklist 的首尾第一个 ziplist 以及首尾第二个 ziplist 都不压缩
    
    - Redis 5.0 又引入了一个新的数据结构 listpack，它是对 ziplist 结构的改进，在存储空间上会更加节省，而且结构上也比 ziplist 要精简。它的整体形式和
      ziplist 还是比较接近的，如果你认真阅读了 ziplist 的内部结构分析，那么 listpack 也是比较容易理解的。
      
    - **使用场景：**
        1. 消息队列
        2. 文章列表
 
#### HASH
3. 哈希 hash：相当于 Java 里的 HashMap 无序字典。
    - 为了节约内存空间使用，zset 和 hash 容器对象在元素个数较**少**的时候，采用**压缩列表 (ziplist)** 进行存储。
    - **压缩列表是一块连续的内存空间，元素之间紧挨着存储，没有任何冗余空隙。**
    ```html
    struct ziplist<T> {
    int32 zlbytes; // 整个压缩列表占用字节数
    int32 zltail_offset; // 最后一个元素距离压缩列表起始位置的偏移量，用于快速定位到最后一个节点
    int16 zllength; // 元素个数
    T[] entries; // 元素内容列表，挨个挨个紧凑存储
    int8 zlend; // 标志压缩列表的结束，值恒为 0xFF
    }
    ```
    - 压缩列表为了支持**双向遍历**，所以才会有 **ztail_offset** 这个字段，用来快速定位到最后一个元素，然后倒着遍历。
    - **entry** 块随着容纳的元素类型不同，也会有不一样的结构。
    ```html
    struct entry {
    t int<var> prevlen; // 前一个 entry 的字节长度
    t int<var> encoding; // 元素类型编码
    optional byte[] content; // 元素内容
    }
    ```
    - 它的 prevlen 字段表示前一个 entry 的字节长度，当压缩列表倒着遍历时，需要通过这个字段来快速定位到下一个元素的位置。它是一个变长的整数，当字符串长
      度小于 254(0xFE) 时，使用一个字节表示；如果达到或超出 254(0xFE) 那就使用 5 个字节来表示。第一个字节是 0xFE(254)，剩余四个字节表示字符串长度。你可
      能会觉得用 5 个字节来表示字符串长度，是不是太浪费了。我们可以算一下，当字符串长度比较长的时候，其实 5 个字节也只占用了不到 (5/(254+5))<2% 的空间。
      
    - 当哈希类型无法满足ziplist条件时，redis会使用**hashtable**作为哈希的内部实现，因为此时ziplist的读写效率会下降，而**hashtable的读写时间复杂度为O(1)**.
    - dict 结构内部包含两个hashtable，通常情况下只有一个hashtable是有值的。但是在dict扩容缩容时，需要分配新的hashtable，然后进行渐进式搬迁，这时候两个hashtable存储的分别时旧的hashtable和新的hashtable。
    - 待搬迁结束后，旧的hashtable被删除，新的hashtable取而代之。
    
    ```html
    struct dictEntry {
    d void* key;
    d void* val;
    dictEntry* next; // 链接下一个 entry
    }
    struct dictht {
    dictEntry** table; // 二维
    long size; // 第一维数组的长度
    long used; // hash 表中的元素个数
    ...
    }
    ```
    - 字典数据结构的精华就落在了hashtable结构上了。hashtable的结构和Java 的 HashMap几乎时一样的，都是通过分桶的方式解决hash冲突。第一维时数组，第二维时链表。数组中存储的是第二维链表的第一个元素的指针。
    - **渐进式rehash**:
        1. 大字典的扩容是比较耗时的，需要重新申请新的数组，然后将旧字典所有链表中的元素重新挂接到新的数组下面，这是一个O(n)级别的操作。
        2. 作为单线程的redis表示很难承受这样耗时的过程。redis 使用渐进式rehash小步搬迁，虽然慢一点，但是肯定可以搬完的。
        3. redis 会在定时任务中对字典进行主动搬迁。
        
#### SET

4. set 相当于 Java语言里的HashSet，它内部的键值对是无序的、唯一的。
    - 内部实现相当于一个特殊的字典，字典中所有的value都是一个值NULL.
    - 当集合中最后一个元素移除之后，数据机构自动删除，内存被回收。
    - set 结构可以用来存储活动中奖用户id，因为有**去重**功能，可以保证同一个用户不会中奖两次。
    - 内部编码：
        1. intset：当集合中元素很少时，通过intset来作为集合的内部实现，从而减少内存的使用。
        2. hashtable：当集合类型无法满足intset条件时，redis会使用hashtable作为集合的内部实现。
    - **使用场景：**
        1. 标签
        2. 生成随机数，比如抽奖
        3. 社交需求
        
#### ZSET

5. zset 有序列表：类似于Java的sortedSet和HashMap的结合体，一方面它是一个set，保证了内部value的唯一性，另一方面可以给每个value赋予一个score，代表这个value的排序权重。
    - 内部编码：
        1. ziplist：当有序集合的元素个数小于设定的zset-max-ziplist-entries（默认128个），ziplist可以有效减少内存使用。
        2. skiplist：当数据比较大时，有序集合会使用 **skiplist** 作为内部实现，因为此时ziplist读写效率会下降。
    - 它的内部实现用的是一种叫做 [skiplist 跳跃列表] 的数据结构。
    - skiplist之所以跳跃，是因为内部的元素可能身兼数职。
    - skiplist是有序列表的底层实现之一。
    - **使用场景：**
        1. 排行榜系统（按照时间、按照播放量、按照获得的赞数）
  
    
### 三、典型使用场景
#### Bitmaps(位图)
1. 假如有一些 boolen 型数据需要存取。比如用户一年的签到记录，1表示签到，0表示没签到，要记录365天。如果使用普通的key/value，每个用户要记录365个，当用户上亿时，存储空间是非常惊人的。
2. 为了解决这个问题，使用Bitmaps可以节省很多空间，尤其是随着时间的推移节省的内存还是非常可观的。
3. getbit/setbit/
    - bitcount：统计用户一共签到了多少天
    - bitpos：查找用户从哪一天开始第一次签到
#### HyperLogLog

1. 只是为了计算独立总数，不需要获得单条数据；
2. 可以容忍一定的**误差率**，毕竟HyperLogLog在内存上的**占用量**有很大的优势。
3. pfadd/pfcount
#### 分布式锁：

1. 在分布式场景下，无法使用**单机环境**下的锁来对多个节点上的进程进行同步。
2. 本质上，就是要实现在redis里面占个"坑位"，当别的进程也要来占时，发现已经有人在了，就只有放弃或者稍后再试。
    ```html
    setnx（set if not exists）
    expire
    ```
3. 如果在setnx和expire之间服务器进程突然挂掉了，可能是因为机器掉电或者认为kill，就会导致expire得不到执行，也会造成**死锁**。
4. 这种问题的根源就在于 setnx 和 expire 是两条指令而不是原子指令。如果这两条指令可以一起执行就不会出现问题。也许你会想到用 Redis 事务来解决。但是这里
   不行，因为 expire 是依赖于 setnx 的执行结果的，如果 setnx 没抢到锁，expire 是不应该执行的。事务里没有 if-else 分支逻辑，事务的特点是一口气执行，
   要么全部执行要么一个都不执行。
    ```html
    set key true ex 5 nx
    ```
5. ***上面这个指令就是 setnx 和 expire 组合在一起的原子指令，它就是分布式锁的奥义所在.
6. 不足之处：不能解决超时问题，如果在加锁和释放锁之间的逻辑执行的太长，以至于超出了锁的超时限制，就会出现问题。因为这时候锁过期了，第二个线程重
        新持有了这把锁，但是紧接着第一个线程执行完了业务逻辑，就把锁给释放了，第三个线程就会在第二个线程逻辑执行完之间拿到了锁。
    - 相对安全的方案：为 set 指令的 value 参数设置为一个随机数，释放锁时先匹配随机数是否一致，然后再删除 key。
    - 但是匹配 value 和删除 key 不是一个原子操作，Redis 也没有提供类似于 delifequals 这样的指令，这就需要使用 Lua 脚本来处理了，因为 Lua 脚本可以保证连续多个指令的原子性执行。
 
#### 延时队列 
 
1. list 常用来作为异步消息队列使用，使用 rpush/lpush 操作入队列，使用 lpop 和 rpop 出队列 。
2. **blpop/brpop** 这两个指令的前缀字符 b 代表的是blocking，也就是阻塞读。
3. 阻塞度在队列没有数据的时候，会立即进入休眠状态，一旦数据到来，则立刻醒过来。消息的延迟几乎为0.
4. 客户端在处理请求时加锁没加成功，一般有3种策略来处理加锁失败：
    1. 直接抛出异常，通知用户稍后重试；
    2. sleep 一会再重试；
    3. 将请求转移至延时队列，过一会再试。
  
#### Bloom Filter(布隆过滤器)
  
1. 对推荐系统用户历史记录去重的有效法宝，在空间上节省90%以上，稍微有点不精确，有一定的误判。
2. 可以理解为一个不怎么精确的 set 结构，当你使用它的 contains 方法判断某个对象是否存在时，它可能会误判。但是布隆过滤器也不是特别不精确，只要参数设置的合理，它的精确度可以控制的相对足够精确，只会有小小的误判概率。
3. redis 4.0 提供了插件功能之后才正式登场。
    ```html
    > docker pull redislabs/rebloom # 拉取镜像
    > docker run -p6379:6379 redislabs/rebloom # 运行容器
    > redis-cli # 连接容器中的 redis 服务
    ```
4. **bf.add/bf.exists/bf.madd/bf.mexists**
### 四、事务
    
   - 一个事务包含了多个命令，服务器在执行事务期间，不会改去执行其它客户端的命令请求。  
   - 事务中的多个命令被一次性发送给服务器，而不是一条一条发送，这种方式被称为流水线，它可以减少客户端与服务器之间的网络通信次数从而提升性能。  
   - Redis 最简单的事务实现方式是使用 **MULTI** 和 **EXEC** 命令将事务操作包围起来。
