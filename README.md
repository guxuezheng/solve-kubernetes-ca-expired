  
记kubernetes证书过期的引发的集群大范围不可用问题,及解决办法



# **问题:**

公司服务器环境突然无法访问,查看kube-apiserver日志报如下错误,

![](file:///C:\Users\ASUS\AppData\Local\Temp\ksohtml28772\wps1.jpg)

apiserver日志报:context deadline exceeded，此时apiserver容器已经退出,平台中大量管理容器启动异常。



# **查看证书过期时间:**

首先查看证书有效期:

\[root@kubemaster manifests\]\# openssl x509 -in /etc/kubernetes/pki/apiserver-etcd-client.crt -noout -dates

notBefore=Sep  7 03:38:14 2018 GMT

notAfter=Sep  8 03:38:09 2019 GMT

\[root@kubemaster manifests\]\# openssl x509 -in /etc/kubernetes/pki/ca.crt -noout -dates

notBefore=Sep  7 03:38:12 2018 GMT

notAfter=Sep  4 03:38:12 2028 GMT

可以发现除了根证书之外，其他证书的有效期都是一年，根证书的有效期为10年,由于二级证书是由根证书签发，所以我们需要把其他所有过期的证书全部替换掉。



# **备份数据:**

首先备份所有证书数据,目录地址/etc/kubernetes/pki/

备份好数据后,将pki目录下的标红文件全部删除，如下:

\[root@kubemaster kubernetes\]\# tree pki

pki

├── apiserver.crt

├── apiserver-etcd-client.crt

├── apiserver-etcd-client.key

├── apiserver.key

├── apiserver-kubelet-client.crt

├── apiserver-kubelet-client.key

├── ca.crt

├── ca.key

├── etcd

│├── ca.crt

│├── ca.key

│├── healthcheck-client.crt

│├── healthcheck-client.key

│├── peer.crt

│├── peer.key

│├── server.crt

│└── server.key

├── front-proxy-ca.crt

├── front-proxy-ca.key

├── front-proxy-client.crt

├── front-proxy-client.key

├── sa.key

├── sa.pub

仅保留根证书和sa相关公钥和私钥。重点说下sa.pub和sa.key，是一对秘钥对，用于加密和解密serviceaccount所携带的请求的，这里千万别进行更新，更新的话，将造成解码失败，注意:etcd目录下的根证书与pki下的相同，删除后的树形结构:

├── ca.crt

├── ca.key

├── etcd

│├── ca.crt

│├── ca.key

├── sa.key

├── sa.pub



# **更新证书:**

由于使用kubeadm构建集群，所有的证书和密码都可以通过kubeadm生成，此处我们也使用kubeadm。

kubeadm创建集群证书时，需要用到一个对集群信息描述的配置文件,最便捷的做法是通过如下命令获取:

kubeadm config view&gt; $file.config

但此命令需要连接apiserver，此时apiserver的容器大概率已经挂掉了，所以我们需要自己手动编写这个配置文件,

我从其他环境找了一个，如果没有其他环境，可以找下自己的安装脚本里，定义kubeadm的配置文件。

配置文件中，重点关注如下两个位置的配置信息，api-advertiseAddress和etcd-dataDir，修改好后，保存到一个文件中，我保存到了kubeadm.config中。

api:

  advertiseAddress: 192.9.200.77

  bindPort: 6443

  controlPlaneEndpoint: ""

...

...

...

...

clusterName: kubernetes

etcd:

  local:

    dataDir: /var/lib/etcd

    image: ""

...

...

...

...

更新秘钥

将所有的需要的证书和密码都进行更新:



kubeadm alpha phase certs etcd-healthcheck-client --configkubeadm.config

kubeadm alpha phase certs etcd-peer --configkubeadm.config

kubeadm alpha phase certs etcd-server --configkubeadm.config

kubeadm alpha phase certs front-proxy-client--cakubeadm.config

kubeadm alpha phase certs front-proxy-client--configkubeadm.config

kubeadm alpha phase certs apiserver-etcd-client --configkubeadm.config

kubeadm alpha phase certs apiserver-kubelet-client --configkubeadm.config

kubeadm alpha phase certs apiserver --configkubeadm.config

此时所有的证书都将重新生成:

树形结构如下:

![](file:///C:\Users\ASUS\AppData\Local\Temp\ksohtml28772\wps2.jpg)

# **更新配置文件:**

此时我们重新生成配置文件:

移除原配置文件

mv /etc/kubernetes/\*conf /etc/kubernetes/conf\_bak/

生成新配置文件:

kubeadm alpha phase kubeconfig all --config cluster.yaml

此时，为保证kubectl命令能正常使用，需要将admin.conf文件替换掉~.kube/config配置文件。

然后所有节点重启docker和kubelet即可.\(不确定是否需要，反正我重启了\)

注意，kubelet不需要重新分发证书，kubelet证书是链接apiserver，只要根证书没变，kubelet就可以完成自动更新。

重启后，通过命令校验:

kubectl get node

kubectl get all --all-namespace



