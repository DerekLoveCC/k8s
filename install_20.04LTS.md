# reference https://blog.csdn.net/zpsimon/article/details/134388092

# k8s版本1.28.3,并使用containerd作为容器运行时，使用calico作为cni插件

# 三台Ubuntu 20.04LTS
192.168.0.220   k8s-master1   Ubuntu 20.04 LTS   4GB
192.168.0.221   k8s-work1     Ubuntu 20.04 LTS   4GB
192.168.0.222   k8s-work2     Ubuntu 20.04 LTS   4GB

# enable root
1. sudo passwd root
2. sudo vim /etc/ssh/sshd_config
3. 修改 PermitRootLogin prohibit-password 为PermitRootLogin yes
4. sudo systemctl restart sshd

# 修改hosts
cat>>/etc/hosts<<eof
192.168.0.220 k8s-master1
192.168.0.221 k8s-work1
192.168.0.222 k8s-work2
eof

# 关闭selinux
echo "SELINUX=disabled" >> /etc/selinux/config

# 关闭swap
swapoff -a && sed -i '/swap/ s/^\(.*\)$/#\1/' /etc/fstab

# 关闭防火墙并禁止开机自启
ufw disable

# 转发 IPv4 并让 iptables 看到桥接流量
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

## 设置所需的 sysctl 参数，参数在重新启动后保持不变
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

## 应用 sysctl 参数而不重新启动
sudo sysctl --system

# 安装容器运行时（containerd）
1. 安装containerd 1.7.8
   下载二进制文件
   root@k8s-master1:~# wget https://github.com/containerd/containerd/releases/download/v1.7.8/containerd-1.7.8-linux-amd64.tar.gz

   解压
   root@k8s-master1:/zpdata/packages# tar Cxzvf /usr/local containerd-1.7.8-linux-amd64.tar.gz

2. 将containerd配置成systemd服务，方便使用systemctl命令管理
   root@k8s-master1:/zpdata/packages# mkdir -p /usr/local/lib/systemd/system/
   root@k8s-master1:/zpdata/packages# cd /usr/local/lib/systemd/system/
   root@k8s-master1:/usr/local/lib/systemd/system# curl -o containerd.service https://github.moeyy.xyz/https://raw.githubusercontent.com/containerd/containerd/main/containerd.service

3. 加载配置文件
   systemctl daemon-reload

4. 设置开启自启并立即启动
   systemctl enable --now containerd

# 配置containerd的config.toml
  containerd的一些默认参数可能不兼容，需要针对性的调整

  root@k8s-master:/zpdata/packages# mkdir /etc/containerd
  root@k8s-master:/zpdata/packages# containerd config default > /etc/containerd/config.toml

 ## 修改“sandbox_image”

  vi /etc/containerd/config.toml

  ```shell
  [plugins."io.containerd.grpc.v1.cri"]
  ……
  sandbox_image = "registry.k8s.io/pause:3.8"  #修改为   sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.8"

  ```

 ## 修改"SystemdCgroup"【重要，需要与kubelet的cGroupDriver匹配】

  ```shell
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ……
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  ……
  SystemdCgroup = false    # 改为  SystemdCgroup = true

  ```

 ## 然后重启containerd
  sudo systemctl restart containerd

# 安装runc

  root@k8s-master:/zpdata/packages# wget http://github.com/opencontainers/runc/releases/download/v1.1.10/runc.amd64
  root@k8s-master:/zpdata/packages# install -m 755 runc.amd64 /usr/local/sbin/runc

# 安装cni
  root@k8s-slave1:/zpdata/packages# wget http://github.com/containernetworking/plugins/releases/download/v1.3.0/cni-plugins-linux-amd64-v1.3.0.tgz
  root@k8s-master:/zpdata/packages# mkdir -p /opt/cni/bin
  root@k8s-master:/zpdata/packages# tar Cxzvf /opt/cni/bin/ cni-plugins-linux-amd64-v1.3.0.tgz

# 安装容器操作的客户端工具（crictl，nerdctl)

* crictl只用于debug

    VERSION="v1.28.0"
    wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz
    sudo tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin
  ## 检查
    root@k8s-master:/zpdata/packages# crictl -v

* nerdctl– General-purpose
    root@k8s-master:/zpdata/packages# wget https://github.com/containerd/nerdctl/releases/download/v1.7.0/nerdctl-1.7.0-linux-amd64.tar.gz
    root@k8s-master:/zpdata/packages# tar Cxzvvf /usr/local/bin nerdctl-1.7.0-linux-amd64.tar.gz
  ## 检查
    root@k8s-master:/usr/local/bin# nerdctl -v

# 安装 kubeadm、kubelet 和 kubectl
  apt-get update
  mkdir -p -m 755  /etc/apt/keyrings
  curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
  echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
  apt-get update
  apt-get install -y kubelet kubeadm kubectl
  apt-mark hold kubelet kubeadm kubectl

# kubelet 配置 cgroup 驱动程序





# 使用 kubeadm init 创建集群
  kubeadm config images list  #查看kubeadm init需要的镜像
  sudo kubeadm config images pull --image-repository=registry.aliyuncs.com/google_containers  # 提前拉取这些镜像
  #查看kubeadm init的默认值
  kubeadm config print init-defaults
  #查看kubeadm join的默认值
  kubeadm config print join-defaults
Master节点
  kubeadm init --image-repository registry.aliyuncs.com/google_containers --kubernetes-version=v1.28.6

# 安装网络插件（Calico）

k8s中网络插件有很中，比如：Calico，Flannel等，calico作为k8s的中的一种资源，只需要在master上执行即可。
以下以安装Calico为例
   ## 1）下载calico.yaml文件
   wget https://github.com/projectcalico/calico/raw/2f38bd7adc4771a1de28bd2545e28d888c017a00/manifests/calico.yaml
   wget https://github.moeyy.xyz/https://raw.githubusercontent.com/projectcalico/calico/2f38bd7adc4771a1de28bd2545e28d888c017a00/manifests/calico.yaml

   ## 2）做如下修改 calico.yaml.default为修改前，calico.yaml为修改后
   root@k8s-master:/zpdata/packages# diff calico.yaml.default calico.yaml
   4931,4932c4931,4932
   <             # - name: CALICO_IPV4POOL_CIDR
   <             #   value: "192.168.0.0/16"
   ---
   >             - name: CALICO_IPV4POOL_CIDR
   >               value: "10.96.0.0/12"


   ## 10.96.0.0/12  值来源于kubeadm init时指定的 --pod-network-cidr 参数，默认是：10.96.0.0/12
    root@k8s-master:/zpdata/packages# kubectl get cm -n kube-system kubeadm-config -o yaml | grep serviceSubnet
        #结果 serviceSubnet: 10.96.0.0/12
   ## 使用root用户运行
   kubectl apply -f calico.yaml
   ## 检验
   使用 kubectl get pods -A 命令查看是不是所有的pod都是running状态

# 使用kubeadm join将node加入k8s集群
【slave1,slave2节点执行】
  kubeadm join 192.168.0.220:6443 --token ejh0fj.pbf5d51qnlw94n9v --discovery-token-ca-cert-hash sha256:5a4bc66483d146cee084542d07fac00296c2b4b559b9914e568f52851f20cc7f
若令牌过期，或丢失上述指令可在控制平面节点使用如下指令重新获取：kubeadm token create --print-join-command

# 验证
【master节点执行】
 kubectl get no -o wide

# References

https://zhuanlan.zhihu.com/p/651200897
https://blog.csdn.net/zpsimon/article/details/134388092
https://blog.csdn.net/MssGuo/article/details/128149704