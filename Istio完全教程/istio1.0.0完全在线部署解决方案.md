## kubernetes实践：istio-1.0.0部署和试用

下载获得istio相应文件。

国内网络环境镜像仓库：https://cr.console.aliyun.com/cn-hangzhou/mirrors

特别重点：
1  master节点上允许hpa通过接口采集数据

    $vi /etc/kubernetes/manifests/kube-controller-manager.yaml
    - --horizontal-pod-autoscaler-use-rest-clients=false
2 master上允许istio的自动注入，修改/etc/kubernetes/manifests/kube-apiserver.yaml

    $vi /etc/kubernetes/manifests/kube-apiserver.yaml
    - --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota

master节点重启服务

### 重启服务

    $systemctl restart kubelet

1.获取安装包

    $wget https://github.com/istio/istio/releases/download/1.0.0/istio-1.0.0-linux.tar.gz


	$tar -zxvf istio-1.0.0-linux.tar.gz

2.安装istioctl

    $cp istio-1.0.0/bin/istioctl /usr/local/bin/

设置环境变量：(增加两条环境变量)

	$vim /etc/profile 

    export ISTIO_HOME=/root/istio-1.0
    
    export PATH=$ISTIO_HOME/bin:$PATH


3.安装istio核心组件

    $kubectl apply -f istio-1.0.0/install/kubernetes/istio-demo.yaml

gcr.io和quay.io相关的镜像下载不了的话可以替换为自己的镜像：(文件中存在istio-image.sh)

	$cat image-istio.sh

    docker pull registry.cn-hangzhou.aliyuncs.com/istio-release/citadel:1.0.0
    docker tag registry.cn-hangzhou.aliyuncs.com/istio-release/citadel:1.0.0 gcr.io/istio-release/citadel:1.0.0
    docker rmi registry.cn-hangzhou.aliyuncs.com/istio-release/citadel:1.0.0
    
    docker pull registry.cn-hangzhou.aliyuncs.com/istio-release/galley:1.0.0
    docker tag registry.cn-hangzhou.aliyuncs.com/istio-release/galley:1.0.0 gcr.io/istio-release/galley:1.0.0
    docker rmi registry.cn-hangzhou.aliyuncs.com/istio-release/galley:1.0.0
    
    docker pull registry.cn-hangzhou.aliyuncs.com/istio-release/pilot:1.0.0
    docker tag registry.cn-hangzhou.aliyuncs.com/istio-release/pilot:1.0.0 gcr.io/istio-release/pilot:1.0.0
    docker rmi registry.cn-hangzhou.aliyuncs.com/istio-release/pilot:1.0.0
    
    docker pull registry.cn-hangzhou.aliyuncs.com/istio-release/sidecar_injector:1.0.0
    docker tag registry.cn-hangzhou.aliyuncs.com/istio-release/sidecar_injector:1.0.0 gcr.io/istio-release/sidecar_injector:1.0.0
    docker rmi registry.cn-hangzhou.aliyuncs.com/istio-release/sidecar_injector:1.0.0
    
    docker pull registry.cn-hangzhou.aliyuncs.com/istio-release/proxyv2:1.0.0
    docker tag registry.cn-hangzhou.aliyuncs.com/istio-release/proxyv2:1.0.0 gcr.io/istio-release/proxyv2:1.0.0
    docker rmi registry.cn-hangzhou.aliyuncs.com/istio-release/proxyv2:1.0.0
    
    
    docker pull registry.cn-hangzhou.aliyuncs.com/istio-release/grafana:1.0.0
    docker tag registry.cn-hangzhou.aliyuncs.com/istio-release/grafana:1.0.0 gcr.io/istio-release/grafana:1.0.0
    docker rmi registry.cn-hangzhou.aliyuncs.com/istio-release/grafana:1.0.0
    
    
    
    docker pull registry.cn-hangzhou.aliyuncs.com/istio-release/mixer:1.0.0
    docker tag registry.cn-hangzhou.aliyuncs.com/istio-release/mixer:1.0.0 gcr.io/istio-release/mixer:1.0.0
    docker rmi registry.cn-hangzhou.aliyuncs.com/istio-release/mixer:1.0.0
    
    
    docker pull registry.cn-hangzhou.aliyuncs.com/istio-release/servicegraph:1.0.0
    docker tag registry.cn-hangzhou.aliyuncs.com/istio-release/servicegraph:1.0.0 gcr.io/istio-release/servicegraph:1.0.0
    docker rmi registry.cn-hangzhou.aliyuncs.com/istio-release/servicegraph:1.0.0
    
    
    docker pull registry.cn-hangzhou.aliyuncs.com/istio-release/pilot:1.0.0
    docker tag registry.cn-hangzhou.aliyuncs.com/istio-release/pilot:1.0.0 gcr.io/istio-release/pilot:1.0.0
    docker rmi registry.cn-hangzhou.aliyuncs.com/istio-release/pilot:1.0.0
    
    docker pull registry.cn-hangzhou.aliyuncs.com/istio-release/proxy_init:1.0.0
    docker tag registry.cn-hangzhou.aliyuncs.com/istio-release/proxy_init:1.0.0 gcr.io/istio-release/proxy_init:1.0.0
    docker rmi registry.cn-hangzhou.aliyuncs.com/istio-release/proxy_init:1.0.0




特别镜像打制：

    $docker tag  registry.cn-hangzhou.aliyuncs.com/kubes/coreos_hyperkube:v1.7.6_coreos.0 quay.io/coreos/hyperkube:v1.7.6_coreos.0
    
    $docker tag  registry.cn-hangzhou.aliyuncs.com/kubes/coreos_hyperkube:v1.7.6_coreos.0 quay.io/coreos/hyperkube:v1.7.6_coreos.0