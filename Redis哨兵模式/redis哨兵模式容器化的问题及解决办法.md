## Redis哨兵模式容器化问题总结

### 容器化哨兵模式问题

1 容器化自身的活性和哨兵模式自身的冲突

我们利用k8s容器化应用服务，但是k8s本身的特性保证了pod节点一旦发生故障便会立即重新启动新的pod，哨兵模式应用在传统的物理或者虚机的一种因为主机节点因为物理或者逻辑故障master节点断开连接后为了保持高可用问题从而进行的主机选举问题，从而保持集群的主从高可用。所以哨兵模式是应用在传统的没有节点活性的传统应用场景，经过大量测试发现，采用外力消除哨兵模式集群中的master的pod节点，短时间内都会快速启动新的pod使得哨兵模式一直无法开始选拔新的master节点就已经出现新的master的pod替代。

总结：哨兵模式应用于传统的网络节点断开连接后无法动态加入主从网络的传统分布式集群问题，然而容器化哨兵模式的哨兵会在master的pod异常死亡以后在剩下的slave节点选择新的master的pod。但是因为k8s的集群中pod即使删除，也会在短时间内启动新的master的pod，所以就呵呵了，哨兵模式的master选举算法压根不会执行到。这是传统应用环境和k8s本身特性部署应用的矛盾所在。


2 master节点和slave节点上下线的问题总结

经过验证，如果master节点没有问题，从节点的上下线动态增删节点都是可以进行的，只要保证足够的pv存储就可以进行无限制的动态扩容。

##### master节点下线以后无法动态加入到现有集群问题

经测试发现，当我们删除原有的master的pod以后，虽然我们是根据服务名进行的服务发现，因为存在先后顺序和信息数据写入问题造成的master节点无法加入原有的分布式集群。
###### 分析：
我们通过service进行服务发布的，首先集群中master节点的配置根据服务名配置ip，slave节点根据集群存在的master的服务名对应master节点ip配置自己的配置文件，这在写入slave的配置文件以后将会保存很长的时间甚至不会动态更新和重新下发，所以在原有集群的master下线重新上线新的master的pod，因为slave中配置文件是根据之前master的pod的service和pod的ip进行的配置，所以重新上线的master的pod不会在集群中存在。

##### 解决办法：

将原来的slave的pod全部删除，因为k8s的本身特性，被删除的pod还会重新起来，所以新启动的slave的pod会通过现有的master的pod（即新启动的master）配置下发配置文件到新的slave中，所以此时所有的节点才会更新加入到完全新的redis集群中。因此对redis容器化集群做数据持久化配置是极为关键的。



### 关于服务名代替ip进行访问的问题


#### 一 背景讨论


理想状态下，我们可以认为Kubernetes Pod是健壮的。但是，理想与现实的差距往往是非常大的。很多情况下，Pod中的容器可能会因为发生故障而死掉。Deployment等Controller会通过动态创建和销毁Pod来保证应用整体的健壮性。众所周知，每个Pod都拥有自己的IP地址，当新的Controller用新的Pod替代发生故障的Pod时，我们会发现，新的IP地址可能跟故障的Pod的IP地址可能不一致。此时，客户端如何访问这个服务呢？Kubernetes中的Service应运而生。

#### 二 举个栗子

##### 2.1 创建Deployment：httpd。

Kubernetes Service 逻辑上代表了一组具有某些label关联的Pod，Service拥有自己的IP，这个IP是不变的。无论后端的Pod如何变化，Service都不会发生改变。创建YAML如下：


    apiVersion: apps/v1beta1
    kind: Deployment
    metadata:
      name: httpd
    spec:
      replicas: 4
      template:
	    metadata:
	      labels:
	    	run: httpd
	    spec:
	      containers:
	      - name: httpd
		    image: httpd
		    ports:
		    - containerPort: 80

配置命令：

    [root@k8s-m ~]# kubectl apply -f Httpd-Deployment.yaml
    deployment.apps/httpd created

稍后片刻：

    [root@k8s-m ~]# kubectl get pod -o wide
    NAME READY   STATUSRESTARTS   AGE IP NODE NOMINATED NODE
    httpd-79c4f99955-dbbx7   1/1 Running   0  7m32s   10.244.2.35k8s-n2   <none>
    httpd-79c4f99955-djv44   1/1 Running   0  7m32s   10.244.1.101   k8s-n1   <none>
    httpd-79c4f99955-npqxz   1/1 Running   0  7m32s   10.244.1.102   k8s-n1   <none>
    httpd-79c4f99955-vkjk6   1/1 Running   0  7m32s   10.244.2.36k8s-n2   <none>
    [root@k8s-m ~]# curl 10.244.2.35
    <html><body><h1>It works!</h1></body></html>
    [root@k8s-m ~]# curl 10.244.2.36
    <html><body><h1>It works!</h1></body></html>
    [root@k8s-m ~]# curl 10.244.1.101
    <html><body><h1>It works!</h1></body></html>
    [root@k8s-m ~]# curl 10.244.1.102
    <html><body><h1>It works!</h1></body></html>

##### 2.2 创建Service：httpd-svc


创建YAML如下：


    apiVersion: v1
    kind: Service
    metadata:
      name: httpd-svc
    spec:
      selector:
    	run: httpd
      ports:
      - protocol: TCP
	    port: 8080
	    targetPort: 80

配置完成并观察：

    [root@k8s-m ~]# kubectl apply -f Httpd-Service.yaml
    service/httpd-svc created
    [root@k8s-m ~]# kubectl get svc
    NAME TYPECLUSTER-IP   EXTERNAL-IP   PORT(S)AGE
    httpd-svcClusterIP   10.110.212.171   <none>8080/TCP   14s
    kubernetes   ClusterIP   10.96.0.1<none>443/TCP11d
    [root@k8s-m ~]# curl 10.110.212.171:8080
    <html><body><h1>It works!</h1></body></html>
    [root@k8s-m ~]# kubectl describe service httpd-svc
    Name:  httpd-svc
    Namespace: default
    Labels:<none>
    Annotations:   kubectl.kubernetes.io/last-applied-configuration:
     {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"name":"httpd-svc","namespace":"default"},"spec":{"ports":[{"port":8080,"...
    Selector:  run=httpd
    Type:  ClusterIP
    IP:10.110.212.171
    Port:  <unset>  8080/TCP
    TargetPort:80/TCP
    Endpoints: 10.244.1.101:80,10.244.1.102:80,10.244.2.35:80 + 1 more...
    Session Affinity:  None
    Events:<none>
    
从以上内容中的Endpoints可以看出服务httpd-svc下面包含我们指定的labels的Pod，cluster-ip通过iptables成功映射到Pod IP，成功。再通过iptables-save命令看一下相关的iptables规则。

    [root@k8s-m ~]# iptables-save |grep "10.110.212.171"
    -A KUBE-SERVICES ! -s 10.244.0.0/16 -d 10.110.212.171/32 -p tcp -m comment --comment "default/httpd-svc: cluster IP" -m tcp --dport 8080 -j KUBE-MARK-MASQ
    -A KUBE-SERVICES -d 10.110.212.171/32 -p tcp -m comment --comment "default/httpd-svc: cluster IP" -m tcp --dport 8080 -j KUBE-SVC-RL3JAE4GN7VOGDGP
    [root@k8s-m ~]# iptables-save|grep -v 'default/httpd-svc'|grep 'KUBE-SVC-RL3JAE4GN7VOGDGP'
    :KUBE-SVC-RL3JAE4GN7VOGDGP - [0:0]
    -A KUBE-SVC-RL3JAE4GN7VOGDGP -m statistic --mode random --probability 0.25000000000 -j KUBE-SEP-R5YBMKYSG56R4KDU
    -A KUBE-SVC-RL3JAE4GN7VOGDGP -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-7G5ANBWSVVLRNZAH
    -A KUBE-SVC-RL3JAE4GN7VOGDGP -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-2PT6QZGNQHS4OL4I
    -A KUBE-SVC-RL3JAE4GN7VOGDGP -j KUBE-SEP-I4PXZ6UARQLLOV4E


我们可以进一步查看相关的转发规则，此处省略。iptables将访问Service的流量转发到后端Pod，使用类似于轮询的的负载均衡策略。

##### 2.3 通过域名访问Service

通过创建一个临时的隔离环境来验证一下DNS是否生效。



    [root@k8s-m ~]# kubectl create -it --rm busybox --image=busybox /bin/sh
    kubectl run --generator=deployment/apps.v1beta1 is DEPRECATED and will be removed in a future version. Use kubectl create instead.
    If you don't see a command prompt, try pressing enter.
    / # wget httpd-svc.default:8080
    Connecting to httpd-svc.default:8080 (10.110.212.171:8080)
    index.html   100% |*******************************************************************************************************************************|45  0:00:00 ETA
    / # cat index.html
    <html><body><h1>It works!</h1></body></html>

在以上例子中，临时的隔离环境的namespace为default，与我们新建的httpd-svc都在同一namespace内，httpd-svc.default的default可以省略。如果跨namespace访问的话，那么namespace是不能省略的。


##### 2.4 外网访问Service。

通常情况下，我们可以通过四种方式来访问Kubeenetes的Service，分别是ClusterIP，NodePort，Loadbalance，ExternalName。在此之前的实验都是基于ClusterIP的，集群内部的Node和Pod均可通过Cluster IP来访问Service。NodePort是通过集群节点的静态端口对外提供服务。
接下来我们将以NodePort为例来进行实际演示。修改之后的Service的YAML如下：

    apiVersion: v1
    kind: Service
    metadata:
      name: httpd-svc
    spec:
      type: NodePort
      selector:
    	run: httpd
      ports:
      - protocol: TCP
	    nodePort: 31688
	    port: 8080
	    targetPort: 80


配置后观察：

    [root@k8s-m ~]# kubectl apply -f  Httpd-Service.yaml
    service/httpd-svc configured
    [root@k8s-m ~]# kubectl get svc
    NAME TYPECLUSTER-IP   EXTERNAL-IP   PORT(S)  AGE
    httpd-svcNodePort10.110.212.171   <none>8080:31688/TCP   117m
    kubernetes   ClusterIP   10.96.0.1<none>443/TCP  12d


Service httpd-svc的端口被映射到了主机的31688端口。YAML文件如果不指定nodePort的话，Kubernetes会在30000-32767范围内为Service分配一个端口。此刻我们就可以通过浏览器来访问我们的服务了。在与node网络互通的环境中，通过任意一个Node的IP:31688即可访问刚刚部署好的Service。


#### 总结：

在k8s上搭建Redis sentinel完全没有意义，经过测试，当master节点宕机后，sentinel选择新的节点当主节点，当原master恢复后，此时无法再次成为集群节点。因为在物理机上部署时，sentinel探测以及更改配置文件都是以IP的形式，集群复制也是以IP的形式，但是在容器中，虽然采用的StatefulSet的Headless Service来建立的主从，但是主从建立后，master、slave、sentinel记录还是解析后的IP，但是pod的IP每次重启都会改变，所有sentinel无法识别宕机后又重新启动的master节点，所以一直无法加入集群，虽然可以通过固定podIP或者使用NodePort的方式来固定，或者通过sentinel获取当前master的IP来修改配置文件，但是个人觉得也是没有必要的，sentinel实现的是高可用Redis主从，检测Redis Master的状态，进行主从切换等操作，但是在k8s中，无论是dc或者ss，都会保证pod以期望的值进行运行，再加上k8s自带的活性检测，当端口不可用或者服务不可用时会自动重启pod或者pod的中的服务，所以当在k8s中建立了Redis主从同步后，相当于已经成为了高可用状态，并且sentinel进行主从切换的时间不一定有k8s重建pod的时间快，所以个人认为在k8s上搭建sentinel没有意义。

我们通过，实践验证了redis高可用集群部署过程中的问题还有解决办法，针对k8s集群中容器化应用项目使用service进行服务通信的实践验证，k8s中使用的service通信就是借用了传统应用场景中域名代替ip进行通信的应用方式。在传统应用和k8s应用系统中我们权衡利弊，大型传统分布式应用在容器化过程中要充分结合应用场景的不同和部署平台的自身特性进行综合性分析才是解决问题的最佳途径。


