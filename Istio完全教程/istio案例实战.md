# istio-1.0.0部署和试用

### 1\. 安装

##### 1.1 获取安装包

到istio的[release地址](https://github.com/istio/istio/releases)下载安装包,我目前使用1.0.0版：

xxxxxxxxxx

$ wget https://github.com/istio/istio/releases/download/1.0.0/istio-1.0.0-linux.tar.gz

注：如果国内网络下载存在问题，那么就直接下载github压缩包。

解压：

```
$ tar -zxvf istio-1.0.0-linux.tar.gz
```

#### 1.3 部署istio

##### 1.3.1 创建istio的自定义资源类型CRD

```
$ kubectl apply -f istio-1.0.0/install/kubernetes/helm/istio/templates/crds.yaml
```

##### 1.3.2 安装istio核心组件

核心组件的部署，官方给出了四种方式，我选择了“without mutual TLS authentication between sidecars”的方式：

```
$ kubectl apply -f istio-1.0.0/install/kubernetes/istio-demo.yaml
```

**注意：**

启动文件默认配置的通过外部LoadBalancer访问istio-ingressgateway，如果没有外部LoadBalancer，需要修改启动文件使用NodePort访问istio-ingressgateway：sed -i 's/LoadBalancer/NodePort/g' istio-1.0.0/install/kubernetes/istio-demo.yaml

gcr.io和quay.io相关的镜像下载不了的话可以替换为我做好的墙内的镜像：

##### 1.3.3 验证安装是否成功

查看是否所有服务和pod都正常：

- kubectl get svc -n istio-system：



参考实战案例：

### [https://blog.csdn.net/liukuan73/article/details/81165716](https://blog.csdn.net/liukuan73/article/details/81165716)




