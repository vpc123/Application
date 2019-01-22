### 第1章 文档目的

本文档的目的是通过对中移在线容器平台集群redis主从集群提供解决部署方案，以保障容器云平台能更好的提供服务,通过对redis的配置参数做解析更深层的了解redis参数如何更好的优化redis集群正常运行。
本文档的长期目标是不断优化、总结中移在线在容器云平台redis主从集群，出现的各种问题深层次原因，在后续的解决方案中避免问题的再次出现。

### 第2章Redis配置参数汇总

    基本配置参数
    daemonize	no 	是否以后台进程启动
    databases 	16 	创建database的数量(默认选中的是database 0)
    save 	900 1	刷新快照到硬盘中，必须满足两者要求才会触发，即900秒之后至少1个关键字发生变化。
    save 	300 10	必须是300秒之后至少10个关键字发生变化。
    save 	60 10000	必须是60秒之后至少10000个关键字发生变化。
    stop-writes-on-bgsave-error 	yes 	后台存储错误停止写。
    rdbcompression 	yes	使用LZF压缩rdb文件。
    rdbchecksum 	yes	存储和加载rdb文件时校验。
    dbfilename 	dump.rdb 	设置rdb文件名。
    dir 	./ 	设置工作目录，rdb文件会写入该目录。
    主从配置
    slaveof 	<masterip> <masterport> 	设为某台机器的从服务器
    masterauth 	<master-password> 	连接主服务器的密码
    slave-serve-stale-data 	yes 	当主从断开或正在复制中,从服务器是否应答
    slave-read-only 	yes 	从服务器只读
    repl-ping-slave-period 	10 	从ping主的时间间隔,秒为单位
    repl-timeout 	60 	主从超时时间(超时认为断线了),要比period大
    slave-priority 	100 	如果master不能再正常工作，那么会在多个slave中，选择优先值最小的一个slave提升为master，优先值为0表示不能提升为master。
    repl-disable-tcp-nodelay 	no	主端是否合并数据,大块发送给slave
    slave-priority 	100 	从服务器的优先级,当主服挂了,会自动挑slave priority最小的为主服
    安全
    requirepass 	foobared 	需要密码
    rename-command 	CONFIG b840fc02d524045429941cc15f59e41cb7be6c52	如果公共环境,可以重命名部分敏感命令 如config
    限制#解释 LRU ttl都是近似算法,可以选N个,再比较最适宜T踢出的数据
    maxclients 	10000 	最大连接数
    maxmemory 	<bytes> 	最大使用内存
    maxmemory-policy 	volatile-lru	内存到极限后的处理
    volatile-lru 	-> 	LRU算法删除过期key
    allkeys-lru 	-> 	LRU算法删除key(不区分过不过期)
    volatile-random  	->	随机删除过期key
    allkeys-random 	-> 	随机删除key(不区分过不过期)
    volatile-ttl	-> 	删除快过期的key
    noeviction 	->	 不删除,返回错误信息
    maxmemory-samples 	3	
    日志模式
    appendonly 	no 	是否仅要日志
    appendfsync 	no 	系统缓冲,统一写,速度快
    appendfsync	always 	系统不缓冲,直接写,慢,丢失数据少
    appendfsync 	everysec 	折衷,每秒写1次
    no-appendfsync-on-rewrite 	no 	为yes,则其他线程的数据放内存里,合并写入(速度快,容易丢失的多)
    auto-AOF-rewrite-percentage 	100 	当前aof文件是上次重写是大N%时重写
    auto-AOF-rewrite-min-size 	64mb aof	重写至少要达到的大小
    慢查询
    slowlog-log-slower-than 	10000 	记录响应时间大于10000微秒的慢查询
    slowlog-max-len 	128 	最多记录128条
    服务端命令
    time 	返回时间戳+微秒
    dbsize 	返回key的数量
    bgrewriteaof 	重写aof
    bgsave 	后台开启子进程dump数据
    save 	阻塞进程dump数据
    lastsave	
    slaveof host port 	做host port的从服务器(数据清空,复制新主内容)
    slaveof no one 	变成主服务器(原数据不丢失,一般用于主服失败后)
    flushdb 	清空当前数据库的所有数据
    flushall 	清空所有数据库的所有数据(误用了怎么办?)
    shutdown [save/nosave] 	关闭服务器,保存数据,修改AOF(如果设置)
    slowlog get 	获取慢查询日志
    slowlog len 	获取慢查询日志条数
    slowlog reset 	清空慢查询
    服务端命令之info []
    config get 选项(支持*通配)	获取配置选项值
    config set 选项 值	设置配置选项值
    config rewrite 	把值写到配置文件
    config restart 	更新info命令的信息
    debug object key	调试选项,看一个key的情况
    debug segfault 	模拟段错误,让服务器崩溃
    object key (refcount|encoding|idletime)	
    monitor 	打开控制台,观察命令(调试用)
    client list 	列出所有连接
    client kill	杀死某个连接 CLIENT KILL 127.0.0.1:43501
    client getname 	获取连接的名称 默认nil
    client setname "名称" 	设置连接名称,便于调试
    连接命令
    auth 密码 	密码登陆(如果有密码)
    ping 	测试服务器是否可用
    echo "some content" 	测试服务器是否正常交互
    select 0/1/2...	选择数据库
    quit 	退出连接


### 第3章Redis配置文件参考

#### 1  主从集群master的redis-master.conf配置示例

    data:
    redis-master.conf: |
      port 6379
      tcp-backlog 511
      timeout 0
      tcp-keepalive 0
      loglevel notice
      databases 16
      save 900 1
      save 300 10
      save 60 10000
      stop-writes-on-bgsave-error yes
      rdbcompression yes
      rdbchecksum yes
      dbfilename dump.rdb
      dir /data/
      slave-serve-stale-data yes
      repl-diskless-sync no
      repl-diskless-sync-delay 5
      repl-disable-tcp-nodelay no
      slave-priority 100
      appendonly no
      appendfilename "appendonly.aof"
      appendfsync everysec
      no-appendfsync-on-rewrite no
      auto-aof-rewrite-percentage 100
      auto-aof-rewrite-min-size 64mb
      aof-load-truncated yes
      lua-time-limit 5000
      slowlog-log-slower-than 10000
      slowlog-max-len 128
      latency-monitor-threshold 0
      notify-keyspace-events ""
      hash-max-ziplist-entries 512
      hash-max-ziplist-value 64
      list-max-ziplist-entries 512
      list-max-ziplist-value 64
      set-max-intset-entries 512
      zset-max-ziplist-entries 128
      zset-max-ziplist-value 64
      hll-sparse-max-bytes 3000
      activerehashing yes
      client-output-buffer-limit normal 0 0 0
      client-output-buffer-limit slave 256mb 64mb 60
      client-output-buffer-limit pubsub 64mb 16mb 60
      hz 10
      aof-rewrite-incremental-fsync yes


#### 2  主从集群slave的redis-slave.conf配置示例

    data:
    redis-slave.conf: |
      port 6379
      slaveof redis-ps-master-ss-0.redis-ps-master-ss.kube-system.svc.cluster.local 6379
      tcp-backlog 511
      timeout 0
      tcp-keepalive 0
      loglevel notice
      databases 16
      save 900 1
      save 300 10
      save 60 10000
      stop-writes-on-bgsave-error yes
      rdbcompression yes
      rdbchecksum yes
      dbfilename dump.rdb
      dir /data/
      slave-serve-stale-data yes
      slave-read-only yes
      repl-diskless-sync no
      repl-diskless-sync-delay 5
      repl-disable-tcp-nodelay no
      slave-priority 100
      appendonly no
      appendfilename "appendonly.aof"
      appendfsync everysec
      no-appendfsync-on-rewrite no
      auto-aof-rewrite-percentage 100
      auto-aof-rewrite-min-size 64mb
      aof-load-truncated yes
      lua-time-limit 5000
      slowlog-log-slower-than 10000
      slowlog-max-len 128
      latency-monitor-threshold 0
      notify-keyspace-events ""
      hash-max-ziplist-entries 512
      hash-max-ziplist-value 64
      list-max-ziplist-entries 512
      list-max-ziplist-value 64
      set-max-intset-entries 512
      zset-max-ziplist-entries 128
      zset-max-ziplist-value 64
      hll-sparse-max-bytes 3000
      activerehashing yes
      client-output-buffer-limit normal 0 0 0
      client-output-buffer-limit slave 256mb 64mb 60
      client-output-buffer-limit pubsub 64mb 16mb 60
      hz 10
      aof-rewrite-incremental-fsync yes

