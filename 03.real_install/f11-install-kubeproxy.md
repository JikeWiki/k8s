# 安装kube-proxy

## 1. 签发ssl证书

我们登录到证书服务器，去签发证书

创建证书目录`/opt/certs/kubeproxy`，


创建证书请求文件`/opt/certs/kubeproxy/kube-proxy-csr.json`，添加如下内容

```json
{
	"CN": "system:kube-proxy",
	"key": {
		"algo": "rsa",
		"size": 2048
	},
	"name": [
		{
			"C": "CN",
			"ST": "Guangdong",
			"L": "Guangzhou",
			"O": "od",
			"OU": "ops"
		}
	]
}
```


创建证书

```shell
cfssl gencert -ca=../ca.pem -ca-key=../ca-key.pem -config=../ca-config.json -profile=client kube-proxy-csr.json | cfssljson -bare kube-proxy-client
```


## 2. 安装kube-proxy


签发好证书之后，我们回到`kb21`和`kb22`两个节点，进行安装操作，具体过程如下


下载证书文件
```shell
scp root@kb200.host.com:/opt/certs/kubeproxy/kube-proxy-client.pem ./
scp root@kb200.host.com:/opt/certs/kubeproxy/kube-proxy-client-key.pem ./
```


使用命令`cd /opt/kubernetes/server/bin/conf`回到集群配置文件目录，执行以下操作

设置集群

```shell
kubectl config set-cluster myk8s \
    --certificate-authority=/opt/kubernetes/server/bin/certs/ca.pem \
    --embed-certs=true \
    --server=https://192.168.14.10:7443 \
    --kubeconfig=kube-proxy.kubeconfig
```


设置客户端证书


```shell
kubectl config set-credentials kube-proxy \
    --client-certificate=/opt/kubernetes/server/bin/certs/kube-proxy-client.pem \
    --client-key=/opt/kubernetes/server/bin/certs/kube-proxy-client-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig
```


设置上下文

```shell
kubectl config set-context myk8s-context \
    --cluster=myk8s \
    --user=kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig
```




使用上下文


以上配置只需在一台主机进行，另一台主机从创建的主机拷贝即可，最后两台主机都执行以下命令

```shell
kubectl config use-context myk8s-context --kubeconfig=kube-proxy.kubeconfig
```



## 3. 启动服务

先创建一个脚本文件`ipvs.sh`，添加以下内容

```shell
#!/bin/bash
ipvs_mods_dir="/usr/lib/modules/$(uname -r)/kernel/net/netfilter/ipvs"
for i in $(ls $ipvs_mods_dir|grep -o "^[^.]*")
do
	/sbin/modinfo -F filename $i &>/dev/null
	if [ $? -eq 0 ]; then
		/sbin/modprobe $i
	fi
done
```

该脚本的作用是加载所有与ipvs相关的模块

我们可以使用以下命令查看当前ip_vs相关的模块，可以发现在执行脚本之前是没有返回内容的
```shell
lsmod | grep ip_vs
```

在两台主机执行以上脚本之后，我们创建`kube-proxy`的启动脚本文件`/opt/kubernetes/server/bin/kube-proxy.sh`

```shell
#!/bin/bash
./kube-proxy \
  --cluster-cidr 172.7.0.0/16 \
  --hostname-override kb21 \
  --proxy-mode=ipvs \
  --ipvs-scheduler=nq \
  --kubeconfig ./conf/kube-proxy.kubeconfig
```

添加可执行权限

```shell
chmod +x kube-proxy.sh
```

创建supervisor的配置文件`/etc/supervisord.d/kube-proxy.ini`文件，添加以下内容

```shell
[program:kube-proxy-21]
directory=/opt/kubernetes/server/bin
command=/opt/kubernetes/server/bin/kube-proxy.sh
numprocs=1
autostart=true
autorestart=true
startsecs=30
startretries=3
exitcodes=0,2
stopsignal=QUIT
stopwaitsecs=10
user=root
redirect_stderr=true
stdout_logfile=/data/logs/kubernetes/kube-proxy/kube-proxy.stdout.log
stdout_logfile_maxbytes=64MB
stdout_logfile_backups=4
stdout_capture_maxbytes=1MB
stdout_event_enabled=false
```

创建日志目录与更新supervisor

```shell
mkdir -p /data/logs/kubernetes/kube-proxy/
supervisorctl update
```


我们还可以安装`ipvsadm`用于管理ipvs，如下命令

```shell
yum install -y ipvsadm
```




## 4. 集群的验证

在两个节点都启动好kube-proxy服务之后，我们来验证集群。创建一个DaemonSet类型的资源，添加`nginx-ds.yaml`文件，添加以下内容

```shell
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: nginx-ds
spec:
  template:
    metadata:
      labels:
        app: nginx-ds
    spec:
      containers:
      - name: my-nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
```

执行资源创建命令

```shell
kubectl create -f nginx-ds.yaml
```

使用以下命令验证pod是否正常运行

```shell
kubectl get pod -o wide
```

如果返回如下内容，代表集群正常

```shell
[root@kb21 yml]# kubectl get pod -o wide
NAME             READY   STATUS    RESTARTS   AGE   IP           NODE   NOMINATED NODE   READINESS GATES
nginx-ds-mzxrv   1/1     Running   0          12s   172.7.22.2   kb22   <none>           <none>
nginx-ds-sdfwt   1/1     Running   0          12s   172.7.21.2   kb21   <none>           <none>
```

加入我们现在在`kb21`这台主机上，使用`curl http://172.7.21.2`命令检测新建的服务是否正常，如果返回如下nginx相关的提示信息，代表kb21这个节点服务正常。


但是如果我们在`kb21`上使用`curl http://172.7.22.2`去检测`kb22`这个节点，会发现访问不到。原因是这两个节点上的容器在各自的虚拟网络内，我们将到后续章节继续解决不同节点的容器网络互相访问的问题！

> Deployment 部署的副本 Pod 会分布在各个 Node 上，每个 Node 都可能运行好几个副本。 DaemonSet 的不同之处在于:每个 Node 上最多只能运行一个副本。

