# Kubernetes 三节点集群部署手册

> 环境:Windows 宿主机 + VMware Workstation;3 台 Ubuntu Server 24.04 LTS 虚拟机
> 容器运行时:containerd 2.4.x;集群版本:Kubernetes v1.37.2(kubeadm);CNI:Flannel
> 网络:VMware VMnet8 NAT,网段 `192.168.150.0/24`,网关 `192.168.150.2`
> 用途:个人学习环境(单 master + 2 worker,不做高可用)

---

## 目录

1. 规划
2. 安装 Ubuntu Server 24.04
3. 配置静态 IP(netplan)
4. 克隆节点与修改节点身份
5. K8s 前置系统配置
6. 安装 containerd 2.x
7. 安装 kubeadm / kubelet / kubectl
8. 初始化 master 节点
9. 安装 CNI(Flannel)
10. worker 节点加入集群
11. 集群验证
12. 常用排错手册
13. 本次部署踩坑记录

---

## 1. 规划

### 1.1 节点规划

| 主机名 | 角色 | IP | 建议规格 |
|---|---|---|---|
| k8s01 | control-plane(master) | 192.168.150.170 | 2 vCPU / 4 GB / 25 GB |
| k8s02 | worker | 192.168.150.171 | 2 vCPU / 4 GB / 25 GB |
| k8s03 | worker | 192.168.150.172 | 2 vCPU / 4 GB / 25 GB |

宿主机建议 ≥ 16 GB 内存;三台合计 12 GB 内存,宿主仍有余量。

### 1.2 版本规划

| 组件 | 版本 | 说明 |
|---|---|---|
| Ubuntu Server | 24.04 LTS(noble) | 内核 6.8,cgroup v2 默认开启 |
| containerd | 2.4.2 | 来自阿里云 Docker CE 镜像 |
| kubeadm / kubelet / kubectl | v1.37.2 | 来自阿里云 Kubernetes 镜像源 |
| CNI | Flannel(最新稳定版) | Pod 网段 `10.244.0.0/16` |

### 1.3 网络规划

- VMware NAT 网段:`192.168.150.0/24`
- NAT 网关 / DNS:`192.168.150.2`
- 备用 DNS:`223.5.5.5`
- 静态 IP 取值要**避开 VMware DHCP 池**,默认池为 `.128 ~ .254`。本手册使用 `.170 ~ .172`;如果沿用池内地址,建议在“虚拟网络编辑器 → VMnet8 → DHCP 设置”里把范围改为 `.200 ~ .254`。

### 1.4 国内网络环境说明(重要)

| 站点 | 可达性 | 用途 / 替代方案 |
|---|---|---|
| download.docker.com | ❌ 连接被重置 | 改用 `mirrors.aliyun.com/docker-ce` |
| pkgs.k8s.io | 可用(302) | 或改用 `mirrors.aliyun.com/kubernetes-new` |
| registry.k8s.io | 可直连(401=可达) | kubeadm 默认镜像仓库 |
| registry-1.docker.io(Docker Hub) | ❌ 超时 | 用 `docker.m.daocloud.io` 加速 |
| raw.githubusercontent.com | ❌ 连不上 | 用 jsDelivr / ghproxy 镜像下载 YAML |

---

## 2. 安装 Ubuntu Server 24.04

### 2.1 创建虚拟机

VMware Workstation:

1. 新建虚拟机 → 典型 → 稍后安装操作系统
2. 客户机系统:Linux → Ubuntu 64 位
3. 名称:`k8s01`,位置自选
4. 磁盘:25 GB,拆分为多个文件
5. 自定义硬件:2 vCPU、4 GB 内存、网络适配器选 **NAT**
6. 挂载 Ubuntu Server 24.04 ISO,启动安装

### 2.2 安装向导注意事项

- 安装类型可选 **Ubuntu Server(minimized)**,最小化安装
- 用户名建议按主机名命名(本手册为 `k8s01`),记住密码
- **不要**勾选 “Install OpenSSH server”(最小安装下后面手动装也一样;勾了更省事)
- 安装器默认创建 4 GB swap 文件,后面会关闭
- 安装完成后重启,登录

### 2.3 系统更新(可选)

```bash
sudo apt update && sudo apt upgrade -y
```

---

## 3. 配置静态 IP(netplan)

Ubuntu 17.10 之后网络统一由 **netplan** 管理,配置文件位于 `/etc/netplan/`。

### 3.1 查看网卡名

```bash
ip a
```

VMware 下通常是 `ens33`(以实际输出为准)。`/etc/netplan/` 为空目录是正常的,netplan 会读取该目录下所有 `*.yaml`。

### 3.2 创建静态 IP 配置

```bash
sudo mkdir -p /etc/netplan
sudo tee /etc/netplan/01-static.yaml > /dev/null <<'EOF'
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      dhcp4: false
      addresses: [192.168.150.170/24]
      routes:
        - to: default
          via: 192.168.150.2
      nameservers:
        addresses: [192.168.150.2, 223.5.5.5]
EOF
sudo chmod 600 /etc/netplan/01-static.yaml
sudo netplan try
sudo netplan apply
```

要点:

- **YAML 只能用空格缩进,不能用 Tab**
- Ubuntu 24.04 已废弃 `gateway4`,必须用 `routes: - to: default via:`
- `chmod 600` 是 netplan 的要求,权限过宽会告警
- `netplan try` 有 120 秒确认窗口,配错会自动回滚;SSH 操作时先用它更安全

### 3.3 验证

```bash
ip a show ens33     # inet 192.168.150.170/24
ip r                # default via 192.168.150.2 dev ens33
ping -c 2 192.168.150.2
ping -c 2 baidu.com
```

---

## 4. 克隆节点与修改节点身份

### 4.1 克隆流程

1. **先关闭 master 虚拟机**(Power Off)
2. VMware 中右键 → 管理 → 克隆 → 完整克隆,命名为 `k8s02`、`k8s03`
3. 启动克隆机,**先不要让它带着 master 的 IP 联网**(可在硬件设置里暂时取消“启动时连接”)
4. 在控制台执行 4.2 的命令修改身份
5. 改完再启动 master,避免 IP 冲突

### 4.2 修改身份(每台克隆机执行)

k8s02(`.171`):

```bash
sudo sed -i 's/192.168.150.170/192.168.150.171/' /etc/netplan/01-static.yaml
sudo hostnamectl set-hostname k8s02
sudo sed -i 's/127.0.1.1 k8s01/127.0.1.1 k8s02/' /etc/hosts
sudo rm -f /etc/machine-id
sudo systemd-machine-id-setup
sudo rm -f /etc/ssh/ssh_host_*
sudo ssh-keygen -A
sudo netplan apply
sudo systemctl restart ssh
```

k8s03(`.172`):把上面的 `171` 换成 `172`、`k8s02` 换成 `k8s03`。

为什么要做:

| 操作 | 原因 |
|---|---|
| 改 IP | 克隆机会带着 master 的静态 IP 启动,导致 IP 冲突 |
| 改 hostname | kubelet 以 hostname 注册节点,重名会覆盖节点 |
| 改 `/etc/hosts` | 保证 `127.0.1.1` 与主机名一致,避免 sudo 解析变慢 |
| 改 machine-id | 每台机器 ID 必须唯一,否则日志/systemd 身份混淆 |
| 重新生成 SSH host key | 避免三台共用同一主机密钥 |
| 改用户名(可选) | 本实验把 worker 默认用户改成 `k8s02`/`k8s03` 与主机名对应 |

### 4.3 修改默认用户名(本实验执行过)

`usermod` 不允许在用户有登录会话/进程时改名,因此需要以 root 身份操作。步骤:

```bash
# 1) 临时允许 root 密码 SSH(00- 前缀保证优先于 cloud-init 的 50- 配置)
sudo sh -c "printf '%s\n' 'PermitRootLogin yes' > /etc/ssh/sshd_config.d/00-root-login.conf"
sudo chmod 644 /etc/ssh/sshd_config.d/00-root-login.conf
sudo systemctl restart ssh

# 2) 从宿主机以 root 登录该机器后执行:
loginctl terminate-user k8s01
pkill -KILL -u 1000
usermod -l k8s02 -d /home/k8s02 -m k8s01
groupmod -n k8s02 k8s01

# 3) 恢复 sshd 默认(root 密码登录关闭)
rm -f /etc/ssh/sshd_config.d/00-root-login.conf
systemctl restart ssh
```

说明:`usermod -l` 只改用户名保留 UID,`-d /home/新名 -m` 同时迁移家目录;原 sudo 等附加组保留。

### 4.4 验证

```bash
hostname
id
ip -4 -o a show ens33 | awk '{print $4}'
cat /etc/machine-id            # 三台必须不同
ls -l /etc/ssh/ssh_host_*      # 三台生成时间应不同
ping -c 2 192.168.150.170      # 三台互通
```

---

## 5. K8s 前置系统配置

**三台都要执行。**

### 5.1 关闭 swap

Kubernetes 要求节点关闭 swap(kubelet 默认拒绝在开启 swap 的节点运行)。

```bash
sudo swapoff -a
sudo sed -i '/swap/s/^/#/' /etc/fstab
```

注意:Ubuntu 的 fstab 用 Tab 分隔,`sed` 匹配 `swap` 即可,不要写成 `'/ swap /'`。

验证:

```bash
free -h          # Swap 行应为 0
grep -c '^[^#].*swap' /etc/fstab   # 应为 0
```

### 5.2 加载内核模块

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

### 5.3 网络参数

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

验证:

```bash
lsmod | grep -E 'overlay|br_netfilter'
sysctl net.ipv4.ip_forward        # 1
stat -fc %T /sys/fs/cgroup        # cgroup2fs(Ubuntu 24.04 默认)
```

---

## 6. 安装 containerd 2.x

Ubuntu 24.04 自带源的 containerd 是 1.7.x;要装 2.x,本手册使用阿里云 Docker CE 镜像。

### 6.1 添加阿里云 Docker CE 源

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg -o /tmp/docker.asc
sudo gpg --batch --yes --dearmor -o /etc/apt/keyrings/docker.gpg < /tmp/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://mirrors.aliyun.com/docker-ce/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" | sudo tee /etc/apt/sources.list.d/docker.list
```

### 6.2 安装 containerd

```bash
sudo apt update
sudo apt install -y containerd.io
containerd --version    # containerd v2.4.2
```

### 6.3 生成配置并开启 SystemdCgroup

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
grep -n 'SystemdCgroup' /etc/containerd/config.toml
```

要点:

- containerd 2.x 生成的配置是 **version 3/4**(新版),与网上大量 1.x 教程的 `plugins."io.containerd.grpc.v1.cri"` 结构不同
- **`SystemdCgroup = true` 必须设置**,否则 kubelet + cgroup v2 下 Pod 会出现各种启动异常
- containerd 2.x 的 CRI 配置拆分为 `io.containerd.cri.v1.runtime` / `io.containerd.cri.v1.images` 等插件

### 6.4 配置 docker.io 镜像加速(国内环境推荐)

Docker Hub 在本环境不可达,Flannel 等镜像走 Docker Hub 会拉取失败。通过 containerd 的 hosts.toml 把 docker.io 指向 DaoCloud 镜像:

```bash
sudo mkdir -p /etc/containerd/certs.d/docker.io
sudo tee /etc/containerd/certs.d/docker.io/hosts.toml > /dev/null <<'EOF'
server = "https://docker.m.daocloud.io"

[host."https://docker.m.daocloud.io"]
  capabilities = ["pull", "resolve"]
EOF
```

然后在 `/etc/containerd/config.toml` 中找到 CRI images 插件的 `config_path = ''`,改为:

```toml
config_path = '/etc/containerd/certs.d'
```

可用以下命令定位:

```bash
grep -n -B3 "config_path" /etc/containerd/config.toml
```

改完重启:

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
systemctl is-active containerd
```

验证镜像加速是否生效:

```bash
sudo ctr -n k8s.io images pull docker.io/library/busybox:latest
```

### 6.5 修改 sandbox 镜像(国内必做)

containerd 默认的 sandbox 镜像是 `registry.k8s.io/pause:3.10.2`。本环境实测:对 `registry.k8s.io` 的拉取请求会被 302 重定向到 `asia-east1-docker.pkg.dev`,而该域名被墙,导致所有 Pod 卡在 `ImagePullBackOff` / `CreateContainerError`。必须把 sandbox 镜像改成阿里云地址:

```bash
sudo sed -i "s|sandbox = 'registry.k8s.io/pause:3.10.2'|sandbox = 'registry.aliyuncs.com/google_containers/pause:3.10.2'|" /etc/containerd/config.toml
sudo systemctl restart containerd
grep -n -A1 'pinned_images' /etc/containerd/config.toml
```

> 与第 8 章的 `--image-repository=registry.aliyuncs.com/google_containers` 保持一致,预拉的 pause 镜像可直接复用。

### 6.6 验证

```bash
containerd --version
systemctl status containerd --no-pager
sudo crictl --version 2>/dev/null || echo "crictl 未安装(可选)"
ss -lntp | grep 2375   # 不应有监听;containerd 不对外暴露
```

---

## 7. 安装 kubeadm / kubelet / kubectl

使用阿里云 Kubernetes 镜像源(当前稳定版 v1.37.2)。

### 7.1 添加 Kubernetes 源

```bash
curl -fsSL https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.37/deb/Release.key | sudo gpg --batch --yes --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.37/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
```

### 7.2 安装并锁定版本

```bash
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

`hold` 防止后续 `apt upgrade` 把版本升乱;升级集群时应按计划显式升级。

### 7.3 验证

```bash
kubeadm version -o short     # v1.37.2
kubelet --version
kubectl version --client
```

注意:此时 kubelet 会反复重启(没有 `/var/lib/kubelet/config.yaml`),属于正常现象,`kubeadm init` 之后会自动正常。

---

## 8. 初始化 master 节点

以下命令**只在 k8s01 上执行**。

### 8.1 预拉取镜像

```bash
sudo kubeadm config images pull --image-repository=registry.aliyuncs.com/google_containers
```

如果阿里云镜像缺版本,可以去掉 `--image-repository`,直接拉 `registry.k8s.io`(本环境直连可达)。

### 8.2 执行 init

```bash
sudo kubeadm init \
  --apiserver-advertise-address=192.168.150.170 \
  --pod-network-cidr=10.244.0.0/16 \
  --image-repository=registry.aliyuncs.com/google_containers
```

参数说明:

| 参数 | 作用 |
|---|---|
| `--apiserver-advertise-address` | API Server 对外通告地址,必须是本机静态 IP |
| `--pod-network-cidr` | Pod 网段,Flannel 要求 `10.244.0.0/16` |
| `--image-repository` | 国内加速镜像仓库 |

init 成功后记录输出中的 `kubeadm join ...` 命令(worker 要用)。

> 前提:第 6.5 步已把 containerd 的 sandbox 镜像改为阿里云地址,否则 init 虽然成功,但所有 Pod 会因拉取 `registry.k8s.io/pause` 失败而卡住。

### 8.3 配置普通用户 kubeconfig

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 8.4 验证

```bash
kubectl get nodes              # 此时 NotReady 正常,只差 CNI
kubectl get pods -A            # coredns 处于 Pending,等待 CNI
```

---

## 9. 安装 CNI(Flannel)

### 9.1 获取 kube-flannel.yml

GitHub raw 在国内不可达。本环境实测可用的镜像源:

| 源 | 结果 |
|---|---|
| `ghfast.top` | ✅ 可用 |
| `ghproxy.net` | ✅ 可用(偶发失败,可重试) |
| jsDelivr(`cdn.`/`fastly.`/`gcore.`) | ❌ 不可用 |
| `raw.gitmirror.com` / `gh-proxy.com` | ❌ 不可用 |

```bash
curl -fsSL -o kube-flannel.yml \
  https://ghfast.top/https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
wc -c kube-flannel.yml    # 正常约 4.8 KB
```

### 9.2 改写镜像地址(国内必做)

最新版清单使用 **ghcr.io** 镜像(`ghcr.io/flannel-io/...`)。本环境实测:Docker Hub 的 DaoCloud 加速可用,但 **ghcr.io 的 DaoCloud 加速会把 blob 重定向到 `image-mirror.r2.daocloud.vip`(被墙)**,拉取超时。Flannel 同时发布在 Docker Hub,因此把清单的 ghcr.io 地址改写为 docker.io:

```bash
sed -i 's#ghcr.io/flannel-io/#docker.io/flannel/#g' kube-flannel.yml
grep image: kube-flannel.yml
# 期望输出:
#   docker.io/flannel/flannel-cni-plugin:v1.9.1-flannel3
#   docker.io/flannel/flannel:v0.28.10
```

### 9.3 部署

```bash
kubectl apply -f kube-flannel.yml
```

### 9.4 验证

```bash
kubectl get pods -n kube-flannel -o wide
kubectl get nodes          # 等待 k8s01 变为 Ready
kubectl get pods -A        # coredns 变为 Running
```

Flannel 容器需要以 `--iface=ens33` 或默认自动识别网卡;若选错网卡,可在 DaemonSet 的 `--kube-subnet-mgr` 参数后加 `--iface=ens33` 后重新 apply。

---

## 10. worker 节点加入集群

### 10.1 生成加入命令(master 上)

```bash
kubeadm token create --print-join-command
```

输出形如:

```bash
kubeadm join 192.168.150.170:6443 --token xxxxxx.xxxxxxxxxxxxxxxx \
  --discovery-token-ca-cert-hash sha256:xxxxxxxx...
```

### 10.2 在 k8s02、k8s03 上执行

```bash
sudo kubeadm join 192.168.150.170:6443 --token xxxxxx.xxxxxxxxxxxxxxxx \
  --discovery-token-ca-cert-hash sha256:xxxxxxxx...
```

### 10.3 验证

在 master 上:

```bash
kubectl get nodes -o wide
# 三台都应为 Ready
```

---

## 11. 集群验证

### 11.1 节点与组件

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl cluster-info
```

### 11.2 跑一个测试 Pod

```bash
kubectl create deployment nginx --image=nginx:alpine --replicas=2
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get pods -o wide
kubectl get svc nginx
```

浏览器访问 `http://192.168.150.170:<NodePort>` 验证。

### 11.3 清理测试资源

```bash
kubectl delete service nginx
kubectl delete deployment nginx
```

---

## 12. 常用排错手册

### 12.1 apt 报 `Temporary failure resolving`

DNS 没配好。检查:

```bash
cat /etc/resolv.conf
ping -c 2 223.5.5.5      # 通 IP 不通域名 = DNS 问题
```

在 netplan 里补 `nameservers`,或临时 `echo 'nameserver 223.5.5.5' > /etc/resolv.conf`。

### 12.2 `ip route add default via ...` 报 `Nexthop has invalid gateway`

网关必须与接口在同一“直连网段”。先确认:

```bash
ip link set ens33 up
ip r
```

没有 `192.168.150.0/24 dev ens33` 直连路由时,先给接口加地址(或直接用 netplan 配置),再加默认路由。

### 12.3 SSH 连不上 root

Ubuntu 默认 `PermitRootLogin prohibit-password`:root 禁止密码登录,仅允许密钥。用普通用户登录后 `sudo -i`:

```bash
sudo sshd -T | grep -i permitrootlogin
```

### 12.4 控制台不能粘贴

VMware 控制台粘贴依赖 open-vm-tools:

```bash
sudo apt install -y open-vm-tools
sudo systemctl restart open-vm-tools
```

更推荐:装 `openssh-server`,从宿主机 SSH 操作,复制粘贴不受限:

```powershell
ssh 用户名@192.168.150.170
```

### 12.5 最小安装没有 vim / vi / nano

不需要编辑器也能写文件:

```bash
sudo tee /path/file > /dev/null <<'EOF'
...内容...
EOF

# 或小范围替换
sudo sed -i 's/old/new/g' /path/file
```

### 12.6 kubelet 一直重启 / Pod 异常

- 检查 `SystemdCgroup = true` 是否设置
- 检查 swap 是否关闭
- 检查内核模块 `overlay`、`br_netfilter`
- 查看日志:`journalctl -u kubelet -n 100 --no-pager`

### 12.7 镜像拉取失败

- kubeadm 镜像:使用 `--image-repository=registry.aliyuncs.com/google_containers`
- **`registry.k8s.io` 直连会 302 到 `asia-east1-docker.pkg.dev`(国内被墙)**:containerd 的 sandbox 镜像也必须换成阿里云地址(见 6.5),否则所有 Pod 无法创建
- **ghcr.io 镜像**:DaoCloud 的 ghcr 加速会把 blob 重定向到 `image-mirror.r2.daocloud.vip`(被墙),建议把清单镜像改写为 Docker Hub 版本(见 9.2)
- docker.io 镜像:按 6.4 配置 `/etc/containerd/certs.d/docker.io/hosts.toml`;DaoCloud 偶发 blob 超时,重试 2~3 次通常可成功
- 查看 kubelet 拉取日志:`journalctl -u kubelet | grep -i pull`

### 12.8 netplan 报 `Invalid YAML`

- 只用空格,不用 Tab
- `addresses` 与方括号内容保持同一行,减少缩进层级
- 用 `netplan try` 会指出具体行号

### 12.9 忘记 root 密码

Console 进 GRUB(UEFI 连按 Esc / BIOS 按住 Shift)→ 编辑 `linux` 行 → `ro` 改 `rw`,行尾加 `init=/bin/bash` → Ctrl+X 启动 → `mount -o remount,rw /` → `passwd root` → `sync; reboot -f`。

---

## 13. 本次部署踩坑记录

1. **最小安装无 vim/vi/nano**:全程用 `tee` heredoc 和 `sed` 完成配置。
2. **VMware 控制台无法复制粘贴**:先临时配网 → 装 openssh-server → 从 Windows SSH 操作。
3. **`/etc/netplan/` 为空**:最小安装正常现象,自行创建 `01-static.yaml` 即可。
4. **apt 报解析失败**:netplan 未配置导致无 DNS;配好后恢复。
5. **`Nexthop has invalid gateway`**:ens33 处于 DOWN 且无直连路由;`netplan apply` 一次性解决,不要手动拆步加路由。
6. **忘记 root 密码**:通过 GRUB `init=/bin/bash` 重置。
7. **克隆后三台用户名/机器身份重复**:改 hostname/IP/machine-id/SSH host key,并用临时 root SSH 完成 `usermod` 改名。
8. **download.docker.com 被重置**:containerd 改用阿里云 Docker CE 源(2.4.2)。
9. **Docker Hub 不可达**:containerd 配 `docker.m.daocloud.io` 镜像加速。
10. **raw.githubusercontent.com / jsDelivr 均不可达**:Flannel 清单最终从 `ghfast.top` 获取(`ghproxy.net` 也可用但偶发失败)。
11. **`registry.k8s.io` 实际拉取被墙**:请求 302 到 `asia-east1-docker.pkg.dev`;containerd sandbox 镜像改为 `registry.aliyuncs.com/google_containers/pause:3.10.2`,否则所有 Pod 无法创建。
12. **ghcr.io 经 DaoCloud 加速时 blob 被重定向到 `image-mirror.r2.daocloud.vip`(被墙)**:Flannel 清单镜像由 `ghcr.io/flannel-io/...` 改写为 `docker.io/flannel/...`。
13. **DaoCloud 的 docker.io 加速偶发 blob 超时**:重试 2~3 次即可(本次 cni-plugin 第一次失败、第二次成功)。

---

## 14. 实际部署结果(2026-11-09)

### 14.1 节点状态

```text
NAME    STATUS   ROLES           AGE   VERSION   INTERNAL-IP       CONTAINER-RUNTIME
k8s01   Ready    control-plane   14m   v1.37.2   192.168.150.170   containerd://2.4.2
k8s02   Ready    <none>          71s   v1.37.2   192.168.150.171   containerd://2.4.2
k8s03   Ready    <none>          20s   v1.37.2   192.168.150.172   containerd://2.4.2
```

### 14.2 系统组件

- `kube-flannel-ds`:三个节点各 1 个 Pod,全部 1/1 Running
- `kube-proxy`:三个节点各 1 个 Pod,全部 1/1 Running
- `coredns`:2 个副本 Running
- `etcd` / `kube-apiserver` / `kube-controller-manager` / `kube-scheduler`:全部 Running

### 14.3 冒烟测试

部署 `nginx:alpine` 2 副本:

- Pod 分别调度到 k8s02(`10.244.1.2`)与 k8s03(`10.244.2.2`)
- NodePort `32762` 通过 `192.168.150.171` / `192.168.150.172` 访问均返回 HTTP 200
- 验证完毕已删除测试资源

### 14.4 本次 join 命令(24 小时有效)

```bash
kubeadm join 192.168.150.170:6443 --token sg4ai7.dmqqbftydxpjyxcy \
  --discovery-token-ca-cert-hash sha256:277cc23a31c694c6c74b712094294a286584c4851efe21b34a8fb2aa40727c0a
```

> token 过期后,在 master 上执行 `kubeadm token create --print-join-command` 重新生成。

---

## 15. 安装 Kubernetes Dashboard(Web 控制台)

### 15.1 部署

```bash
curl -fsSL -o dashboard.yaml \
  https://ghfast.top/https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
kubectl apply -f dashboard.yaml
kubectl -n kubernetes-dashboard rollout status deployment/kubernetes-dashboard
kubectl -n kubernetes-dashboard rollout status deployment/dashboard-metrics-scraper
```

镜像为 `docker.io/kubernetesui/dashboard:v2.7.0` 与 `docker.io/kubernetesui/metrics-scraper:v1.0.9`,走 6.4 配置的 Docker Hub 加速即可。可先在三台预拉:

```bash
sudo ctr -n k8s.io images pull --hosts-dir /etc/containerd/certs.d docker.io/kubernetesui/dashboard:v2.7.0
sudo ctr -n k8s.io images pull --hosts-dir /etc/containerd/certs.d docker.io/kubernetesui/metrics-scraper:v1.0.9
```

### 15.2 创建管理员账号与长期 Token

```bash
kubectl -n kubernetes-dashboard create serviceaccount dashboard-admin
kubectl create clusterrolebinding dashboard-admin \
  --clusterrole=cluster-admin \
  --serviceaccount=kubernetes-dashboard:dashboard-admin

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: dashboard-admin-token
  namespace: kubernetes-dashboard
  annotations:
    kubernetes.io/service-account.name: dashboard-admin
type: kubernetes.io/service-account-token
EOF

# 读取 Token(登录用)
kubectl -n kubernetes-dashboard get secret dashboard-admin-token \
  -o jsonpath='{.data.token}' | base64 -d; echo
```

> `kubectl create token` 生成的 Token 默认 1 小时过期;用上面的 Secret 方式生成的 Token 长期有效,除非删除 Secret。

### 15.3 暴露 NodePort

```bash
kubectl -n kubernetes-dashboard patch svc kubernetes-dashboard -p '{"spec":{"type":"NodePort"}}'
kubectl -n kubernetes-dashboard get svc kubernetes-dashboard
```

本环境实际分配到的 NodePort 为 `32005`,访问地址:

`https://192.168.150.170:32005`(或 `.171` / `.172` 任一节点 IP)

### 15.4 登录

1. 浏览器打开上述地址,自签名证书告警 → “高级” → “继续前往”
2. 选择 **Token** 登录方式,粘贴 15.2 输出的 Token
3. 登录后可查看/管理所有 Namespace、Deployment、Pod、Service 等资源

### 15.5 安全提示

- `dashboard-admin` 绑定了 `cluster-admin`,拿到 Token 等于拿到集群最高权限
- 只适合内网实验环境,不要暴露到公网
- 卸载:Dashboard 资源 `kubectl delete -f dashboard.yaml`,再删除 `dashboard-admin` 的 ServiceAccount / ClusterRoleBinding / Token Secret

---

## 16. 在集群上部署 AI-OJ 微服务项目(实际案例)

集群搭好后,可在其之上部署实际业务项目 `D:\code\oj\ai-oj`(Spring Cloud 微服务 + Vue 前端 + Rust 判题沙箱)。

### 16.1 部署内容

- 命名空间:`ai-oj`
- 中间件:MySQL 8.0.40、Redis 7.4、Nacos 2.4.3、RocketMQ 5.3.2、Seata 2.3.0、MinIO、Elasticsearch 8.19.21
- 微服务:gateway / auth / problem / judge / contest / file / ai / log(共 8 个)
- 前端:nginx(NodePort 30080)
- 判题沙箱:Rust 服务 + DinD(特权 Pod,节点 k8s03)

### 16.2 访问入口

| 入口 | 地址 |
|---|---|
| 前端 | http://192.168.150.170:30080 |
| 网关 | http://192.168.150.170:30088 |
| MinIO Console | http://192.168.150.170:30901 |
| 登录账号 | admin / admin123 |

### 16.3 部署文档

完整清单、镜像分发、初始化 Job、沙箱设计与排错手册见:

- `D:\code\oj\ai-oj\k8s\README.md`
- 清单目录:`D:\code\oj\ai-oj\k8s\*.yaml`

### 16.4 该案例涉及的额外知识点

- 镜像通过 `docker save | gzip | ssh ... ctr images import -` 流式导入节点
- RocketMQ 镜像 uid 3000 与 hostPath 权限冲突 → `runAsUser: 0`
- Seata 默认堆 2G 超过容器限制 → `JVM_XMS/JVM_XMX`
- Elasticsearch 不允许 root 运行 → initContainer chown
- K8s 环境变量名不允许点号 → ES 改用 ConfigMap 挂载 `elasticsearch.yml`
- 判题沙箱需要 Docker → Pod 内 DinD + hostPath 卷共享
- 语言镜像需预载到 DinD(`/data/ai-oj/sandbox/images/*.tar`)

---

## 附录:关键版本与状态

| 项目 | 实际值 |
|---|---|
| Ubuntu | 24.04.4 LTS(noble) |
| 内核 | 6.8.0-100-generic |
| containerd | v2.4.2(`containerd.io 2.4.2-3~ubuntu.24.04~noble`) |
| kubeadm / kubelet / kubectl | v1.37.2 |
| CNI | Flannel v0.28.10(cni-plugin v1.9.1-flannel3) |
| Pod CIDR | 10.244.0.0/16 |
| Service CIDR | 10.96.0.0/12(kubeadm 默认) |
| 节点状态 | k8s01 / k8s02 / k8s03 全部 Ready |
| 冒烟测试 | nginx 2 副本跨节点 Running,NodePort 32762 返回 200 |
