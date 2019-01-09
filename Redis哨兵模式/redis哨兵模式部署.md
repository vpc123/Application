## k8s持久化安装Redis Sentinel(哨兵模式)


### 1、PV创建

在nfs或者其他类型后端存储创建pv，首先创建共享目录


    [root@nfs ~]# vi /etc/exports
    /k8s/redis-sentinel/0 *(rw,sync,no_subtree_check,no_root_squash)
    /k8s/redis-sentinel/1 *(rw,sync,no_subtree_check,no_root_squash)
    /k8s/redis-sentinel/2 *(rw,sync,no_subtree_check,no_root_squash)
    [root@nfs ~]# vi /etc/exports

nfs服务器上配置生效:

    [root@bogon lys]# exportfs -r

启动rpcbind、nfs服务

    [root@bogon lys]# service rpcbind start
    正在启动 rpcbind： [确定]
    [root@bogon lys]# service nfs start
    启动 NFS 服务：[确定]
    启动 NFS mountd：  [确定]
    启动 NFS 守护进程：[确定]
    正在启动 RPC idmapd：  [确定]

统计目录下存在一个文件夹:k8s-redis-sentinel包含了所有的启用配置文件。

创建pv，注意Redis的空间大小按需修改


    $kubectl create -f redis-sentinel-pv.yaml
	$kubectl get pv | grep redis
	pv-redis-sentinel-0   4Gi        RWX            Recycle          Bound     public-service/redis-sentinel-master-storage-redis-sentinel-master-ss-0   redis-sentinel-storage-class             16h
	pv-redis-sentinel-1   4Gi        RWX            Recycle          Bound     public-service/redis-sentinel-slave-storage-redis-sentinel-slave-ss-0     redis-sentinel-storage-class             16h
	pv-redis-sentinel-2   4Gi        RWX            Recycle          Bound     public-service/redis-sentinel-slave-storage-redis-sentinel-slave-ss-1     redis-sentinel-storage-class             16h

### 2、创建namespace

默认是在public-service中创建Redis哨兵模式

    $kubectl create namespace public-service
    # 如果不使用public-service，需要更改所有yaml文件的public-service为你namespace。
    # sed -i "s#public-service#YOUR_NAMESPACE#g" *.yaml


### 3、创建ConfigMap

Redis配置按需修改，默认使用的是rdb存储模式


    [root@k8s-master01 redis-sentinel]# kubectl create -f redis-sentinel-configmap.yaml
    [root@k8s-master01 redis-sentinel]# kubectl get configmap -n public-service
    NAMEDATA  AGE
    redis-sentinel-config   2 17h

注意，此时configmap中redis-slave.conf的slaveof的master地址为ss里面的Headless Service地址。

4、创建service

　　service主要提供pods之间的互访，StatefulSet主要用Headless Service通讯，格式： 

	   statefulSetName-{0..N-1}.serviceName.namespace.svc.cluster.local
    
    　　- serviceName为Headless Service的名字
    
    　　- 0..N-1为Pod所在的序号，从0开始到N-1
    
    　　- statefulSetName为StatefulSet的名字
    
    　　- namespace为服务所在的namespace，Headless Servic和StatefulSet必须在相同的namespace
    
    　　- .cluster.local为Cluster Domain

　　如本集群的HS为：

    　　　　Master：
    
    　　　　　　redis-sentinel-master-ss-0.redis-sentinel-master-ss.public-service.svc.cluster.local:6379
    
    　　　　Slave：
    
    　　　　　　redis-sentinel-slave-ss-0.redis-sentinel-slave-ss.public-service.svc.cluster.local:6379
    
    　　　　　　redis-sentinel-slave-ss-1.redis-sentinel-slave-ss.public-service.svc.cluster.local:6379

　　创建Service

    [root@k8s-master01 redis-sentinel]# kubectl create -f redis-sentinel-service-master.yaml -f redis-sentinel-service-slave.yaml
    
    [root@k8s-master01 redis-sentinel]# kubectl get service -n public-service
    NAME   TYPECLUSTER-IP  EXTERNAL-IP   PORT(S)  AGE
    redis-sentinel-master-ss   ClusterIP   None<none>6379/TCP 16h
    redis-sentinel-slave-ssClusterIP   None<none>6379/TCP <invalid>

### 5、创建StatefulSet

    [root@k8s-master01 redis-sentinel]# kubectl create -f redis-sentinel-rbac.yaml -f redis-sentinel-ss-master.yaml -f redis-sentinel-ss-slave.yaml

    [root@k8s-master01 redis-sentinel]# kubectl get statefulset -n public-service
    NAME   DESIRED   CURRENT   AGE
    redis-sentinel-master-ss   1 1 16h
    redis-sentinel-slave-ss2 2 16h
    rmq-cluster3 3 3d
    [root@k8s-master01 redis-sentinel]# kubectl get pods -n public-service
    NAME READY STATUSRESTARTS   AGE
    redis-sentinel-master-ss-0   1/1   Running   0  16h
    redis-sentinel-slave-ss-01/1   Running   0  16h
    redis-sentinel-slave-ss-11/1   Running   0  16h

此时相当于已经在k8s上创建了Redis的主从模式。

### 6、主从集群测试查看

　　状态查看

　　pods通讯测试

　　master连接slave测试

    [root@k8s-master01 redis-sentinel]# kubectl exec -ti redis-sentinel-master-ss-0 -n public-service -- redis-cli -h redis-sentinel-slave-ss-0.redis-sentinel-slave-ss.public-service.svc.cluster.local  ping
    PONG
    [root@k8s-master01 redis-sentinel]# kubectl exec -ti redis-sentinel-master-ss-0 -n public-service -- redis-cli -h redis-sentinel-slave-ss-1.redis-sentinel-slave-ss.public-service.svc.cluster.local  ping
    PONG

slave连接master测试


    [root@k8s-master01 redis-sentinel]# kubectl exec -ti redis-sentinel-slave-ss-0 -n public-service -- redis-cli -h redis-sentinel-master-ss-0.redis-sentinel-master-ss.public-service.svc.cluster.local  ping
    PONG
    [root@k8s-master01 redis-sentinel]# kubectl exec -ti redis-sentinel-slave-ss-1 -n public-service -- redis-cli -h redis-sentinel-master-ss-0.redis-sentinel-master-ss.public-service.svc.cluster.local  ping
    PONG

同步状态查看

    [root@k8s-master01 redis-sentinel]# kubectl exec -ti redis-sentinel-slave-ss-1 -n public-service -- redis-cli -h redis-sentinel-master-ss-0.redis-sentinel-master-ss.public-service.svc.cluster.local  info replication
    # Replication
    role:master
    connected_slaves:2
    slave0:ip=172.168.5.94,port=6379,state=online,offset=80410,lag=1
    slave1:ip=172.168.6.113,port=6379,state=online,offset=80410,lag=0
    master_replid:ad4341815b25f12d4aeb390a19a8bd8452875879
    master_replid2:0000000000000000000000000000000000000000
    master_repl_offset:80410
    second_repl_offset:-1
    repl_backlog_active:1
    repl_backlog_size:1048576
    repl_backlog_first_byte_offset:1
    repl_backlog_histlen:80410

同步测试

    # master写入数据
    [root@k8s-master01 redis-sentinel]# kubectl exec -ti redis-sentinel-slave-ss-1 -n public-service -- redis-cli -h redis-sentinel-master-ss-0.redis-sentinel-master-ss.public-service.svc.cluster.local  set test test_data
    OK
    # master获取数据
    [root@k8s-master01 redis-sentinel]# kubectl exec -ti redis-sentinel-slave-ss-1 -n public-service -- redis-cli -h redis-sentinel-master-ss-0.redis-sentinel-master-ss.public-service.svc.cluster.local  get test
    "test_data"
    # slave获取数据
    [root@k8s-master01 redis-sentinel]# kubectl exec -ti redis-sentinel-slave-ss-1 -n public-service -- redis-cli   get test
    "test_data"

从节点无法写入数据

    [root@k8s-master01 redis-sentinel]# kubectl exec -ti redis-sentinel-slave-ss-1 -n public-service -- redis-cli   set k v
    (error) READONLY You can't write against a read only replica.

NFS查看数据存储

    [root@nfs redis-sentinel]# tree .
    .
    ├── 0
    │   └── dump.rdb
    ├── 1
    │   └── dump.rdb
    └── 2
    └── dump.rdb
    
    3 directories, 3 files

###### 网络观点：在k8s上搭建Redis sentinel完全没有意义，经过测试，当master节点宕机后，sentinel选择新的节点当主节点，当原master恢复后，此时无法再次成为集群节点。因为在物理机上部署时，sentinel探测以及更改配置文件都是以IP的形式，集群复制也是以IP的形式，但是在容器中，虽然采用的StatefulSet的Headless Service来建立的主从，但是主从建立后，master、slave、sentinel记录还是解析后的IP，但是pod的IP每次重启都会改变，所有sentinel无法识别宕机后又重新启动的master节点，所以一直无法加入集群，虽然可以通过固定podIP或者使用NodePort的方式来固定，或者通过sentinel获取当前master的IP来修改配置文件，但是个人觉得也是没有必要的，sentinel实现的是高可用Redis主从，检测Redis Master的状态，进行主从切换等操作，但是在k8s中，无论是dc或者ss，都会保证pod以期望的值进行运行，再加上k8s自带的活性检测，当端口不可用或者服务不可用时会自动重启pod或者pod的中的服务，所以当在k8s中建立了Redis主从同步后，相当于已经成为了高可用状态，并且sentinel进行主从切换的时间不一定有k8s重建pod的时间快，所以个人认为在k8s上搭建sentinel没有意义。所以下面搭建sentinel的步骤无需在看。 


### 7、创建sentinel


    [root@k8s-master01 redis-sentinel]# kubectl create -f redis-sentinel-ss-sentinel.yaml -f redis-sentinel-service-sentinel.yaml
    
    [root@k8s-master01 redis-sentinel]# kubectl get service -n public-servicve
    No resources found.
    [root@k8s-master01 redis-sentinel]# kubectl get service -n public-service
    NAME TYPECLUSTER-IP  EXTERNAL-IP   PORT(S)  AGE
    redis-sentinel-master-ss ClusterIP   None<none>6379/TCP 17h
    redis-sentinel-sentinel-ss   ClusterIP   None<none>26379/TCP36m
    redis-sentinel-slave-ss  ClusterIP   None<none>6379/TCP 1h
    rmq-cluster  ClusterIP   None<none>5672/TCP 3d
    rmq-cluster-balancer NodePort10.107.221.85   <none>15672:30051/TCP,5672:31892/TCP   3d
    [root@k8s-master01 redis-sentinel]# kubectl get statefulset -n public-service
    NAME DESIRED   CURRENT   AGE
    redis-sentinel-master-ss 1 1 17h
    redis-sentinel-sentinel-ss   3 3 8m
    redis-sentinel-slave-ss  2 2 17h
    rmq-cluster  3 3 3d
    [root@k8s-master01 redis-sentinel]# kubectl get pods -n public-service | grep sentinel
    redis-sentinel-master-ss-0 1/1   Running   0  17h
    redis-sentinel-sentinel-ss-0   1/1   Running   0  2m
    redis-sentinel-sentinel-ss-1   1/1   Running   0  2m
    redis-sentinel-sentinel-ss-2   1/1   Running   0  2m
    redis-sentinel-slave-ss-0  1/1   Running   0  17h
    redis-sentinel-slave-ss-1  1/1   Running   0  17h

### 8、查看日志

查看哨兵状态

    [root@k8s-master01 ~]# kubectl exec -ti redis-sentinel-sentinel-ss-0 -n public-service -- redis-cli -h 127.0.0.1 -p 26379 info Sentinel
    # Sentinel
    sentinel_masters:1
    sentinel_tilt:0
    sentinel_running_scripts:0
    sentinel_scripts_queue_length:0
    sentinel_simulate_failure_flags:0
    master0:name=mymaster,status=ok,address=172.168.6.111:6379,slaves=2,sentinels=3

### 9、容灾测试

    # 查看当前数据
    [root@k8s-master01 ~]# kubectl exec -ti redis-sentinel-master-ss-0 -n public-service -- redis-cli -h 127.0.0.1 -p 6379 get test
    "test_data"

关闭master节点

通过dashboard将副本数量调整为0.

至此redis的哨兵模式部署讲解完毕！