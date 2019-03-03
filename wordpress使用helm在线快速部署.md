## helm实战应用部署wordpress

作者：流雨声

根据之前的教程我们应该部署过了helm并且可以进行国内helm源的替换

1 我们执行部署命令(我们禁止使用存储，因为资源有限)：

    $helm install --name wordpress-test --set "persistence.enabled=false,mariadb.persistence.enabled=false" stable/wordpress

2 根据提示命令执行如下的操作得到访问url和登录用户名密码：

    NOTES:
    1. Get the WordPress URL:
    
      Or running:
    
      $export NODE_PORT=$(kubectl get --namespace default -o jsonpath="{.spec.ports[0].nodePort}" services wordpress-test-wordpress)
      $export NODE_IP=$(kubectl get nodes --namespace default -o jsonpath="{.items[0].status.addresses[0].address}")
      echo http://$NODE_IP:$NODE_PORT/admin
    
    2. Login with the following credentials to see your blog
    
      $echo Username: user
      $echo Password: $(kubectl get secret --namespace default wordpress-test-wordpress -o jsonpath="{.data.wordpress-password}" | base64 --decode)

实验验证我们得到的url地址为：

    http://192.168.131.10:30264/admin

之后通过浏览器进行访问。

签到名单: 2019/3/3

签到名单:2019/3/4
