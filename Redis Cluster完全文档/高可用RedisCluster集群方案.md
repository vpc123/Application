### 第1章 部署准备：

#### 1 数据持久化前的准备-PV创建
参考pv创建文件：
所有文件都在Nk8s-redis-cluster目录下

    #pv创建说明,示例为nfs网络存储作为pv，存储需要提前做好关联
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: pv-redis-cluster-1  #redis创建pv块的名字
      namespace: kube-system#pv创建所在的命名空间
    spec:
      capacity:
    storage: 1Gi  #创建pv块的大小,可以按照需求进行自定义大小设置
      accessModes:
    - ReadWriteOnce   #读写模式
      volumeMode: Filesystem  #挂载模式，用文件系统模式，hostpath等
      persistentVolumeReclaimPolicy: Recycle#定义为可回收
      storageClassName: "redis-cluster-storage-class"
      nfs:#根据系统存储设备的要求，灵活选用相应的存储，验证使用的nfs作为存储。
    # real share directory
    path: /k8s/redis-cluster/1
    # nfs real ip
    server: 192.168.2.2

#### 2 命名规则说明
所需的yaml文档都按照redis-cluster-*.yaml规则命名。

#### 3 部署方案所在的namespace说明
文中redis cluster集群的部署都会部署在命名空间kube-system的命名空间下，也可以自定义namespace，此时就需要修改所有yaml的namespace字段改成自定义的命名。
#### 4 redis版本说明
部署redis集群的yaml为Redis Cluster模式,用于简单部署持久化Redis Cluster,Redis版本为4.0.8。

### 部署Redis：
#### 1 创建redis cluster集群的configmap

    #查看部署所需的yaml文件
    [root@k8s-master01 redis-cluster]# ls
    failover.py README.md redis-cluster-configmap.yaml redis-cluster-rbac.yaml redis-cluster-ss.yaml redis-cluster-pv.yaml redis-cluster-service.yaml
    #应用部署文档，创建ss,svc,configmap,rbac,pv
    [root@k8s-master01 redis-cluster]# kubectl apply -f .


### Redis配置解析：

##### 1.1 namespace根据自己定义进行修改


##### 1.2 configmap配置解析

    kind: ConfigMap
    apiVersion: v1
    metadata:
      name: redis-cluster-config
      namespace: kube-system
      labels:
    addonmanager.kubernetes.io/mode: Reconcile
    data:
    redis-cluster.conf: |
      # 节点端口
      port 6379
      # #  开启集群模式
      cluster-enabled yes
      # #  节点超时时间，单位毫秒
      cluster-node-timeout 15000
      # #  集群内部配置文件
      cluster-config-file "nodes.conf"

##### 1.3 Service文件解析

    kind: Service
    apiVersion: v1
    metadata:
      labels:
    app: redis-cluster-ss
      name: redis-cluster-ss
      namespace: kube-system
    spec:
      clusterIP: None
      ports:
      - name: redis
    port: 6379
    targetPort: 6379
      selector:
    app: redis-cluster-ss

#### 2 检查redis cluster集群的pod状态并新建slot分配槽位

    #检查pod运行状态
    [root@k8s-master ~]# kubectl get pod -nkube-system
    NAME READY   STATUSRESTARTS   AGE
    redis-cluster-ss-0   1/1 Running   0  22h
    redis-cluster-ss-1   1/1 Running   0  22h
    redis-cluster-ss-2   1/1 Running   0  22h
    redis-cluster-ss-3   1/1 Running   0  22h
    redis-cluster-ss-4   1/1 Running   0  22h
    redis-cluster-ss-5   1/1 Running   0  22h
    #等待所有pod启动完毕后，直接执行以下命令。
    $v=""
    $for i in `kubectl get po -n public-service -o wide | awk  '{print $6}' | grep -v IP`; do v="$v $i:6379";done
    $kubectl exec -ti redis-cluster-ss-5 -n public-service -- redis-trib.rb create --replicas 1 $v
    #检查redis cluster集群状态信息
    [root@k8s-master ~]# kubectl exec -ti redis-cluster-ss-0 -n public-service -- redis-cli cluster info
    cluster_state:ok
    cluster_slots_assigned:16384 #被分配的槽位数
    cluster_slots_ok:16384
    cluster_slots_pfail:0
    cluster_slots_fail:0
    cluster_known_nodes:6
    cluster_size:3
    cluster_current_epoch:7
    cluster_my_epoch:1
    cluster_stats_messages_ping_sent:7258
    cluster_stats_messages_pong_sent:6557
    cluster_stats_messages_fail_sent:30
    cluster_stats_messages_sent:13845
    cluster_stats_messages_ping_received:6552
    cluster_stats_messages_pong_received:7178
    cluster_stats_messages_meet_received:5
    cluster_stats_messages_fail_received:6
    cluster_stats_messages_auth-req_received:1
    cluster_stats_messages_received:13742

### 第2章redis cluster集群验证测试

#### 1 redis  Cluster集群node状态

    [root@k8s-master ~]# kubectl exec -ti redis-cluster-ss-2 -n public-service -- redis-cli cluster nodes
    fdb86230272da69e25929cae8e803bc101957d02 10.244.1.53:6379@16379 slave 817fe5a9a6e396dcbdc37b9aef22cefa1e161080 0 1548317000000 5 connected
    47ac3aefbdeb453cd2125cb80a57b3a8dbe03f65 10.244.1.52:6379@16379 myself,master - 0 1548316996000 3 connected 10923-16383
    8833f7f471cc0fe428b5b07b50ce3faad583c660 10.244.0.9:6379@16379 master - 0 1548316999004 2 connected 5461-10922
    817fe5a9a6e396dcbdc37b9aef22cefa1e161080 10.244.1.51:6379@16379 master - 0 1548317000009 1 connected 0-5460
    574d24a4d5da4a51a7271b1bdebfb44c2dc16df3 10.244.0.10:6379@16379 slave 47ac3aefbdeb453cd2125cb80a57b3a8dbe03f65 0 1548316999874 4 connected
    737615e0be457b866dc9fd4bdc3190ce590d4249 10.244.0.11:6379@16379 slave 8833f7f471cc0fe428b5b07b50ce3faad583c660 0 1548317001021 2 connected

#### 2 redis  cluster状态信息

    [root@k8s-master ~]# kubectl exec -ti redis-cluster-ss-0 -n public-service -- redis-cli cluster info
    cluster_state:ok
    cluster_slots_assigned:16384 #被分配的槽位数
    cluster_slots_ok:16384
    cluster_slots_pfail:0
    cluster_slots_fail:0
    cluster_known_nodes:6
    cluster_size:3
    cluster_current_epoch:7
    cluster_my_epoch:1
    cluster_stats_messages_ping_sent:7258
    cluster_stats_messages_pong_sent:6557
    cluster_stats_messages_fail_sent:30
    cluster_stats_messages_sent:13845
    cluster_stats_messages_ping_received:6552
    cluster_stats_messages_pong_received:7178
    cluster_stats_messages_meet_received:5
    cluster_stats_messages_fail_received:6
    cluster_stats_messages_auth-req_received:1
    cluster_stats_messages_received:13742

#### 3 Redis cluster集群数据读写

    #数据节点信息写入操作
    $kubectl exec -ti redis-cluster-ss-0 -n public-service -- redis-cli  -c -p 6379 set zhongyi zaixian
    
    #从其中一个节点查询所有的已经写入信息（注释：*取值范围为0-5）
    $kubectl exec -ti redis-cluster-ss-*-n public-service -- redis-cli  -c -p 6379 set zhongyi zaixian*



总结：redis cluster模式方案文档已经部署验证结束，更多配置请参阅同级目录下redis参数解析通过修改redis集群优化配置参数达到redis满足自己项目环境需求的目的。