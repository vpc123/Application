#### Redis集群模式升级手册



1   创建redis集群运行的命名空间：

```
$cd  k8s-redis-cluster

$kubectl   apply   -f  ./
```

确认检查pod状态:

```
$kubectl   get   pod   -n  public-service 
NAME                 READY   STATUS    RESTARTS   AGE   IP            NODE         NOMINATED NODE   READINESS GATES
redis-cluster-ss-0   1/1     Running   0          16s   10.244.0.5    k8s-master   <none>           <none>
redis-cluster-ss-1   1/1     Running   0          13s   10.244.1.13   k8s-node01   <none>           <none>
redis-cluster-ss-2   1/1     Running   0          11s   10.244.0.6    k8s-master   <none>           <none>
redis-cluster-ss-3   1/1     Running   0          9s    10.244.1.14   k8s-node01   <none>           <none>
redis-cluster-ss-4   1/1     Running   0          7s    10.244.0.7    k8s-master   <none>           <none>
redis-cluster-ss-5   1/1     Running   0          4s    10.244.1.15   k8s-node01   <none>           <none>
```

2  确认所有pod已经正常运行起来，然后执行redis集群进行集群关联功能。



```
[root@k8s-master k8s-redis-cluster]# echo $v

[root@k8s-master k8s-redis-cluster]# for i in `kubectl get po -n public-service -o wide | awk  '{print $6}' | grep -v IP`; do v="$v $i:6379";done
[root@k8s-master k8s-redis-cluster]# echo $v
10.244.0.5:6379 10.244.1.13:6379 10.244.0.6:6379 10.244.1.14:6379 10.244.0.7:6379 10.244.1.15:6379
[root@k8s-master k8s-redis-cluster]# kubectl exec -ti redis-cluster-ss-5 -n public-service -- redis-trib.rb create --replicas 1 $v

> > > Creating cluster
> > > Performing hash slots allocation on 6 nodes...
> > > Using 3 masters:
> > > 10.244.0.5:6379
> > > 10.244.1.13:6379
> > > 10.244.0.6:6379
> > > Adding replica 10.244.0.7:6379 to 10.244.0.5:6379
> > > Adding replica 10.244.1.15:6379 to 10.244.1.13:6379
> > > Adding replica 10.244.1.14:6379 to 10.244.0.6:6379
> > > M: 52bae65db294c4e5c4c6e1d6ee5ea0729e9cf4f4 10.244.0.5:6379
> > >    slots:0-5460 (5461 slots) master
> > > M: 8e436c036f85aac5fa222806e92fdb5454e09e46 10.244.1.13:6379
> > >    slots:5461-10922 (5462 slots) master
> > > M: e7479bd22fe887be2fea2e6863b7a13c35ed6dfe 10.244.0.6:6379
> > >    slots:10923-16383 (5461 slots) master
> > > S: 762b9e917a431eda2082a9e7ed23a6602d27ccca 10.244.1.14:6379
> > >    replicates e7479bd22fe887be2fea2e6863b7a13c35ed6dfe
> > > S: 6103f4a18bbfa2c2e39a25cfb84c122a42e406c5 10.244.0.7:6379
> > >    replicates 52bae65db294c4e5c4c6e1d6ee5ea0729e9cf4f4
> > > S: ec6b2a388743465826d067cdf01984197668a8a5 10.244.1.15:6379
> > >    replicates 8e436c036f85aac5fa222806e92fdb5454e09e46
> > > 
> > > ###########default : yes>>> Nodes configuration updated
> > > Assign a different config epoch to each node
> > > Sending CLUSTER MEET messages to join the cluster
> > > Waiting for the cluster to join.....
> > > Performing Cluster Check (using node 10.244.0.5:6379)
> > > M: 52bae65db294c4e5c4c6e1d6ee5ea0729e9cf4f4 10.244.0.5:6379
> > >    slots:0-5460 (5461 slots) master
> > >    1 additional replica(s)
> > > S: 762b9e917a431eda2082a9e7ed23a6602d27ccca 10.244.1.14:6379
> > >    slots: (0 slots) slave
> > >    replicates e7479bd22fe887be2fea2e6863b7a13c35ed6dfe
> > > S: 6103f4a18bbfa2c2e39a25cfb84c122a42e406c5 10.244.0.7:6379
> > >    slots: (0 slots) slave
> > >    replicates 52bae65db294c4e5c4c6e1d6ee5ea0729e9cf4f4
> > > M: 8e436c036f85aac5fa222806e92fdb5454e09e46 10.244.1.13:6379
> > >    slots:5461-10922 (5462 slots) master
> > >    1 additional replica(s)
> > > S: ec6b2a388743465826d067cdf01984197668a8a5 10.244.1.15:6379
> > >    slots: (0 slots) slave
> > >    replicates 8e436c036f85aac5fa222806e92fdb5454e09e46
> > > M: e7479bd22fe887be2fea2e6863b7a13c35ed6dfe 10.244.0.6:6379
> > >    slots:10923-16383 (5461 slots) master
> > >    1 additional replica(s)
> > > [OK] All nodes agree about slots configuration.
> > > Check for open slots...
> > > Check slots coverage...
> > > [OK] All 16384 slots covered.
```

注：至此已经完成了Redis集群模式的搭建，没有做数据持久化。



3  检查redis项目集群的健康状态和关联情况



```
[root@k8s-master k8s-redis-cluster]# kubectl exec -ti redis-cluster-ss-0 -n public-service -- redis-cli cluster nodes
52bae65db294c4e5c4c6e1d6ee5ea0729e9cf4f4 10.244.0.5:6379@16379 myself,master - 0 1551186064000 1 connected 0-5460
762b9e917a431eda2082a9e7ed23a6602d27ccca 10.244.1.14:6379@16379 slave e7479bd22fe887be2fea2e6863b7a13c35ed6dfe 0 1551186065000 4 connected
6103f4a18bbfa2c2e39a25cfb84c122a42e406c5 10.244.0.7:6379@16379 slave 52bae65db294c4e5c4c6e1d6ee5ea0729e9cf4f4 0 1551186065569 5 connected
8e436c036f85aac5fa222806e92fdb5454e09e46 10.244.1.13:6379@16379 master - 0 1551186066577 2 connected 5461-10922
ec6b2a388743465826d067cdf01984197668a8a5 10.244.1.15:6379@16379 slave 8e436c036f85aac5fa222806e92fdb5454e09e46 0 1551186064563 6 connected
e7479bd22fe887be2fea2e6863b7a13c35ed6dfe 10.244.0.6:6379@16379 master - 0 1551186064000 3 connected 10923-16383
[root@k8s-master k8s-redis-cluster]# kubectl exec -ti redis-cluster-ss-0 -n public-service -- redis-cli cluster info
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:6
cluster_my_epoch:1
cluster_stats_messages_ping_sent:69
cluster_stats_messages_pong_sent:79
cluster_stats_messages_sent:148
cluster_stats_messages_ping_received:74
cluster_stats_messages_pong_received:69
cluster_stats_messages_meet_received:5
cluster_stats_messages_received:148
```

目前的redis集群并没有设置密码信息，我们需要更改redis的configmap配置。



4  设置redis集群密码验证，并测试redis集群的可用性验证。



更改configmap的配置添加验证密码

    [root@k8s-master k8s-redis-cluster]# cat redis-cluster-configmap.yaml
     kind: ConfigMap
    apiVersion: v1
    metadata:
     name: redis-cluster-config
     namespace: public-service
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
      requirepass 123456
      masterauth 123456
      # #  集群内部配置文件
      cluster-config-file "nodes.conf"

更新最新配置到集群中

```
[root@k8s-master k8s-redis-cluster]# kubectl replace -f redis-cluster-configmap.yaml 
configmap/redis-cluster-config replaced
[root@k8s-master k8s-redis-cluster]# 
```

然后更新pod加载最新的配置信息，需要手动删除redis的pod，但是注意不可以一次性全部删除，需要删除一个完成以后并且已经运行再删除另外一个。

```
[pod -npublic-service
NAME                 READY   STATUS    RESTARTS   AGE
redis-cluster-ss-0   1/1     Running   0          34m
redis-cluster-ss-1   1/1     Running   0          34m
redis-cluster-ss-2   1/1     Running   0          34m
redis-cluster-ss-3   1/1     Running   0          34m
redis-cluster-ss-4   1/1     Running   0          34m
redis-cluster-ss-5   1/1     Running   0          34m
[root@k8s-master k8s-redis-cluster]# kubectl delete  pod redis-cluster-ss-0  -npublic-service
pod "redis-cluster-ss-0" deleted
[root@k8s-master k8s-redis-cluster]# kubectl get pod -npublic-service
NAME                 READY   STATUS    RESTARTS   AGE
redis-cluster-ss-0   1/1     Running   0          3m44s
redis-cluster-ss-1   1/1     Running   0          38m
redis-cluster-ss-2   1/1     Running   0          38m
redis-cluster-ss-3   1/1     Running   0          38m
redis-cluster-ss-4   1/1     Running   0          38m
redis-cluster-ss-5   1/1     Running   0          38m
```

将6个pod删除完全就可以。之后新建的pod就是用的最新配置。


