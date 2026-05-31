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

## 节点规划

| 角色 | 主机名 | IP | 说明 |
|------|--------|----|------|
| 控制机（Ansible） | ollama | 192.168.88.118 | 不加入集群 |
| 控制节点 | master1 | 192.168.88.119 | 初始主节点 |
| 控制节点 | master2 | 192.168.88.120 | |
| 控制节点 | master3 | 192.168.88.124 | |
| 工作节点 | worker1 | 192.168.88.121 | |
| 工作节点 | worker2 | 192.168.88.122 | |
| 工作节点 | worker3 | 192.168.88.123 | |
| VIP | — | 192.168.88.200 | 同网段空闲 IP |

---

## 软件版本

| 组件 | 版本 |
|------|------|
| OS | CentOS Stream 9 |
| Kubernetes | v1.35.5 |
| containerd | v2.2.x |
| Cilium | v1.18.10 |
| Helm | v3.17.3 |

---

## 快速开始

### 前置条件

```bash
# 控制机安装 Ansible
dnf install -y ansible

# 配置所有节点 SSH 免密
for ip in 192.168.88.119 192.168.88.120 192.168.88.121 \
          192.168.88.122 192.168.88.123 192.168.88.124; do
  ssh-copy-id root@$ip
done
```

### 修改配置

```bash
vim group_vars/all.yml   # 修改 VIP 等变量
vim inventory.ini        # 修改节点 IP
```

### 部署

```bash
# 全新部署
ansible-playbook -i inventory.ini site.yml

# 重置后重新部署
ansible-playbook -i inventory.ini reset.yml
rm -f /tmp/k8s_worker_join.sh /tmp/k8s_master_join.sh
ansible-playbook -i inventory.ini site.yml
```

> 注意：Cilium 镜像较大（约 300MB），首次部署约需 20-30 分钟。

---

## 部署流程

```
Step 1  所有节点系统初始化
Step 2  所有节点安装 containerd
Step 3  所有节点安装 kubelet/kubeadm/cri-tools + 预拉取 Cilium 镜像
Step 4  控制节点启动 Keepalived
Step 5  master1 启动 HAProxy
Step 6  master1 kubeadm init（跳过 kube-proxy）+ 安装 Helm + 部署 Cilium
Step 7  master2/master3 串行加入控制平面
Step 8  master2/master3 启动 HAProxy
Step 9  worker 加入集群
Step 10 等待全部 6 个节点 Ready
Step 11 输出最终状态报告
```

---

## Cilium 关键配置

```yaml
# kube-proxy 完全替换（eBPF 实现 Service 路由）
kubeProxyReplacement: true
k8sServiceHost: "192.168.88.200"
k8sServicePort: "6443"

# 隧道模式（兼容 VMware 环境）
tunnelProtocol: "vxlan"

# Hubble 可观测性（默认开启）
hubble:
  enabled: true
  relay:
    enabled: true
  ui:
    enabled: true
```

---

## 镜像来源

所有 Cilium 镜像通过华为云 SWR 拉取（`image.override` 方式完全覆盖 chart 默认地址）：

| 组件 | 镜像 |
|------|------|
| Cilium Agent | swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/cilium/cilium:v1.18.10 |
| Cilium Operator | swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/cilium/operator-generic:v1.18.10 |
| Hubble Relay | swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/cilium/hubble-relay:v1.18.10 |
| Hubble UI | swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/cilium/hubble-ui:v0.13.3 |
| K8s 核心组件 | registry.aliyuncs.com/google_containers |

---

## 验证集群

```bash
# 节点状态
kubectl get nodes -o wide

# Cilium Pod 状态
kubectl get pods -n kube-system -l k8s-app=cilium -o wide

# Cilium 健康检查
kubectl exec -n kube-system ds/cilium -- cilium status

# 确认 kube-proxy 已被替换（应无 kube-proxy Pod）
kubectl get pods -n kube-system | grep kube-proxy

# Hubble 流量监控
kubectl exec -n kube-system ds/cilium -- hubble observe --last 20
```

---

## 目录结构

```
k8s-Cilium-v5/
├── inventory.ini
├── group_vars/all.yml         # Cilium 镜像、版本等变量
├── site.yml
├── reset.yml                  # 含 Cilium 网络接口清理
└── roles/
    ├── common/                # sysctl 含 eBPF 所需参数
    ├── containerd/
    ├── kubernetes/            # 预拉取 Cilium 镜像
    ├── keepalived_only/
    ├── haproxy_only/
    ├── master_init/           # kubeadm init（skip kube-proxy）+ Helm + Cilium
    ├── master_join/
    └── worker_join/
```

---，欢迎 Issue 和 PR。
