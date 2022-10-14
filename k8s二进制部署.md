

## 1. 环境准备

### 1.1 主机规划

| IP地址                            | 机器名                            | 机器配置 | 机器角色     | 安装软件                                                     |
| --------------------------------- | --------------------------------- | -------- | ------------ | ------------------------------------------------------------ |
| 10.10.0.74 /10.10.0.80            | /                                 | /        | VIP          | 由HAProxy和keepalived组成的LB                                |
| 10.10.0.75 -77 /10.0.0.83         | master01                          | 4c * 4G  | master/node  | kube-apiserver、kube-controller-manager、kube-scheduler、etcd、haproxy、keepalived、nfs-client |
| 10.10.0.78-79 10.0.0.84/10.0.0.85 | master02/master03  /node02/node03 | 4c * 2G  | master/node  | kubelet、kube-proxy、nfs-client                              |
| 10.10.0.81/待定                   | data01                            | 2c * 2G  | data-volumes | nfs server                                                   |



### 1.2 软件版本

注意版本，不同版本可能配置发生变化，具体的可以去官网翻阅

| 软件                                                         | 版本    |
| ------------------------------------------------------------ | ------- |
| centos 7.9.2009                                              | 内核    |
| kube-apiserver、kube-controller-manager、kube-scheduler、kubelet、kube-proxy | v1.22.1 |
| etcd                                                         | v3.5.0  |
| calico                                                       | v3.19.1 |
| coredns                                                      | 1.8.4   |
| docker                                                       | 20.10.8 |
| haproxy                                                      | 1.5.18  |
| keepalived                                                   | v1.3.5  |

### 1.3 网络分配

| 网段信息    | 配置           |
| ----------- | -------------- |
| Pod网段     | 172.168.0.0/12 |
| Service网段 | 10.96.0.0/16   |

## 2. 搭建集群

### 2.1 所有机器准备工作

#### 2.1.1 修改主机名

主机名称见1.1表

```shell
hostnamectl set-hostname master01
```

#### 2.1.2 配置hosts文件

```shell
cat >> /etc/hosts << EOF
10.0.0.83 master01 data01
10.0.0.84 master02
10.0.0.85 master03
EOF
```

#### 2.1.3 关闭防火墙和selinux

```shell
systemctl stop firewalld && systemctl disable firewalld
setenforce 0
sed -i 's/^SELINUX=.\*/SELINUX=disabled' /etc/selinux/config
#查看结果
 sestatus
```

#### 2.1.4 关闭交换分区

```shell
swapoff -a
sed -ri 's/.*swap.*/#&/' /etc/fstab
echo "vm.swappiness = 0" >> /etc/sysctl.conf 
sysctl -p
```

#### 2.1.5 时间同步

```shell
1、设置时区：
查看时区：timedatectl status|grep 'Time zone'
#设置硬件时钟调整为与本地时钟一致
timedatectl set-local-rtc 1
#设置时区为上海
timedatectl set-timezone Asia/Shanghai

2、使用ntpdate同步时间
#安装ntpdate
yum -y install ntpdate
#同步时间
ntpdate -u cn.ntp.org.cn
#同步完成后,date命令查看时间是否正确
date
```

同步时间后可能部分服务器过一段时间又会出现偏差，因此最好设置crontab来定时同步时间，方法如下：

```shell
#安装crontab
yum -y install crontab
#创建crontab任务
crontab -e
#添加定时任务
*/20 * * * * /usr/sbin/ntpdate cn.ntp.org.cn > /dev/null 2>&1
# */5 * * * * /usr/sbin/ntpdate ntp1.aliyun.com
#重启crontab
service crond reload
#上面的计划任务会在每20分钟进行一次时间同步，注意/usr/sbin/ntpdate为ntpdate命令所在的绝对路径，不同的服务器可能路径不一样，可以使用which命令来找到绝对路径
```

#### 2.1.6 系统配置

```shell
#limit优化
ulimit -SHn 65535

cat <<EOF >> /etc/security/limits.conf
* soft nofile 655360
* hard nofile 131072
* soft nproc 655350
* hard nproc 655350
* soft memlock unlimited
* hard memlock unlimited
EOF
```

#### 2.1.7 加载ipvs

```shell
# 配置阿里镜像源：https://developer.aliyun.com/mirror/centos/?spm=a2c6h.25603864.0.0.132666edc041sa

yum install ipvsadm ipset sysstat conntrack libseccomp -y 

#所有节点配置ipvs模块，在内核4.19+版本nf_conntrack_ipv4已经改为nf_conntrack， 4.18以下使用nf_conntrack_ipv4即可： 
 
modprobe -- ip_vs 
modprobe -- ip_vs_rr 
modprobe -- ip_vs_wrr 
modprobe -- ip_vs_sh 
modprobe -- nf_conntrack 

 
#创建 /etc/modules-load.d/ipvs.conf 并加入以下内容： 
cat >/etc/modules-load.d/ipvs.conf <<EOF 
ip_vs 
ip_vs_lc 
ip_vs_wlc 
ip_vs_rr 
ip_vs_wrr 
ip_vs_lblc 
ip_vs_lblcr 
ip_vs_dh 
ip_vs_sh 
ip_vs_fo 
ip_vs_nq 
ip_vs_sed 
ip_vs_ftp 
ip_vs_sh 
nf_conntrack 
ip_tables 
ip_set 
xt_set 
ipt_set 
ipt_rpfilter 
ipt_REJECT 
ipip 
EOF

#设置为开机启动
systemctl enable --now systemd-modules-load.service
```

#### 2.1.8 K8s内核优化

```shell
cat <<EOF > /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
fs.may_detach_mounts = 1
vm.overcommit_memory=1
vm.panic_on_oom=0
fs.inotify.max_user_watches=89100
fs.file-max=52706963
fs.nr_open=52706963
net.netfilter.nf_conntrack_max=2310720

net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_keepalive_intvl =15
net.ipv4.tcp_max_tw_buckets = 36000
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_max_orphans = 327680
net.ipv4.tcp_orphan_retries = 3
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.ip_conntrack_max = 131072
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.tcp_timestamps = 0
net.core.somaxconn = 16384
EOF
sysctl --system

#所有节点配置完内核后，重启服务器，保证重启后内核依旧加载
reboot -h now

#重启后查看结果：
lsmod | grep --color=auto -e ip_vs -e nf_conntrack
```

#### 2.1.9 安装其他工具（可选）

```shell
yum install wget jq psmisc vim net-tools telnet yum-utils device-mapper-persistent-data lvm2 git lrzsz -y
```

### 2.2 master节点安装

#### 2.2.1 配置免密

```shell
# 在master01上操作，ssh端口进行过修改，为4956
cd /root
ssh-keygen -t rsa
for i in master02 master03;do ssh-copy-id $i;done
#测试
ssh  master03
```

#### 2.2.2 haproxy和keepalived部署高可用

##### 2.2.2.1 安装

```shell
yum install keepalived haproxy -y
```

##### 2.2.2.2 配置haproxy

```shell
cat >/etc/haproxy/haproxy.cfg<<"EOF"
global
 maxconn 2000
 ulimit-n 16384
 log 127.0.0.1 local0 err
 stats timeout 30s

defaults
 log global
 mode http
 option httplog
 timeout connect 5000
 timeout client 50000
 timeout server 50000
 timeout http-request 15s
 timeout http-keep-alive 15s

frontend k8s-master
 bind 0.0.0.0:16443
 bind 127.0.0.1:16443
 mode tcp
 option tcplog
 tcp-request inspect-delay 5s
 default_backend k8s-master

backend k8s-master
 mode tcp
 option tcplog
 option tcp-check
 balance roundrobin
 default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
 server  master01  10.0.0.83:6443 check
 server  master02  10.0.0.84:6443 check
 server  master03  10.0.0.85:6443 check
EOF
```

```shell
多余的配置：
frontend monitor-in
 bind *:33305
 mode http
 option httplog
 monitor-uri /monitor
```

##### 2.2.2.3 配置KeepAlived

每个masrer配置不一样，注意区分（网卡名称+ip）

```shell
#master01 配置：
cat >/etc/keepalived/keepalived.conf<<"EOF"
! Configuration File for keepalived
global_defs {
   router_id LVS_DEVEL
   script_user root
   enable_script_security
}
vrrp_script chk_apiserver {
   script "/etc/keepalived/check_apiserver.sh"
   interval 5
   weight -5
   fall 2
   rise 1 #检测一次成功，则认为在线
}
vrrp_instance VI_1 {
   state MASTER
   interface eth0
   mcast_src_ip 10.0.0.83
   virtual_router_id 51
   priority 100
   nopreempt
   advert_int 2
   authentication {
       auth_type PASS
       auth_pass K8SHA_KA_AUTH
   }
   virtual_ipaddress {
       10.0.0.80
   }
   track_script {
      chk_apiserver
   }
}
EOF

#Master02 配置：
cat >/etc/keepalived/keepalived.conf<<"EOF"
! Configuration File for keepalived
global_defs {
   router_id LVS_DEVEL
   script_user root
   enable_script_security
}
vrrp_script chk_apiserver {
   script "/etc/keepalived/check_apiserver.sh"
   interval 5
   weight -5
   fall 2 
   rise 1
}
vrrp_instance VI_1 {
   state BACKUP
   interface eth0
   mcast_src_ip 10.0.0.84
   virtual_router_id 51
   priority 99
   nopreempt
   advert_int 2
   authentication {
       auth_type PASS
       auth_pass K8SHA_KA_AUTH
   }
   virtual_ipaddress {
       10.0.0.80
   }
   track_script {
      chk_apiserver
   }
}
EOF

#Master03 配置：
cat >/etc/keepalived/keepalived.conf<<"EOF"
! Configuration File for keepalived
global_defs {
   router_id LVS_DEVEL
   script_user root
   enable_script_security
}
vrrp_script chk_apiserver {
   script "/etc/keepalived/check_apiserver.sh"
   interval 5
   weight -5
   fall 2 
   rise 1
}
vrrp_instance VI_1 {
   state BACKUP
   interface eth0
   mcast_src_ip 10.0.0.85
   virtual_router_id 51
   priority 98
   nopreempt
   advert_int 2
   authentication {
       auth_type PASS
       auth_pass K8SHA_KA_AUTH
   }
   virtual_ipaddress {
       10.0.0.80
   }
    track_script {
      chk_apiserver
   }
EOF
```

##### 2.2.2.4 健康检查脚本

```shell
cat > /etc/keepalived/check_apiserver.sh <<"EOF"
#!/bin/bash
err=0
for k in $(seq 1 3)
do
   check_code=$(pgrep haproxy)
   if [[ $check_code == "" ]]; then
       err=$(expr $err + 1)
       sleep 1
       continue
   else
       err=0
       break
   fi
done

if [[ $err != "0" ]]; then
   echo "systemctl stop keepalived"
   /usr/bin/systemctl stop keepalived
   exit 1
else
   exit 0
fi
EOF

chmod u+x /etc/keepalived/check_apiserver.sh
```

##### 2.2.2.5 启动服务

```shell
systemctl daemon-reload
systemctl enable --now haproxy
systemctl enable --now keepalived
```

##### 2.2.2.6  检查状态

```shell
#master01，看到vip
ip addr

#各节点测试
ping 10.0.0.80 -c 4
telnet  10.0.0.80 16443
systemctl status keepalived haproxy 
```

如果过一段时间后不能PING通VIP

```shell
# 先清理master的arp，将vip切回至master，ping vip正常,再清理slave的arp
arp -n|awk '/^[1-9]/{system("arp -d "$1)}'
```

#### 2.2.3 搭建etcd集群

##### 2.2.3.1 配置工作目录

```shell
# 在master01上创建工作目录
mkdir -p /data/k8s-work
```

##### 2.2.3.2 生成cfssl证书

安装cfssl工具

```shell
cd /data/k8s-work
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64

chmod +x cfssl*
mv cfssl_linux-amd64 /usr/local/bin/cfssl
mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
mv cfssl-certinfo_linux-amd64 /usr/local/bin/cfssl-certinfo
```

配置ca请求文件

```shell
cat > ca-csr.json <<"EOF"
{
  "CN": "kubernetes",
  "key": {
      "algo": "rsa",
      "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Guangdong",
      "L": "shenzhen",
      "O": "k8s",
      "OU": "system"
    }
  ],
  "ca": {
          "expiry": "87600h"
  }
}
EOF
```

**创建ca证书**

```shell
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

**配置ca证书策略**

```shell
cat > ca-config.json <<"EOF"
{
  "signing": {
      "default": {
          "expiry": "87600h"
        },
      "profiles": {
          "kubernetes": {
              "usages": [
                  "signing",
                  "key encipherment",
                  "server auth",
                  "client auth"
              ],
              "expiry": "87600h"
          }
      }
  }
}
EOF
```

**配置etcd请求csr文件**

```javascript
cat > etcd-csr.json <<"EOF"
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "10.0.0.83",
    "10.0.0.84",
    "10.0.0.85"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [{
    "C": "CN",
    "ST": "Guangdong",
    "L": "shenzhen",
    "O": "k8s",
    "OU": "system"
  }]
}
EOF
```

**生成证书**

```shell
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes etcd-csr.json | cfssljson  -bare etcd

ls etcd*.pem
# etcd-key.pem  etcd.pem
```

### **2.2.3.3 部署etcd集群**

**下载分发etcd软件包**

```shell
wget https://github.com/etcd-io/etcd/releases/download/v3.5.0/etcd-v3.5.0-linux-amd64.tar.gz
tar -xvf etcd-v3.5.0-linux-amd64.tar.gz
cp -p etcd-v3.5.0-linux-amd64/etcd* /usr/local/bin/
scp  etcd-v3.5.0-linux-amd64/etcd* master02:/usr/local/bin/
scp  etcd-v3.5.0-linux-amd64/etcd* master03:/usr/local/bin/
```

**创建配置文件**

```shell
cat >  etcd.conf <<"EOF"
#[Member]
ETCD_NAME="etcd1"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://10.0.0.83:2380"
ETCD_LISTEN_CLIENT_URLS="https://10.0.0.83:2379,http://127.0.0.1:2379"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://10.0.0.83:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://10.0.0.83:2379"
ETCD_INITIAL_CLUSTER="etcd1=https://10.0.0.83:2380,etcd2=https://10.0.0.84:2380,etcd3=https://10.0.0.85:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
EOF
```

```
ETCD_NAME：节点名称，集群中唯一 ETCD_DATA_DIR：数据目录 ETCD_LISTEN_PEER_URLS：集群通信监听地址 ETCD_LISTEN_CLIENT_URLS：客户端访问监听地址 ETCD_INITIAL_ADVERTISE_PEER_URLS：集群通告地址 ETCD_ADVERTISE_CLIENT_URLS：客户端通告地址 ETCD_INITIAL_CLUSTER：集群节点地址 ETCD_INITIAL_CLUSTER_TOKEN：集群Token ETCD_INITIAL_CLUSTER_STATE：加入集群的当前状态，new是新集群，existing表示加入已有集群
```

**创建启动service**

```shell
cat > etcd.service <<"EOF"
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
EnvironmentFile=-/etc/etcd/etcd.conf
WorkingDirectory=/var/lib/etcd/
ExecStart=/usr/local/bin/etcd \
  --cert-file=/etc/etcd/ssl/etcd.pem \
  --key-file=/etc/etcd/ssl/etcd-key.pem \
  --trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-cert-file=/etc/etcd/ssl/etcd.pem \
  --peer-key-file=/etc/etcd/ssl/etcd-key.pem \
  --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-client-cert-auth \
  --client-cert-auth
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

**各节点创建etcd目录**

```shell
mkdir -p /etc/etcd
mkdir -p /etc/etcd/ssl
mkdir -p /var/lib/etcd/default.etcd
```

**同步到各个节点**

```shell
cp ca*.pem /etc/etcd/ssl/
cp etcd*.pem /etc/etcd/ssl/
cp etcd.conf /etc/etcd/
cp etcd.service /usr/lib/systemd/system/
for i in master02 master03;do scp  etcd.conf $i:/etc/etcd/;done
for i in master02 master03;do scp  etcd*.pem ca*.pem $i:/etc/etcd/ssl/;done
for i in master02 master03;do scp  etcd.service $i:/usr/lib/systemd/system/;done
```

**master2和master3分别修改配置文件中etcd名字和ip**

```shell
vim /etc/etcd/etcd.conf
# etcd名字：etcd2、etcd3
```

**启动etcd集群**

```shell
systemctl daemon-reload
systemctl enable --now etcd.service
systemctl status etcd
```

**查看集群状态**

```shell
ETCD_API=3 etcdctl --write-out=table \
--cacert=/etc/etcd/ssl/ca.pem \
--cert=/etc/etcd/ssl/etcd.pem \
--key=/etc/etcd/ssl/etcd-key.pem \
--endpoints=https://10.0.0.83:2379,https://10.0.0.84:2379,https://10.0.0.85:2379 endpoint health
```

#### **2.2.4 kubernetes 部署**

[kubernetes二进制包下载页面](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.22.md#server-binaries)

##### **2.2.4.1 下载分发安装包**

```shell
wget https://dl.k8s.io/v1.22.1/kubernetes-server-linux-amd64.tar.gz
tar -xvf kubernetes-server-linux-amd64.tar.gz
cd kubernetes/server/bin/
cp kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
scp   kube-apiserver kube-controller-manager kube-scheduler kubectl master02:/usr/local/bin/
scp   kube-apiserver kube-controller-manager kube-scheduler kubectl master03:/usr/local/bin/
for i in master02 master03 ;do scp  kubelet kube-proxy $i:/usr/local/bin/;done
# for i in node01 node02 ;do scp  kubelet kube-proxy $i:/usr/local/bin/;done
```

##### **2.2.4.2 所有节点创建工作目录**

```shell
mkdir -p /etc/kubernetes/        
mkdir -p /etc/kubernetes/ssl     
mkdir -p /var/log/kubernetes    
```

##### **2.2.4.3 部署api-server**

创建apiserver-csr 

```shell
cat > kube-apiserver-csr.json << "EOF"
{
"CN": "kubernetes",
  "hosts": [
    "127.0.0.1",
    "10.0.0.80",
    "10.0.0.83",
    "10.0.0.84",
    "10.0.0.85",
    "10.0.0.86",
    "10.0.0.87",
    "10.0.0.88",
    "10.0.0.89",
    "10.0.0.90",
    "10.96.0.1",
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Guangdong",
      "L": "shenzhen",
      "O": "k8s",
      "OU": "system"
    }
  ]
}
EOF
```

```
如果 hosts 字段不为空则需要指定授权使用该证书的 IP 或域名列表。由于该证书被 集群使用，需要将节点的IP都填上，为了方便后期扩容可以多写几个预留的IP。同时还需要填写 service 网络的首个IP(一般是 kube-apiserver 指定的 service-cluster-ip-range 网段的第一个IP，如 10.96.0.1)。
```

**生成证书和token文件**

```shell
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-apiserver-csr.json | cfssljson -bare kube-apiserver

cat > token.csv << EOF
$(head -c 16 /dev/urandom | od -An -t x | tr -d ' '),kubelet-bootstrap,10001,"system:kubelet-bootstrap"
EOF
```

**创建配置文件**

```shell
cat > kube-apiserver.conf << "EOF"
KUBE_APISERVER_OPTS="--enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \
  --anonymous-auth=false \
  --bind-address=10.0.0.83 \
  --secure-port=6443 \
  --advertise-address=10.0.0.83 \
  --insecure-port=0 \
  --authorization-mode=Node,RBAC \
  --runtime-config=api/all=true \
  --enable-bootstrap-token-auth \
  --service-cluster-ip-range=10.96.0.0/16 \
  --token-auth-file=/etc/kubernetes/token.csv \
  --service-node-port-range=30000-50000 \
  --tls-cert-file=/etc/kubernetes/ssl/kube-apiserver.pem  \
  --tls-private-key-file=/etc/kubernetes/ssl/kube-apiserver-key.pem \
  --client-ca-file=/etc/kubernetes/ssl/ca.pem \
  --kubelet-client-certificate=/etc/kubernetes/ssl/kube-apiserver.pem \
  --kubelet-client-key=/etc/kubernetes/ssl/kube-apiserver-key.pem \
  --service-account-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --service-account-signing-key-file=/etc/kubernetes/ssl/ca-key.pem  \
  --service-account-issuer=api \
  --etcd-cafile=/etc/etcd/ssl/ca.pem \
  --etcd-certfile=/etc/etcd/ssl/etcd.pem \
  --etcd-keyfile=/etc/etcd/ssl/etcd-key.pem \
  --etcd-servers=https://10.0.0.83:2379,https://10.0.0.84:2379,https://10.0.0.85:2379 \
  --enable-swagger-ui=true \
  --allow-privileged=true \
  --apiserver-count=3 \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=3 \
  --audit-log-maxsize=100 \
  --audit-log-path=/var/log/kube-apiserver-audit.log \
  --event-ttl=1h \
  --alsologtostderr=true \
  --logtostderr=false \
  --log-dir=/var/log/kubernetes \
  --v=4"
EOF
```

[kube-apiserver命令行参考](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kube-apiserver/)

**创建apiserver服务启动文件**

```shell
cat > kube-apiserver.service << "EOF"
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
After=etcd.service
Wants=etcd.service

[Service]
EnvironmentFile=-/etc/kubernetes/kube-apiserver.conf
ExecStart=/usr/local/bin/kube-apiserver $KUBE_APISERVER_OPTS
Restart=on-failure
RestartSec=5
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

**同步相关文件到各个节点**

```shell
cp ca*.pem /etc/kubernetes/ssl/
cp kube-apiserver*.pem /etc/kubernetes/ssl/
cp token.csv /etc/kubernetes/
cp kube-apiserver.conf /etc/kubernetes/ 
cp kube-apiserver.service /usr/lib/systemd/system/
scp   token.csv master02:/etc/kubernetes/
scp   token.csv master03:/etc/kubernetes/
scp   kube-apiserver*.pem master02:/etc/kubernetes/ssl/
scp   kube-apiserver*.pem master03:/etc/kubernetes/ssl/
scp   ca*.pem master02:/etc/kubernetes/ssl/
scp   ca*.pem master03:/etc/kubernetes/ssl/
scp   kube-apiserver.conf master02:/etc/kubernetes/
scp   kube-apiserver.conf master03:/etc/kubernetes/
scp   kube-apiserver.service master02:/usr/lib/systemd/system/
scp   kube-apiserver.service master03:/usr/lib/systemd/system/
```

**master2和master3配置文件的IP地址修改为实际的本机IP**

```shell
vim /etc/kubernetes/kube-apiserver.conf
```

**启动服务**

```shell
systemctl daemon-reload
systemctl enable --now kube-apiserver

systemctl status kube-apiserver
# 测试
curl --insecure https://10.0.0.83:6443/
curl --insecure https://10.0.0.84:6443/
curl --insecure https://10.0.0.85:6443/
curl --insecure https://10.0.0.80:16443/
```

##### **2.2.4.4 部署kubectl**

**创建csr请求文件**

```shell
cat > admin-csr.json << "EOF"
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Guangdong",
      "L": "shenzhen",
      "O": "system:masters",             
      "OU": "system"
    }
  ]
}
EOF
```

```
说明： 后续 kube-apiserver 使用 RBAC 对客户端(如 kubelet、kube-proxy、Pod)请求进行授权； kube-apiserver 预定义了一些 RBAC 使用的 RoleBindings，如 cluster-admin 将 Group system:masters 与 Role cluster-admin 绑定，该 Role 授予了调用kube-apiserver 的所有 API的权限； O指定该证书的 Group 为 system:masters，kubelet 使用该证书访问 kube-apiserver 时 ，由于证书被 CA 签名，所以认证通过，同时由于证书用户组为经过预授权的 system:masters，所以被授予访问所有 API 的权限； 注： 这个admin 证书，是将来生成管理员用的kube config 配置文件用的，现在我们一般建议使用RBAC 来对kubernetes 进行角色权限控制， kubernetes 将证书中的CN 字段 作为User， O 字段作为 Group； "O": "system:masters", 必须是system:masters，否则后面kubectl create clusterrolebinding报错。
```

**生成证书**

```shell
[root@master1 work]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin
[root@master1 work]# cp admin*.pem /etc/kubernetes/ssl/
```

**kubeconfig配置**

kube.config 为 kubectl 的配置文件，包含访问 apiserver 的所有信息，如 apiserver 地址、CA 证书和自身使用的证书

```shell
kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=https://10.0.0.80:16443 --kubeconfig=kube.config
# 创建kube.config文件，并定义此文件中一个k8s集群名为kubernetes的地址为https://10.0.0.80:16443

kubectl config set-credentials admin --client-certificate=admin.pem --client-key=admin-key.pem --embed-certs=true --kubeconfig=kube.config
# 在kube.config文件中创建一个名为admin的用户凭证，此凭证使用的admin证书

kubectl config set-context kubernetes --cluster=kubernetes --user=admin --kubeconfig=kube.config
# 在kube.config文件中设置一个上下文名为kubernetes，这个上下文环境中用的admin凭证连接的kubernetes集群

kubectl config use-context kubernetes --kubeconfig=kube.config
# 设置kube.config文件中默认使用上下文环境kubernetes

mkdir ~/.kube
cp kube.config ~/.kube/config
# kubectl create clusterrolebinding kube-apiserver:kubelet-apis --clusterrole=system:kubelet-api-admin --user kubernetes --kubeconfig=~/.kube/config
# 上面这行不用执行，可能k8s新版本已经自动创建了这个clusterrolebinding 
# --kubeconfig指定认证的kubeconfig文件，不指定默认去家目录下找
```

查看集群状态

```shell
export KUBECONFIG=$HOME/.kube/config

kubectl cluster-info
kubectl get componentstatuses
kubectl get all --all-namespaces
```

**同步kubectl配置文件到其他节点**

```shell
scp    /root/.kube/config master02:/root/.kube/
scp    /root/.kube/config master03:/root/.kube/
```

**配置kubectl子命令补全**

```shell
yum install -y bash-completion
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
kubectl completion bash > ~/.kube/completion.bash.inc
source '/root/.kube/completion.bash.inc'  
source $HOME/.bash_profile
```

##### **2.2.4.5 部署kube-controller-manager**

**创建csr请求文件**

```shell
cat > kube-controller-manager-csr.json << "EOF"
{
    "CN": "system:kube-controller-manager",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "hosts": [
      "127.0.0.1",
      "10.0.0.83",
      "10.0.0.84",
      "10.0.0.85"
    ],
    "names": [
      {
        "C": "CN",
        "ST": "Guangdong",
        "L": "shenzhen",
        "O": "system:kube-controller-manager",
        "OU": "system"
      }
    ]
}
EOF
```

```
hosts 列表包含所有 kube-controller-manager 节点 IP； CN 为 system:kube-controller-manager、O 为 system:kube-controller-manager，kubernetes 内置的 ClusterRoleBindings system:kube-controller-manager 赋予 kube-controller-manager 工作所需的权限
```

**生成证书**

```javascript
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager

ls kube-controller-manager*.pem
```

**创建kube-controller-manager的kube-controller-manager.kubeconfig**

```shell
kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=https://10.0.0.80:16443 --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-credentials system:kube-controller-manager --client-certificate=kube-controller-manager.pem --client-key=kube-controller-manager-key.pem --embed-certs=true --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-context system:kube-controller-manager --cluster=kubernetes --user=system:kube-controller-manager --kubeconfig=kube-controller-manager.kubeconfig

kubectl config use-context system:kube-controller-manager --kubeconfig=kube-controller-manager.kubeconfig
```

**创建配置文件kube-controller-manager.conf**

```shell
cat > kube-controller-manager.conf << "EOF"
KUBE_CONTROLLER_MANAGER_OPTS="--port=0 \
  --secure-port=10257 \
  --bind-address=127.0.0.1 \
  --kubeconfig=/etc/kubernetes/kube-controller-manager.kubeconfig \
  --service-cluster-ip-range=10.96.0.0/16 \
  --cluster-name=kubernetes \
  --cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem \
  --cluster-signing-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --allocate-node-cidrs=true \
  --cluster-cidr=172.168.0.0/16 \
  --experimental-cluster-signing-duration=87600h \
  --root-ca-file=/etc/kubernetes/ssl/ca.pem \
  --service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --leader-elect=true \
  --feature-gates=RotateKubeletServerCertificate=true \
  --controllers=*,bootstrapsigner,tokencleaner \
  --horizontal-pod-autoscaler-sync-period=10s \
  --tls-cert-file=/etc/kubernetes/ssl/kube-controller-manager.pem \
  --tls-private-key-file=/etc/kubernetes/ssl/kube-controller-manager-key.pem \
  --use-service-account-credentials=true \
  --alsologtostderr=true \
  --logtostderr=false \
  --log-dir=/var/log/kubernetes \
  --v=2"
EOF
```

[kube-controller-manager命令行参考](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kube-controller-manager/)

**创建启动文件**

```shell
cat > kube-controller-manager.service << "EOF"
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=-/etc/kubernetes/kube-controller-manager.conf
ExecStart=/usr/local/bin/kube-controller-manager $KUBE_CONTROLLER_MANAGER_OPTS
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

**同步相关文件到各个节点**

```shell
cp kube-controller-manager*.pem /etc/kubernetes/ssl/
cp kube-controller-manager.kubeconfig /etc/kubernetes/
cp kube-controller-manager.conf /etc/kubernetes/
cp kube-controller-manager.service /usr/lib/systemd/system/
scp  kube-controller-manager*.pem master02:/etc/kubernetes/ssl/
scp  kube-controller-manager*.pem master03:/etc/kubernetes/ssl/
scp  kube-controller-manager.kubeconfig  master02:/etc/kubernetes/
scp  kube-controller-manager.kubeconfig master03:/etc/kubernetes/
scp  kube-controller-manager.service master02:/usr/lib/systemd/system/
scp  kube-controller-manager.service master03:/usr/lib/systemd/system/
scp  kube-controller-manager.conf master02:/etc/kubernetes/
scp  kube-controller-manager.conf master03:/etc/kubernetes/

#查看证书
openssl x509 -in /etc/kubernetes/ssl/kube-controller-manager.pem -noout -text
```

**启动服务**

```shell
systemctl daemon-reload 
systemctl enable --now kube-controller-manager
systemctl status kube-controller-manager
```

##### **2.2.4.6 部署kube-scheduler**

**创建csr请求文件**

```shell
cat > kube-scheduler-csr.json << "EOF"
{
    "CN": "system:kube-scheduler",
    "hosts": [
      "127.0.0.1",
      "10.0.0.83",
      "10.0.0.84",
      "10.0.0.85"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
      {
        "C": "CN",
        "ST": "Guangdong",
        "L": "shenzhen",
        "O": "system:kube-scheduler",
        "OU": "system"
      }
    ]
}
EOF
```

**生成证书**

```shell
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-scheduler-csr.json | cfssljson -bare kube-scheduler

ls kube-scheduler*.pem
```

**创建kube-scheduler的kubeconfig**

```shell
kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=https://10.0.0.80:16443 --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler --client-certificate=kube-scheduler.pem --client-key=kube-scheduler-key.pem --embed-certs=true --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-context system:kube-scheduler --cluster=kubernetes --user=system:kube-scheduler --kubeconfig=kube-scheduler.kubeconfig

kubectl config use-context system:kube-scheduler --kubeconfig=kube-scheduler.kubeconfig
```

**创建配置文件**

```shell
cat > kube-scheduler.conf << "EOF"
KUBE_SCHEDULER_OPTS="--address=127.0.0.1 \
--kubeconfig=/etc/kubernetes/kube-scheduler.kubeconfig \
--leader-elect=true \
--alsologtostderr=true \
--logtostderr=false \
--log-dir=/var/log/kubernetes \
--v=2"
EOF
```

[kube-scheduler命令行参考](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kube-scheduler/)

**创建服务启动文件**

```shell
cat > kube-scheduler.service << "EOF"
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=-/etc/kubernetes/kube-scheduler.conf
ExecStart=/usr/local/bin/kube-scheduler $KUBE_SCHEDULER_OPTS
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

**同步相关文件到各个节点**

```shell
cp kube-scheduler*.pem /etc/kubernetes/ssl/
cp kube-scheduler.kubeconfig /etc/kubernetes/
cp kube-scheduler.conf /etc/kubernetes/
cp kube-scheduler.service /usr/lib/systemd/system/
scp  kube-scheduler*.pem master02:/etc/kubernetes/ssl/
scp  kube-scheduler*.pem master03:/etc/kubernetes/ssl/
scp  kube-scheduler.kubeconfig kube-scheduler.conf master02:/etc/kubernetes/
scp  kube-scheduler.kubeconfig kube-scheduler.conf master03:/etc/kubernetes/
scp  kube-scheduler.service master02:/usr/lib/systemd/system/
scp  kube-scheduler.service master03:/usr/lib/systemd/system/
```

**启动服务**

```shell
systemctl daemon-reload
systemctl enable --now kube-scheduler
systemctl status kube-scheduler
```

### 2.3 work节点安装

#### **2.3.1** [**docker**](https://cloud.tencent.com/product/tke?from=10680)**安装配置**

```shell
sudo yum remove docker-ce docker-ce-cli  -y
下载docker-ce 对应的安装包
 wget https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/centos/7/x86_64/stable/Packages/docker-ce-cli-20.10.8-3.el7.x86_64.rpm
 wget https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/centos/7/x86_64/stable/Packages/containerd.io-1.4.9-3.1.el7.x86_64.rpm  
 wget https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/centos/7/x86_64/stable/Packages/docker-ce-rootless-extras-20.10.8-3.el7.x86_64.rpm
wget https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/centos/7/x86_64/stable/Packages/docker-ce-20.10.8-3.el7.x86_64.rpm  
 wget https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/centos/7/x86_64/stable/Packages/docker-scan-plugin-0.8.0-3.el7.x86_64.rpm
 
yum install *.rpm -y
systemctl enable --now docker

#curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://f1361db2.m.daocloud.io

cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ],
   "registry-mirrors": ["http://f1361db2.m.daocloud.io"]
}
EOF

systemctl restart docker
```

#### **2.3.2 kubernetes部署**

##### **2.3.2.1 部署kubelet**

以下操作在master01上操作  **创建kubelet-bootstrap.kubeconfig**

```shell
# kubelet-bootstrap.kubeconfig用于给新增node节点自动颁发证书的。新增node节点先检查家目录/.kube/config是否存在，如果不存在则会用kubelet-bootstrap.kubeconfig连接k8s集群请求自动为其颁发证书。证书颁发后，后面kubelet就用颁发的新证书和apiserver通信。（因为node节点比较多，如果每个node节点的kubelet都手动创建证书，管理很吃力，因此需要用bootstrap自动颁发证书）
BOOTSTRAP_TOKEN=$(awk -F "," '{print $1}' /etc/kubernetes/token.csv)

kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=https://10.0.0.80:16443 --kubeconfig=kubelet-bootstrap.kubeconfig

kubectl config set-credentials kubelet-bootstrap --token=${BOOTSTRAP_TOKEN} --kubeconfig=kubelet-bootstrap.kubeconfig

kubectl config set-context default --cluster=kubernetes --user=kubelet-bootstrap --kubeconfig=kubelet-bootstrap.kubeconfig

kubectl config use-context default --kubeconfig=kubelet-bootstrap.kubeconfig

# kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap --kubeconfig=/root/.kube/config
# 上面这行不用执行，可能k8s新版本已经自动创建了这个clusterrolebinding
```

**创建配置文件**

```shell
cat > kubelet.json << "EOF"
{
  "kind": "KubeletConfiguration",
  "apiVersion": "kubelet.config.k8s.io/v1beta1",
  "authentication": {
    "x509": {
      "clientCAFile": "/etc/kubernetes/ssl/ca.pem"
    },
    "webhook": {
      "enabled": true,
      "cacheTTL": "2m0s"
    },
    "anonymous": {
      "enabled": false
    }
  },
  "authorization": {
    "mode": "Webhook",
    "webhook": {
      "cacheAuthorizedTTL": "5m0s",
      "cacheUnauthorizedTTL": "30s"
    }
  },
  "address": "10.0.0.84",
  "port": 10250,
  "readOnlyPort": 10255,
  "cgroupDriver": "systemd",                    
  "hairpinMode": "promiscuous-bridge",
  "serializeImagePulls": false,
  "clusterDomain": "cluster.local.",
  "clusterDNS": ["10.96.0.2"]
}
EOF
```

*clusterDNS的配置，后面配置coredns会用到*

**创建启动文件**

```shell
cat > kubelet.service << "EOF"
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
ExecStart=/usr/local/bin/kubelet \
  --bootstrap-kubeconfig=/etc/kubernetes/kubelet-bootstrap.kubeconfig \
  --cert-dir=/etc/kubernetes/ssl \
  --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \
  --config=/etc/kubernetes/kubelet.json \
  --network-plugin=cni \
  --rotate-certificates \
  --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.2 \
  --alsologtostderr=true \
  --logtostderr=false \
  --log-dir=/var/log/kubernetes \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

[kubelet命令行参考](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kubelet/)

**同步相关文件到各个节点**

```shell
cp kubelet-bootstrap.kubeconfig /etc/kubernetes/
cp kubelet.json /etc/kubernetes/
cp kubelet.service /usr/lib/systemd/system/

for i in  master02 master03 ;do scp  kubelet-bootstrap.kubeconfig kubelet.json $i:/etc/kubernetes/;done
#for i in  master02 master03 ;do scp  ca.pem $i:/etc/kubernetes/ssl/;done
for i in master02 master03 ;do scp  kubelet.service $i:/usr/lib/systemd/system/;done

#for i in  node01 node02 ;do scp  kubelet-bootstrap.kubeconfig kubelet.json $i:/etc/kubernetes/;done
#for i in  node01 node02 ;do scp  ca.pem $i:/etc/kubernetes/ssl/;done
#for i in node01 node02 ;do scp  kubelet.service $i:/usr/lib/systemd/system/;done
```

**每个node节点修改kubelet.json中的ip**

在各个节点执行

```shell
mkdir -p /var/lib/kubelet
mkdir -p /var/log/kubernetes
systemctl daemon-reload
systemctl enable --now kubelet

systemctl status kubelet
```

确认kubelet服务启动成功后，接着到master上Approve一下bootstrap请求。  `kubectl get csr | grep Pending | awk '{print $1}' | xargs kubectl certificate approve`  查看一下node是否加入成功  `kubectl get nodes`

##### **2.3.2.2 部署kube-proxy**

**创建csr请求文件**

```shell
cat > kube-proxy-csr.json << "EOF"
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Guangdong",
      "L": "shenzhen",
      "O": "k8s",
      "OU": "system"
    }
  ]
}
EOF
```

**生成证书**

```javascript
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy
ls kube-proxy*.pem
```

**创建kubeconfig文件**

```shell
kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=https://10.0.0.80:16443 --kubeconfig=kube-proxy.kubeconfig

kubectl config set-credentials kube-proxy --client-certificate=kube-proxy.pem --client-key=kube-proxy-key.pem --embed-certs=true --kubeconfig=kube-proxy.kubeconfig

kubectl config set-context default --cluster=kubernetes --user=kube-proxy --kubeconfig=kube-proxy.kubeconfig

kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

**创建kube-proxy配置文件**

```shell
cat > kube-proxy.yaml << "EOF"
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 10.0.0.84
clientConnection:
  kubeconfig: /etc/kubernetes/kube-proxy.kubeconfig
clusterCIDR: 172.168.0.0/12
healthzBindAddress: 10.0.0.84:10256
kind: KubeProxyConfiguration
metricsBindAddress: 10.0.0.84:10249
mode: "ipvs"
EOF
```

[kube-proxy命令行参考](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kube-proxy/)

**创建服务启动文件**

```shell
cat >  kube-proxy.service << "EOF"
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
WorkingDirectory=/var/lib/kube-proxy
ExecStart=/usr/local/bin/kube-proxy \
  --config=/etc/kubernetes/kube-proxy.yaml \
  --alsologtostderr=true \
  --logtostderr=false \
  --log-dir=/var/log/kubernetes \
  --v=2
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

**同步文件到各个节点**

```shell
cp kube-proxy*.pem /etc/kubernetes/ssl/
cp kube-proxy.kubeconfig kube-proxy.yaml /etc/kubernetes/
cp kube-proxy.service /usr/lib/systemd/system/

for i in master02 master03;do scp  kube-proxy.kubeconfig kube-proxy.yaml $i:/etc/kubernetes/;done
for i in master02 master03;do scp  kube-proxy.service $i:/usr/lib/systemd/system/;done
# kube-proxy*.pem不用同步，kube-proxy服务启动不需要

# for i in node01 node02;do scp  kube-proxy.kubeconfig kube-proxy.yaml $i:/etc/kubernetes/;done
# for i in node01 node02;do scp  kube-proxy.service $i:/usr/lib/systemd/system/;done
```

**在各node修改kube-proxy.yaml中address修改为各节点的实际IP**

```shell
vim /etc/kubernetes/kube-proxy.yaml
```

**启动服务**

```shell
mkdir -p /var/lib/kube-proxy
systemctl daemon-reload
systemctl enable --now kube-proxy

systemctl status kube-proxy
```

### 2.4 部署网络组件

#### **2.4.1 安装calico**

```shell
wget https://docs.projectcalico.org/v3.19/manifests/calico.yaml
kubectl apply -f calico.yaml 
```

查看状态，各个节点，均为Ready状态

```shell
kubectl get pods -A
kubectl get nodes
```

#### **2.4.2  部署coredns**

```shell
cat >  coredns.yaml << "EOF"
apiVersion: v1
kind: ServiceAccount
metadata:
  name: coredns
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:coredns
rules:
  - apiGroups:
    - ""
    resources:
    - endpoints
    - services
    - pods
    - namespaces
    verbs:
    - list
    - watch
  - apiGroups:
    - discovery.k8s.io
    resources:
    - endpointslices
    verbs:
    - list
    - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:coredns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:coredns
subjects:
- kind: ServiceAccount
  name: coredns
  namespace: kube-system
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
          lameduck 5s
        }
        ready
        kubernetes cluster.local  in-addr.arpa ip6.arpa {
          fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . /etc/resolv.conf {
          max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/name: "CoreDNS"
spec:
  # replicas: not specified here:
  # 1. Default is 1.
  # 2. Will be tuned in real time if DNS horizontal auto-scaling is turned on.
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  selector:
    matchLabels:
      k8s-app: kube-dns
  template:
    metadata:
      labels:
        k8s-app: kube-dns
    spec:
      priorityClassName: system-cluster-critical
      serviceAccountName: coredns
      tolerations:
        - key: "CriticalAddonsOnly"
          operator: "Exists"
      nodeSelector:
        kubernetes.io/os: linux
      affinity:
         podAntiAffinity:
           preferredDuringSchedulingIgnoredDuringExecution:
           - weight: 100
             podAffinityTerm:
               labelSelector:
                 matchExpressions:
                   - key: k8s-app
                     operator: In
                     values: ["kube-dns"]
               topologyKey: kubernetes.io/hostname
      containers:
      - name: coredns
        image: coredns/coredns:1.8.4
        imagePullPolicy: IfNotPresent
        resources:
          limits:
            memory: 170Mi
          requests:
            cpu: 100m
            memory: 70Mi
        args: [ "-conf", "/etc/coredns/Corefile" ]
        volumeMounts:
        - name: config-volume
          mountPath: /etc/coredns
          readOnly: true
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        - containerPort: 9153
          name: metrics
          protocol: TCP
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            add:
            - NET_BIND_SERVICE
            drop:
            - all
          readOnlyRootFilesystem: true
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
        readinessProbe:
          httpGet:
            path: /ready
            port: 8181
            scheme: HTTP
      dnsPolicy: Default
      volumes:
        - name: config-volume
          configMap:
            name: coredns
            items:
            - key: Corefile
              path: Corefile
---
apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
  annotations:
    prometheus.io/port: "9153"
    prometheus.io/scrape: "true"
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: "CoreDNS"
spec:
  selector:
    k8s-app: kube-dns
  clusterIP: 10.96.0.2
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
  - name: metrics
    port: 9153
    protocol: TCP
 
EOF
```

*clusterIP为：10.96.0.2（kubelet配置文件中的clusterDNS）*

```shell
kubectl apply -f coredns.yaml
```

#### **2.4.3  部署nginx验证**

```shell
cat >  nginx.yaml  << "EOF"
---
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx-controller
spec:
  replicas: 2
  selector:
    name: nginx
  template:
    metadata:
      labels:
        name: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.19.6
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service-nodeport
spec:
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30001
      protocol: TCP
  type: NodePort
  selector:
    name: nginx
EOF
```

部署

```shell
kubectl apply -f nginx.yaml
kubectl get svc
kubectl get pods -o wide
```

访问nginx验证

删除nginx

```shell
kubectl delete -f nginx.yaml
```