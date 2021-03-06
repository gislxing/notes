### Redis 学习笔记

#### 1. Linux 安装

```bash
$ wget http://download.redis.io/releases/redis-5.0.3.tar.gz
$ tar xzf redis-5.0.3.tar.gz
$ cd redis-5.0.3
$ make MALLOC=libc

# 启动redis, 进入redis目录
$ ./src/redis-server

# 进入 redis 客户端，所有命令都只能在客户端对 redis 操作, 进入redis目录
$ ./src/redis-cli
```

注意：

  		1. 报错 `/bin/sh: 1: cc: not found`则需要安装 `gcc`  : `sudo apt install gcc`

#### 2. 总体概念

       1、每个命令都相对应于一种特定的数据结构。例如，当你使用set命令，你就是将值存储到一个字符串数据结构里
    
    3、存储器和持久化
        默认情况下，如果1000个或更多的关键字已变更，Redis会每隔60秒存储数据库；而如果9个或更少的关键字已变更，Redis会每隔15分钟存储数据库
    4、数据类型及常用操作
        1、strings类型及操作
            1、set
                设置key对应的值为string类型的value
                    set name xx
            2、setnx
                设置key对应的值为string类型的value。如果key已经存在，返回0，nx是not exist的意思
            3、setex
                设置key对应的值为string类型的value，并指定此键值对应的有效期
                    set name 10 xx
            4、setrange
                设置指定key的value值的子字符串（即从指定的下标开始替换值），例如：
                    127.0.0.1:6379> set email 123@163.com
                    127.0.0.1:6379> setrange email 4 qq.com
                    127.0.0.1:6379> get email
                    "123@qq.comm"
                    说明：其中的 4 是指从下标为 4（包含4）的字符开始替换（下标从0开始mg）
            5、mset
                一次设置多个key的值，成功返回ok表示所有的值都设置了，失败返回0表示没有任何值被设置
                mset key1 value1 key2 value2
            6、msetnx
                一次设置多个key的值，成功返回ok表示所有的值都设置了，失败返回0表示没有任何值被设置(只要其中一个失败则所有的操作都失败)，但是不会覆盖已经存在的key
            7、get
                获取key对应的string值,如果key不存在返回nil
            8、getset
                设置key的值，并返回key的旧值
            9、getrange
                获取指定key的value值的子字符串
            10、mget
                一次获取多个key的值，如果对应key不存在，则对应返回nil
            11、incr
                对key的值做加加操作,并返回新的值。注意：incr 一个不是int的value会返回错误，incr一不存在的key，则设置key为1
            12、incrby
                同incr类似，加指定值，key不存在时候会设置key，并认为原来的value是0
            11、decr
                对key的值做减减，decr一个不存在key，则设置key为-1
            12、decrby
                同decr，减指定值
            13、append
                给指定key的字符串值追加value,返回新字符串值的长度
            14、strlen
                取指定key的value值的长度
        2、hashes类型及操作
            1、hset
                设置hash field为指定值，如果key不存在，则先创建
            2、hsetnx
                设置hash field为指定值，如果key不存在，则先创建。如果field已经存在，返回0，nx是not exist的意思
            3、hmset
                同时设置hash的多个field
            4、hget
                获取指定的hash field
            5、hmget
                获取全部指定的hash filed
            6、hincrby
                指定的hash filed 加上给定值
            7、hexists
                测试指定field是否存在
            8、hlen
                返回指定hash的field数量
            9、hdel
                删除指定hash的field
            10、hkeys
                返回hash的所有field
            11、hvals
                返回hash的所有value
            12、hgetall
                获取某个hash中全部的filed及value
        3、lists类型及操作
            list是一个链表结构，Redis的list类型其实就是一个每个子元素都是string类型的双向链表。链表的最大长度是(2的32次方)
            1、lpush
                在key对应list的头部添加字符串元素
            2、rpush
                在key对应list的尾部添加字符串元素
            3、linsert
                在key对应list的特定位置之前或之后添加字符串元素
            4、lset
                设置list中指定下标的元素值(下标从0开始)
            5、lrem
                从key对应list中删除count个和value相同的元素（count>0 时，按从头到尾的顺序删除；count<0 时，按从尾到头的顺序删除；count=0 时，删除全部）
            6、ltrim
                保留指定 key 的值范围内的数据
            7、lpop
                从list的头部删除元素，并返回删除元素
            8、rpop
                从list的尾部删除元素，并返回删除元素
            9、rpoplpush
                从第一个list的尾部移除元素并添加到第二个list的头部,最后返回被移除的元素值，整个操作是原子的.如果第一个list是空或者不存在返回nil
            10、lindex
                返回名称为key的list中index位置的元素
            11、llen
                返回key对应list的长度
        4、sets类型及操作
            和 Java 中 set 集合的概念差不多，是不重复的 string 集合
            1、sadd
                向名称为key的set中添加元素
            2、srem
                删除名称为key的set中的元素member
            3、spop
                随机返回并删除名称为key的set中一个元素
            4、sdiff
                第一个 key 减去 第二个 key 的差集
            5、sdiffstore
                返回所有给定key与第一个key的差集，并将结果存到第一个key中
            6、sinter
                返回所有给定key的交集
            7、sinterstore
                返回所有给定key的交集，并将结果存为另一个key
            8、sunion
                返回所有给定key的并集
            9、sunionstore
                返回所有给定key的并集，并将结果存为另一个key
            10、smove
                从第一个key对应的set中移除member并添加到第二个对应set中
            11、scard
                返回名称为key的set的元素个数
            12、sismember
                测试member是否是名称为key的set的元素
            13、srandmember
                随机返回名称为key的set的一个元素，但是不删除元素
        5、sorted sets 类型及操作
            sorted set是set的一个升级版本，它在set的基础上增加了一个顺序属性，这一属性在添加修改元素的时候可以指定，每次指定后，zset会自动重新按新的值调整顺序
            1、zadd
                向名称为key的zset中添加元素member，score用于排序。如果该元素已经存在，则返回0，添加失败
            2、zrem
                删除名称为key的zset中的元素member
            3、zincrby
                如果在名称为key的zset中已经存在元素member，则该元素的score增加increment；否则向集合中添加该元素，其score的值为increment
            4、zrank
                返回名称为key的zset中member元素的排名(按score从小到大排序)即下标
            5、zrevrank
                返回名称为key的zset中member元素的排名(按score从大到小排序)即下标
            6、zrevrange
                返回名称为key的zset（按score从大到小排序）中的index从start到end的所有元素
            7、zrangebyscore
                返回集合中score在给定区间的元素
            8、zcount
                返回集合中score在给定区间的数量
            9、zcard
                返回集合中元素个数
            10、zscore
                返回给定元素对应的score
            11、zremrangebyrank
                删除集合中排名在给定区间的元素
            12、zremrangebyscore
                删除集合中score在给定区间的元素
    3、Redis 常用命令
        1、键值相关命令
            1、keys
                返回满足给定pattern的所有key，例如返回所有的 key：
                    keys *
            2、exists
                确认一个key是否存在
            3、del
                删除一个key
            4、expire
                设置一个key的过期时间(单位:秒)
            5、move
                将当前数据库中的key转移到其它 redis 数据库中
            6、persist
                移除给定key的过期时间
            7、randomkey
                随机返回key空间的一个key
            8、rename
                重命名key
            9、type
                返回值的类型
        2、服务器相关命令
            1、ping
                测试连接是否存活
            2、echo
                在命令行打印一些内容
            3、select
                选择数据库。Redis数据库编号从0~15，我们可以选择任意一个数据库来进行数据的存取
            4、quit
                退出连接
            5、dbsize
                返回当前数据库中key的数目
            6、info
                获取服务器的信息和统计
            7、monitor
                实时转储收到的请求
            8、config get
                获取服务器配置信息
            9、flushdb
                删除当前选择数据库中的所有key
            10、flushall
                删除所有数据库中的所有key