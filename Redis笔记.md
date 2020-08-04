# Redis笔记

> Redis 是一个开源的 使用C 语言编写、支持网络、可基于内存亦可持久化的日志型、Key-Value 数据库。
>
> 内存中的数据结构存储系统，它可以用作数据库、缓存和消息中间件。 它支持多种类型的数据结构，如
> ①字符串（strings）
> ②散列（hashes）
> ③列表（lists）
> ④ 集合（sets）
> ⑤有序集合（sorted sets）
> 与范围查询， bitmaps， hyperloglogs 和 地理空间（geospatial） 索引半径查询。 Redis 内置了 复制（replication），LUA脚本（Lua scripting）， LRU驱动事件（LRU eviction），事务（transactions） 和不同级别的 磁盘持久化（persistence）， 并通过 Redis哨兵（Sentinel）和自动 分区（Cluster）提供高可用性（high availability）。

***

## redis的key

> Redis 的 key 是字符串类型，但是 key 中不能包括边界字符 ，由于 key 不是 binary safe 的字符串，所以像"my key"和"mykey\n"这样包含空格和换行的 key 是不允许的。
>
> key最大为512MB。

| 命令                     | 说明                                                         |
| ------------------------ | ------------------------------------------------------------ |
| **exists** key           | 检测指定key是否存在，返回1表示存在，0不存在。                |
| **del** key1 key2 … keyN | 删除给定key,返回删除key的数目，0表示给定key都不存在<br />注意：这里的keys要在**同一个slot**中，否则报错<br />(error) CROSSSLOT Keys in request don't hash to the same slot |
| **type** key             | 返回给定 key 值的类型。返回 none 表示 key 不存在,string 字符类型，list 链表 类型 set 无序集合类型… |
| **keys** pattern         | 返回匹配指定模式的所有 key。如：keys * 返回所有key<br />注意：这里的keys，只能返回当前slot中的key集合【集群当前节点的所有key】 |
| randomkey                | 返回从当前数据库中随机选择的一个 key,如果当前数据库是空的，返回空串(nil) |
| rename oldkey newkey     | 重命名一个 key,如果 newkey 存在，将会被覆盖，返回 1 表示成功， 0 失败。可能是 oldkey 不存在或者和 newkey 相同。 |
| renamenx oldkey newkey   | 同上。newkey存在，报错。                                     |
| **expire** key seconds   | 为 key 指定过期时间，单位是秒。返回 1 成功，0 表示 key 已经设置过过 期时间或者不存在。 |
| **ttl** key              | 返回设置过过期时间key的剩余过期秒数。-1表示key不存在或者未设置过期时间。 |
| select db-index          | 通过索引选择数据库，默认连接的数据库是 0,默认数据库数是 16 个。返回 1 表示成功，0 失败。 |
| move key db-index        | 将key从当前数据库移动到指定数据库。返回 1表示成功。0表示key 不存在或者已经在指定数据库中 。 |
| dbsize                   | 查看当前数据库的key的数量                                    |
| flushdb                  | 清空当前库(**慎用！**）当前集群节点                          |
| flushall                 | 清空全部库（**删库跑路！！！**）当前集群节点                 |

***

## 五大数据类型

### ==**string**==

> Redis 的字符串是动态字符串，是可以修改的字符串，内部结构实现上类似于 Java 的 ArrayList，采用预分配冗余空间的方式来减少内存的频繁分配，，内部为当前字符串实际分配的空间 capacity 一般要高于实际字符串长度 len。当字符串长度小于 1M 时，扩容都是加倍现有的空间，如果超过 1M，扩容时一次只会多扩 1M 的空间。需要注意的是**字符串最大长度为 512M**。
>
> 而且 string 类型是**二进制安全的**。意思是 redis 的 string 可以包含任何数据。比如 jpg 图片或者序列化的对象。
>
> [什么是二进制安全](https://www.zhihu.com/question/28705562)

#### string-底层实现

> Redis 的字符串叫着「SDS」，也就是Simple Dynamic String。它的结构是一个带长度信息的字节数组。

![在这里插入图片描述](https://gitee.com/wangigor/typora-images/raw/master/redis-sds.png)

```c
//SDS的结构：
struct SDS {
    T capacity; // 数组容量
    T len; // 数组长度
    byte flags; // 特殊标识位，不理睬它
    byte[] content; // 数组内容
}
```

> SDS 结构使用了范型 T，为什么不直接用 int 呢，这是因为当字符串比较短时，len 和 capacity 可以使用 byte 和 short 来表示，Redis 为了**对内存做极致的优化**，**不同长度的字符串使用不同的结构体来表示**。
>
> **创建字符串时 len 和 capacity 一样长**，不会多分配冗余空间，这是因为绝大多数场景下我们不会使用 append 操作来修改字符串。

#### string-使用

| 命令                                         | 说明                                                 |
| -------------------------------------------- | ---------------------------------------------------- |
| **set** key value                            | 设置key的值                                          |
| **get** key                                  | 获取对应的key的值                                    |
| strlen key                                   | 获取key的值的长度                                    |
| append key                                   | 在原有的value的基础上追加内容                        |
| **incr** key                                 | 将key存储的内容加1                                   |
| **incrby** key 步长                          | 将key存储的内容加指定的值                            |
| incrbyfloat key float                        | 将key存储的内容累加一个float类型的数据               |
| **decr** key                                 | 将key存储的内容减1                                   |
| **decrby** key 步长                          | 将key存储的内容减去指定的值                          |
| getrange key start end                       | 截取value的值【相当于substring】                     |
| setrange key offset value                    | 修改value的部分内容，根据偏移量offset修改            |
| **getset** key newValue                      | 获取设置key的值并返回原来的旧值                      |
| mget key1 key2 ... keyN                      | 批量获取值                                           |
| mset key1 value1 key2 value2 ... keyN valueN | 批量设置值                                           |
| setex key expireTime value                   | 设置key对应的value，同时设置过期时间，单位是**秒**   |
| psetex key expireTime value                  | 设置key对应的value，同时设置过期时间，单位是**毫秒** |
| **==setnx==**                                | 只有在 key 不存在时设置 key 的值,set if not exists   |
| msetnx                                       | 兼具了mset和setnx的特性                              |

#### string-bitmap

> [ascii码对照表](http://ascii.911cha.com/)
>
> <官网初始分配>：在一台2010MacBook Pro上，offset为2^32-1（分配512MB）需要～300ms，offset为2^30-1(分配128MB)需要～80ms，offset为2^28-1（分配32MB）需要～30ms，offset为2^26-1（分配8MB）需要8ms。

| 命令                               | 说明                                                         |
| ---------------------------------- | ------------------------------------------------------------ |
| **getbit** key offset              | 获取二级制中对应偏移量的值                                   |
| **setbit** key offset 0/1          | 设置对应二进制位的值                                         |
| **bitcount** key                   | 统计二进制中位中为1的个数                                    |
| bitop 运算符 key argKey1 [argKey2] | 对二进制数据做位元操作，AND(与) 、 OR(或) 、 NOT(非) 、 XOR(异或) |
| bitpos key 0/1                     | 返回字符串里面第一个被设置为1或者0的bit位                    |

> 举例：
>
> a的ascii码为97，对应的二进制是 0110 0001。
> c的ascii码为99，对应的二进制是 0110 0011。

![image-20200716161619194](https://gitee.com/wangigor/typora-images/raw/master/redis-getbit-a.png)

注意：当偏移量比字符串长度大 或 key 不存在，都返回0.

反向操作也是可以的。

> 设置一个新key，或者使用不存在的key，按位操作，也可以得到正向字符串。

![image-20200716162058879](https://gitee.com/wangigor/typora-images/raw/master/redis-setbit-make-a.png)

##### bitmap实战-用户签到

todo

##### bitmap实战-统计活跃用户

todo

##### bitmap实战-用户在线状态

todo



### **==list==**

> list是一个链表结构，可以理解为每个子元素都是string类型的双向链表。
>
> 它的底层实际是个双向链表，对两端的操作性能很高，通过索引下标的操作中间的节点性能会较差。操作的时间复杂度为 O(1)，但是索引定位很慢，时间复杂度为 O(n)。

#### list-底层原理

> 具体一点，其实底层用的是**ZipList（压缩链表）**和**QuickList（快速链表）**。
>
> 首先在列表元素较少的情况下会使用一块连续的内存存储，这个结构是 ziplist，也即是压缩列表。它将所有的元素紧挨着一起存储，分配的是一块连续的内存。（感觉好像数组啊，连续的内存空间）
>
> 当数据量比较多的时候才会改成 quickList。Redis 将链表和 ziplist 结合起来组成了 quickList。

##### 压缩链表ZipList

```c
//压缩链表
struct ziplist {
    int32 zlbytes; // 整个压缩列表占用字节数
    int32 zltail_offset; // 最后一个元素距离压缩列表起始位置的偏移量，用于快速定位到最后一个节点
    int16 zllength; // 元素个数
    T[] entries; // 元素内容列表，挨个挨个紧凑存储
    int8 zlend; // 标志压缩列表的结束，值恒为 0xFF
}
//元素内容Entry
struct entry {
    int prevlen; // 前一个 entry 的字节长度
    int encoding; // 元素类型编码
    optional byte[] content; // 元素内容
}
```

![在这里插入图片描述](https://gitee.com/wangigor/typora-images/raw/master/redis-ziplist.png)

> 增加元素：
>
> 因为 ziplist 都是紧凑存储，没有冗余空间 (对比一下 Redis 的字符串结构)。意味着插入一个新的元素就需要调用 realloc 扩展内存。取决于内存分配器算法和当前的 ziplist 内存大小，realloc 可能会重新分配新的内存空间，并将之前的内容一次性拷贝到新的地址，也可能在原有的地址上进行扩展，这时就不需要进行旧内容的内存拷贝。
>
> 如果 ziplist 占据内存太大，重新分配内存和拷贝内存就会有很大的消耗。所以 ziplist 不适合存储大型字符串，存储的元素也不宜过多。



##### 快速链表QuickList

> 基础双向链表。

```c
// 链表的节点
struct listNode {
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

> 链表的附加空间相对太高，prev 和 next 指针就要占去 16 个字节 (64bit 系统的指针是 8 个字节)，另外每个节点的内存都是单独分配，会加剧内存的碎片化，影响内存管理效率。
>
> 为了解决上述问题，就引出了quicklist：
> quicklist 是 ziplist 和 linkedlist 的混合体，它将 linkedlist 按段切分，每一段使用 ziplist 来紧凑存储，多个 ziplist 之间使用双向指针串接起来。

![在这里插入图片描述](https://gitee.com/wangigor/typora-images/raw/master/redis-quicklist.png)

> zipList因为是连续的内存空间，所以不需要prev 和 next 指针，空间占用大幅减少。同样，比起原来每个节点的一盘散沙，现在zipList是一坨一坨的沙砖，碎片化的问题也解决了。

```c
//快速链表数据结构
struct quicklistNode {
    quicklistNode* prev;
    quicklistNode* next;
    ziplist* zl; // 指向压缩列表
    int32 size; // ziplist 的字节总数
    int16 count; // ziplist 中的元素数量
    int2 encoding; // 存储形式 2bit，原生字节数组还是 LZF 压缩存储
    …
}
struct quicklist {
    quicklistNode* head;
    quicklistNode* tail;
    long count; // 元素总数
    int nodes; // ziplist 节点的个数
    int compressDepth; // LZF 算法压缩深度
    …
}
```

> quicklist 内部默认单个 ziplist 长度为 8k 字节，超出了这个字节数，就会新起一个 ziplist。ziplist 的长度由配置参数list-max-ziplist-size决定，是可以配置的。
> 为了更进一步的压缩，在原来zipList的基础上，还有压缩的zipList使用 LZF 算法压缩，可以选择压缩深度
>
> ```c
> struct ziplist_compressed {
>     int32 size;
>     byte[] compressed_data;
> }
> ```
> quicklist 默认的压缩深度是 0，也就是不压缩。压缩的实际深度由配置参数list-compress-depth决定。为了支持快速的 push/pop 操作，quicklist 的首尾两个 ziplist 不压缩，此时深度就是 1。如果深度为 2，就表示 quicklist 的首尾第一个 ziplist 以及首尾第二个 ziplist 都不压缩。

![在这里插入图片描述](https://gitee.com/wangigor/typora-images/raw/master/redis-quicklist-compressed.png)

#### list-使用

| 命令                                             | 说明                                                         |
| ------------------------------------------------ | ------------------------------------------------------------ |
| **lpush==/==rpush** key value1 value2 ... valueN | 向列表头部【左】/尾部【右】插入一个或多个元素。<br />如果 key 不存在，那么在进行 push 操作前会创建一个空列表。 <br />如果 key 对应的值不是一个 list 的话，那么会返回一个错误。 |
| **lpop==/==rpop** key                            | 移除列表头部【左】/尾部【右】的一个元素，并返回。            |
| **lrange** key start end                         | 按照索引下标获得元素集合(从左到右)<br />start 和 end 偏移量都是基于0的下标，即list的**第一个元素下标是0（list的表头）**，第二个元素下标是1，以此类推。<br/>**偏移量也可以是负数**，表示偏移量是从list尾部开始计数。 例如，**-1 表示列表的最后一个元素**，-2 是倒数第二个，以此类推。 |
| **lindex** key index                             | 获取列表中对应下标的值。没有key或者数组下标越界返回空。      |
| **llen** key                                     | 获取列表长度                                                 |
| **lset** key index value                         | 通过index设置列表的值                                        |
| **ltrim** key start end                          | 截取列表对应的元素 相当于 list=list.subList(start,end)       |
| **blpop==/==brpop** key timeout                  | lpop和rpop的阻塞版。<br />blpop/brpop是阻塞式列表的弹出原语。 <br />它是命令 lpop/rpop 的阻塞版本，这是因为当给定列表内没有任何元素可供弹出的时候， 连接将被 blpop/brpop 命令阻塞。 <br />当给定多个 key 参数时，按参数 key 的先后顺序依次检查各个列表，弹出第一个非空列表的头元素。<br />同时在使用此命令的时候也需要指定过期时间，**单位是秒**。返回的接口是key和列表元素值<br/> |
| rpoplpush source destination                     | 移除一个列表的最后一个元素，并将该元素添加到另一个列表的头部<br />原子性地返回并移除存储在 source 的列表的最后一个元素（列表尾部元素）， 并把该元素放入存储在 destination 的列表的第一个元素位置（列表头部） |
| brpoplpush source destination timeout            | rpoplpush的阻塞版本                                          |



### **==hash==**

> Redis hash 是一个键值对集合。
> Redis hash是一个string类型的field和value的映射表，hash特别适合用于存储对象。
> 在实际开发过程中我们肯定会碰到很多需要存储对象的需求，此时hash就比较合适了。hash 是一个string类型的field和value的映射表，hash特别适合用于存储对象。
> Redis 中每个 hash 可以存储 2^32 - 1 键值对（40多亿）。

#### hash-底层原理

> 跟HashMap一样。

```c
//node
struct dictEntry {
    void key;
    void val;
    dictEntry* next; // 链接下一个 entry
}
//table
struct dictht {
    dictEntry** table; // 二维
    long size; // 第一维数组的长度
    long used; // hash 表中的元素个数
    …
}
```

> 大字典的扩容是比较耗时间的，需要重新申请新的数组，然后将旧字典所有链表中的元素重新挂接到新的数组下面，这是一个O(n)级别的操作，作为单线程的Redis表示很难承受这样耗时的过程。步子迈大了会扯着蛋，所以Redis使用渐进式rehash小步搬迁。虽然慢一点，但是肯定可以搬完。

![在这里插入图片描述](https://gitee.com/wangigor/typora-images/raw/master/redis-hash-rehash.png)

> 搬迁操作埋伏在当前字典的后续指令中(来自客户端的hset/hdel指令等),实际上**在redis中每一个增删改查命令中都会判断数据库字典中的哈希表是否正在进行渐进式rehash，如果是则帮助执行一次**。但是有可能客户端闲下来了，没有了后续指令来触发这个搬迁，那么Redis就置之不理了么？当然不会，优雅的Redis怎么可能设计的这样潦草。**Redis还会在定时任务中对字典进行主动搬迁**。
>
> - redis使用的Hash函数：
>   hashtable 的性能好不好完全取决于 hash 函数的质量。hash 函数如果可以将 key 打散的比较均匀，那么这个 hash 函数就是个好函数。**Redis 的字典默认的 hash 函数是 siphash**。siphash 算法即使在输入 key 很小的情况下，也可以产生随机性特别好的输出，而且它的性能也非常突出。对于 Redis 这样的单线程来说，字典数据结构如此普遍，字典操作也会非常频繁，hash 函数自然也是越快越好。
>
> - 扩容条件：
>   正常情况下，当 hash 表中元素的个数等于第一维数组的长度时，就会开始扩容，**扩容的新数组是原数组大小的 2 倍**。不过如果 Redis 正在做 bgsave（持久化），为了减少内存页的过多分离 (Copy On Write)，Redis 尽量不去扩容 (dict_can_resize)，但是如果 hash 表已经非常满了，**元素的个数已经达到了第一维数组长度的 5 倍 (dict_force_resize_ratio)，说明 hash 表已经过于拥挤了，这个时候就会强制扩容**。
>
> - 缩容条件：
>   当 hash 表因为元素的逐渐删除变得越来越稀疏时，Redis 会对 hash 表进行缩容来减少 hash 表的第一维数组空间占用。缩容的条件是元素个数低于数组长度的 10%。缩容不会考虑 Redis 是否正在做 bgsave。

#### hash-使用



| 命令                                         | 说明                                    |
| -------------------------------------------- | --------------------------------------- |
| **hset** key field value                     | 设置key中字段field的值                  |
| **hget** key field                           | 获取key中字段field的值                  |
| **hmset** key field1 value1 field2 value ... | 批量设置key中的字段                     |
| **hmget** key field1 field2 ...              | 批量获取key中字段的值                   |
| **hdel** key field1 field2 ...               | 删除key中指定的字段                     |
| **hsetnx** key field value                   | 设置key中的字段的值，如果字段存在就失败 |
| **hvals** key                                | 获取key中所有的字段的值                 |
| **hkeys** key                                | 获取key中的所有的字段                   |
| **hgetall** key                              | 获取key中的所有的字段及值               |
| **hexists** key field                        | 判断key中的字段是否存在                 |
| **hincrby** key field value                  | 将key中的字段增加特定的值               |
| hincrbyfloat key field floatValue            | 和hincrby类似增加的float类型的数据      |
| hlen key                                     | 获取key中的字段的个数                   |
| hstrlen key field                            | 获取key中某个字段的值得长度             |



### **==set==**

> Redis set对外提供的功能与list类似是一个列表的功能，特殊之处在于set是可以自动排重的，当你需要存储一个列表数据，又不希望出现重复数据时，set是一个很好的选择，并且set提供了判断某个成员是否在一个set集合内的重要接口，这个也是list所不能提供的。
>
> Redis的Set是string类型的无序集合。它底层其实是一个value为null的hash表,所以添加，删除，查找的复杂度都是O(1)。

#### set-底层原理

Set的实现参考hash的底层实现，Set就是Value都为空的Hash。

#### set-使用

| 命令                                                 | 使用                                                         |
| ---------------------------------------------------- | ------------------------------------------------------------ |
| **sadd** key value1 value2 ...                       | 添加一个或多个元素到集合中，如果集合中存在该元素则忽略，返回成功数量 |
| **scard** key                                        | 返回集合中的元素的个数                                       |
| **sismember** key value                              | 判断集合中是否含有某元素                                     |
| **smembers** key                                     | 获取集合中的所有的元素                                       |
| **srem** key value                                   | 删除集合中指定的元素                                         |
| **srandmember** key count                            | 随机返回集合中的元素<br />redis2.6之后可以在命令后加一个count参数，指定随机返回的元素的个数<br />如果count大于集合中的个数，则返回所有的元素。负数的话取绝对值 |
| **spop** key count                                   | 和srandmember类似，**只是spop会将获取的元素移除而srandmember不会移除元素** |
| **smove** sourceKey destinationKey value             | 将元素从一个集合移动到另一个集合中                           |
| **sdiff** key1 key2                                  | 返回两个集合的差集                                           |
| **sdiffstore** destinationKey sourceKey1 sourceKey2  | 和sdiff类似，不同的是会将差集结果保存起来                    |
| **sinter** key1 key2                                 | 获取两个集合的交集                                           |
| **sinterstore** destinationKey sourceKey1 sourceKey2 | 和sinter类似，不同的是将结果保存起来了                       |
| **sunion** key1 key2                                 | 获取两个集合的并集                                           |
| **sunionstore** destinationKey sourceKey1 sourceKey2 | 获取两个集合的并集并保存起来                                 |



### **==zset==**

> ### sorted set。
>
> Redis有序集合zset与普通集合set非常相似，是一个没有重复元素的字符串集合。不同之处是有序集合的每个成员都关联了一个评分（score） ，这个评分（score）被用来按照从最低分到最高分的方式排序集合中的成员。集合的成员是唯一的，但是评分可以是重复了 。
>
> 因为元素是有序的, 所以你也可以很快的根据评分（score）或者次序（position）来获取一个范围的元素。访问有序集合的中间元素也是非常快的,因此你能够使用有序集合作为一个没有重复成员的智能列表。

#### zset-底层原理

> **跳跃链表skipList**。
>
> 插入过程:
>
> ![skiplist插入形成过程](https://gitee.com/wangigor/typora-images/raw/master/skiplist_insertions.png)
>
> ```c
> #define ZSKIPLIST_MAXLEVEL 32
> #define ZSKIPLIST_P 0.25
> 
> typedef struct zskiplistNode {
>     robj *obj;
>     double score;
>     struct zskiplistNode *backward;
>     struct zskiplistLevel {
>         struct zskiplistNode *forward;
>         unsigned int span;
>     } level[];
> } zskiplistNode;
> 
> typedef struct zskiplist {
>     struct zskiplistNode *header, *tail;
>     unsigned long length;
>     int level;
> } zskiplist;
> ```
>
> 以学生成绩为例
>
> ![Redis skiplist结构举例](https://gitee.com/wangigor/typora-images/raw/master/redis_skiplist_example.png)
>
> 跳跃列表采取一个**随机策略**来决定新元素可以兼职到第几层。
> 首先 L0 层肯定是 100% 了，L1 层只有 50% 的概率，L2 层只有 25% 的概率，L3 层只有 12.5% 的概率，一直随机到最顶层 L31 层2*-31。绝大多数元素都过不了几层，只有极少数元素可以深入到顶层。列表中的元素越多，能够深入的层次就越深，能进入到顶层的概率就会越大。

#### zset-使用

| 命令                                                         | 使用                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| **zadd** key score1 value1 store2 value2 ...                 | 向有序集合中添加一个或多个 分数/元素对                       |
| **zscore** key value                                         | 获取有序集合中元素对应的分数                                 |
| **zrange** key start end [withscores]                        | **【下标】**获取集合中的元素，如果加上withscores则会连同分数一并返回**【从小到大】** |
| **zrevrange** key start end [withscores]                     | **【下标】**和zrange类似，只是将结果倒序了**【从大到小】**   |
| **zcard** key                                                | 返回集合中元素的个数                                         |
| **zcount** key [(]min [(]max                                 | 统计集合中分数在min和max之间的元素个数，加上"("标识开区间，默认闭区间 |
| **zrangebyscore** key start end [withscores]                 | **【score】**可以根据score范围查找元素                       |
| **zrank** key value                                          | 获取元素在集合中的排名，**从小到大**，最小的是0              |
| **zrevrank** key value                                       | 获取元素在集合中的排序，**从大到小**                         |
| **zincrby** key score value                                  | 给元素增加分数，如果不存在就新创建元素，并赋予对应的分数     |
| **zinterstore** newKey num oldKey1 ... oldKeyNum [weights weight1...weightNum] | 计算给定的一个或多个有序集的交集并将结果集存储在新的有序集合 key 中。<br />**根据value值取交集**<br />**新key中的value的score为多个相加**<br />**weight后面标识权重。新key中的value为对应的score*对应的权重，再相加。** |
| **zrem** key value                                           | 从集合中弹出一个元素**【删除】**                             |
| **zlexcount** key preValue postValue                         | 计算有序集合中指定字典区间内成员数量<br />**-+表示最小值和最大值，如果我们需要通过元素查找的话需要加[** |
| **zrangebylex** key preValue postValue                       | 获取指定区间的元素，分数必须相同<br />**-+表示最小值和最大值，如果我们需要通过元素查找的话需要加[** |

***

## 持久化

> redis是一个内存数据库，数据保存在内存中，但是我们都知道内存的数据变化是很快的，也容易发生丢失。幸好Redis还为我们提供了持久化的机制，分别是**RDB**(Redis DataBase)和**AOF**(Append Only File)。

### RDB

> RDB是一种**快照存储**持久化方式，具体就是**将Redis某一时刻的内存数据保存到硬盘的文件当中**，默认保存的文件名为dump.rdb，而在Redis服务器启动时，会重新加载dump.rdb文件的数据到内存当中恢复数据。

#### RDB快照生成方式

> 可以通过五种方式触发RDB快照生成。
>
> - **客户端save命令**
>
> save命令是个**阻塞命令**。当服务器接收到save命令时，开始「拍摄」快照，在此期间不处理其他请求，其他请求将会被挂起直到备份结束。
>
> ![image-20200721151325113](https://gitee.com/wangigor/typora-images/raw/master/redis-save-rm.png)
>
> ![image-20200721151425000](https://gitee.com/wangigor/typora-images/raw/master/redis-save.png)
>
> ![image-20200721151452070](https://gitee.com/wangigor/typora-images/raw/master/redis-save-after-dump.png)
>
> > 先删除dump.rdb文件。
> >
> > 客户端发送save请求。
> >
> > 返回OK后，dump.rdb生成。
>
> - **客户端bgsave命令**
>
> bgsave命令也是「拍摄」快照。**非阻塞命令**，而是fork一个子线程，子线程进行备份操作，父线程继续处理客户端请求。
>
> ![image-20200721151834899](https://gitee.com/wangigor/typora-images/raw/master/redis-bgsave.png)
>
> - **服务器配置文件**
>
> ```properties
> save 900 1   #900秒内至少有1个key被更改就执行快照
> save 300 10  #300内描述至少有10个key被更改就执行快照
> save 60 10000  #60秒内至少有10000个key被更改就执行快照
> ```
>
> - **shutdown命令**
>
> 手动shutdown命令是，服务器会自动发送一条save命令来完成快照，并在完成备份后关闭服务。
>
> - **sync命令**
>
> 主从环境中，进行主从复制操作：
>
> ①**从节点**向主节点发送**sync命令**。
>
> ②**主节点**执行**bgsave命令**，fork一个新线程完成快照。
>
> ③主节点将dump文件发送至从节点。
>
> ④从节点完成数据同步。

#### RDB快照文件

> 快照文件的生成过程是，生成临时文件 -> 写入数据 -> 临时文件替换正式文件。
>
> 默认文件名是dump.rdb，可以通过配置文件修改。

| 参数                        | 默认值   | 说明                           |
| --------------------------- | -------- | ------------------------------ |
| stop-writes-on-bgsave-error | yes      | 拍摄快照失败是否继续执行写命令 |
| rdbcompression              | yes      | 是否对快照文件进行压缩         |
| rdbchecksum                 | yes      | 是否数据校验                   |
| dbfilename                  | dump.rdb | 快照文件存储的名称             |
| dir                         | ./       | 快照文件存储的位置             |

#### 禁用快照持久化

- 在redis.conf配置文件中注释掉所有的save配置
- 追加 save "" 配置。

#### 优缺点

##### 优点

- RDB是一个**非常紧凑的文件**,**它保存了某个时间点得数据集,非常适用于数据集的备份**,比如你可以在每个小时报保存一下过去24小时内的数据,同时每天保存过去30天的数据,这样即使出了问题你也可以根据需求恢复到不同版本的数据集.
- RDB是一个紧凑的**单一文件**,很**方便传送**到另一个远端数据中心或者亚马逊的S3（可能加密），非常**适用于灾难恢复**.
- RDB在保存RDB文件时父进程唯一需要做的就是**fork出一个子进程**,接下来的工作全部由子进程来做，父进程不需要再做其他IO操作，所以RDB持久化方式可以最大化redis的性能.
- **与AOF相比,在恢复大的数据集的时候，RDB方式会更快一些**.

##### 缺点

- 如果服务器宕机，依然会有某个时间段内数据丢失。
- save命令会造成服务阻塞。
- bgsave命令即便使用fork子线程进行备份，当数据集较大时，fork过程耗时，可能会导致redis在毫秒级不能响应的请求。

### AOF

> RDB快照功能不是非常耐久，如果redis因为某些原因造成故障停机，服务器将丢失最近写入、且没有保存到快照中的那些数据。
>
> 从 1.1 版本开始， Redis 增加了一种完全耐久的持久化方式： AOF 持久化。
>
> 每当 Redis 执行一个改变数据集的命令时（比如 SET）， 这个命令就会被追加到 AOF 文件的末尾。这样的话， 当 Redis 重新启时， 程序就可以通过重新执行 AOF 文件中的命令来达到重建数据集的目的。

#### AOF配置

> 默认关闭。

| 参数                        | 值               | 说明                                                         |
| --------------------------- | ---------------- | ------------------------------------------------------------ |
| appendonly                  | yes              | 是否开启AOF持久化                                            |
| appendfilename              | “appendonly.aof” | 存储的文件的名称                                             |
| appendfsync                 | everysec         | 同步频率<br />everysec 每隔一秒钟持久化一次<br/>always 每执行一条命令持久化一次<br/>no 持久化的时机交给操作系统处理 |
| no-appendfsync-on-rewrite   | no               | 执行写入操作时是否进行重写                                   |
| auto-aof-rewrite-percentage | 100              | 当前AOF文件超过上次AOF文件的百分比后才进行持久化操作         |
| auto-aof-rewrite-min-size   | 64mb             | 自定执行AOF操作文件最小的大小要达到的大小                    |

##### 写入策略

- **everysec** 【默认】

每秒写入一次，最多丢失1s的数据

- **always**

每一个写操作都保存到aof中，很安全，所以慢。

- **no**

将数据交给操作系统来处理。更快，也更不安全的选择

#### 日志重写

> AOF将客户端的每一个写操作都追加到aof文件末尾，比如对一个key多次执行incr命令，这时候，aof保存每一次命令到aof文件中，aof文件会变得非常大。
>
> aof文件太大，加载aof文件恢复数据时，就会非常慢，为了解决这个问题，Redis支持aof文件重写，通过重写aof，可以生成一个恢复当前数据的最少命令集。
>
> - **压缩aof文件。减少磁盘占用**。
> - **压缩成更少的指令集，加快数据恢复速度**。

就是保留结果。不保留中间过程，对多条指令进行合并。

##### 日志重写触发

- no-appendfsync-on-rewrite 这个参数指定了写入aof操作时，是否执行重写。默认和推荐是no。因为每次fsync都执行重写，影响性能。
- 客户端发起bgrewriteaof命令。

##### 重写原理

- Redis 执行 fork() ，现在同时拥有父进程和子进程。
- 子进程开始将新 AOF 文件的内容写入到临时文件。
- 对于所有新执行的写入命令，父进程一边将它们累积到一个内存缓存中，一边将这些改动追加到现有 AOF 文件的末尾,这样样即使在重写的中途发生停机，现有的 AOF 文件也还是安全的。
- 当子进程完成重写工作时，它给父进程发送一个信号，父进程在接收到信号之后，将内存缓存中的所有数据追加到新 AOF 文件的末尾。
- 搞定！现在 Redis 原子地用新文件替换旧文件，之后所有命令都会直接追加到新 AOF 文件的末尾。



#### AOF文件损坏

服务器可能在程序正在对 AOF 文件进行写入时停机， 如果停机造成了 AOF 文件出错（corrupt）， 那么 Redis 在重启时会拒绝载入这个 AOF 文件， 从而确保数据的一致性不会被破坏。当发生这种情况时， 可以用以下方法来修复出错的 AOF 文件：

- 为现有的 AOF 文件创建一个备份。

- 使用 Redis 附带的 redis-check-aof 程序，对原来的 AOF 文件进行修复:

  $ redis-check-aof –fix

- （可选）使用 diff -u 对比修复后的 AOF 文件和原始 AOF 文件的备份，查看两个文件之间的不同之处。

- 重启 Redis 服务器，等待服务器载入修复后的 AOF 文件，并进行数据恢复。

#### 优缺点

##### 优点

- 使用AOF 会让你的Redis**更加耐久**: 你可以使用不同的fsync策略：无fsync,每秒fsync,每次写的时候fsync.使用默认的每秒fsync策略,Redis的性能依然很好(fsync是由后台线程进行处理的,主线程会尽力处理客户端请求),一旦出现故障，你最多丢失1秒的数据.
- AOF文件是一个只进行追加的日志文件,所以不需要写入seek,即使由于某些原因(磁盘空间已满，写的过程中宕机等等)未执行完整的写入命令,你也也可使用redis-check-aof工具修复这些问题.
- Redis 可以在 AOF 文件体积变得过大时，自动地在后台对 AOF 进行**重写**： 重写后的新 AOF 文件包含了恢复当前数据集所需的最小命令集合。 整个重写操作是绝对安全的，因为 Redis 在创建新 AOF 文件的过程中，会继续将命令追加到现有的 AOF 文件里面，即使重写过程中发生停机，现有的 AOF 文件也不会丢失。 而一旦新 AOF 文件创建完毕，Redis 就会从旧 AOF 文件切换到新 AOF 文件，并开始对新 AOF 文件进行追加操作。
- AOF 文件**有序**地保存了对数据库执行的所有写入操作， 这些写入操作以 Redis 协议的格式保存， 因此 AOF 文件的内容非常容易被人读懂， 对文件进行分析（parse）也很轻松。 导出（export） AOF 文件也非常简单： 举个例子， 如果你不小心执行了 FLUSHALL 命令， 但只要 AOF 文件未被重写， 那么只要停止服务器， 移除 AOF 文件末尾的 FLUSHALL 命令， 并重启 Redis ， 就可以将数据集恢复到 FLUSHALL 执行之前的状态。

##### 缺点

- 对于相同的数据集来说，**AOF 文件的体积通常要大于 RDB 文件的体积**。
- 根据所使用的 fsync 策略，**AOF 的速度可能会慢于 RDB** 。 在一般情况下， 每秒 fsync 的性能依然非常高， 而关闭 fsync 可以让 AOF 的速度和 RDB 一样快， 即使在高负荷之下也是如此。 不过在处理巨大的写入载入时，RDB 可以提供更有保证的最大延迟时间（latency）。

### 选择RDB还是AOF

> 一般来说， 如果想达到足以媲美 PostgreSQL 的数据安全性， 你应该同时使用两种持久化功能。
>
> 如果你非常关心你的数据， 但仍然可以承受数分钟以内的数据丢失， 那么你可以只使用 RDB 持久化。
>
> 有很多用户都只使用 AOF 持久化， 但我们并不推荐这种方式： 因为定时生成 RDB 快照（snapshot）非常便于进行数据库备份， 并且 RDB 恢复数据集的速度也要比 AOF 恢复的速度要快， 除此之外， 使用 RDB 还可以避免之前提到的 AOF 程序的 bug 。

![img](https://gitee.com/wangigor/typora-images/raw/master/compara-rdb-aofpng)

当RDB与AOF两种方式都开启时，Redis会优先使用AOF日志来恢复数据，因为AOF保存的文件比RDB文件更完整。

- 如果仅作为缓存服务器，可以不使用任何持久化。
- 开启AOF的情况下，主从同步是时候必然会带来IO的性能影响，此时我们可以调大auto-aof-rewrite-min-size的值，比如5GB。来减少IO的频率
- 不开启AOF的情况下，可以节省IO的性能影响，这是主从建通过RDB持久化同步，但如果主从都挂掉，影响较大

- **混合持久化开启**

> 4.0版本的混合持久化默认关闭的，通过**aof-use-rdb-preamble**配置参数控制，yes则表示开启，no表示禁用，默认是禁用的，可通过config set修改。
>
> 混合持久化同样也是通过bgrewriteaof完成的，不同的是当开启混合持久化时，**fork出的子进程先将共享的内存副本全量的以RDB方式写入aof文件，然后在将重写缓冲区的增量命令以AOF方式写入到文件，写入完成后通知主进程更新统计信息，并将新的含有RDB格式和AOF格式的AOF文件替换旧的的AOF文件。**简单的说：**新的AOF文件前半段是RDB格式的全量数据后半段是AOF格式的增量数据**

***

## 主从复制

> - 实现读写分离
> - 降低master压力
> - 实现数据备份

### 同步方式

#### 增量同步

> 增量同步，同步的是指令流。
>
> - 主节点将那些对自己状态产生修改性影响的指令记录在本地内存buffer中。
> - 异步将buffer中的指令同步到从节点。
> - 从节点 一边执行同步指令，一边向主节点反馈同步位置【偏移量】

<img src="https://gitee.com/wangigor/typora-images/raw/master/redis-master-slave-buffer.jpg" alt="img" style="zoom:50%;" />

- 内存buffer有限。内存buffer是一个定长的环状数组，如果数组内容满了，就会从头开始覆盖前面的内容。
- 如果网络状态不好，从节点短时间内无法与主节点同步。网络恢复时，可能会有没有同步的指令在buffer已经被后续指令覆盖了，这时通过指令流同步的方式就会有问题，需要进行快照同步【由主节点控制】。



#### 快照同步

> 快照同步是一个非常耗费资源的操作。
>
> - 主库进行bgsave，将内存数据快照到磁盘文件中。
> - 将快照文件内容全部传送到从节点。
> - 从节点接收文件完毕之后，先清空当前内存数据，执行一次全量加载。
> - 通知树节点进行增量同步。
>
> 问题：在整个快照同步过程中，主节点的复制buffer还在不停的向前移动，如果快照同步时间过长 或 复制buffer太小，都会导致同步期间的增量指令被覆盖。这到导致快照同步之后无法进行增量同步，需要再次发起快照同步。
>
> 如此极有可能陷入**持续快照同步的死循环**。

### 主从配置



#### 一主多从式

> - master节点可读可写，但是slave节点**只读不可写**(如果非要写可以修改redis.conf文件中的slave-read-only的值来实现)
> - 在当前的这个主从结构中，如果master挂点，**重启后依然还是master**，主从操作依然可用。

![技术分享图片](https://gitee.com/wangigor/typora-images/raw/master/redis-master-3slave.png)

- 命令配置

> 两个节点手动通过slaveof命令配置

```shell
127.0.0.1:6380> slaveof 127.0.0.1 6379
127.0.0.1:6381> slaveof 127.0.0.1 6379
```

- 配置文件

> 在conf文件中添加master节点配置。

```properties
slaveof 127.0.0.1 6379
```

可以使用命令查看当前节点的主从信息

```shell
127.0.0.1:6380> info replication
```



#### 薪火相传式

> 为了减轻master的写压力。
>
> 上一个Slave可以是下一个Slave的Master，Slave同样可以接收其他slaves的连接和同步请求，那么该slave作为了
>
> 链条中下一个slave的Master。
>
> 命令还是同样的命令。

![技术分享图片](https://gitee.com/wangigor/typora-images/raw/master/redis-master-slave-slave.png)

#### 取消同步

> 当Master挂掉后，Slave可键入命令 slaveof no one使当前redis停止与其他Master redis数据同步，转成
> Master redis。
>
> ```shell
> 127.0.0.1:6380> slaveof no one
> ```

***

## 集群和哨兵机制

### 环境准备

> 先准备6个redis独立实例。
>
> 使用docker-compose比较简单。都是些通用配置，only端口号不同，使用shell生成。

```shell
#init.sh
#!/usr/bin/env bash

#在当前文件夹下生成docker-compose头
cat >> ./docker-compose.yml << EOF
version: '3'
services:
EOF

#redis端口 7001~7006
#循环创建 每个端口对应的redis实例需要的工作目录和conf
for port in $(seq 7001 7006);
do
mkdir -p ./node-${port}/conf
touch ./node-${port}/conf/redis.conf
cat >> ./node-${port}/conf/redis.conf << EOF
port ${port}
cluster-enabled yes
cluster-config-file nodes.conf #nodes.conf会自动生成记录集群信息
cluster-node-timeout 5000
cluster-announce-ip 192.168.8.100 #本机ip
cluster-announce-port ${port}
cluster-announce-bus-port 1${port}
appendonly yes #开启AOF
EOF
#把当前节点写入docker-compose中，方便之后一起启动
cat >> ./docker-compose.yml << EOF
  redis-${port}:
    image: redis:5.0.7
    container_name: redis-${port}
    volumes:
      - "./node-${port}/data:/data"
      - "./node-${port}/conf/redis.conf:/etc/redis/redis.conf"
    ports:
      - "${port}:${port}"
      - "1${port}:1${port}"
    restart: always
    hostname: redis-${port}
    command:
      redis-server /etc/redis/redis.conf
EOF
done
```

```shell
#赋予所有用户可执行权限
chmod a+x init.sh
#执行
./init.sh
```

![image-20200722161509357](https://gitee.com/wangigor/typora-images/raw/master/redis-init-result.png)

初始化目录结构生成完成。

```shell
#启动全部6个redis实例
docker-compose up -d
```

![image-20200722161743755](https://gitee.com/wangigor/typora-images/raw/master/redis-docker-compose-ps.png)

六个实例生成完成。每个登录进去都是独立无关联的redis服务端。

### 哨兵机制

> 哨兵机制的产生，是因为前面的主从复制存在问题：
>
> 虽然，主从复制解决了数据备份问题。
>
> ①可以做到主节点故障不可达时，从节点可以后备顶上来保证数据尽量不丢失。
>
> ②从节点扩展了主节点的读能力。分担了主节点的读压力。
>
> 但是。
>
> 一旦主节点出现了故障，需要**手动**将一个节点晋升为主节点。
>
> 同时，需要**修改应用方**的主节点地址。
>
> 还需要**手动**命令去调整整个「集群」的结构。
>
> 都需要**人工干预**。
>
> ***
>
> **Sentinel方案**是一个分布式架构。
>
> - 包含了若干个Sentinel节点和Redis节点。
> - Sentinel节点不存储数据，Redis节点是数据节点。
> - sentinel节点对redis节点和其他sentinel节点进行监控。
> - 当发现节点不可达时，对节点做下线标识。
> - 如果被标记的节点是主节点，会和其他sentinel节点进行「协商」，当大多数sentinel节点都认为主节点不可达时，选出一个sentinel节点完成自动故障转移工作，同时会将这个变化适时通知给redis应用方。
> - 整个过程完全自动。
>
> ![img](https://gitee.com/wangigor/typora-images/raw/master/redis-sentinel-架构图.png)

#### 部署

> 三个sentinel节点分别为 7001、7002、7003。
>
> 三个redis节点分别为 【master】7004 和【slave】7005、7006.

##### 配置一对多主从

分别修改两个slave节点7005、7006的配置文件，追加对7004的slaveof配置

注意：注释掉三个节点cluster模式的属性。

```properties
#cluster-enabled yes
#cluster-config-file nodes.conf
#cluster-node-timeout 5000
#cluster-announce-ip 192.168.8.100
#cluster-announce-port 7004
#cluster-announce-bus-port 17004
slaveof 192.168.8.100 7004
```

重启7004、7005、7006

```shell
wangke@wangkedeMacBook-Pro:~/docker/redis> docker restart redis-7004 redis-7005 redis-7006
redis-7004
redis-7005
redis-7006
```

验证主从关系

![](https://gitee.com/wangigor/typora-images/raw/master/redis-sentinel-验证主从关系.png)

主节点看到有两个slave。

![image-20200722171958984](https://gitee.com/wangigor/typora-images/raw/master/redis-sentinel-验证主从关系-slave.png)

从节点也能看到对应的master节点的情况

##### 部署sentinel节点

7001、7002、7003节点增加sentinel配置

> sentinel monitor myMaster 192.168.8.100 7004 2配置代表sentinel节点需要监控192.168.8.100:7004这个主节点，2代表判断主节点失败至少需要2个Sentinel节点同意，myMaster是主节点的别名

```properties
sentinel monitor myMaster 192.168.8.100 7004 2
sentinel down-after-milliseconds myMaster 30000
sentinel parallel-syncs myMaster 1
sentinel failover-timeout myMaster 180000
```

> 注意：修改7001、7002、7003三个实例的启动方式。
> 之前定义在docker-compose.yml中的启动方式是「redis-server」。
> 需要修改成**「redis-sentinel」**。
>
> 否则报错
>
> ```log
> *** FATAL CONFIG FILE ERROR ***
> Reading the configuration file, at line 9
> >>> 'sentinel monitor myMaster 192.168.8.100 7004 2'
> sentinel directive while not in sentinel mode
> ```
>
> 要**删除节点**重新启动。
>
> ```shell
> docker-compose down
> docker-compose up -d
> ```

查看sentinel状态

![image-20200723100344425](https://gitee.com/wangigor/typora-images/raw/master/redis-sentinel-状态.png)

> **可观察到master地址，端口正确。slave节点，sentinel节点数量正确。**
>
> 三个sentinel节点的配置文件，都被自动修改
>
> ```properties
> port 7001
> #cluster-enabled yes
> #cluster-config-file nodes.conf
> #cluster-node-timeout 5000
> #cluster-announce-ip 192.168.8.100
> #cluster-announce-port 7001
> #cluster-announce-bus-port 17001
> appendonly yes
> sentinel myid eae6e67b9ce0f97c62444b86e1c7c389ba572fd6
> sentinel deny-scripts-reconfig yes
> sentinel monitor myMaster 192.168.8.100 7004 2
> sentinel config-epoch myMaster 0
> # Generated by CONFIG REWRITE
> dir "/data"
> sentinel leader-epoch myMaster 0
> sentinel known-replica myMaster 172.28.0.1 7006
> sentinel known-replica myMaster 172.28.0.1 7005
> sentinel known-sentinel myMaster 172.28.0.4 7002 2f03ac8ac0323f7c12e055a10fcf82b990f19a8e
> sentinel known-sentinel myMaster 172.28.0.3 7003 08cfa6527022dabe24f15160ab6279c37062d363
> sentinel current-epoch 0
> ```
>
> - Sentinel节点自动发现了从节点、其余Sentinel节点。
> - 去掉了默认配置，例如parallel-syncs、failover-timeout参数。
> - 添加了配置纪元相关参数。

##### springboot-Jedis-sentinel测试

> 之前。docker分配的都内网ip，需要每个redis.conf设置 **slave-announce-ip 192.168.8.100**。
> 经过 sentinel解析后，会变成**replica-announce-ip "192.168.8.100"**。

- redis配置类

redis.properties

```properties
redis.nodes=192.168.8.100:7001,192.168.8.100:7002,192.168.8.100:7003
redis.masterName=myMaster
redis.maxTotal=10000
redis.maxIdle=100
redis.minIdle=50
redis.timeout=30000
```

RedisProperties

```java
@Data
@ToString
@Configuration
@PropertySource("classpath:redis.properties")
@ConfigurationProperties(prefix = "redis")
public class RedisProperties {
    /**
     * 节点名称
     */
    private String nodes;

    /**
     * Redis服务名称
     */
    private String masterName;

    /**
     * 最大连接数
     */
    private int maxTotal;

    /**
     * 最大空闲数
     */
    private int maxIdle;

    /**
     * 最小空闲数
     */
    private int minIdle;

    /**
     * 连接超时时间
     */
    private int timeout;
}
```

RedisConfig

```java
@Configuration
@Slf4j
public class RedisConfig {

    @Autowired
    private RedisProperties redisProperties;

    @Bean
    public JedisPoolConfig jedisPoolConfig() {
        JedisPoolConfig config = new JedisPoolConfig();

        config.setMaxTotal(redisProperties.getMaxTotal());
        config.setMaxIdle(redisProperties.getMaxIdle());
        config.setMinIdle(redisProperties.getMinIdle());

        config.setTestOnBorrow(true);
        config.setTestOnReturn(true);
        return config;
    }

    @Bean
    public JedisSentinelPool jedisSentinelPool() {

        String propertiesNodes = redisProperties.getNodes();

        HashSet<String> nodeSet = Sets.newHashSet(
                propertiesNodes.split(",")
        );

        return new JedisSentinelPool(redisProperties.getMasterName(), nodeSet, jedisPoolConfig(), redisProperties.getTimeout());
    }

}
```

RedisService

> 简单的String的get方法

```java
@Component
public class StringRedisService {
    @Autowired
    private JedisSentinelPool jedisSentinelPool;


    private Jedis getJedis() {
        return jedisSentinelPool.getResource();
    }

    public String getString(String key) {
        Jedis jedis = getJedis();
        return jedis.get(key);
    }
}
```

测试类

```java
@Slf4j
@SpringBootTest
public class StringRedisServiceTest {
    @Autowired
    private StringRedisService redisService;

    @SneakyThrows
    @Test
    public void testSentinel() {
        while (true) {

            TimeUnit.SECONDS.sleep(1);
          	//这里要try-catch住异常。
            try {
                log.info("test = {}", redisService.getString("test"));
            } catch (Exception e) {
                log.info("test = {}", e.getMessage());
            }

        }
    }
}
```

再确认一下节点信息

![image-20200727102101491](https://gitee.com/wangigor/typora-images/raw/master/redis-sentinel-test-info-before.png)

目前主节点在7006实例。

启动测试

> 每秒打印一条value

![image-20200727102009358](https://gitee.com/wangigor/typora-images/raw/master/redis-sentinel-test-java-console-before.png)

**==手动关闭master节点==**

日志变化

![image-20200727102345439](https://gitee.com/wangigor/typora-images/raw/master/redis-sentinel-test-console-after.png)

已经切换到节点7005

![image-20200727102445730](https://gitee.com/wangigor/typora-images/raw/master/redis-sentinel-test-info-after.png)

集群节点也已经切换至7005

sentinel日志

![image-20200727102713757](https://gitee.com/wangigor/typora-images/raw/master/redis-sentinel-log-switch-master.png)

#### sentinel 配置说明

- ### sentinel monitor mymaster

```properties
sentinel monitor <master-name> <ip> <port> <quorum>
```

哨兵要监控名称为<master-name>，暴露的ip和端口是<ip>:<port>的主节点。

quorum是判断主节点不可达的票数。建议是sentinel节点数量的一半+1。

同时也跟sentinel的领导选举有关，至少要有max（quorum，num（sentinels）/2+1）个Sentinel节点参与选举，才能选出领导者Sentinel，从而完成故障转移。

- ### sentinel down-after-milliseconds

```properties
sentinel down-after-milliseconds <master-name> <times>
```

每个Sentinel节点都要通过定期发送**ping命令来判断Redis数据节点和其余Sentinel节点是否可达**，如果超过了down-after-milliseconds配置的时间且没有有效的回复，则判定节点不可达，（单位为毫秒）就是超时时间。这个配置是对节点失败判定的重要依据。

- ### sentinel parallel-syncs

```properties
sentinel parallel-syncs <master-name> <nums>
```

当Sentinel节点集合对主节点故障判定达成一致时，Sentinel领导者节点会做故障转移操作，选出新的主节点，原来的从节点会向新的主节点发起复制操作，parallel-syncs就是用来限制在一次故障转移之后，每次向新的主节点发起复制操作的从节点个数。如果这个参数配置的比较大，那么多个从节点会向新的主节点同时发起复制操作，尽管复制操作通常不会阻塞主节点，但是同时向主节点发起复制，必然会对主节点所在的机器造成一定的网络和磁盘IO开销。

- ### sentinel failover-timeout

```properties
sentinel failover-timeout <master-name> <times>
```

failover-timeout通常被解释成故障转移超时时间，但实际上它作用于故
障转移的各个阶段：
a）选出合适从节点。
b）晋升选出的从节点为主节点。
c）命令其余从节点复制新的主节点。
d）等待原主节点恢复后命令它去复制新的主节点。

- ### sentinel auth-pass

```properties
sentinel auth-pass <master-name> <password>
```

如果Sentinel监控的主节点配置了密码，sentinel auth-pass配置通过添加
主节点的密码，防止Sentinel节点对主节点无法监控。

- ### sentinel notification-script

```properties
sentinel notification-script <master-name> <script-path>
```

sentinel notification-script的作用是在故障转移期间，当一些警告级别的Sentinel事件发生（指重要事件，例如-sdown：客观下线、-odown：主观下线）时，会触发对应路径的脚本，并向脚本发送相应的事件参数。

- ### sentinel client-reconfig-script

```properties
sentinel client-reconfig-script <master-name> <script-path>
```

sentinel client-reconfig-script的作用是在故障转移结束后，会触发对应路
径的脚本，并向脚本发送故障转移结果的相关参数。

#### 实现原理

哨兵启动后会与要监控的主数据库建立两条连接

![img](https://gitee.com/wangigor/typora-images/raw/master/redis-sentinel-master-tcp.png)

和主数据库连接建立完成后，哨兵会使用连接2发送如下命令

- 每10秒钟哨兵会向主数据库和从数据库发送INFO 命令
- 每2秒钟哨兵会向主数据库和从数据的_sentinel_:hello频道发送自己的消息。
- 每1秒钟哨兵会向主数据、从数据库和其他哨兵节点发送PING命令。

首先,发送INFO命令会返回当前数据库的相关信息(运行id，从数据库信息等)从而实现新节点的自动发现，前面提到的配置哨兵时只需要监控Redis主数据库即可，因为哨兵可以借助INFO命令来获取所有的从数据库信息(slave),进而和这两个从数据库分别建立两个连接。在此之后哨兵会每个10秒钟向已知的主从数据库发送INFO命令来获取信息更新并进行相应的操作。

接下来哨兵向主从数据库的_sentinel_:hello 频道发送信息来与同样监控该数据库的哨兵分享自己的信息。发送信息内容为:

```text
<哨兵的地址>，<哨兵的端口>，<哨兵的运行ID>，<哨兵的配置版本>，<主数据库的名字>，<主数据库的地址>，<主数据库的端口>，<主数据库的配置版本>
```

![在这里插入图片描述](https://gitee.com/wangigor/typora-images/raw/master/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9kcGItYm9ib2thb3lhLXNtLmJsb2cuY3Nkbi5uZXQ=,size_16,color_FFFFFF,t_70-20200727105436610.png)

哨兵通过监听的_sentinel_:hello频道接收到其他哨兵发送的消息后会判断哨兵是不是新发现的哨兵，如果是则将其加入已发现的哨兵列表中并创建一个到其的连接(哨兵与哨兵只会创建用来发送PING命令的连接，不会创建订阅频道的连接)。

  实现了自定发现从数据库和其他哨兵节点后，哨兵要做的就是定时监控这些数据和节点运行情况，每隔一定时间向这些节点发送PING命令来监控。间隔时间和down-after-milliseconds选项有关，down-after-milliseconds的值小于1秒时，哨兵会每隔down-after-milliseconds指定的时间发送一次PING命令，当down-after-milliseconds的值大于1秒时，哨兵会每隔1秒发送一次PING命令。

```properties
// 每隔1秒发送一次PING命令
sentinel down-after-milliseconds mymaster 60000
// 每隔600毫秒发送一次PING命令
sentinel down-after-milliseconds othermaster 600
```

##### 主管下线

当超过down-after-milliseconds指定时间后，如果**被PING的数据库或节点仍然未回复**，则哨兵认为其主观下线(subjectively down),主观下线表示从当前的哨兵进程看来，该节点已经下线。

##### 客观下线

在主观下线后，如果该节点是主数据库，则哨兵会进一步判断是否需要对其进行故障恢复，哨兵发送SENTINEL is-master-down-by-addr 命令询问其他哨兵节点以了解他们是否也认为该主数据库主观下线，如果达到指定数量时，哨兵会认为其客观下线(objectively down),并选举领头的哨兵节点对主从系统发起故障恢复。这个指定数量就是前面配置的 quorum参数。

该配置表示只有当至少有两个Sentinel节点(包括当前节点)认为该主数据库主观下线时，当前哨兵节点才会认为该主数据库客观下线。接下来选举领头哨兵。

##### 选举领头哨兵

当前哨兵虽然发现了主数据客观下线，需要故障恢复，但故障恢复需要由领头哨兵来完成。这样来保证**同一时间只有一个哨兵来执行故障恢复**，选举领头哨兵的过程使用了==**Raft算法**==，具体过程如下:

- 发现主数据库客观下线的哨兵节点(A)向每个哨兵节点发送命令，要求对象选择自己成为领头哨兵
- 如果目标哨兵节点没有选过其他人，则会同样将A设置为领头哨兵
- 如果A发现有超过半数且超过quorum参数值的哨兵节点同样选择自己成为领头哨兵，则A成功成为领头哨兵
- 当有多个哨兵节点同时参选领头哨兵，则会出现没有任何节点当选的可能，此时每个参选节点将等待一个随机事件重新发起参选请求进行下一轮选举，直到选举成功。

> Raft选举：
>
> 规则：群众发起投票成为候选人，候选人得到大多数票至少(n/2)+1，才能成为领导人，（自己可以投自己，当没有接受到请求节点的选票时，发起投票节点才能自己选自己），领导人负责处理所有与客户端交互，是数据唯一入口，协调指挥群众节点。
>
> 选举过程：考虑最简单情况，abc三个节点，每个节点只有一张票，当N个节点发出投票请求，其他节点必须投出自己的一票，不能弃票，**最差的情况是每个人都有一票**，那么随机设置一个timeout时间，就像**加时赛**一样，这时同时的概率大大降低，**谁最先恢复过来，就向其他两个节点发出投票请求**，获得大多数选票，成为领导人。选出 Leader 后，Leader 通过定期向所有 Follower 发送心跳信息维持其统治。若 Follower 一段时间未收到 Leader 的心跳则认为 Leader 可能已经挂了再次发起选主过程。

##### 故障恢复

选出领头哨兵后，领头哨兵将会开始对主数据库进行故障恢复.

- 首先领头哨兵将从停止服务的主数据库的从数据库中挑选一个来充当新的主数据库。

|      | 挑选依据                                                     |
| ---- | ------------------------------------------------------------ |
| 1    | 所有在线的从数据库中，选择优先级最高的从数据库。**优先级通过replica-priority参数设置** |
| 2    | 优先级相同，则复制的命令偏移量越大**(复制越完整)**越优先     |
| 3    | 如果以上都一样，则选择**运行ID较小**的从数据库               |

- 选出一个从数据库后，领头哨兵将向从数据库发送SLAVEOF NO ONE命令使其升格为主数据库，而后领头哨兵向其他从数据库发送 SLAVEOF命令来使其成为新主数据库的从数据库，最后一步则是更新内部的记录，将已经停止服务的旧的主数据库更新为新的主数据库的从数据库，使得当其恢复服务时自动以从数据库的身份继续服务

### cluster集群

> 哨兵模式仍然只在master节点进行写操作。
>
> 为了**分担写操作的压力**，Cluster集群应运而生。
>
> ![img](https://gitee.com/wangigor/typora-images/raw/master/redis-cluster-架构.jpg)
>
> - 所有的redis节点彼此互联(PING-PONG机制),内部使用二进制协议优化传输速度和带宽.
> - 节点的fail是通过集群中超过半数的master节点检测失效时才生效.
> - 客户端与redis节点直连,不需要中间proxy层.客户端不需要连接集群所有节点,连接集群中任何一个可用节点即可
> - redis-cluster把所有的物理节点映射到[0-16383]slot上,cluster 负责维护node<->slot<->key
>
> ![img](https://gitee.com/wangigor/typora-images/raw/master/redis-cluster-容错.jpg)
>
> - 选举过程是集群中所有master参与,如果半数以上master节点与故障节点通信超过(cluster-node-timeout),认为该节点故障，自动触发故障转移操作.
> - 如果集群任意master挂掉,且当前master没有slave.集群进入fail状态,也可以理解成集群的slot映射[0-16383]不完成时进入fail状态. ps : redis-3.0.0.rc1加入cluster-require-full-coverage参数,默认关闭,打开集群兼容部分失败.
> - 如果集群超过半数以上master挂掉，无论是否有slave集群进入fail状态.
> - 当集群不可用时,所有对集群的操作做都不可用，收到((error) CLUSTERDOWN The cluster is down)错误

#### 部署

> 清空6个redis实例的备份数据。修改回原来的配置文件。重启。
>
> docker-compose.yml
>
> ```shell
> redis-sentinel /etc/redis/redis.conf
> #改为
> redis-server /etc/redis/redis.conf
> ```
>
> redis.conf
>
> ```properties
> port ${port}
> cluster-enabled yes
> cluster-config-file nodes.conf 
> cluster-node-timeout 5000
> cluster-announce-ip 192.168.8.100 
> cluster-announce-port ${port}
> cluster-announce-bus-port 1${port}
> appendonly yes 
> ```
> 或者删除 **除init.sh**的所有文件。重新生成。

进入任何一个redis实例节点。执行集群命令

```shell
redis-cli --cluster create 192.168.8.100:7001 192.168.8.100:7002 192.168.8.100:7003 192.168.8.100:7004 192.168.8.100:7005 192.168.8.100:7006 --cluster-replicas 1
```

![image-20200727163023213](https://gitee.com/wangigor/typora-images/raw/master/redis-cluster-部署.png)

> 三个主节点 【**M**】7001（5461个槽位 [0-5460]）、7002（5462个槽位 [5461-10922]）、7003（5461个槽位 [10923-16383]），分别对应三个从节点【**S**】7004、7005、7006。

可以使用集群方式进入客户端redis-cli，查看集群节点信息。

```shell
redis-cli -c -p 7001
```

cluster info

```properties
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:6
cluster_my_epoch:1
cluster_stats_messages_ping_sent:830
cluster_stats_messages_pong_sent:814
cluster_stats_messages_sent:1644
cluster_stats_messages_ping_received:809
cluster_stats_messages_pong_received:830
cluster_stats_messages_meet_received:5
cluster_stats_messages_received:1644
```

cluster nodes

```shell
8fb45ffa7b0798a690634e240b64343f33390927 192.168.8.100:7002@17002 master - 0 1595839021466 2 connected 5461-10922
ba30e565ea0784a2c67f7dd6388cb2c49c802f6d 192.168.8.100:7005@17005 slave 8fb45ffa7b0798a690634e240b64343f33390927 0 1595839022478 5 connected
9ddb7965b6b294a36dee60d88604899e5bc63c47 192.168.8.100:7006@17006 slave f44bf3d87ba99711e86ae34379843175a6aa456f 0 1595839021466 6 connected
f44bf3d87ba99711e86ae34379843175a6aa456f 192.168.8.100:7003@17003 master - 0 1595839021000 3 connected 10923-16383
df6f8dd9c19421f30c91b1794aee526332bbebcf 192.168.8.100:7001@17001 myself,master - 0 1595839021000 1 connected 0-5460
5722a6d20fcea553d3c38bfcc0e1d432bab8588f 192.168.8.100:7004@17004 slave df6f8dd9c19421f30c91b1794aee526332bbebcf 0 1595839021000 4 connected
```

部署完毕。

#### 客户端测试

![image-20200727164213523](https://gitee.com/wangigor/typora-images/raw/master/redis-cluster-cli-test.png)

#### springboot测试

pom文件

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

application.properties文件

```properties
spring.redis.cluster.nodes[0]=192.168.8.100:7001
spring.redis.cluster.nodes[1]=192.168.8.100:7002
spring.redis.cluster.nodes[2]=192.168.8.100:7003
spring.redis.cluster.nodes[3]=192.168.8.100:7004
spring.redis.cluster.nodes[4]=192.168.8.100:7005
spring.redis.cluster.nodes[5]=192.168.8.100:7006
```

测试类

```java
@Slf4j
@SpringBootTest
public class RedisClusterTest {

    @Autowired
    private RedisTemplate redisTemplate;

    @Test
    public void test() {
        ValueOperations valueOperations = redisTemplate.opsForValue();

        valueOperations.set("a1", "a1");
        log.info("a1={}", valueOperations.get("a1"));
        valueOperations.set("b1", "b1");
        log.info("b1={}", valueOperations.get("b1"));
        valueOperations.set("c1", "c1");
        log.info("c1={}", valueOperations.get("c1"));

    }
}
```

#### cluster操作

| 操作                                       | 说明                                                         |
| ------------------------------------------ | ------------------------------------------------------------ |
| CLUSTER INFO                               | 打印集群的信息                                               |
| CLUSTER NODES                              | 列出集群当前已知的所有节点（node），以及这些节点的相关信息。 |
| CLUSTER MEET <ip> <port>                   | 将 ip 和 port 所指定的节点添加到集群当中，让它成为集群的一份子。 |
| CLUSTER FORGET <node_id>                   | 从集群中移除 node_id 指定的节点。                            |
| CLUSTER REPLICATE <node_id>                | 将当前节点设置为 node_id 指定的节点的从节点。                |
| CLUSTER SAVECONFIG                         | 将节点的配置文件保存到硬盘里面。                             |
| CLUSTER ADDSLOTS <slot> [slot ...]         | 将一个或多个槽（slot）指派（assign）给当前节点。             |
| CLUSTER DELSLOTS <slot> [slot ...]         | 移除一个或多个槽对当前节点的指派。                           |
| CLUSTER FLUSHSLOTS                         | 移除指派给当前节点的所有槽，让当前节点变成一个没有指派任何槽的节点。 |
| CLUSTER SETSLOT <slot> NODE <node_id>      | 将槽 slot 指派给 node_id 指定的节点，如果槽已经指派给另一个节点，那么先让另一个节点删除该槽>，然后再进行指派。 |
| CLUSTER SETSLOT <slot> MIGRATING <node_id> | 将本节点的槽 slot 迁移到 node_id 指定的节点中。              |
| CLUSTER SETSLOT <slot> IMPORTING <node_id> | 从 node_id 指定的节点中导入槽 slot 到本节点。                |
| CLUSTER SETSLOT <slot> STABLE              | 取消对槽 slot 的导入（**import**）或者迁移（migrate）。      |
| CLUSTER KEYSLOT <key>                      | 计算键 key 应该被放置在哪个槽上。                            |
| CLUSTER COUNTKEYSINSLOT <slot>             | 返回槽 slot 目前包含的键值对数量。                           |
| CLUSTER GETKEYSINSLOT <slot> <count>       | 返回 count 个 slot 槽中的键。                                |

***

## 事务

> 事务只支持单机版
>
> redis不支持事务：
>
> - 从实用性的角度来说，失败的命令是由编程错误造成的，而这些错误应该在开发的过程中被发现，而不应该出现在生产环境中。
> - 因为不需要对回滚进行支持，所以 Redis 的内部可以保持简单且快速
>   

- 通过multi开启事务
- 通过exec执行事务
- 开启事务后的命令进入队列中，不会被立即执行
- 命令出错或者命令执行出错，不会回滚事务，继续执行
- 事务没有回滚

```shell
# 开启事务
127.0.0.1:6379> multi
OK
# 持续添加事务内命令
127.0.0.1:6379> set k1 bbb
QUEUED
127.0.0.1:6379> set k2 ccc
QUEUED
127.0.0.1:6379> set k3 ddd
QUEUED
127.0.0.1:6379> set k4 eee
QUEUED
127.0.0.1:6379> get k1
QUEUED
# 执行
127.0.0.1:6379> exec
1) OK
2) OK
3) OK
4) OK
5) "bbb"
```

### watch

> watch命令可以为 Redis 事务提供 check-and-set （CAS）行为。

watch命令可以监控一个或多个键，一旦其中有一个键被修改（或删除），之后的事务就不会执行。监控一直持续到exec命令（事务中的命令是在exec之后才执行的，所以在multi命令后可以修改watch监控的键值）。假设我们通过watch命令在事务执行之前监控了多个Keys，倘若在watch之后有任何Key的值发生了变化，exec命令执行的事务都将被放弃，同时返回Null multi-bulk应答以通知调用者事务执行失败。

```shell
127.0.0.1:6379> set b2 bbb
OK
127.0.0.1:6379> watch b2
OK
127.0.0.1:6379> multi
OK
127.0.0.1:6379> set b2 d111
QUEUED
127.0.0.1:6379> exec
1) OK
127.0.0.1:6379> set b3 bbb
OK
127.0.0.1:6379> watch b3
OK
127.0.0.1:6379> set b3 aaaa
OK
127.0.0.1:6379> set b3 avava
OK
127.0.0.1:6379> multi
OK
127.0.0.1:6379> set b3 aaaa
QUEUED
127.0.0.1:6379> exec
(nil)
```

exec后会自动执行unwatch命令，撤销监控

### UnWatch

撤销对一个key的监控

```shell
127.0.0.1:6379> set n1 aaa
OK
127.0.0.1:6379> watch n1
OK
127.0.0.1:6379> set n1 bbbb
OK
127.0.0.1:6379> unwatch
OK
127.0.0.1:6379> multi
OK
127.0.0.1:6379> set n1 bbbb
QUEUED
127.0.0.1:6379> exec
1) OK
127.0.0.1:6379> get n1
"bbbb"
```



***

## 缓存淘汰策略

### 最大缓存

> 在 redis 中，允许用户设置最大使用内存大小 **server.maxmemory**，默认为0，没有指定最大缓存，如果有新的数据添加，超过最大内存，则会使redis崩溃，所以一定要设置。redis 内存数据集大小上升到一定大小的时候，就会实行数据淘汰策略。

### 主键失效

> 作为一种定期清理无效数据的重要机制，在 Redis 提供的诸多命令中，EXPIRE、EXPIREAT、PEXPIRE、PEXPIREAT 以及 SETEX 和 PSETEX 均可以用来设置一条 Key-Value 对的失效时间，而一条 Key-Value 对一旦被关联了失效时间就会在到期后自动删除（或者说变得无法访问更为准确）

可以给redis中的key设置过期时间，当key过期的时候（生存期为0）,它会被删除。

- 对key进行覆盖会修改数据的生存时间，如set和getSet命令

- 修改key对应的value和使用另外相同的key和value来覆盖以后，当前数据的生存时间不同。 如对一个 key 执行INCR命令，对一个列表进行LPUSH命令，或者对一个哈希表执行HSET命令，这类操作都不会修改 key 本身的生存时间。

- 使用rename对一个 key 进行改名，那么改名后的 key 的生存时间和改名前一样。

- 使用persist命令可以在不删除 key 的情况下，移除 key 的生存时间，让 key 重新成为一个persistent key
- 使用expire命令可以更新key的生存时间
  

### 淘汰机制

> 随着不断的向redis中保存数据，当内存剩余空间无法满足添加的数据时，redis 内就会施行数据淘汰策略，清除一部分内容然后保证新的数据可以保存到内存中。
>
> 内存淘汰机制是为了更好的使用内存，用一定得miss来换取内存的利用率，保证redis缓存中保存的都是热点数据。

redis淘汰策略配置：maxmemory-policy voltile-lru，支持热配置

- **voltile-lru**：从已设置过期时间的数据集（server.db[i].expires）中挑选最近最少使用的数据淘汰
- **volatile-ttl**：从已设置过期时间的数据集（server.db[i].expires）中挑选将要过期的数据淘汰
- **volatile-random**：从已设置过期时间的数据集（server.db[i].expires）中任意选择数据淘汰
- **allkeys-lru**：从数据集（server.db[i].dict）中挑选最近最少使用的数据淘汰
- **allkeys-random**：从数据集（server.db[i].dict）中任意选择数据淘汰
- **no-enviction**（驱逐）：禁止驱逐数据

> 策略规则：
>
> -  如果数据呈现**幂律分布**，也就是**一部分数据访问频率高，一部分数据访问频率低**，则使用**allkeys-lru**
> - 如果数据呈现**平等分布**，也就是**所有的数据访问频率都相同**，则使用**allkeys-random**
> - volatile-lru策略和volatile-random策略适合我们将一个Redis实例既应用于缓存和又应用于持久化存储的时候，然而我们也可以通过使用两个Redis实例来达到相同的效果，
> -  将key设置过期时间实际上会消耗更多的内存，因此我们建议使用allkeys-lru策略从而更有效率的使用内存

#### 非精准LRU

> 上面提到的LRU（Least Recently Used）策略，实际上Redis实现的LRU并不是可靠的LRU，也就是名义上我们使用LRU算法淘汰键，但是**实际上被淘汰的键并不一定是真正的最久没用的**，这里涉及到一个权衡的问题，如果需要**在全部键空间内搜索最优解，则必然会增加系统的开销**，Redis是单线程的，也就是同一个实例在每一个时刻只能服务于一个客户端，所以耗时的操作一定要谨慎 。为了在一定成本内实现相对的LRU，早期的Redis版本是**基于采样的LRU**，也就是放弃全部键空间内搜索解改为采样空间搜索最优解。自从Redis3.0版本之后，Redis作者对于基于采样的LRU进行了一些优化，目的是在一定的成本内让结果更靠近真实的LRU。

#### 触发时机

> - 消极方法（passive way），在主键被访问时如果发现它已经失效，那么就删除它
>
> - 积极方法（active way），周期性地从设置了失效时间的主键中选择一部分失效的主键删除
>
> - 主动删除：当前已用内存超过maxmemory限定时，触发主动清理策略，该策略由启动参数的配置决定
>
>   主键具体的失效时间全部都维护在expires这个字典表中。
>   

***

## 管道

> Redis是一种基于客户端-服务端模型以及请求/响应协议的TCP服务。
>
> 这意味着通常情况下一个请求会遵循以下步骤：
>
> - 客户端向服务端发送一个查询请求，并监听Socket返回，通常是以阻塞模式，等待服务端响应。
> - 服务端处理命令，并将结果返回给客户端。
>
> 因此，例如下面是4个命令序列执行情况：
>
> - *Client:* INCR X
> - *Server:* 1
> - *Client:* INCR X
> - *Server:* 2
> - *Client:* INCR X
> - *Server:* 3
> - *Client:* INCR X
> - *Server:* 4
>
> 客户端和服务器通过网络进行连接。这个连接可以很快（loopback接口）或很慢（建立了一个多次跳转的网络连接）。无论网络延如何延时，数据包总是能从客户端到达服务器，并从服务器返回数据回复客户端。
>
> 这个时间被称之为 RTT (Round Trip Time - 往返时间). 当客户端需要在一个批处理中执行多次请求时很容易看到这是如何影响性能的（例如添加许多元素到同一个list，或者用很多Keys填充数据库）。例如，如果RTT时间是250毫秒（在一个很慢的连接下），即使服务器每秒能处理100k的请求数，我们每秒最多也只能处理4个请求。
>
> 如果采用loopback接口，RTT就短得多（比如我的主机ping 127.0.0.1只需要44毫秒），但它任然是一笔很多的开销在一次批量写入操作中。
>
> 幸运的是有一种方法可以改善这种情况。
>
> 一次请求/响应服务器能实现处理新的请求即使旧的请求还未被响应。这样就可以将*多个命令*发送到服务器，而不用等待回复，最后在一个步骤中读取该答复。
>
> 这就是管道（pipelining），是一种几十年来广泛使用的技术。例如许多POP3协议已经实现支持这个功能，大大加快了从服务器下载新邮件的过程。
>
> Redis很早就支持管道（pipelining）技术，因此无论你运行的是什么版本，你都可以使用管道（pipelining）操作Redis。下面是一个使用的例子：
>
> ```
> $ (printf "PING\r\nPING\r\nPING\r\n"; sleep 1) | nc localhost 6379
> +PONG
> +PONG
> +PONG
> ```
>
> 这一次我们没有为每个命令都花费了RTT开销，而是只用了一个命令的开销时间。
>
> 非常明确的，用管道顺序操作的第一个例子如下：
>
> - *Client:* INCR X
> - *Client:* INCR X
> - *Client:* INCR X
> - *Client:* INCR X
> - *Server:* 1
> - *Server:* 2
> - *Server:* 3
> - *Server:* 4
>
> **重要说明**: 使用管道发送命令时，服务器将被迫回复一个队列答复，占用很多内存。所以，如果你需要发送大量的命令，最好是把他们按照合理数量分批次的处理，例如10K的命令，读回复，然后再发送另一个10k的命令，等等。这样速度几乎是相同的，但是在回复这10k命令队列需要非常大量的内存用来组织返回数据内容。

实例

> 以Jedis+redis cluster为例。
>
> #### 设计思路
>
> 1.首先要根据key计算出此次pipeline会使用到的节点对应的连接（也就是jedis对象，通常每个节点对应一个Pool）。
> 2.相同槽位的key，使用同一个jedis.pipeline去执行命令。
> 3.合并此次pipeline所有的response返回。
> 4.连接释放返回到池中。
>
> 也就是将一个JedisCluster下的pipeline分解为每个单节点下独立的jedisPipeline操作，最后合并response返回。具体实现就是通过JedisClusterCRC16.getSlot(key)计算key的slot值，通过每个节点的slot分布，就知道了哪些key应该在哪些节点上。再获取这个节点的JedisPool就可以使用pipeline进行读写了。

上面提到的过程，其实在JedisClusterInfoCache对象中都已经帮助开发人员实现了，但是这个对象在JedisClusterConnectionHandler中为protected并没有对外开放，而且通过JedisCluster的API也无法拿到JedisClusterConnectionHandler对象。所以通过下面两个类将这些对象暴露出来，这样使用getJedisPoolFromSlot就可以知道每个key对应的JedisPool了。

```java
class JedisClusterPipeline extends JedisCluster {
        public JedisClusterPipeline(Set<HostAndPort> jedisClusterNode, int connectionTimeout, int soTimeout, int maxAttempts, String password, final GenericObjectPoolConfig poolConfig) {
            super(jedisClusterNode, connectionTimeout, soTimeout, maxAttempts, password, poolConfig);
            super.connectionHandler = new JedisSlotAdvancedConnectionHandler(jedisClusterNode, poolConfig,
                    connectionTimeout, soTimeout, password);
        }

        public JedisSlotAdvancedConnectionHandler getConnectionHandler() {
            return (JedisSlotAdvancedConnectionHandler) this.connectionHandler;
        }

        /**
         * 刷新集群信息，当集群信息发生变更时调用
         *
         * @param
         * @return
         */
        public void refreshCluster() {
            connectionHandler.renewSlotCache();
        }
    }

    class JedisSlotAdvancedConnectionHandler extends JedisSlotBasedConnectionHandler {

        public JedisSlotAdvancedConnectionHandler(Set<HostAndPort> nodes, GenericObjectPoolConfig poolConfig, int connectionTimeout, int soTimeout, String password) {
            super(nodes, poolConfig, connectionTimeout, soTimeout, password);
        }

        public JedisPool getJedisPoolFromSlot(int slot) {
            JedisPool connectionPool = cache.getSlotPool(slot);
            if (connectionPool != null) {
                // It can't guaranteed to get valid connection because of node
                // assignment
                return connectionPool;
            } else {
                renewSlotCache(); //It's abnormal situation for cluster mode, that we have just nothing for slot, try to rediscover state
                connectionPool = cache.getSlotPool(slot);
                if (connectionPool != null) {
                    return connectionPool;
                } else {
                    throw new JedisNoReachableClusterNodeException("No reachable node in cluster for slot " + slot);
                }
            }
        }
    }
```

逐条执行的测试方法

> 10000条测试。

```java
private void testJedisCluster(Set<HostAndPort> nodes, JedisPoolConfig config) {
    JedisCluster jc = new JedisCluster(nodes, 20000, 20000, 1,null, config);

    IntStream.range(0, 10000).parallel().forEach(i -> {
        jc.set("n_key_" + i, "n_value_" + i);
    });

}
```

pipeline测试方法

> 10000条测试。

```java
private void testJedisClusterPipeline(Set<HostAndPort> nodes, JedisPoolConfig config) {
    JedisClusterPipeline jedisClusterPipeline = new JedisClusterPipeline(nodes, 20000, 20000, 1, null, config);
    JedisSlotAdvancedConnectionHandler jedisSlotAdvancedConnectionHandler = jedisClusterPipeline.getConnectionHandler();

    Map<JedisPool, List<String>> poolKeys = new HashMap<>();

    jedisClusterPipeline.refreshCluster();

    IntStream.range(0, 10000).forEach(i -> {
        String key = "p_key_" + i;
        int slot = JedisClusterCRC16.getSlot(key);
        JedisPool jedisPool = jedisSlotAdvancedConnectionHandler.getJedisPoolFromSlot(slot);
        if (poolKeys.keySet().contains(jedisPool)) {
            List<String> keys = poolKeys.get(jedisPool);
            keys.add(key);
        } else {
            List<String> keys = new CopyOnWriteArrayList<>();
            keys.add(key);
            poolKeys.put(jedisPool, keys);
        }
    });


    poolKeys.forEach((jedisPool, strings) -> {
        Jedis jedis = jedisPool.getResource();

        Pipeline pipeline = jedis.pipelined();
				
      	//这里注意不能使用parallel()。因为pipeline.set是向流中写命令，非线程安全
        strings.forEach(key -> {
            pipeline.set(key, "value");
        });
        pipeline.sync();//同步提交
        pipeline.close();
    });

}
```

测试方法

> 使用spring的StopWatch记录两个任务的执行时间。

```java
@Test
public void testPipeline() {

    StopWatch watch = new StopWatch();

    Set<HostAndPort> nodes = new HashSet<>();
    nodes.add(new HostAndPort("192.168.8.100", 7001));
    nodes.add(new HostAndPort("192.168.8.100", 7002));
    nodes.add(new HostAndPort("192.168.8.100", 7003));
    nodes.add(new HostAndPort("192.168.8.100", 7004));
    nodes.add(new HostAndPort("192.168.8.100", 7005));
    nodes.add(new HostAndPort("192.168.8.100", 7006));

    JedisPoolConfig config = new JedisPoolConfig();
    config.setMaxIdle(0);
    config.setTestOnBorrow(true);
    config.setTestWhileIdle(true);

    watch.start("testJedisCluster");
    testJedisCluster(nodes, config);
    watch.stop();
  
    watch.start("testJedisClusterPipeline");
    testJedisClusterPipeline(nodes, config);
    watch.stop();

    System.out.println(watch.prettyPrint());
    for (StopWatch.TaskInfo taskInfo : watch.getTaskInfo()) {
        System.out.println(taskInfo.getTaskName()+"执行时间："+taskInfo.getTimeSeconds()+"s");
    }

}
```

执行结果：

```log
StopWatch '': running time = 7458599145 ns
---------------------------------------------
ns         %     Task name
---------------------------------------------
7348143992  099%  testJedisCluster
110455153  001%  testJedisClusterPipeline

testJedisCluster执行时间：7.348143992s
testJedisClusterPipeline执行时间：0.110455153s
```

***

## 应用缓存

> 手动的方式就不介绍了，这里使用spring-cache实现。

CacheConfig

> Spring-cache配置

```java
@Configuration
class CacheConfig extends CachingConfigurerSupport {

    @Bean
    public CacheManager cacheManager(RedisConnectionFactory factory, RedisSerializer<Object> redisSerializer) {
        RedisCacheManager cacheManager = RedisCacheManager.builder(factory)
          			//默认60秒超时时间
                .cacheDefaults(getRedisCacheConfigurationWithTtl(60, redisSerializer))
                .build();
        return cacheManager;
    }

    private RedisCacheConfiguration getRedisCacheConfigurationWithTtl(Integer minutes, RedisSerializer<Object> redisSerializer) {

        RedisCacheConfiguration redisCacheConfiguration = RedisCacheConfiguration.defaultCacheConfig();
        redisCacheConfiguration = redisCacheConfiguration
                .prefixKeysWith("data:test:") //设置数据key前缀
                .serializeValuesWith(
                        RedisSerializationContext.SerializationPair.fromSerializer(redisSerializer))
                .entryTtl(Duration.ofMinutes(minutes));

        return redisCacheConfiguration;
    }

    @Override
    public KeyGenerator keyGenerator() {
        // 当没有指定缓存的 key时来根据类名、方法名和方法参数来生成key
        return (target, method, params) -> {
            StringBuilder sb = new StringBuilder();
            sb.append(target.getClass().getName())
                    .append(':')
                    .append(method.getName());
            if (params.length > 0) {
                sb.append('[');
                for (Object obj : params) {
                    if (obj != null) {
                        sb.append(obj.toString());
                    }
                }
                sb.append(']');
            }
            return sb.toString();
        };
    }
}
```



Redis-cluster 配置

```java
@Configuration
@EnableCaching
public class RedisClusterConfig {


    @Bean
    public RedisClusterConfiguration getRedisCluster() {
        RedisClusterConfiguration redisClusterConfiguration = new RedisClusterConfiguration();
        Set<RedisNode> jedisClusterNodes = new HashSet<RedisNode>();
        String[] add = new String[]{
                "192.168.8.100:7001",
                "192.168.8.100:7002",
                "192.168.8.100:7003",
                "192.168.8.100:7004",
                "192.168.8.100:7005",
                "192.168.8.100:7006"
        };
        for (String temp : add) {
            String[] hostAndPort = temp.split(":");
            jedisClusterNodes.add(new RedisNode(hostAndPort[0], Integer.parseInt(hostAndPort[1])));
        }
        redisClusterConfiguration.setClusterNodes(jedisClusterNodes);
        return redisClusterConfiguration;
    }

    @Bean
    public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory, RedisSerializer<Object> redisSerializer) {

        RedisTemplate<Object, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory);
        template.setDefaultSerializer(redisSerializer);
        template.setValueSerializer(redisSerializer);
        template.setHashValueSerializer(redisSerializer);
        template.setKeySerializer(StringRedisSerializer.UTF_8);
        template.setHashKeySerializer(StringRedisSerializer.UTF_8);
        template.afterPropertiesSet();
        return template;
    }

    @Bean
    public RedisConnectionFactory redisConnectionFactory(RedisClusterConfiguration redisClusterConfiguration) {
        JedisConnectionFactory jedisConnectionFactory = new JedisConnectionFactory(redisClusterConfiguration);
        jedisConnectionFactory.afterPropertiesSet();
        return jedisConnectionFactory;
    }

    @Bean
    public RedisSerializer<Object> redisSerializer() {

        ObjectMapper objectMapper = new ObjectMapper();
        //反序列化时候遇到不匹配的属性并不抛出异常
        objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
        //序列化时候遇到空对象不抛出异常
        objectMapper.configure(SerializationFeature.FAIL_ON_EMPTY_BEANS, false);
        //反序列化的时候如果是无效子类型,不抛出异常
        objectMapper.configure(DeserializationFeature.FAIL_ON_INVALID_SUBTYPE, false);
        //不使用默认的dateTime进行序列化,
        objectMapper.configure(SerializationFeature.WRITE_DATE_KEYS_AS_TIMESTAMPS, false);
        //使用JSR310提供的序列化类,里面包含了大量的JDK8时间序列化类
        objectMapper.registerModule(new JavaTimeModule());
        //启用反序列化所需的类型信息,在属性中添加@class
        objectMapper.activateDefaultTyping(LaissezFaireSubTypeValidator.instance, ObjectMapper.DefaultTyping.NON_FINAL, JsonTypeInfo.As.PROPERTY);
        //配置null值的序列化器
        GenericJackson2JsonRedisSerializer.registerNullValueSerializer(objectMapper, null);
        return new GenericJackson2JsonRedisSerializer(objectMapper);

    }

}
```

CacheService

```java
@Component
public class CacheService {

    public static String getSign() {
        return String.valueOf(LocalDateTime.now().toEpochSecond(ZoneOffset.of("+8")));
    }
		
  	//简单hello服务 使用秒级时间戳作为key
    @Cacheable(value = "test", key = "T(org.example.dubbo.provider.cache.CacheService).getSign()")
    public String hello() {
        log.info("hello method invoke...")
        return "hello cache";
    }
    
}
```

测试类

```java
@Slf4j
@SpringBootTest
public class TestCache {

    @Autowired
    CacheService cacheService;

    @SneakyThrows
    @Test
    public void cacheTest() {
      	//总共执行10次，一秒五次。
        for (int i = 0; i < 10; i++) {
            TimeUnit.MILLISECONDS.sleep(200);
            log.info(cacheService.hello());
        }
    }
}
```

执行结果

```log
15:33:16.462 [main] INFO  org.example.dubbo.provider.cache.CacheService - hello method invoke...
15:33:16.473 [main] INFO  org.example.dubbo.provider.TestCache - hello cache
15:33:16.703 [main] INFO  org.example.dubbo.provider.TestCache - hello cache
15:33:16.911 [main] INFO  org.example.dubbo.provider.TestCache - hello cache
15:33:17.120 [main] INFO  org.example.dubbo.provider.cache.CacheService - hello method invoke...
15:33:17.122 [main] INFO  org.example.dubbo.provider.TestCache - hello cache
15:33:17.326 [main] INFO  org.example.dubbo.provider.TestCache - hello cache
15:33:17.531 [main] INFO  org.example.dubbo.provider.TestCache - hello cache
15:33:17.737 [main] INFO  org.example.dubbo.provider.TestCache - hello cache
15:33:17.943 [main] INFO  org.example.dubbo.provider.TestCache - hello cache
15:33:18.148 [main] INFO  org.example.dubbo.provider.cache.CacheService - hello method invoke...
15:33:18.150 [main] INFO  org.example.dubbo.provider.TestCache - hello cache
15:33:18.356 [main] INFO  org.example.dubbo.provider.TestCache - hello cache
```

### @Cacheable

> @Cacheable可以标记在一个方法上，也可以标记在一个类上。当标记在一个方法上时表示该方法是支持缓存的，当标记在一个类上时则表示该类所有的方法都是支持缓存的。

| 参数      | 解释                                                         | example                                                      |
| :-------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| value     | 缓存的名称，在 spring 配置文件中定义，必须指定至少一个       | 例如: <br />@Cacheable(value=”mycache”) <br />@Cacheable(value={”cache1”,”cache2”} |
| key       | 缓存的 key，可以为空，如果指定要按照 SpEL 表达式编写，如果不指定，则缺省使用key生成器。 | @Cacheable(value=”testcache”,key=”#userName”)                |
| condition | 缓存的条件，可以为空，使用 SpEL 编写，返回 true 或者 false，只有为 true 才进行缓存 | @Cacheable(value=”testcache”,condition=”#userName.length()>2”) |

内置属性

> 可以将“#root”省略

| 属性名称    | 描述                        | 示例                 |
| ----------- | --------------------------- | -------------------- |
| methodName  | 当前方法名                  | #root.methodName     |
| method      | 当前方法                    | #root.method.name    |
| target      | 当前被调用的对象            | #root.target         |
| targetClass | 当前被调用的对象的class     | #root.targetClass    |
| args        | 当前方法参数组成的数组      | #root.args[0]        |
| caches      | 当前被调用的方法使用的Cache | #root.caches[0].name |

### 

### @CachePut

> 使用@CachePut标注的方法在执行前不会去检查缓存中是否存在之前执行过的结果，而是每次都会执行该方法，并将执行结果以键值对的形式存入指定的缓存中。

### @CacheEvict

>  @CacheEvict是用来标注在需要清除缓存元素的方法或类上的。当标记在一个类上时表示其中所有的方法的执行都会触发缓存的清除操作。

@CacheEvict可以指定的属性有value、key、condition、allEntries和beforeInvocation。

其中value、key和condition的语义与@Cacheable对应的属性类似。即value表示清除操作是发生在哪些Cache上的（对应Cache的名称）；key表示需要清除的是哪个key，如未指定则会使用默认策略生成的key；condition表示清除操作发生的条件。下面我们来介绍一下新出现的两个属性allEntries和beforeInvocation。

##### allEntries属性

   allEntries是boolean类型，表示是否需要清除缓存中的所有元素。默认为false，表示不需要。当指定了allEntries为true时，Spring Cache将忽略指定的key。有的时候我们需要Cache一下清除所有的元素，这比一个一个清除元素更有效率。

```java
@CacheEvict(value="users", allEntries=true)
   public void delete(Integer id) {
      System.out.println("delete user by id: " + id);
   }
```

##### beforeInvocation属性

​    清除操作默认是在对应方法成功执行之后触发的，即方法如果因为抛出异常而未能成功返回时也不会触发清除操作。使用beforeInvocation可以改变触发清除操作的时间，当我们指定该属性值为true时，Spring会在调用该方法之前清除缓存中的指定元素。

```java
@CacheEvict(value="users", beforeInvocation=true)
   public void delete(Integer id) {
      System.out.println("delete user by id: " + id);
   }
```



### @Caching

> @Caching注解可以让我们在一个方法或者类上同时指定多个Spring Cache相关的注解。其拥有三个属性：cacheable、put和evict，分别用于指定@Cacheable、@CachePut和@CacheEvict。

```java
@Caching(cacheable = @Cacheable("users"), evict = { @CacheEvict("cache2"),
   @CacheEvict(value = "cache3", allEntries = true) })
   public User find(Integer id) {
      returnnull;
   }
```



### 缓存击穿

> 缓存穿透，是指查询一个数据库一定不存在的数据。正常的使用缓存流程大致是，数据查询先进行缓存查询，如果key不存在或者key已经过期，再对数据库进行查询，并把查询到的对象，放进缓存。如果数据库查询对象为空，则不放进缓存。

**采用缓存空值的方式**，如果从数据库查询的对象为空，也放入缓存，只是设定的缓存过期时间较短，比如设置为60秒。

### 缓存穿透

> 缓存雪崩，是指在某一个时间段，缓存集中过期失效.解决方式就是上面设置过期时间中使用的方式，**灵活设置过期时间。**

### 缓存雪崩

> 缓存击穿，是指一个key非常热点，在不停的扛着大并发，大并发集中对这一个点进行访问，当这个key在失效的瞬间，持续的大并发就穿破缓存，直接请求数据库，就像在一个屏障上凿开了一个洞。解决方式**直接设置为永久key就可以了**。**mutex key互斥锁**可以学习下，但一般情况下用不上！



***

## 订阅和发布

> Redis 发布订阅(pub/sub)是一种消息通信模式：
> 发送者(pub)发送消息
> 订阅者(sub)接收消息
> Redis 客户端可以订阅任意数量的频道
>
> **Redis发布订阅的问题：**
>
> **不落盘，有监听就发送，没有监听就丢弃消息。**

两个订阅者

```shell
# 订阅1 topic = test_channel
127.0.0.1:7005> subscribe test_channel
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "test_channel"
3) (integer) 1
1) "message"
#接收到 test_channel的hello消息
2) "test_channel"
3) "hello"
```

```shell
# 订阅2 topic = test_channel,test_channel_2
127.0.0.1:7004> subscribe test_channel test_channel_2
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "test_channel"
3) (integer) 1
1) "message"
# 接收到 test_channel的hello消息
2) "test_channel"
3) "hello"
```

一个发布者

```shell
# 向test_channel 发送hello消息
127.0.0.1:7006> publish test_channel hello
(integer) 1
```

> 这里的topic可以是个正则表达式，监听使用**psubscribe <表达式>** 命令。

### spring-boot 实现

增加redis消息监听组件

消息接收器

```java
@Slf4j
@Component
public class RedisReceiver implements MessageListener {
		
  	//进行简单的消息接收，打印
    @Override
    public void onMessage(Message message, byte[] pattern) {
        log.info("接收到数据...");
        log.info(new String(pattern));
        log.info(new String(message.getBody()));
        log.info(new String(message.getChannel()));
    }

}
```

消息配置

```java
@Configuration
public class RedisMessageConfig {

    @Bean
    public RedisMessageListenerContainer redisMessageListenerContainer(RedisConnectionFactory redisConnectionFactory, MessageListenerAdapter messageListenerAdapter) {
        RedisMessageListenerContainer redisMessageListenerContainer = new RedisMessageListenerContainer();
        redisMessageListenerContainer.setConnectionFactory(redisConnectionFactory);

        //可以绑定多个 messageListenerAdapter,配置不同的topic
        redisMessageListenerContainer.addMessageListener(messageListenerAdapter, new PatternTopic("test_channel"));
        return redisMessageListenerContainer;
    }

  	//第二个参数指定反射调用的方法名。
    @Bean
    public MessageListenerAdapter messageListenerAdapter(RedisReceiver redisReceiver) {
        return new MessageListenerAdapter(redisReceiver, "onMessage");
    }

}
```



测试方法

```java
    @Test
    public void testTopic(){
        redisTemplate.convertAndSend("test_channel","hello");
    }
```

结果输出

```java
14:56:49.048 [redisMessageListenerContainer-2] INFO  org.example.dubbo.provider.cache.RedisReceiver - test_channel
14:56:49.048 [redisMessageListenerContainer-2] INFO  org.example.dubbo.provider.cache.RedisReceiver - hello
14:56:49.048 [redisMessageListenerContainer-2] INFO  org.example.dubbo.provider.cache.RedisReceiver - test_channel
```

***

## 布隆过滤器

> 布隆过滤器是用来判断一个元素是否出现在给定集合中的重要工具，具有快速，比哈希表更节省空间等优点，而缺点在于有一定的误识别率（false-positive，假阳性），亦即，它可能会把不是集合内的元素判定为存在于集合内，不过这样的概率相当小，在大部分的生产环境中是可以接受的；
>
> [官网Quick Start](https://oss.redislabs.com/redisbloom/)
>
> 其原理比较简单，如下图所示，S集合中有n个元素，利用k个哈希函数，将S中的每个元素映射到一个长度为m的位（bit）数组B中不同的位置上，这些位置上的二进制数均置为1，如果待检测的元素经过这k个哈希函数的映射后，发现其k个位置上的二进制数不全是1，那么这个元素一定不在集合S中，反之，该元素可能是S中的某一个元素
>
> ![img](https://gitee.com/wangigor/typora-images/raw/master/redis-bloom.jpg)
>
> [在线bloom过滤计算器](https://hur.st/bloomfilter/?n=100000000&p=1.0E-7&m=&k=)
>
> 计算公式：
>
> - 【预计存储数量】n = ceil(m / (-k / log(1 - exp(log(p) / k))))
> - 【错误率】p = pow(1 - exp(-k / (m / n)), k)
> - 【bitmap占用空间】m = ceil((n * log(p)) / log(1 / pow(2, log(2))));
> - 【hash函数数量】k = round((m / n) * log(2));

### Redis bloom 模块

```shell
git clone https://github.com/RedisBloom/RedisBloom.git
cd redisbloom
make
```

文件夹下产生一个redisbloom.so

> 注意：编译环境默认使用本机环境，**windows、mac、linux各不相同**，不能跨平台使用。

配置近redis中有两种方式

- 一个是redis启动命令增加模块 

```shell
redis-server --loadmodule /data/redisbloom.so /etc/redis/redis.conf
```



- 一个是redis配置模块增加模块配置

```properties
loadmodule /data/redisbloom.so
```

### redis client 使用

```shell
# 添加 bf.add <key> <value>
127.0.0.1:7001> bf.add test_items item1
(integer) 1

# 批量添加 bf.madd <key> <value1 value2 ... valueN>
192.168.8.100:7006> bf.madd test_items a b c
-> Redirected to slot [3881] located at 192.168.8.100:7004
1) (integer) 1
2) (integer) 1
3) (integer) 1

# 检查是否存在 bf.exists <key> <value>
192.168.8.100:7004> bf.exists test_items a
(integer) 1
192.168.8.100:7004> bf.exists test_items d
(integer) 0

# 批量检查是否存在 bf.exists <key> <value1 value2 ... valueN>
192.168.8.100:7004> bf.mexists test_items b c d
1) (integer) 1
2) (integer) 1
3) (integer) 0

# 自定义布隆过滤器属性 bf.reserve <key> <错误率> <预计总数>
# 如果key存在，就会报错。
192.168.8.100:7004> bf.reserve test_key 0.00001 10000
-> Redirected to slot [15118] located at 192.168.8.100:7006
(error) ERR item exists
192.168.8.100:7006> bf.reserve test_bf 0.00001 10000
OK

```



### spring-boot集成

> redis配置不变。只增加两个组件即刻。

BloomFilter组件

```java
@Component
public class BloomFilterComponent {

    @Autowired
    private RedisTemplate redisTemplate;

    public <T> void add(BloomFilterConfig bloomFilterConfig, String key, T value) {

        Assert.notNull(bloomFilterConfig, "bloomFilterConfig不能为空");
      
      	//通过murmurHashOffset计算多个位置坐标
        int[] offset = bloomFilterConfig.murmurHashOffset(value);
        for (int i : offset) {
            redisTemplate.opsForValue().setBit(key, i, true);
        }

    }

    public <T> boolean exists(BloomFilterConfig bloomFilterConfig, String key, T value) {
      
        Assert.notNull(bloomFilterConfig, "bloomFilterConfig不能为空");
      
      	//通过murmurHashOffset计算多个位置坐标
        int[] offset = bloomFilterConfig.murmurHashOffset(value);
        for (int i : offset) {
            if (!redisTemplate.opsForValue().getBit(key, i)) {
                return false;
            }
        }
        return true;
    }
}
```

BloomFilter配置

> 目前是原型类，使用的时候实例化。

```java
public class BloomFilterConfig<T> {

    private int num_HashFunctions;
    private int size_bitMap;
    private Funnel<T> funnel;

    public BloomFilterConfig(Funnel<T> funnel, int expectedInsertions, double fpp) {
        Assert.notNull(funnel, "funnel is null!");
        this.funnel = funnel;
        size_bitMap = calculateSizeofBitmap(expectedInsertions, fpp);
        num_HashFunctions = calculateNumofHashFunction(expectedInsertions, size_bitMap);
    }

    private int calculateNumofHashFunction(long expectedInsertions, long size_bitMap) {
        return Math.max(1, (int) Math.round((double) size_bitMap / expectedInsertions * Math.log(2)));
    }

    private int calculateSizeofBitmap(long expectedInsertions, double fpp) {
        if (fpp == 0) {
            fpp = Double.MIN_VALUE;
        }
        return (int) (-expectedInsertions * Math.log(fpp) / (Math.log(2) * Math.log(2)));
    }


    public int[] murmurHashOffset(T value) {
        int[] offset = new int[num_HashFunctions];

        long hash64 = Hashing.murmur3_128().hashObject(value, funnel).asLong();
        int hash1 = (int) hash64;
        int hash2 = (int) (hash64 >>> 32);
        for (int i = 1; i <= num_HashFunctions; i++) {
            int nextHash = hash1 + i * hash2;
            if (nextHash < 0) {
                nextHash = ~nextHash;
            }
            offset[i - 1] = nextHash % size_bitMap;
        }

        return offset;


    }
}
```



测试类

```java
@Slf4j
@SpringBootTest
public class BloomFilterTest {

    @Autowired
    private BloomFilterComponent bloomFilterComponent;


    @Test
    public void test() {

      	//自定义配置
        BloomFilterConfig<String> stringBloomFilterConfig = new BloomFilterConfig<>(
                (Funnel<String>) (from, into) -> into.putString(from, Charsets.UTF_8),
                1000,
                0.001);
				
      	//塞值
        bloomFilterComponent.add(stringBloomFilterConfig,"test_springboot_bloom","a");
        bloomFilterComponent.add(stringBloomFilterConfig,"test_springboot_bloom","b");
        bloomFilterComponent.add(stringBloomFilterConfig,"test_springboot_bloom","c");

      	//检测存在
        boolean exists = bloomFilterComponent.exists(stringBloomFilterConfig, "test_springboot_bloom", "c");
        boolean exists1 = bloomFilterComponent.exists(stringBloomFilterConfig, "test_springboot_bloom", "d");
        boolean exists2 = bloomFilterComponent.exists(stringBloomFilterConfig, "test_springboot_bloom", "e");

        log.info(String.valueOf(exists));
        log.info(String.valueOf(exists1));
        log.info(String.valueOf(exists2));

    }

}
```

输出结果

```log
09:23:04.292 [main] INFO  org.example.dubbo.provider.org.example.dubbo.provider.BloomFilterTest - true
09:23:04.292 [main] INFO  org.example.dubbo.provider.org.example.dubbo.provider.BloomFilterTest - false
09:23:04.292 [main] INFO  org.example.dubbo.provider.org.example.dubbo.provider.BloomFilterTest - false
```





***

## 分布式锁

***

## IO多路复用

***

# 参考文档

[Redis实战（通俗易懂，超详细攻略） V2.0版本](https://blog.csdn.net/bernkafly/article/details/89553711)
[《redis学习》-- 缓存淘汰策略](https://blog.csdn.net/lizhi_java/article/details/68953179?locationNum=1&fps=1)
[官方文档](http://www.redis.cn/documentation.html)
[波波烤鸭redis专栏](https://dpb-bobokaoya-sm.blog.csdn.net/column/info/33752)
[Redis内部数据结构详解(6)——skiplist](http://zhangtielei.com/posts/blog-redis-skiplist.html)
[深入理解跳跃链表](https://zhuanlan.zhihu.com/p/91753863)