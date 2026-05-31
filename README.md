# Cilium
k8s-Cilium— K8s HA 集群自动化部署（Cilium CNI）
## 集群架构

```
                    ┌─────────────────────────────────────────┐
                    │         VIP: 192.168.88.200:6443         │
                    │         Keepalived VRRP 漂移              │
                    └──────────────────┬──────────────────────┘
                                       │
              ┌────────────────────────┼────────────────────────┐
              │                        │                        │
    ┌─────────▼────────┐   ┌──────────▼───────┐   ┌───────────▼──────┐
    │     master1      │   │     master2      │   │     master3      │
    │  192.168.88.119  │   │  192.168.88.120  │   │  192.168.88.124  │
    │  HAProxy+KA      │   │  HAProxy+KA      │   │  HAProxy+KA      │
    │  kube-apiserver  │   │  kube-apiserver  │   │  kube-apiserver  │
    │  etcd            │◄──┤  etcd            ├──►│  etcd            │
    └──────────────────┘   └──────────────────┘   └──────────────────┘

              ┌────────────────────────────────────────────────┐
              │                  Worker 节点                    │
    ┌─────────▼────────┐   ┌──────────▼───────┐   ┌───────────▼──────┐
    │     worker1      │   │     worker2      │   │     worker3      │
    │  192.168.88.121  │   │  192.168.88.122  │   │  192.168.88.123  │
    └──────────────────┘   └──────────────────┘   └──────────────────┘
```

**etcd quorum = 2（3节点）**：任意一个控制节点宕机，集群写入不受影响。

---

## 节点规划

| 角色 | 主机名 | IP |
|------|--------|----|
| Ansible 控制机 | ollama | 192.168.88.118 |
| 控制节点 | master1 | 192.168.88.119 |
| 控制节点 | master2 | 192.168.88.120 |
| 控制节点 | master3 | 192.168.88.124 |
| 工作节点 | worker1 | 192.168.88.121 |
| 工作节点 | worker2 | 192.168.88.122 |
| 工作节点 | worker3 | 192.168.88.123 |
| VIP | — | 192.168.88.200 |

---

## 核心技术亮点

### 1. VMware ARP 问题修复

VMware 虚拟交换机过滤 Gratuitous ARP，导致 worker 节点无法通过 VIP 访问 API Server。

**解决方案：动态 DNAT**

```bash
# 每10秒通过 arping 检测 VIP 当前持有者
# 自动更新 iptables DNAT 规则
VIP_HOLDER=$(arping -c 2 -I $NIC $VIP | grep "Unicast reply" | awk '{print $5}')
iptables -t nat -I OUTPUT 1 -d $VIP -p tcp --dport 6443 \
  -j DNAT --to-destination ${VIP_HOLDER}:6443
```

### 2. kubeadm JSON Patch

kube-apiserver 需要绑定节点 IP（而非 0.0.0.0），以避免与 HAProxy 端口冲突：

```json
[{"op": "add", "path": "/spec/containers/0/command/-",
  "value": "--bind-address=节点IP"}]
```

只 patch kube-apiserver，不影响 controller-manager 和 scheduler。

### 3. kubeconfig 直连节点 IP

每个 master 的 kubeconfig `server` 字段指向本节点 IP，而非 VIP。  
master 宕机时，其他 master 的 kubectl 不受影响。

### 4. Keepalived notify 脚本

VIP 漂移时自动管理 DNAT：
- `notify_master`：本节点持有 VIP → 清除所有 DNAT（本地 HAProxy 直接处理）
- `notify_backup`：本节点失去 VIP → 设置 DNAT 指向 VIP 新持有者

---

## 快速开始

### 环境准备

```bash
# 控制机（192.168.88.118）安装 Ansible
dnf install -y ansible

# 所有节点 SSH 免密
for ip in 119 120 121 122 123 124; do
  ssh-copy-id root@192.168.88.$ip
done

# 验证连通性
ansible -i k8s-final9/inventory.ini all -m ping
```
# Cilium 方案（高性能）
cd k8s-Cilium-v5
ansible-playbook -i inventory.ini site.yml

## 重置集群

```bash
ansible-playbook -i inventory.ini reset.yml

## 软件版本

| 组件 | 版本 |
|------|------|
| OS | CentOS Stream 9 |
| Kubernetes | v1.35.5 |
| containerd | v2.2.x |
| Cilium | v1.18.10 |
| Helm | v4.2.0 |

## 镜像来源汇总

所有镜像均来自国内可访问的镜像仓库：

| 组件 | 镜像源 |
|------|--------|
| K8s 核心 | registry.aliyuncs.com/google_containers |
| Cilium | swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/cilium |


---

## 目录结构

```
.
── k8s-Cilium-v5/             # Cilium 方案
    ├── README.md
    ├── inventory.ini
    ├── group_vars/all.yml
    ├── site.yml
    ├── reset.yml
    └── roles/
```

---

## 常见问题

**Q: master 宕机后 worker 节点 NotReady？**  
A: 等待约 10-20 秒，动态 DNAT 定时器会自动更新路由。  
查看日志：`cat /var/log/k8s-dnat.log`

**Q: crictl 不可用？**  
A: cri-tools 已在 Step 3 安装。若仍不可用：  
`ansible -i inventory.ini all -m dnf -a "name=cri-tools state=present"`

**Q: Cilium 部署超时？**  
A: Cilium 镜像约 300MB，首次拉取需要时间。部署超时不代表失败，  
检查：`kubectl get pods -n kube-system -l k8s-app=cilium`

**Q: 如何验证 HA？**  
```bash
# 在 master1 上执行
poweroff

# 在 master2 上验证（约30秒后）
kubectl get nodes
kubectl get pods -A
```

---

## License

MIT License

---

> 本项目经过多轮实际部署验证，欢迎 Issue 和 PR。
