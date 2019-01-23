### 第1章 文档目的
本文档的目的是通过对中移在线容器平台集群redis主从集群提供解决部署方案，以保障容器云平台能更好的提供服务,通过对redis的配置参数做优化调整，更好利用redis参数的调整使其更好的配合redis集群工作，redis参数如何更好的优化redis集群正常运行。
本文档的长期目标是不断优化、总结中移在线在容器云平台redis主从集群，出现的各种问题深层次原因，在后续的解决方案中避免问题的再次出现。
第2章Redis配置参数优化汇总

#### 1 databases 参数说明

Databases：redis 对于不同的库（database）没有提供任何隔离机制，完全依赖于应用（app）部署时约定使用不同的库（database）
不要将 redis 和 mysql 混为一谈，通过对比mysql,更好理解redis的databases:

	mysql 的实体分层由上至下依次是：	redis 的实体分层由上至下依次是：

    库（database）	                实例（instance）redis 进程
    表（table）						库（database）
    记录（row）						键值（Key-Value）
    字段（field）	

- 1.1 redis 的实例（instance） 等同于 mysql 的库（database）
- 1.2 redis 的库（database） 等同于 mysql 的表（table）
- 1.3 redis使用不同的库（database）可以从而避免键命名冲突

#### 2  dbfilename参数优化说明  

redis有两种持久化方式，Rdb 和 Aof,本地数据库文件名，默认值为dump.rdb,这时dump.rdb存放位置是不固定的，而是存放在启动redis时的当前目录,该文件是一个二进制文件，无法直接正常打开。

    // 本地数据库文件名，默认值为dump.rdb  
    dbfilename dump6379.rdb 

##### 2.1 RDB方式
是指在指定的时间间隔内将内存中的数据集快照写入磁盘，实际的操作过程是fork一个子进程，先将数据集写入临时文件，写入成功后，再替换之前的文件，用二进制压缩存储（binlog）。存储的文件为：dum.rdb

##### 2.2 RDB的优缺点
优点:RDB是一个紧凑压缩的二进制文件，代表Redis在某一个时间点上的数据快照。非常适合用于备份，全量复制等场景。比如每6小时执行bgsave备份，并把RDB文件拷贝到远程机器或者文件系统中（如hdfs），用于灾难恢复。Redis加载RDB恢复数据远远快于AOF方式。
缺点:RDB方式数据没办法做到实时持久化/秒级持久化。因为bgsave每次运行都要执行fork操作创建子进程，属于重量级操作，频繁执行成本过高。
RDB文件使用特定二进制格式保存，Redis版本演进过程中有多个格式的RDB笨笨，存在老版本Redis服务无法兼容新版RDB格式的问题。针对RDB不适合实时持久化的问题，Redis提供了AOF持久化方式来解决。

##### 2.3 AOF(append only file)持久化
以独立日志的方式记录每次写命令，重启时再重新执行AOF文件中命令达到恢复数据的目的,AOF的主要作用是解决了数据持久化的实时性，目前已经是Redis持久化的主流方式。

##### 2.4 AOF(append only file)优缺点

优点:使用 AOF 持久化会让 Redis 变得非常耐久（much more durable）：你可以设置不同的 fsync 策略，比如无 fsync ，每秒钟一次 fsync ，或者每次执行写入命令时 fsync 。 AOF 的默认策略为每秒钟 fsync 一次，在这种配置下，Redis 仍然可以保持良好的性能，并且就算发生故障停机，也最多只会丢失一秒钟的数据（ fsync 会在后台线程执行，所以主线程可以继续努力地处理命令请求）。


缺点:对于相同的数据集来说，AOF 文件的体积通常要大于 RDB 文件的体积。根据所使用的 fsync 策略，AOF 的速度可能会慢于 RDB 。 在一般情况下，每秒 fsync 的性能依然非常高， 而关闭 fsync 可以让 AOF 的速度和 RDB 一样快，即使在高负荷之下也是如此。

#### 3  redis日志存放位置参数调优

    loglevel notice 	日志记录等级，有4个可选值，debug，verbose，notice，warning  
    logfile ""  	指定日志输出的文件名，可设为/dev/null屏蔽日志  

#### 4  REPLICATION Redis的复制配置之masterauth配置解析

    /************* REPLICATION Redis的复制配置 ************/   
      
    // 当本机为从服务时，设置主服务的IP及端口  
    // slaveof <masterip> <masterport>   
      
    // 当本机为从服务时，设置主服务的连接密码  
    // masterauth <master-password>
    
    5 slave-serve-stale-data的配置参数解析
    默认配置slave-serve-stale-data yes
    当从库同主机失去连接或者复制正在进行，从机库有两种运行方式  
    1) 如果slave-serve-stale-data设置为yes(默认设置)，从库会继续相应客户端的请求  
    2) 如果slave-serve-stale-data是指为no，除去INFO和SLAVOF命令之外的任何请求都会返回一个错误"SYNC with master in progress" 
      
    6 slave-read-only的配置参数优化
    作为从服务器，默认情况下是只读的（yes），可以修改成NO，用于写（不建议）。
    
    7 slave-priority的配置参数优化
    作用:如果master不能再正常工作，那么会在多个slave中，选择优先值最小的一个slave提升为master，优先值为0表示不能提升为master。
    此配置参数仅在哨兵模式下才有作用的，单一主从是无法进行选举算法的进行，鉴于部署平台为k8s，并且配置了探针，目前k8s平台的redis主从就已经是高可用集群，所以参数配置在主从集群中并没有太大的作用。
    
    8 redis配置参数优化限制系统的资源使用
    
    /*************************** LIMITS 约束 ***************************/  
    // 最大可用内存 maxmemory <bytes> 536870912，即512M  
    maxmemory 536870912   
      
    // 当内存达到最大值的时候Redis会选择删除哪些数据？有五种方式可供选择  
    //  
    // volatile-lru -> 利用LRU算法移除设置过过期时间的key (LRU:最近使用 Least Recently Used )  
    // allkeys-lru -> 利用LRU算法移除任何key  
    // volatile-random -> 移除设置过过期时间的随机key  
    // allkeys->random -> remove a random key, any key  
    // volatile-ttl -> 移除即将过期的key(minor TTL)  
    // noeviction -> 不移除任何可以，只是返回一个写错误  
    maxmemory-policy allkeys-lru   
      
    // LRU 和 minimal TTL 算法都不是精准的算法，但是相对精确的算法(为了节省内存)，随意你可以选择样本大小进行检测。  
    // Redis默认的会选择3个样本进行检测，你可以通过maxmemory-samples进行设置  
    maxmemory-samples 3   
    /********************************* VM *********************************/  
      
    // 是否使用虚拟内存，默认值为no  
    vm-enabled yes  
      
    // 虚拟内存文件路径，默认值为/tmp/redis.swap，不可多个Redis实例共享  
    vm-swap-file /tmp/redis.swap  
      
    // 将所有大于vm-max-memory的数据存入虚拟内存,无论vm-max-memory设置多小,所有索引数据都是内存存储的  
    (Redis的索引数据就是keys),也就是说,当vm-max-memory设置为0的时候,其实是所有value都存在于磁盘。默认值为0。  
    vm-max-memory 0  
      
    // 虚拟内存文件以块存储，每块32bytes  
    vm-page-size 32  
      
    // 虚拟内在文件的最大数  
    vm-pages 134217728  
      
    // 可以设置访问swap文件的线程数,设置最好不要超过机器的核数,如果设置为0,那么所有对swap文件的操作都是串行的.  
    可能会造成比较长时间的延迟,但是对数据完整性有很好的保证.  
    vm-max-threads 4  


#### 9 redis日志参数解析与慢查询

###### 慢查询定义:慢查询是指执行时间超过慢查询时间的sql语句。


    /********************************* SLOW LOG *********************************/  
      
    // Redis Slow Log 记录超过特定执行时间的命令。执行时间不包括I/O计算比如连接客户端，返回结果等，只是命令执行时间  
    //  
    // 可以通过两个参数设置slow log：一个是告诉Redis执行超过多少时间被记录的参数slowlog-log-slower-than(微妙)，  
    // 另一个是slow log 的长度。当一个新命令被记录的时候最早的命令将被从队列中移除   
      
    // 下面的时间以微妙微单位，因此1000000代表一分钟。  
    // 注意制定一个负数将关闭慢日志，而设置为0将强制每个命令都会记录  
    slowlog-log-slower-than 10000   
      
    // 对日志长度没有限制，只是要注意它会消耗内存  
    // 可以通过 SLOWLOG RESET 回收被慢日志消耗的内存  
    slowlog-max-len 128   
      
    /*************************** LATENCY MONITOR ************************/   
      
    latency-monitor-threshold 0   
      
    /*************************** EVENT NOTIFICATION *********************/  
      
    notify-keyspace-events ""   
      
    /********************** ADVANCED CONFIG ***************************/   
      
    // 当hash中包含超过指定元素个数并且最大的元素没有超过临界时，  
    // hash将以一种特殊的编码方式（大大减少内存使用）来存储，这里可以设置这两个临界值  
    // Redis Hash对应Value内部实际就是一个HashMap，实际这里会有2种不同实现，  
    // 这个Hash的成员比较少时Redis为了节省内存会采用类似一维数组的方式来紧凑存储，  
    而不会采用真正的HashMap结构，对应的value redisObject的encoding为zipmap,   
      
    // 当成员数量增大时会自动转成真正的HashMap,此时encoding为ht。   
      
    hash-max-ziplist-entries 512  
    hash-max-ziplist-value 64   
      
    // list数据类型多少节点以下会采用去指针的紧凑存储格式。  
    // list数据类型节点值大小小于多少字节会采用紧凑存储格式。  
    list-max-ziplist-entries 512  
    list-max-ziplist-value 64   
      
    // set数据类型内部数据如果全部是数值型，且包含多少节点以下会采用紧凑格式存储。  
    set-max-intset-entries 512   
      
    // zsort数据类型多少节点以下会采用去指针的紧凑存储格式。  
    // zsort数据类型节点值大小小于多少字节会采用紧凑存储格式。  
    zset-max-ziplist-entries 128  
    zset-max-ziplist-value 64   
      
    hll-sparse-max-bytes 3000   
      
    // Redis将在每100毫秒时使用1毫秒的CPU时间来对redis的hash表进行重新hash，可以降低内存的使用  
    //  
    // 当你的使用场景中，有非常严格的实时性需要，不能够接受Redis时不时的对请求有2毫秒的延迟的话，把这项配置为no。  
    //  
    // 如果没有这么严格的实时性要求，可以设置为yes，以便能够尽可能快的释放内存  
    activerehashing yes   
      
    client-output-buffer-limit normal 0 0 0  
    client-output-buffer-limit slave 256mb 64mb 60  
    client-output-buffer-limit pubsub 32mb 8mb 60   
      
    hz 10   

#### 10 redis的aof文件持久化存储

      Redis 分别提供了 RDB 和 AOF 两种持久化机制：
      RDB 将数据库的快照（snapshot）以二进制的方式保存到磁盘中。

AOF 则以协议文本的方式，将所有对数据库进行过写入的命令（及其参数）记录到 AOF 文件，以此达到记录数据库状态的目的。


###### AOF持久化缺点：

	Redis会不断地将被执行的命令记录到AOF文件里面，所以随着Redis不断运行，AOF文件的体积也会不断增长。在极端情况下，体积不断增大的AOF文件甚至可能会用完硬盘的所有可用空间。

	Redis在重启之后需要通过重新执行AOF文件记录的所有写命令来还原数据集，所以如果AOF文件的体积非常大，那么还原操作执行的时间就可能会非常长。

    /****************** APPEND ONLY MODE *******************/   
      
    // 启用aof持久化方式  
    // 因为redis本身同步数据文件是按上面save条件来同步的，所以有的数据会在一段时间内只存在于内存中。默认值为no  
    appendonly yes   
      
    // 更新日志文件名，默认值为appendonly.aof  
    appendfilename "appendonly.aof"   
      
    // 收到写命令立即写入磁盘，最慢，保证完全的持久化  
    appendfsync always  
    // 每秒写入一次  
    appendfsync everysec  
    // 完全依赖OS，性能最好，持久化没保证  
    appendfsync no   
      
    // 部署在同一机器的redis实例，把auto-aof-rewrite打开，因为cluster环境下内存占用基本一致  
    #关闭在aof rewrite的时候对新的写操作进行fsync  
    no-appendfsync-on-rewrite yes   
      
    // Automatic rewrite of the append only file.  
    // AOF 自动重写  
    // 当AOF文件增长到一定大小的时候Redis能够调用 BGREWRITEAOF 对日志文件进行重写  
    //  
    // 它是这样工作的：Redis会记住上次进行些日志后文件的大小(如果从开机以来还没进行过重写，那日子大小在开机的时候确定)  
    //  
    // 基础大小会同现在的大小进行比较。如果现在的大小比基础大小大制定的百分比，重写功能将启动  
    // 同时需要指定一个最小大小用于AOF重写，这个用于阻止即使文件很小但是增长幅度很大也去重写AOF文件的情况  
    // 设置 percentage 为0就关闭这个特性  
    auto-aof-rewrite-percentage 100  
    auto-aof-rewrite-min-size 64mb   
      
    aof-load-truncated yes   



### 第3章 Redis配置参数参考示例

    /************************ GENERAL *********************/  
    // 是否作为守护进程运行  
    daemonize yes   
      
    // 如以后台进程运行，则需指定一个pid，默认为/var/run/redis.pid  
    pidfile /var/run/redis.pid   
      
    // Redis默认监听端口  
    port 6379   
      
    tcp-backlog 511   
      
    // 客户端闲置多少秒后，断开连接  
    timeout 0   
      
    tcp-keepalive 0   
      
    // 日志记录等级，有4个可选值，debug，verbose，notice，warning  
    loglevel notice   
      
    // 指定日志输出的文件名，可设为/dev/null屏蔽日志  
    logfile ""   
      
    // 可用数据库数，默认值为16，默认数据库为0  
    databases 16   
      
    /****************** SNAPSHOTTING 快照 *******************/  
    // 保存数据到disk的策略  
    // 900 秒有 1 条改变保存到disk  
    save 900 1  
    // 300 秒有 10 条改变保存到disk  
    save 300 10  
    // 60 秒有 10000 条改变保存到disk  
    save 60 10000   
      
    stop-writes-on-bgsave-error yes   
      
    // 当dump .rdb数据库的时候是否压缩数据对象  
    rdbcompression yes   
      
    rdbchecksum yes   
      
    // 本地数据库文件名，默认值为dump.rdb  
    dbfilename dump.rdb   
      
    // 本地数据库存放路径，默认值为 ./  
    dir ./   
      
    /**************** REPLICATION Redis的复制配置 *****************/   
      
    // 当本机为从服务时，设置主服务的IP及端口  
    // slaveof <masterip> <masterport>   
      
    // 当本机为从服务时，设置主服务的连接密码  
    // masterauth <master-password>   
      
    // 当从库同主机失去连接或者复制正在进行，从机库有两种运行方式  
    // 1) 如果slave-serve-stale-data设置为yes(默认设置)，从库会继续相应客户端的请求  
    // 2) 如果slave-serve-stale-data是指为no，出去INFO和SLAVOF命令之外的任何请求都会返回一个错误"SYNC with master in progress"  
    slave-serve-stale-data yes   
      
    slave-read-only yes   
      
    repl-diskless-sync no   
      
    repl-diskless-sync-delay 5   
      
    // 从库会按照一个时间间隔向主库发送PINGs.可以通过repl-ping-slave-period设置这个时间间隔，默认是10秒  
    repl-ping-slave-period 10   
      
    // repl-timeout 设置主库批量数据传输时间或者ping回复时间间隔，默认值是60秒  
    // 一定要确保repl-timeout大于repl-ping-slave-period  
    repl-timeout 60   
      
    // 采用无延迟同步 默认no  
    repl-disable-tcp-nodelay yes   
      
    slave-priority 100   
      
    /******************* SECURITY 安全 ********************/   
      
    // 设置客户端连接后进行任何其他指定前需要使用的密码。  
    // 警告：因为redis速度相当快，所以在一台比较好的服务器下，一个外部的用户可以在一秒钟进行150K次的密码尝试，  
    这意味着你需要指定非常非常强大的密码来防止暴力破解  
    // requirepass foobared   
      
    // 命令重命名.  
    // 在一个共享环境下可以重命名相对危险的命令。比如把CONFIG重名为一个不容易猜测的字符。  
    // 举例:  
    // rename-command CONFIG b840fc02d524045429941cc15f59e41cb7be6c52  
    // 如果想删除一个命令，直接把它重命名为一个空字符""即可，如下：  
    // rename-command CONFIG ""   
      
    /********************* LIMITS 约束 *********************/  
    // 最大可用内存 maxmemory <bytes> 536870912，即512M  
    maxmemory 536870912   
      
    // 当内存达到最大值的时候Redis会选择删除哪些数据？有五种方式可供选择  
    //  
    // volatile-lru -> 利用LRU算法移除设置过过期时间的key (LRU:最近使用 Least Recently Used )  
    // allkeys-lru -> 利用LRU算法移除任何key  
    // volatile-random -> 移除设置过过期时间的随机key  
    // allkeys->random -> remove a random key, any key  
    // volatile-ttl -> 移除即将过期的key(minor TTL)  
    // noeviction -> 不移除任何可以，只是返回一个写错误  
    maxmemory-policy allkeys-lru   
      
    // LRU 和 minimal TTL 算法都不是精准的算法，但是相对精确的算法(为了节省内存)，随意你可以选择样本大小进行检测。  
    // Redis默认的灰选择3个样本进行检测，你可以通过maxmemory-samples进行设置  
    maxmemory-samples 3   
      
    /******************** APPEND ONLY MODE *********************/   
      
    // 启用aof持久化方式  
    // 因为redis本身同步数据文件是按上面save条件来同步的，所以有的数据会在一段时间内只存在于内存中。默认值为no  
    appendonly yes   
      
    // 更新日志文件名，默认值为appendonly.aof  
    appendfilename "appendonly.aof"   
      
    // 收到写命令立即写入磁盘，最慢，保证完全的持久化  
    appendfsync always  
    // 每秒写入一次  
    appendfsync everysec  
    // 完全依赖OS，性能最好，持久化没保证  
    appendfsync no   
      
    // 部署在同一机器的redis实例，把auto-aof-rewrite打开，因为cluster环境下内存占用基本一致  
    #关闭在aof rewrite的时候对新的写操作进行fsync  
    no-appendfsync-on-rewrite yes   
      
    // Automatic rewrite of the append only file.  
    // AOF 自动重写  
    // 当AOF文件增长到一定大小的时候Redis能够调用 BGREWRITEAOF 对日志文件进行重写  
    //  
    // 它是这样工作的：Redis会记住上次进行些日志后文件的大小(如果从开机以来还没进行过重写，那日子大小在开机的时候确定)  
    //  
    // 基础大小会同现在的大小进行比较。如果现在的大小比基础大小大制定的百分比，重写功能将启动  
    // 同时需要指定一个最小大小用于AOF重写，这个用于阻止即使文件很小但是增长幅度很大也去重写AOF文件的情况  
    // 设置 percentage 为0就关闭这个特性  
    auto-aof-rewrite-percentage 100  
    auto-aof-rewrite-min-size 64mb   
      
    aof-load-truncated yes   
      
    /******************** LUA SCRIPTING ********************/  
    lua-time-limit 5000   
      
    /******************* REDIS CLUSTER 集群********************/  
    // 打开redis集群  
    cluster-enabled yes   
      
    // cluster配置文件(启动自动生成)  
    cluster-config-file nodes-6379.conf   
      
    // 节点互连超时的阀值  
    cluster-node-timeout 15000   
      
    cluster-slave-validity-factor 10   
      
    cluster-migration-barrier 1   
      
    // 集群兼容部分失败  
    cluster-require-full-coverage yes   
      
    /******************* SLOW LOG **********************/  
      
    // Redis Slow Log 记录超过特定执行时间的命令。执行时间不包括I/O计算比如连接客户端，返回结果等，只是命令执行时间  
    //  
    // 可以通过两个参数设置slow log：一个是告诉Redis执行超过多少时间被记录的参数slowlog-log-slower-than(微妙)，  
    // 另一个是slow log 的长度。当一个新命令被记录的时候最早的命令将被从队列中移除   
      
    // 下面的时间以微妙微单位，因此1000000代表一分钟。  
    // 注意制定一个负数将关闭慢日志，而设置为0将强制每个命令都会记录  
    slowlog-log-slower-than 10000   
      
    // 对日志长度没有限制，只是要注意它会消耗内存  
    // 可以通过 SLOWLOG RESET 回收被慢日志消耗的内存  
    slowlog-max-len 128   
      
    /******************* LATENCY MONITOR ********************/   
      
    latency-monitor-threshold 0   
      
    /********************* EVENT NOTIFICATION *********************/  
      
    notify-keyspace-events ""   
      
    /******************** ADVANCED CONFIG ******************/   
      
    // 当hash中包含超过指定元素个数并且最大的元素没有超过临界时，  
    // hash将以一种特殊的编码方式（大大减少内存使用）来存储，这里可以设置这两个临界值  
    // Redis Hash对应Value内部实际就是一个HashMap，实际这里会有2种不同实现，  
    // 这个Hash的成员比较少时Redis为了节省内存会采用类似一维数组的方式来紧凑存储，  
    而不会采用真正的HashMap结构，对应的value redisObject的encoding为zipmap,   
      
    // 当成员数量增大时会自动转成真正的HashMap,此时encoding为ht。   
      
    hash-max-ziplist-entries 512  
    hash-max-ziplist-value 64   
      
    // list数据类型多少节点以下会采用去指针的紧凑存储格式。  
    // list数据类型节点值大小小于多少字节会采用紧凑存储格式。  
    list-max-ziplist-entries 512  
    list-max-ziplist-value 64   
      
    // set数据类型内部数据如果全部是数值型，且包含多少节点以下会采用紧凑格式存储。  
    set-max-intset-entries 512   
      
    // zsort数据类型多少节点以下会采用去指针的紧凑存储格式。  
    // zsort数据类型节点值大小小于多少字节会采用紧凑存储格式。  
    zset-max-ziplist-entries 128  
    zset-max-ziplist-value 64   
      
    hll-sparse-max-bytes 3000   
      
    // Redis将在每100毫秒时使用1毫秒的CPU时间来对redis的hash表进行重新hash，可以降低内存的使用  
    //  
    // 当你的使用场景中，有非常严格的实时性需要，不能够接受Redis时不时的对请求有2毫秒的延迟的话，把这项配置为no。  
    //  
    // 如果没有这么严格的实时性要求，可以设置为yes，以便能够尽可能快的释放内存  
    activerehashing yes   
      
    client-output-buffer-limit normal 0 0 0  
    client-output-buffer-limit slave 256mb 64mb 60  
    client-output-buffer-limit pubsub 32mb 8mb 60   
      
    hz 10   
      
    aof-rewrite-incremental-fsync yes   
      
    /******************* VM ***********************/  
      
    // 是否使用虚拟内存，默认值为no  
    vm-enabled yes  
      
    // 虚拟内存文件路径，默认值为/tmp/redis.swap，不可多个Redis实例共享  
    vm-swap-file /tmp/redis.swap  
      
    // 将所有大于vm-max-memory的数据存入虚拟内存,无论vm-max-memory设置多小,所有索引数据都是内存存储的  
    (Redis的索引数据就是keys),也就是说,当vm-max-memory设置为0的时候,其实是所有value都存在于磁盘。默认值为0。  
    vm-max-memory 0  
      
    // 虚拟内存文件以块存储，每块32bytes  
    vm-page-size 32  
      
    // 虚拟内在文件的最大数  
    vm-pages 134217728  
      
    // 可以设置访问swap文件的线程数,设置最好不要超过机器的核数,如果设置为0,那么所有对swap文件的操作都是串行的.  
    可能会造成比较长时间的延迟,但是对数据完整性有很好的保证.  
    vm-max-threads 4  
      
    /******************** INCLUDES ********************/  
    // 包含通用配置  
    include /etc/redis/redis-common.conf   
      
    /******************** GENERAL *********************/  
    // 如以后台进程运行，则需指定一个pid，默认为/var/run/redis.pid  
    pidfile /var/run/redis_6379.pid   
      
    // Redis默认监听端口  
    port 6379   
      
    // 指定日志输出的文件名，可设为/dev/null屏蔽日志  
    logfile /var/log/redis_6379.log   
      
    /******************* SNAPSHOTTING 快照 ********************/   
      
    // 本地数据库文件名，默认值为dump.rdb  
    dbfilename dump6379.rdb   
      
    // 本地数据库存放路径，默认值为 ./  
    dir /var/redis/6379   
      
    /**************** REPLICATION Redis的复制配置 *****************/   
      
    // 当本机为从服务时，设置主服务的IP及端口  
    // slaveof <masterip> <masterport>   
      
    // 当本机为从服务时，设置主服务的连接密码  
    // masterauth <master-password>   
      
    /******************* APPEND ONLY MODE ********************/   
      
    // 更新日志文件名，默认值为appendonly.aof  
    appendfilename "appendonly6379.aof"   
      
    /********************* REDIS CLUSTER 集群 **********************/  
      
    // cluster配置文件(启动自动生成)  
    cluster-config-file nodes-6379.conf  
    