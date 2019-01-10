## redis主从集群哨兵模式从节点动态扩容方案

前提部署文档参考:（同级目录下参考并且已经完成redis哨兵模式部署.md）

在已经正常运行的redis哨兵模式开始进行从节点动态接入到整个redis接入到集群的流程过程。

创建更多的pv存储块，参考前面的存储模式。
新增加两条数据选项，并在对应的目录下创建文件。
	
	[root@nfs ~]# cd /data/lys/nfs/redis-sentinel
	[root@nfs ~]# mkdir {1..4}
	[root@nfs ~]# ls
	0  1  2  3
    [root@nfs ~]# vi /etc/exports
    /data/lys/nfs/redis-sentinel/3 *(rw,sync,no_subtree_check,no_root_squash)
    /data/lys/nfs/redis-sentinel/4 *(rw,sync,no_subtree_check,no_root_squash)

配置生效

    [root@bogon lys]# exportfs -r
	[root@bogon lys]# service rpcbind start
	[root@bogon lys]# service nfs start

新增配置文件增加pv块，示例(我们增加两块)：

    ---
    
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: pv-redis-sentinel-4
    spec:
      capacity:
    	storage: 4Gi
      accessModes:
    	- ReadWriteMany
      volumeMode: Filesystem
      persistentVolumeReclaimPolicy: Recycle
      storageClassName: "redis-sentinel-storage-class"
      nfs:
	    # real share directory
	    path: /data/lys/nfs/redis-sentinel/4
	    # nfs real ip
	    server: 192.168.131.20


更新系统中的pv存储块的数量：

    $kubectl create -f redis-sentinel-pv.yaml

查看新增的pv存储数量：

	$kubectl get pv | grep redis
	[root@k8s-master k8s-redis-sentinel]# kubectl get pv | grep redis
	pv-redis-sentinel-0   4Gi        RWX            Recycle          Bound       public-service/redis-sentinel-master-storage-redis-sentinel-master-ss-0   redis-sentinel-storage-class            20h
	pv-redis-sentinel-1   4Gi        RWX            Recycle          Bound       public-service/redis-sentinel-slave-storage-redis-sentinel-slave-ss-0     redis-sentinel-storage-class            20h
	pv-redis-sentinel-2   4Gi        RWX            Recycle          Bound       public-service/redis-sentinel-slave-storage-redis-sentinel-slave-ss-1     redis-sentinel-storage-class            20h
	pv-redis-sentinel-3   4Gi        RWX            Recycle          Available                                                                             redis-sentinel-storage-class            35s
	pv-redis-sentinel-4   4Gi        RWX            Recycle          Available                                                                             redis-sentinel-storage-class            35s


这时候我们已经成功挂载了另外两个pv存储块。


`[root@k8s-master k8s-redis-sentinel]# vi redis-sentinel-ss-slave.yaml` 

增加slave副本数量，之前配置为2，现在修改slave为3或者4（替换更新操作）.

    [root@k8s-master k8s-redis-sentinel]# kubectl replace -f  redis-sentinel-ss-slave.yaml 
    statefulset.apps/redis-sentinel-slave-ss replaced
    [root@k8s-master k8s-redis-sentinel]# 

查看slave效果：

    [root@k8s-master k8s-redis-sentinel]# kubectl get pod -npublic-service
    NAME   READY   STATUSRESTARTS   AGE
    redis-sentinel-master-ss-0 1/1 Running   0  20h
    redis-sentinel-sentinel-ss-0   1/1 Running   0  18h
    redis-sentinel-sentinel-ss-1   1/1 Running   0  18h
    redis-sentinel-sentinel-ss-2   1/1 Running   0  18h
    redis-sentinel-slave-ss-0  1/1 Running   0  20h
    redis-sentinel-slave-ss-1  1/1 Running   0  20h
    redis-sentinel-slave-ss-2  1/1 Running   0  53s

为了测试更加明显，我们增加slave节点为4进行测试验证。

    [root@k8s-master k8s-redis-sentinel]# kubectl get pod -npublic-service -owide
    NAME   READY   STATUSRESTARTS   AGE IPNODE NOMINATED NODE   READINESS GATES
    redis-sentinel-master-ss-0 1/1 Running   0  20h 10.244.1.24   k8s-node01   <none>   <none>
    redis-sentinel-sentinel-ss-0   1/1 Running   0  18h 10.244.0.19   k8s-master   <none>   <none>
    redis-sentinel-sentinel-ss-1   1/1 Running   0  18h 10.244.1.26   k8s-node01   <none>   <none>
    redis-sentinel-sentinel-ss-2   1/1 Running   0  18h 10.244.0.20   k8s-master   <none>   <none>
    redis-sentinel-slave-ss-0  1/1 Running   0  20h 10.244.1.25   k8s-node01   <none>   <none>
    redis-sentinel-slave-ss-1  1/1 Running   0  20h 10.244.0.18   k8s-master   <none>   <none>
    redis-sentinel-slave-ss-2  1/1 Running   0  5m32s   10.244.0.21   k8s-master   <none>   <none>
    redis-sentinel-slave-ss-3  1/1 Running   0  9s  10.244.1.27   k8s-node01   <none>   <none>

我们可以看到redis的节点个数是在动态新增和替换的，但是要保证足够多的pv存储块和系统资源我们就可以没有限制的进行节点新增。

至此我们完全验证了我们搭建的哨兵模式redis集群的高可用，我们可以根据验证结果确保了redis的高可用！

打完收工！回家吃饭。