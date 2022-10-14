## 1、随记

`kubectl api-resources`来查看所有api资源

`kubectl api-versions`来查看api的版本

`kubectl explain <资源名对象名>`查看资源对象拥有的字段

```
kubectl explain pod: 列出api资源的一级字段
kubectl explain pod --recursive: 列出api资源的所有字段(包括二级、三级等)
```

`kubectl explain <资源对象名称.字段名称>`：查看api资源字段的详细信息

```
kubectl explain svc.spec.ports
```



helm查看历史：helm list -n ingress-nginx

helm删除历史：helm uninstall ingress-nginx -n ingress-nginx



查看某个命名空间下的所有资源：kubectl get all -n ingress-nginx



## 2、排错

serviceport/nodeport无法访问，service配置文件中没有配置selector或者selector的标签没匹配到。

用telnet 127.0.0.1 30001不通，telnet 10.0.0.83 30001可以。k8s nodeport好像不能用回环ip访问。

强制删除pod $ kubectl delete pod <your-pod-name> -n <name-space> --force --grace-period=0

节点挂掉or资源不够，k8s会自动为其添加taint污点，因此需要注意下，及时去除污点

节点执行kubectl命令很慢，查看cpu/内存都充足，基本上是存储的问题。etcd处理太慢，考虑将etcd迁移到SSD。



docker pull k8s.gcr.io/ingress-nginx/controller:v1.1.0报错？

1.1、Error response from daemon: Get https://k8s.gcr.io/v2/: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)

1.2、Error response from daemon: Get https://asia-east1-docker.pkg.dev/v2/k8s-artifacts-prod/images/ingress-nginx/kube-webhook-certgen/manifests/sha256:39c5b2e3310dc4264d638ad28d9d1d96c4cbb2b2dcfb52368fe4e3c63f61e10f: dial tcp 142.251.8.82:443: connect: connection refused

```
上面两种报错都是需要为docker配置网络代理（翻墙）：
# 不用设置 https-proxy，只需设置HTTP_PROXY
sudo mkdir -p /etc/systemd/system/docker.service.d 
sudo touch /etc/systemd/system/docker.service.d/proxy.conf
sudo chmod 777 /etc/systemd/system/docker.service.d/proxy.conf
sudo echo '
[Service]
Environment="HTTP_PROXY=http://proxy.xxx.com:8888/" 
Environment="HTTPS_PROXY=http://proxy.xxx.com:8888/"
' >> /etc/systemd/system/docker.service.d/proxy.conf
sudo systemctl daemon-reload
sudo systemctl restart docker
sudo systemctl restart kubelet
```

2、Error response from daemon: Get https://k8s.gcr.io/v2/: proxyconnect tcp: net/http: TLS handshake timeout

```
我的机器有点慢，尤其是在启动时，当我想启动服务时。因此，短暂的超时开始并终止了我的请求。
这是我的解决方法：
docker pull $IMAGE || docker pull $IMAGE ||  docker pull $IMAGE || docker pull $IMAGE
拉取镜像慢：加多个|| or 换个节点试试
```





## 3、k8s常用功能插件

ingress-nging安装

```
用k8s官网的helm安装，安装后需要将ingress-controller的service类型由loadbanlance改为nodeport
```

rancher安装

```
用k8s官网的helm安装，安装cert-manager卡住，可能是node节点资源不够，后面扩容后再重新安装试试
就是node资源不够，尝试扩容节点or升级节点资源。还有可能看看其他节点是否有污点，要手动去除。
后面接着安装rancher了
```

