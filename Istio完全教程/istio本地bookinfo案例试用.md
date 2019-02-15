### istio 1.0 BookInfo使用教程

##### 安装方式：

1  helm

2  yaml文件

注释参考:

###### BookInfo官方案例分析

![5c6611ea21451](https://i.loli.net/2019/02/15/5c6611ea21451.png)

- productpage 调用details和reviews渲染页面
- details包含书本信息
- reviews 书本反馈，调用ratings服务
- ratings 书本租借信息

reviews服务有三个版本:

- V1 不请求ratings
- V2 请求ratings，返回1到5个黑星
- V3 请求ratings，返回1到5个红星

数据平面：

![5c6611faf1864](https://i.loli.net/2019/02/15/5c6611faf1864.png)

`$kubectl apply -n istio-test -f istio-1.0.0/samples/bookinfo/platform/kube/bookinfo.yaml`

使用treafik将服务访问放出来。

`apiVersion: extensions/v1beta1`

`kind: Ingress`

`metadata:`

  `name: istio-test-ingress`

  `namespace: istio-test`

  `annotations:`

```
`kubernetes.io/ingress.class: traefik`
```

`spec:`

  `rules:`

  `\- host: traefix-productpage.com`

```
`http:`

 `paths:`

 `\- backend:`

    `serviceName: productpage`

   `servicePort: 9080`
```

#### 智能路由

##### 请求路由 request routing

把所有路由切换到v1版本

```
$kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

这样执行完后，不管怎么刷页面，我们都看不到星星，因为v1版本没星。

##### 根据用户路由

```
$kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml
```

你会发现用正常用户登录就能看到黑星星，而其它方式看到的页面都是无星星

![C:\Users\Administrator\Desktop\他](C:\Users\Administrator\Desktop\他.png)

##### 故障注入 Fault injection

```
$kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml

$kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml
```

假设代码里有个bug，用户jason, reviews:v2 访问ratings时会卡10s, 我们任然希望端到端的测试能正常走完

```
$kubectl apply -f samples/bookinfo/networking/virtual-service-ratings-test-delay.yaml
```

这时访问页面显然会出错，因为我们希望7s内能返回,这样我们就发现了一个延迟的bug。

所以我们就可能通过故障注入去发现这些异常现象。

#### 链路切换 Traffic Shifting

我们先把50%流量发送给reviews:v1 50%流量发送给v3，然后再把100%的流量都切给v3

##### 把100%流量切到v1

```
$kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

不论刷几遍，都没有星星

##### v1 v3各50%流量

```
$kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml
```

##### 全切v3

```
$kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-v3.yaml
```

这时不管怎么刷都是红心了.
