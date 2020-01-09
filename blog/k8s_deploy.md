# /etc/hosts
```
172.19.49.166	alchemystar004	alchemystar004
172.19.49.164 alchemystar001 alchemystar001
172.19.49.165 alchemystar002 alchemystar002
172.19.156.209 alchemystar003 alchemystar003
```
# mod bridge
```
modprobe br_netfilter
ls /proc/sys/net/bridge
```
# master start
```
kubeadm init --kubernetes-version=1.14.2 \
--apiserver-advertise-address=172.19.49.166 \
--image-repository registry.aliyuncs.com/google_containers \
--service-cidr=10.1.0.0/16 \
--pod-network-cidr=10.244.0.0/16
```
# master return
```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.19.49.166:6443 --token b8ew1s.0jj9imj0b49zfcii \
    --discovery-token-ca-cert-hash sha256:66b3d43b832870867d25ec100d11bf5e76a3562bbb3d75e9f19d2e956752423b
```
# docker阿里云加速
```

/etc/docker/daemon.json
{
  "registry-mirrors": ["https://72idtxd8.mirror.aliyuncs.com"]
}
systemctl daemon-reload
systemctl  restart docker
```

```
sudo docker login --username=alchemystar123 registry.cn-hangzhou.aliyuncs.com
Lzy202503
docker tag 4673199e94fe registry.cn-hangzhou.aliyuncs.com/alchemystar/archer:1.0
docker push registry.cn-hangzhou.aliyuncs.com/alchemystar/archer:1.0
```


```
kubectl create secret docker-registry registry-secret --docker-server=registry.cn-hangzhou.aliyuncs.com --docker-username=alchemystar123 --docker-password=Lzy202503 --docker-email=652433935@qq.com -n default
--docker-server: 仓库地址
--docker-username: 仓库登陆账号
--docker-password: 仓库登陆密码
--docker-email: 邮件地址(选填)
```
# image pull secrets
```
yaml:
    // 注意递进关系
    imagePullSecrets:
      - name: your-registry-key
```