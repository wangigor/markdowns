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



## 持久化

## 主从复制和集群

## 哨兵机制

## 事务

## 缓存淘汰策略

## 管道

## 应用缓存

## RedisTemplate

## Jedis

## 分布式锁

## IO多路复用













# 参考文档

[Redis实战（通俗易懂，超详细攻略） V2.0版本](https://blog.csdn.net/bernkafly/article/details/89553711)
[《redis学习》-- 缓存淘汰策略](https://blog.csdn.net/lizhi_java/article/details/68953179?locationNum=1&fps=1)
[官方文档](http://www.redis.cn/documentation.html)
[波波烤鸭redis专栏](https://dpb-bobokaoya-sm.blog.csdn.net/column/info/33752)
[Redis内部数据结构详解(6)——skiplist](http://zhangtielei.com/posts/blog-redis-skiplist.html)
[深入理解跳跃链表](https://zhuanlan.zhihu.com/p/91753863)


