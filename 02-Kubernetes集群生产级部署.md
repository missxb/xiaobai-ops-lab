# Kubernetes 集群生产级部署

> 基于 kubeadm + Ansible + 二进制部署的高可用 Kubernetes 集群
> 参考方案：阿里云 ACK、字节跳动 K8s 集群部署实践

---

## 一、架构设计

### 1.1 高可用架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                  Kubernetes 高可用集群架构                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────── 负载均衡层 ───────────────────────┐   │
│  │         Keepalived + Nginx (VIP: 10.0.0.100)             │   │
│  └─────────────────────────────────────────────────────────┘   │
│         │                  │                  │                  │
│  ┌──────┴──────┐  ┌──────┴──────┐  ┌──────┴──────┐          │
│  │ Master-1    │  │ Master-2    │  │ Master-3    │          │
│  │ 10.0.0.11   │  │ 10.0.0.12   │  │ 10.0.0.13   │          │
│  │             │  │             │  │             │          │
│  │ kube-apiserver│ │ kube-apiserver│ │ kube-apiserver│        │
│  │ kube-ctrl-mgr│  │ kube-ctrl-mgr│  │ kube-ctrl-mgr│        │
│  │ kube-sched   │  │ kube-sched   │  │ kube-sched   │        │
│  │ etcd         │  │ etcd         │  │ etcd         │        │
│  │ kube-proxy   │  │ kube-proxy   │  │ kube-proxy   │        │
│  │ kubelet      │  │ kubelet      │  │ kubelet      │        │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘          │
│         │                  │                  │                  │
│  ┌──────┴──────────────────┴──────────────────┴──────┐        │
│  │              etcd 集群 (3节点)                     │        │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐          │        │
│  │  │ etcd-1  │  │ etcd-2  │  │ etcd-3  │          │        │
│  │  │10.0.0.21│  │10.0.0.22│  │10.0.0.23│          │        │
│  │  └─────────┘  └─────────┘  └─────────┘          │        │
│  └──────────────────────────────────────────────────┘        │
│         │                  │                  │                  │
│  ┌──────┴──────┐  ┌──────┴──────┐  ┌──────┴──────┐          │
│  │ Worker-1    │  │ Worker-2    │  │ Worker-N    │          │
│  │ 10.0.0.31   │  │ 10.0.0.32   │  │ 10.0.0.3N   │          │
│  │ kubelet     │  │ kubelet     │  │ kubelet     │          │
│  │ kube-proxy  │  │ kube-proxy  │  │ kube-proxy  │          │
│  │ containerd  │  │ containerd  │  │ containerd  │          │
│  │ Calico/Cilium│ │ Calico/Cilium│ │ Calico/Cilium│        │
│  └─────────────┘  └─────────────┘  └─────────────┘          │
│                                                                 │
│  ┌─────────────────────── 存储层 ───────────────────────┐    │
│  │   NFS / Ceph / Local Path Provisioner                │    │
│  └──────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 节点规划

| 角色 | 主机名 | IP | 规格 | 用途 |
|------|--------|-----|------|------|
| Master | k8s-master-1 | 10.0.0.11 | 8C16G | 控制平面 |
| Master | k8s-master-2 | 10.0.0.12 | 8C16G | 控制平面 |
| Master | k8s-master-3 | 10.0.0.13 | 8C16G | 控制平面 |
| etcd | k8s-etcd-1 | 10.0.0.21 | 4C8G | etcd 集群 |
| etcd | k8s-etcd-2 | 10.0.0.22 | 4C8G | etcd 集群 |
| etcd | k8s-etcd-3 | 10.0.0.23 | 4C8G | etcd 集群 |
| Worker | k8s-worker-1 | 10.0.0.31 | 16C64G | 工作节点 |
| Worker | k8s-worker-2 | 10.0.0.32 | 16C64G | 工作节点 |
| Worker | k8s-worker-3 | 10.0.0.33 | 16C64G | 工作节点 |

### 1.3 网络规划

| 网段 | CIDR | 用途 |
|------|------|------|
| Pod 网段 | 10.244.0.0/16 | Pod IP 地址池 |
| Service 网段 | 10.96.0.0/12 | Service ClusterIP |
| Node 网段 | 10.0.0.0/24 | 节点物理网络 |

---

## 二、环境准备 Ansible Playbook

### 2.1 Inventory 配置

```ini
# inventory/hosts.ini
[master]
k8s-master-1 ansible_host=10.0.0.11
k8s-master-2 ansible_host=10.0.0.12
k8s-master-3 ansible_host=10.0.0.13

[etcd]
k8s-etcd-1 ansible_host=10.0.0.21
k8s-etcd-2 ansible_host=10.0.0.22
k8s-etcd-3 ansible_host=10.0.0.23

[worker]
k8s-worker-1 ansible_host=10.0.0.31
k8s-worker-2 ansible_host=10.0.0.32
k8s-worker-3 ansible_host=10.0.0.33

[all:vars]
ansible_user=root
ansible_ssh_private_key_file=~/.ssh/id_rsa
ansible_python_interpreter=/usr/bin/python3
```

### 2.2 系统初始化 Playbook

```yaml
# playbooks/01-system-init.yml
---
- name: 系统初始化
  hosts: all
  become: yes
  vars:
    kubernetes_version: "1.29.3"
    containerd_version: "1.7.15"
    calico_version: "v3.27.2"
    
  tasks:
    # ===== 基础配置 =====
    - name: 设置主机名
      hostname:
        name: "{{ inventory_hostname }}"

    - name: 添加 /etc/hosts
      lineinfile:
        path: /etc/hosts
        line: "{{ hostvars[item].ansible_host }} {{ item }}"
      loop: "{{ groups['all'] }}"

    - name: 关闭 SELinux
      selinux:
        state: disabled
      when: ansible_os_family == "RedHat"

    - name: 关闭防火墙
      systemd:
        name: firewalld
        state: stopped
        enabled: no
      ignore_errors: yes

    - name: 禁用 swap
      command: swapoff -a
      changed_when: false

    - name: 永久禁用 swap
      lineinfile:
        path: /etc/fstab
        regexp: '.*swap.*'
        state: absent

    # ===== 内核参数调优 =====
    - name: 加载必要内核模块
      modprobe:
        name: "{{ item }}"
        state: present
      loop:
        - overlay
        - br_netfilter
        - ip_vs
        - ip_vs_rr
        - ip_vs_wrr
        - ip_vs_sh
        - nf_conntrack

    - name: 持久化内核模块
      copy:
        dest: /etc/modules-load.d/kubernetes.conf
        content: |
          overlay
          br_netfilter
          ip_vs
          ip_vs_rr
          ip_vs_wrr
          ip_vs_sh
          nf_conntrack

    - name: 设置内核参数
      sysctl:
        name: "{{ item.name }}"
        value: "{{ item.value }}"
        sysctl_file: /etc/sysctl.d/kubernetes.conf
        reload: yes
      loop:
        - { name: "net.bridge.bridge-nf-call-iptables", value: "1" }
        - { name: "net.bridge.bridge-nf-call-ip6tables", value: "1" }
        - { name: "net.ipv4.ip_forward", value: "1" }
        - { name: "net.ipv4.conf.all.forwarding", value: "1" }
        - { name: "net.ipv6.conf.all.forwarding", value: "1" }
        - { name: "net.ipv4.tcp_keepalive_time", value: "600" }
        - { name: "net.ipv4.tcp_keepalive_intvl", value: "30" }
        - { name: "net.ipv4.tcp_keepalive_probes", value: "10" }
        - { name: "net.ipv4.tcp_max_syn_backlog", value: "8096" }
        - { name: "net.ipv4.tcp_max_tw_buckets", value: "5000" }
        - { name: "net.ipv4.tcp_tw_reuse", value: "1" }
        - { name: "net.ipv4.ip_local_port_range", value: "1024 65535" }
        - { name: "net.netfilter.nf_conntrack_max", value: "1048576" }
        - { name: "fs.inotify.max_user_watches", value: "524288" }
        - { name: "fs.inotify.max_user_instances", value: "8192" }

    # ===== 安装 containerd =====
    - name: 安装 containerd 依赖
      yum:
        name:
          - yum-utils
          - device-mapper-persistent-data
          - lvm2
        state: present

    - name: 添加 Docker 仓库
      yum_repository:
        name: docker-ce
        description: Docker CE Stable
        baseurl: https://mirrors.aliyun.com/docker-ce/linux/centos/$releasever/$basearch/stable/
        gpgcheck: yes
        gpgkey: https://mirrors.aliyun.com/docker-ce/linux/centos/gpg
        enabled: yes

    - name: 安装 containerd
      yum:
        name: "containerd.io-{{ containerd_version }}*"
        state: present
        update_cache: yes

    - name: 生成 containerd 配置
      shell: containerd config default > /etc/containerd/config.toml
      args:
        creates: /etc/containerd/config.toml

    - name: 配置 containerd 使用 systemd cgroup
      replace:
        path: /etc/containerd/config.toml
        regexp: 'SystemdCgroup = false'
        replace: 'SystemdCgroup = true'

    - name: 配置 containerd 使用阿里云镜像
      replace:
        path: /etc/containerd/config.toml
        regexp: 'sandbox_image = "registry.k8s.io/pause:.*"'
        replace: 'sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.9"'

    - name: 启动 containerd
      systemd:
        name: containerd
        state: restarted
        enabled: yes

    # ===== 安装 kubeadm/kubelet/kubectl =====
    - name: 添加 Kubernetes 仓库
      yum_repository:
        name: kubernetes
        description: Kubernetes
        baseurl: https://mirrors.aliyun.com/kubernetes-new/core/stable/v{{ kubernetes_version }}/rpm/
        gpgcheck: yes
        gpgkey: https://mirrors.aliyun.com/kubernetes-new/core/stable/v{{ kubernetes_version }}/rpm/repodata/repomd.xml.key
        enabled: yes

    - name: 安装 kubeadm、kubelet、kubectl
      yum:
        name:
          - "kubelet-{{ kubernetes_version }}*"
          - "kubeadm-{{ kubernetes_version }}*"
          - "kubectl-{{ kubernetes_version }}*"
        state: present
        update_cache: yes

    - name: 启动 kubelet
      systemd:
        name: kubelet
        enabled: yes
        state: started

    # ===== 配置 crictl =====
    - name: 配置 crictl
      copy:
        dest: /etc/crictl.yaml
        content: |
          runtime-endpoint: unix:///run/containerd/containerd.sock
          image-endpoint: unix:///run/containerd/containerd.sock
          timeout: 10
          debug: false

    # ===== 时间同步 =====
    - name: 安装 chrony
      yum:
        name: chrony
        state: present

    - name: 配置 chrony 服务器
      lineinfile:
        path: /etc/chrony.conf
        line: "server ntp.aliyun.com iburst"
        insertbefore: "^server"

    - name: 启动 chrony
      systemd:
        name: chronyd
        state: restarted
        enabled: yes

    # ===== 关闭不需要的服务 =====
    - name: 关闭不需要的服务
      systemd:
        name: "{{ item }}"
        state: stopped
        enabled: no
      loop:
        - postfix
        - rpcbind
      ignore_errors: yes
```

### 2.3 Ansible 主 Playbook

```yaml
# site.yml - 主入口
---
- import_playbook: playbooks/01-system-init.yml
- import_playbook: playbooks/02-etcd-deploy.yml
- import_playbook: playbooks/03-k8s-init.yml
- import_playbook: playbooks/04-network-plugin.yml
- import_playbook: playbooks/05-worker-join.yml
- import_playbook: playbooks/06-cluster-addons.yml
```

---

## 三、etcd 集群部署

### 3.1 etcd 二进制部署

```yaml
# playbooks/02-etcd-deploy.yml
---
- name: 部署 etcd 集群
  hosts: etcd
  become: yes
  vars:
    etcd_version: "3.5.12"
    etcd_data_dir: /var/lib/etcd
    etcd_cert_dir: /etc/kubernetes/pki/etcd
    
  tasks:
    - name: 创建 etcd 目录
      file:
        path: "{{ item }}"
        state: directory
        mode: '0700'
      loop:
        - /opt/etcd
        - "{{ etcd_data_dir }}"
        - "{{ etcd_cert_dir }}"

    - name: 下载 etcd 二进制
      get_url:
        url: "https://github.com/etcd-io/etcd/releases/download/v${etcd_version}/etcd-v${etcd_version}-linux-amd64.tar.gz"
        dest: /tmp/etcd.tar.gz
        mode: '0644'

    - name: 解压 etcd
      unarchive:
        src: /tmp/etcd.tar.gz
        dest: /opt/etcd/
        remote_src: yes

    - name: 复制 etcd 二进制
      copy:
        src: "/opt/etcd/etcd-v{{ etcd_version }}-linux-amd64/{{ item }}"
        dest: "/usr/local/bin/{{ item }}"
        mode: '0755'
        remote_src: yes
      loop:
        - etcd
        - etcdctl
        - etcdutl

    - name: 创建 etcd 用户
      user:
        name: etcd
        system: yes
        shell: /bin/false
        home: /var/lib/etcd

    - name: 生成 etcd systemd 服务
      template:
        src: etcd.service.j2
        dest: /etc/systemd/system/etcd.service
      notify: restart etcd

  handlers:
    - name: restart etcd
      systemd:
        name: etcd
        state: restarted
        daemon_reload: yes
```

### 3.2 etcd systemd 服务模板

```jinja2
# templates/etcd.service.j2
[Unit]
Description=etcd
Documentation=https://github.com/etcd-io/etcd
After=network.target

[Service]
Type=notify
User=etcd
ExecStart=/usr/local/bin/etcd \
  --name {{ inventory_hostname }} \
  --data-dir {{ etcd_data_dir }} \
  --listen-client-urls https://{{ ansible_host }}:2379 \
  --advertise-client-urls https://{{ ansible_host }}:2379 \
  --listen-peer-urls https://{{ ansible_host }}:2380 \
  --initial-advertise-peer-urls https://{{ ansible_host }}:2380 \
  --initial-cluster {% for host in groups['etcd'] %}{{ host }}=https://{{ hostvars[host].ansible_host }}:2380{% if not loop.last %},{% endif %}{% endfor %} \
  --initial-cluster-state new \
  --initial-cluster-token etcd-cluster-2026 \
  --client-cert-auth \
  --trusted-ca-file {{ etcd_cert_dir }}/ca.crt \
  --cert-file {{ etcd_cert_dir }}/server.crt \
  --key-file {{ etcd_cert_dir }}/server.key \
  --peer-client-cert-auth \
  --peer-trusted-ca-file {{ etcd_cert_dir }}/ca.crt \
  --peer-cert-file {{ etcd_cert_dir }}/peer.crt \
  --peer-key-file {{ etcd_cert_dir }}/peer.key \
  --snapshot-count 10000 \
  --heartbeat-interval 100 \
  --election-timeout 1000 \
  --quota-backend-bytes 8589934592 \
  --auto-compaction-retention "8" \
  --max-snapshots 5 \
  --max-wals 5
Restart=always
RestartSec=10
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

### 3.3 etcd 备份脚本

```bash
#!/bin/bash
# etcd-backup.sh - etcd 定时备份脚本
# 用法: crontab: 0 2 * * * /opt/scripts/etcd-backup.sh

set -euo pipefail

# 配置
ETCDCTL="/usr/local/bin/etcdctl"
BACKUP_DIR="/data/etcd-backup"
KEEP_DAYS=30
ETCD_ENDPOINTS="https://10.0.0.21:2379,https://10.0.0.22:2379,https://10.0.0.23:2379"
CERT_DIR="/etc/kubernetes/pki/etcd"

# 生成文件名
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
SNAPSHOT_FILE="${BACKUP_DIR}/snapshot_${TIMESTAMP}.db"
BACKUP_FILE="${BACKUP_DIR}/etcd-backup-${TIMESTAMP}.tar.gz"
LOG_FILE="${BACKUP_DIR}/backup.log"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a ${LOG_FILE}; }

mkdir -p ${BACKUP_DIR}

# 执行快照
log "开始 etcd 快照备份..."
${ETCDCTL} snapshot save ${SNAPSHOT_FILE} \
  --endpoints=${ETCD_ENDPOINTS} \
  --cacert=${CERT_DIR}/ca.crt \
  --cert=${CERT_DIR}/server.crt \
  --key=${CERT_DIR}/server.key

# 验证快照
log "验证快照..."
${ETCDCTL} snapshot status ${SNAPSHOT_FILE} --write-out=table | tee -a ${LOG_FILE}

# 压缩
log "压缩备份..."
tar -czf ${BACKUP_FILE} -C ${BACKUP_DIR} $(basename ${SNAPSHOT_FILE})

# 清理旧备份
log "清理 ${KEEP_DAYS} 天前的备份..."
find ${BACKUP_DIR} -name "etcd-backup-*.tar.gz" -mtime +${KEEP_DAYS} -delete
find ${BACKUP_DIR} -name "snapshot_*.db" -mtime +${KEEP_DAYS} -delete

# 验证备份完整性
log "验证备份完整性..."
tar -tzf ${BACKUP_FILE} > /dev/null 2>&1
if [ $? -eq 0 ]; then
  log "✅ 备份成功: ${BACKUP_FILE}"
else
  log "❌ 备份验证失败"
  exit 1
fi

# 输出备份信息
BACKUP_SIZE=$(du -sh ${BACKUP_FILE} | awk '{print $1}')
log "备份大小: ${BACKUP_SIZE}"
log "=== 备份完成 ==="
```

### 3.4 etcd 恢复脚本

```bash
#!/bin/bash
# etcd-restore.sh - etcd 恢复脚本
# 用法: ./etcd-restore.sh <snapshot-file>

set -euo pipefail

SNAPSHOT_FILE=${1:?用法: $0 <snapshot-file>}
ETCDCTL="/usr/local/bin/etcdctl"
ETCD_DATA_DIR="/var/lib/etcd"
CERT_DIR="/etc/kubernetes/pki/etcd"
ETCD_ENDPOINTS="https://10.0.0.21:2379,https://10.0.0.22:2379,https://10.0.0.23:2379"

RED='\033[0;31m'
GREEN='\033[0;32m'
NC='\033[0m'

log() { echo -e "${GREEN}[$(date '+%H:%M:%S')]${NC} $1"; }
error() { echo -e "${RED}[$(date '+%H:%M:%S')] ❌ $1${NC}"; exit 1; }

# 确认操作
read -p "⚠️  确认恢复 etcd？这将覆盖当前所有数据！(y/N): " CONFIRM
if [[ ! "${CONFIRM}" =~ ^[Yy]$ ]]; then
  log "已取消"
  exit 0
fi

# 停止 etcd
log "停止 etcd 服务..."
systemctl stop etcd

# 备份当前数据
log "备份当前数据..."
mv ${ETCD_DATA_DIR} ${ETCD_DATA_DIR}.bak.$(date +%Y%m%d%H%M%S) 2>/dev/null || true

# 恢复快照
log "恢复快照..."
${ETCDCTL} snapshot restore ${SNAPSHOT_FILE} \
  --data-dir=${ETCD_DATA_DIR} \
  --name=$(hostname) \
  --initial-cluster="{% for host in groups['etcd'] %}{{ host }}=https://{{ hostvars[host].ansible_host }}:2380{% if not loop.last %},{% endif %}{% endfor %}" \
  --initial-cluster-token=etcd-cluster-2026 \
  --initial-advertise-peer-urls="https://$(hostname -I | awk '{print $1}'):2380"

# 修复权限
chown -R etcd:etcd ${ETCD_DATA_DIR}

# 启动 etcd
log "启动 etcd 服务..."
systemctl start etcd

# 验证
log "验证 etcd 状态..."
sleep 5
${ETCDCTL} endpoint health \
  --endpoints=${ETCD_ENDPOINTS} \
  --cacert=${CERT_DIR}/ca.crt \
  --cert=${CERT_DIR}/server.crt \
  --key=${CERT_DIR}/server.key

log "✅ etcd 恢复完成"
```

---

## 四、Kubernetes 控制平面部署

### 4.1 kubeadm 配置

```yaml
# kubeadm-config.yml - Kubernetes 初始化配置
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: v1.29.3
controlPlaneEndpoint: "10.0.0.100:6443"  # Keepalived VIP
imageRepository: registry.aliyuncs.com/google_containers
certificatesDir: /etc/kubernetes/pki
networking:
  serviceSubnet: "10.96.0.0/12"
  podSubnet: "10.244.0.0/16"
  dnsDomain: "cluster.local"
apiServer:
  extraArgs:
    enable-admission-plugins: "NodeRestriction,PodSecurity,ServiceAccount"
    audit-log-path: "/var/log/kubernetes/audit.log"
    audit-log-maxage: "30"
    audit-log-maxbackup: "10"
    audit-log-maxsize: "100"
    profiling: "false"
    request-timeout: "300s"
    service-account-lookup: "true"
  extraVolumes:
    - name: audit-log
      hostPath: /var/log/kubernetes
      mountPath: /var/log/kubernetes
      pathType: DirectoryOrCreate
controllerManager:
  extraArgs:
    node-monitor-grace-period: "40s"
    pod-eviction-timeout: "5m0s"
    terminated-pod-gc-threshold: "100"
    use-service-account-credentials: "true"
    feature-gates: "RotateKubeletServerCertificate=true"
scheduler:
  extraArgs:
    profiling: "false"
etcd:
  external:
    endpoints:
      - https://10.0.0.21:2379
      - https://10.0.0.22:2379
      - https://10.0.0.23:2379
    caFile: /etc/kubernetes/pki/etcd/ca.crt
    certFile: /etc/kubernetes/pki/etcd/etcd-client.crt
    keyFile: /etc/kubernetes/pki/etcd/etcd-client.key
```

### 4.2 初始化第一个 Master

```bash
#!/bin/bash
# init-master-1.sh - 初始化第一个 Master 节点

set -euo pipefail

echo "=== 初始化第一个 Master 节点 ==="

# 使用 kubeadm 初始化
kubeadm init \
  --config=kubeadm-config.yml \
  --upload-certs \
  --skip-phases=addon/kube-proxy

# 配置 kubectl
mkdir -p $HOME/.kube
cp -f /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

# 安装 Calico 网络插件
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.2/manifests/calico.yaml

# 等待所有 Pod 就绪
echo "等待所有 Pod 就绪..."
kubectl wait --for=condition=Ready pods --all -n kube-system --timeout=300s

# 验证集群
echo "=== 集群状态 ==="
kubectl get nodes
kubectl get pods -n kube-system

# 输出 join 命令
echo ""
echo "=== 其他节点加入命令 ==="
kubeadm token create --print-join-command
```

### 4.3 加入其他 Master

```bash
#!/bin/bash
# join-master.sh - 加入其他 Master 节点
# 用法: ./join-master.sh <certificate-key> <discovery-token>

set -euo pipefail

CERTIFICATE_KEY=${1:?用法: $0 <certificate-key>}
TOKEN=${2:?用法: $0 <token>}

echo "=== 加入其他 Master 节点 ==="

kubeadm join 10.0.0.100:6443 \
  --token ${TOKEN} \
  --discovery-token-ca-cert-hash sha256:$(openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //') \
  --control-plane \
  --certificate-key ${CERTIFICATE_KEY} \
  --apiserver-advertise-address $(hostname -I | awk '{print $1}')

# 配置 kubectl
mkdir -p $HOME/.kube
cp -f /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

# 验证
kubectl get nodes
```

---

## 五、网络插件部署

### 5.1 Calico 完整配置

```yaml
# calico-installation.yaml
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  registry: quay.io
  imagePullSecrets: []
  controlPlaneReplicas: 3
  typhaDeployment:
    metadata:
      labels:
        app: calico-typha
    spec:
      minReadySeconds: 60
      strategy:
        type: RollingUpdate
        rollingUpdate:
          maxSurge: 1
          maxUnavailable: 1
      template:
        metadata:
          labels:
            app: calico-typha
        spec:
          affinity:
            podAntiAffinity:
              preferredDuringSchedulingIgnoredDuringExecution:
                - weight: 100
                  podAffinityTerm:
                    labelSelector:
                      matchExpressions:
                        - key: app
                          operator: In
                          values:
                            - calico-typha
                    topologyKey: kubernetes.io/hostname
          tolerations:
            - key: node-role.kubernetes.io/master
              effect: NoSchedule
          containers:
            - name: typha
              resources:
                requests:
                  cpu: 100m
                  memory: 100Mi
                limits:
                  cpu: 500m
                  memory: 200Mi
  calicoNodeDaemonSet:
    metadata:
      labels:
        app: calico-node
    spec:
      template:
        spec:
          containers:
            - name: calico-node
              resources:
                requests:
                  cpu: 200m
                  memory: 200Mi
                limits:
                  cpu: "1"
                  memory: 1Gi
  calicoKubeControllersDeployment:
    metadata:
      labels:
        app: calico-kube-controllers
    spec:
      template:
        spec:
          containers:
            - name: calico-kube-controllers
              resources:
                requests:
                  cpu: 100m
                  memory: 100Mi
                limits:
                  cpu: 500m
                  memory: 500Mi

---
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
```

### 5.2 IP Pool 配置

```yaml
# calico-ippool.yaml
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: default-ipv4-ippool
spec:
  cidr: 10.244.0.0/16
  natOutgoing: true
  nodeSelector: all()
  blockSize: 26  # 每个节点 /26 子网（64个IP）

---
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  logSeverityScreen: Info
  nodeToNodeMeshEnabled: true
  asNumber: 64512
  serviceClusterIPs:
    - cidr: 10.96.0.0/12
  serviceExternalIPs: []
```

### 5.3 网络策略示例

```yaml
# network-policy-default-deny.yaml
# 默认拒绝所有流量
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress

---
# 允许 DNS 查询
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53

---
# 允许同一 namespace 内的 Pod 通信
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector: {}
  egress:
    - to:
        - podSelector: {}
```

---

## 六、Ingress Controller 部署

### 6.1 Nginx Ingress 完整配置

```yaml
# nginx-ingress-values.yaml
controller:
  replicaCount: 3
  
  # 资源限制
  resources:
    requests:
      cpu: 200m
      memory: 256Mi
    limits:
      cpu: "2"
      memory: 2Gi
  
  # HPA 自动扩缩
  autoscaling:
    enabled: true
    minReplicas: 3
    maxReplicas: 20
    targetCPUUtilizationPercentage: 70
    targetMemoryUtilizationPercentage: 80
  
  # Pod 反亲和性
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchExpressions:
                - key: app.kubernetes.io/name
                  operator: In
                  values:
                    - ingress-nginx
            topologyKey: kubernetes.io/hostname
  
  # 安全配置
  config:
    # SSL 配置
    ssl-protocols: "TLSv1.2 TLSv1.3"
    ssl-ciphers: "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256"
    ssl-prefer-server-ciphers: "true"
    
    # 安全头
    add-headers: "ingress-nginx/custom-headers"
    
    # 限流
    limit-req-status-code: 429
    
    # 代理配置
    proxy-body-size: "50m"
    proxy-read-timeout: "60"
    proxy-send-timeout: "60"
    proxy-connect-timeout: "10"
    
    # Gzip 压缩
    use-gzip: "true"
    gzip-level: "5"
    gzip-min-length: "256"
    
    # 日志格式
    log-format-upstream: '$remote_addr - $remote_user [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent" $request_length $request_time [$proxy_upstream_name] [$proxy_alternative_upstream_name] $upstream_addr $upstream_response_length $upstream_response_time $upstream_status $req_id'
    
    # 性能优化
    keep-alive: "75"
    keep-alive-requests: "1000"
    upstream-keepalive-connections: "64"
    enable-real-ip: "true"
  
  # 指标
  metrics:
    enabled: true
    serviceMonitor:
      enabled: true
      namespace: monitoring
  
  # Pod 注解
  podAnnotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "10254"
    prometheus.io/path: "/metrics"

# 默认后端
defaultBackend:
  enabled: true
  replicaCount: 2
  resources:
    requests:
      cpu: 10m
      memory: 20Mi
    limits:
      cpu: 50m
      memory: 64Mi

# TCP/UDP 服务
tcp: {}
udp: {}
```

### 6.2 自定义安全头

```yaml
# custom-headers-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: custom-headers
  namespace: ingress-nginx
data:
  X-Frame-Options: "DENY"
  X-Content-Type-Options: "nosniff"
  X-XSS-Protection: "1; mode=block"
  Referrer-Policy: "strict-origin-when-cross-origin"
  Content-Security-Policy: "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self' data:;"
  Strict-Transport-Security: "max-age=31536000; includeSubDomains; preload"
  Permissions-Policy: "camera=(), microphone=(), geolocation=()"
```

### 6.3 灰度发布配置

```yaml
# canary-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-canary
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"  # 10% 流量到金丝雀
spec:
  ingressClassName: nginx
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-canary
                port:
                  number: 80
```

---

## 七、集群组件部署

### 7.1 CoreDNS 调优

```yaml
# coredns-custom.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-custom
  namespace: kube-system
data:
  custom.server: |
    # 自定义 DNS 解析
    example.com:53 {
        errors
        cache 30
        forward . 10.0.0.10  # 内部 DNS
    }
  ready.override: |
    # 就绪探针配置
    ready
  log.override: |
    # 日志配置
    log . {
        class error
    }
  prometheus.override: |
    # Prometheus 指标
    prometheus :9153
  cache.override: |
    # 缓存配置
    cache 30 {
        success 9984 30
        denial 9984 5
        prefetch 10 60s 10%
    }
  loadbalance.override: |
    # 负载均衡策略
    loadbalance round_robin
```

### 7.2 Metrics Server

```yaml
# metrics-server-values.yaml
args:
  - --metric-resolution=15s
  - --kubelet-insecure-tls
  - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
resources:
  requests:
    cpu: 100m
    memory: 200Mi
  limits:
    cpu: 300m
    memory: 400Mi
replicas: 2
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: metrics-server
          topologyKey: kubernetes.io/hostname
```

---

## 八、节点管理脚本

### 8.1 添加 Worker 节点

```bash
#!/bin/bash
# add-worker.sh - 添加 Worker 节点
# 用法: ./add-worker.sh <node-ip> <node-name> [labels]

set -euo pipefail

NODE_IP=${1:?用法: $0 <node-ip> <node-name> [labels]}
NODE_NAME=${2:?}
LABELS=${3:-""}

# 获取 join 命令
JOIN_CMD=$(kubeadm token create --print-join-command)

echo "=== 添加 Worker 节点 ==="
echo "节点: ${NODE_NAME} (${NODE_IP})"

# 在目标节点执行
ssh root@${NODE_IP} "${JOIN_CMD}"

# 等待节点就绪
echo "等待节点就绪..."
kubectl wait --for=condition=Ready node/${NODE_NAME} --timeout=300s

# 添加标签
if [ -n "${LABELS}" ]; then
  IFS=',' read -ra LABEL_ARRAY <<< "${LABELS}"
  for label in "${LABEL_ARRAY[@]}"; do
    kubectl label node ${NODE_NAME} ${label} --overwrite
  done
fi

# 验证
kubectl get node ${NODE_NAME} -o wide
echo "✅ 节点添加完成"
```

### 8.2 维护模式脚本

```bash
#!/bin/bash
# maintenance-mode.sh - 节点维护模式
# 用法: ./maintenance-mode.sh <node-name> <enter|exit>

set -euo pipefail

NODE_NAME=${1:?用法: $0 <node-name> <enter|exit>}
ACTION=${2:?}

case ${ACTION} in
  enter)
    echo "=== 进入维护模式: ${NODE_NAME} ==="
    
    # 标记不可调度
    kubectl cordon ${NODE_NAME}
    
    # 驱逐 Pod（保留 DaemonSet）
    kubectl drain ${NODE_NAME} \
      --ignore-daemonsets \
      --delete-emptydir-data \
      --force \
      --grace-period=60 \
      --timeout=300s
    
    echo "✅ 已进入维护模式"
    ;;
    
  exit)
    echo "=== 退出维护模式: ${NODE_NAME} ==="
    
    # 恢复调度
    kubectl uncordon ${NODE_NAME}
    
    echo "✅ 已退出维护模式"
    ;;
    
  *)
    echo "用法: $0 <node-name> <enter|exit>"
    exit 1
    ;;
esac
```

### 8.3 节点升级脚本

```bash
#!/bin/bash
# upgrade-node.sh - 升级节点组件
# 用法: ./upgrade-node.sh <node-name> <target-version>

set -euo pipefail

NODE_NAME=${1:?用法: $0 <node-name> <target-version>}
TARGET_VERSION=${2:?}

echo "=== 升级节点: ${NODE_NAME} ==="
echo "目标版本: ${TARGET_VERSION}"

# 获取节点 IP
NODE_IP=$(kubectl get node ${NODE_NAME} -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')

# 进入维护模式
echo "进入维护模式..."
kubectl cordon ${NODE_NAME}
kubectl drain ${NODE_NAME} --ignore-daemonsets --delete-emptydir-data --force

# SSH 到节点升级
echo "升级 kubelet 和 kubeadm..."
ssh root@${NODE_IP} "
  yum install -y kubelet-${TARGET_VERSION}* kubeadm-${TARGET_VERSION}* kubectl-${TARGET_VERSION}*
  systemctl daemon-reload
  systemctl restart kubelet
"

# 等待节点就绪
echo "等待节点就绪..."
kubectl wait --for=condition=Ready node/${NODE_NAME} --timeout=300s

# 退出维护模式
kubectl uncordon ${NODE_NAME}

echo "✅ 节点升级完成"
kubectl get node ${NODE_NAME} -o wide
```

---

## 九、集群升级方案

### 9.1 升级流程

```bash
#!/bin/bash
# upgrade-cluster.sh - Kubernetes 集群升级
# 用法: ./upgrade-cluster.sh <current-version> <target-version>
# 示例: ./upgrade-cluster.sh 1.28 1.29

set -euo pipefail

CURRENT_VERSION=${1:?用法: $0 <current-version> <target-version>}
TARGET_VERSION=${2:?}

echo "=== Kubernetes 集群升级 ==="
echo "当前版本: ${CURRENT_VERSION}"
echo "目标版本: ${TARGET_VERSION}"

# 升级检查
echo "--- 升级前检查 ---"
kubectl get nodes -o wide
kubectl get pods -A --field-selector=status.phase!=Running

# 升级第一个 Master
echo "--- 升级第一个 Master ---"
MASTER1=$(kubectl get nodes -l node-role.kubernetes.io/master -o jsonpath='{.items[0].metadata.name}')
ssh root@$(kubectl get node ${MASTER1} -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}') "
  kubeadm upgrade plan
  kubeadm upgrade apply v${TARGET_VERSION}.0 --yes
  kubeletVersion=\$(kubelet --version | awk '{print \$2}' | cut -d'v' -f2)
  yum install -y kubelet-\${kubeletVersion}* kubeadm-\${kubeletVersion}* kubectl-\${kubeletVersion}*
  systemctl daemon-reload
  systemctl restart kubelet
"

# 升级其他 Master
echo "--- 升级其他 Master ---"
kubectl get nodes -l node-role.kubernetes.io/master -o jsonpath='{.items[*].metadata.name}' | \
  tr ' ' '\n' | tail -n +2 | while read MASTER; do
  echo "升级 ${MASTER}..."
  MASTER_IP=$(kubectl get node ${MASTER} -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')
  ssh root@${MASTER_IP} "
    kubeadm upgrade node
    kubeletVersion=\$(kubelet --version | awk '{print \$2}' | cut -d'v' -f2)
    yum install -y kubelet-\${kubeletVersion}* kubeadm-\${kubeletVersion}* kubectl-\${kubeletVersion}*
    systemctl daemon-reload
    systemctl restart kubelet
  "
done

# 升级 Worker 节点（逐个升级）
echo "--- 升级 Worker 节点 ---"
kubectl get nodes -l '!node-role.kubernetes.io/master' -o jsonpath='{.items[*].metadata.name}' | \
  tr ' ' '\n' | while read WORKER; do
  echo "升级 ${WORKER}..."
  ./upgrade-node.sh ${WORKER} ${TARGET_VERSION}.0
  sleep 30
done

# 验证
echo "--- 升级后验证 ---"
kubectl get nodes -o wide
kubectl get pods -A
kubectl version --short

echo "=== 升级完成 ==="
```

### 9.2 回滚方案

```bash
#!/bin/bash
# rollback-cluster.sh - 集群回滚
# 注意：Kubernetes 不支持原生回滚，需要使用备份恢复

set -euo pipefail

TARGET_VERSION=${1:?用法: $0 <target-version>}

echo "=== 集群回滚到 ${TARGET_VERSION} ==="

# 1. 停止调度
echo "停止集群调度..."
kubectl cordon --all

# 2. 恢复 etcd
echo "恢复 etcd..."
/opt/scripts/etcd-restore.sh /data/etcd-backup/latest-snapshot.db

# 3. 降级 Master 节点
echo "降级 Master 节点..."
for master in $(kubectl get nodes -l node-role.kubernetes.io/master -o jsonpath='{.items[*].metadata.name}'); do
  MASTER_IP=$(kubectl get node ${master} -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')
  ssh root@${MASTER_IP} "
    yum install -y kubelet-${TARGET_VERSION}* kubeadm-${TARGET_VERSION}* kubectl-${TARGET_VERSION}*
    systemctl daemon-reload
    systemctl restart kubelet
  "
done

# 4. 恢复调度
echo "恢复集群调度..."
kubectl uncordon --all

echo "=== 回滚完成 ==="
```

---

## 十、安全加固

### 10.1 PodSecurityStandards

```yaml
# pod-security-standards.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted

---
# 允许特定应用绕过限制
apiVersion: v1
kind: Namespace
metadata:
  name: system
  labels:
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/audit: privileged
    pod-security.kubernetes.io/warn: privileged
```

### 10.2 RBAC 最佳实践

```yaml
# rbac-best-practices.yaml
# 只读角色
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: readonly-all
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "endpoints", "namespaces"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets", "daemonsets"]
    verbs: ["get", "list", "watch"]

---
# 运维角色
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: operator-all
rules:
  - apiGroups: [""]
    resources: ["*"]
    verbs: ["*"]
  - apiGroups: ["apps"]
    resources: ["*"]
    verbs: ["*"]
  - apiGroups: ["networking.k8s.io"]
    resources: ["*"]
    verbs: ["*"]
  - apiGroups: ["autoscaling"]
    resources: ["*"]
    verbs: ["*"]
  - apiGroups: ["policy"]
    resources: ["*"]
    verbs: ["*"]

---
# 开发者角色 - 限定命名空间
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: development
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log", "services", "configmaps", "secrets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

### 10.3 审计日志配置

```yaml
# audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # 不记录只读请求
  - level: None
    resources:
      - group: ""
        resources: ["endpoints", "services", "services/status"]
    users: ["system:kube-proxy"]
    verbs: ["watch"]
  
  # 不记录 kube-system 命名空间的请求
  - level: None
    userGroups: ["system:serviceaccounts:kube-system"]
  
  # 记录 Secret 的请求
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets", "configmaps"]
  
  # 记录所有其他请求
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["pods", "services", "namespaces"]
      - group: "apps"
        resources: ["deployments", "statefulsets"]
  
  # 默认规则
  - level: Metadata
    omitStages:
      - "RequestReceived"
```

---

## 十一、运维手册

### 11.1 日常巡检脚本

```bash
#!/bin/bash
# cluster-health-check.sh - 集群健康检查

set -euo pipefail

echo "=== Kubernetes 集群健康检查 ==="
echo "时间: $(date '+%Y-%m-%d %H:%M:%S')"

# 1. 节点状态
echo -e "\n--- 节点状态 ---"
kubectl get nodes -o wide
echo ""
echo "节点资源使用:"
kubectl top nodes 2>/dev/null || echo "(metrics-server 未就绪)"

# 2. 系统组件
echo -e "\n--- 系统组件状态 ---"
kubectl get componentstatuses 2>/dev/null || echo "(componentstatuses 已弃用)"

# 3. 系统 Pod 状态
echo -e "\n--- kube-system Pod 状态 ---"
kubectl get pods -n kube-system -o wide

# 4. 异常 Pod 检查
echo -e "\n--- 异常 Pod 检查 ---"
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded 2>/dev/null || echo "所有 Pod 正常运行"

# 5. 资源使用
echo -e "\n--- 资源使用 TOP 10 ---"
echo "CPU 使用 TOP 10:"
kubectl top pods -A --sort-by=cpu 2>/dev/null | head -11 || echo "(metrics-server 未就绪)"
echo ""
echo "内存使用 TOP 10:"
kubectl top pods -A --sort-by=memory 2>/dev/null | head -11 || echo "(metrics-server 未就绪)"

# 6. 证书检查
echo -e "\n--- 证书有效期 ---"
for cert in /etc/kubernetes/pki/*.crt; do
  EXPIRY=$(openssl x509 -in ${cert} -noout -enddate 2>/dev/null | cut -d= -f2)
  echo "$(basename ${cert}): ${EXPIRY}"
done

# 7. etcd 状态
echo -e "\n--- etcd 状态 ---"
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://10.0.0.21:2379,https://10.0.0.22:2379,https://10.0.0.23:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/etcd-client.crt \
  --key=/etc/kubernetes/pki/etcd/etcd-client.key 2>/dev/null || echo "etcd 检查失败"

echo ""
echo "=== 巡检完成 ==="
```

### 11.2 故障排查手册

```bash
#!/bin/bash
# troubleshoot.sh - Kubernetes 故障排查

set -euo pipefail

echo "=== Kubernetes 故障排查 ==="

# 排查 Pod 启动失败
troubleshoot_pod() {
  local pod_name=${1:?需要 Pod 名称}
  local namespace=${2:-default}
  
  echo "--- Pod ${pod_name} 详情 ---"
  kubectl describe pod ${pod_name} -n ${namespace}
  
  echo ""
  echo "--- Pod 日志 ---"
  kubectl logs ${pod_name} -n ${namespace} --previous --tail=100 2>/dev/null || \
  kubectl logs ${pod_name} -n ${namespace} --tail=100
  
  echo ""
  echo "--- Pod 事件 ---"
  kubectl get events -n ${namespace} --field-selector involvedObject.name=${pod_name}
}

# 排查节点问题
troubleshoot_node() {
  local node_name=${1:?需要节点名称}
  
  echo "--- 节点 ${node_name} 详情 ---"
  kubectl describe node ${node_name}
  
  echo ""
  echo "--- 节点条件 ---"
  kubectl get node ${node_name} -o jsonpath='{range .status.conditions[*]}{.type}: {.status}{"\n"}{end}'
}

# 排查网络问题
troubleshoot_network() {
  echo "--- 网络排查 ---"
  
  echo "DNS 解析测试:"
  kubectl run dns-test --image=busybox:1.36 --rm -it --restart=Never -- nslookup kubernetes.default
  
  echo ""
  echo "Pod 网络测试:"
  kubectl run net-test --image=nicolaka/netshoot --rm -it --restart=Never -- ping -c 3 10.96.0.1
}

# 主菜单
echo "选择排查项:"
echo "1. Pod 启动失败"
echo "2. 节点问题"
echo "3. 网络问题"
echo "4. 全部检查"

read -p "选择: " choice
case $choice in
  1) read -p "Pod 名称: " pod; read -p "命名空间 (default): " ns; troubleshoot_pod ${pod} ${ns:-default} ;;
  2) read -p "节点名称: " node; troubleshoot_node ${node} ;;
  3) troubleshoot_network ;;
  4) echo "请指定具体问题" ;;
  *) echo "无效选择" ;;
esac
```

### 11.3 容量规划

```bash
#!/bin/bash
# capacity-planning.sh - 集群容量规划

set -euo pipefail

echo "=== Kubernetes 集群容量规划 ==="

# 节点资源总量
echo "--- 节点资源总量 ---"
kubectl get nodes -o json | jq -r '
  .items[] |
  "\(.metadata.name): CPU=\(.status.capacity.cpu) MEM=\(.status.capacity.memory)"
'

# 已分配资源
echo ""
echo "--- 已分配资源 ---"
kubectl get pods -A -o json | jq -r '
  .items[] |
  select(.spec.containers != null) |
  .spec.containers[] |
  "\(.resources.requests.cpu // "0") \(.resources.requests.memory // "0")"
' | awk '{
  cpu += $1; mem += $2;
  print "总 CPU 请求:", cpu, "核, 总内存请求:", mem, "Mi"
}'

# 资源使用率
echo ""
echo "--- 资源使用率 ---"
kubectl top nodes 2>/dev/null || echo "(metrics-server 未就绪)"

# 建议
echo ""
echo "--- 容量建议 ---"
TOTAL_NODES=$(kubectl get nodes --no-headers | wc -l)
TOTAL_CPU=$(kubectl get nodes -o json | jq -r '.items[].status.capacity.cpu' | awk '{sum+=$1} END{print sum}')
TOTAL_MEM=$(kubectl get nodes -o json | jq -r '.items[].status.capacity.memory' | sed 's/Ki//' | awk '{sum+=$1} END{print sum/1024/1024}')

echo "节点总数: ${TOTAL_NODES}"
echo "总 CPU: ${TOTAL_CPU} 核"
echo "总内存: ${TOTAL_MEM} GiB"
echo ""
echo "建议: 保持资源使用率在 70% 以下"
```

---

## 十二、面试题与思考

### Q1: Kubernetes 高可用架构中，为什么需要至少 3 个 Master 节点？

**答：** 需要 3 个 Master 是为了保证控制平面的高可用和容错能力：

1. **etcd 集群**：etcd 使用 Raft 共识算法，需要奇数个节点（3、5、7）来实现容错。3 个 etcd 节点可以容忍 1 个节点故障，集群仍能正常工作。

2. **API Server**：多个 API Server 通过负载均衡器（如 Keepalived + Nginx）对外提供服务，任何一个 Master 故障，流量自动切换到其他 Master。

3. **Controller Manager 和 Scheduler**：通过 Leader Election 机制选举一个 Leader，其他为 Standby。Leader 故障时自动选举新的 Leader。

4. **避免脑裂**：奇数节点可以避免脑裂问题（Split Brain）。3 个节点可以容忍 1 个故障，而 2 个节点无法容忍任何故障。

### Q2: Calico 和 Flannel 的区别？生产环境如何选择？

**答：**

| 特性 | Calico | Flannel |
|------|--------|---------|
| 网络模式 | BGP/VXLAN | VXLAN/Host-GW |
| 网络策略 | 完整支持 | 不支持 |
| 性能 | BGP 模式性能最优 | VXLAN 模式有封装开销 |
| 复杂度 | 较高 | 简单 |
| 适用场景 | 大规模集群、需要网络策略 | 小规模集群、快速部署 |

**生产环境选择**：
- 需要 NetworkPolicy → 选 Calico
- 大规模集群（>500 节点）→ 选 Calico（BGP 模式）
- 小规模集群、快速部署 → 选 Flannel
- 混合云环境 → 选 Calico（支持 BGP 跨子网）

### Q3: 如何实现 Kubernetes 集群的零停机升级？

**答：**

1. **控制平面升级**：逐个升级 Master 节点，使用 `kubeadm upgrade apply/upgrade node`。由于 Master 有多副本，升级期间其他 Master 仍能提供服务。

2. **Worker 升级**：
   - 使用 `kubectl cordon` 标记节点不可调度
   - 使用 `kubectl drain` 驱逐 Pod（设置 `--ignore-daemonsets --delete-emptydir-data`）
   - 升级 kubelet 和 kubeadm
   - 使用 `kubectl uncordon` 恢复调度

3. **Pod 保障**：
   - 确保所有 Deployment 有 PDB（PodDisruptionBudget）
   - 确保有足够副本数满足 PDB
   - 使用 `maxUnavailable: 0` 的 RollingUpdate 策略

4. **升级顺序**：etcd → Master → Worker（逐个）

### Q4: Kubernetes 的资源请求(requests)和限制(limits)有什么区别？

**答：**

- **requests**：调度依据。Kubernetes 根据 requests 值来决定 Pod 调度到哪个节点。如果节点的可用资源不满足 requests，Pod 不会被调度到该节点。

- **limits**：运行时限制。Pod 实际使用的资源不能超过 limits。如果超过 CPU limits，Pod 会被限流（throttle）；如果超过 Memory limits，Pod 会被 OOM Kill。

**最佳实践**：
- CPU：requests 设为平均值，limits 设为峰值
- Memory：requests 和 limits 设为相同值（避免 OOM Kill）
- 使用 VPA（Vertical Pod Autoscaler）自动调整 requests

### Q5: 如何排查 Kubernetes 网络不通的问题？

**答：** 按照 OSI 模型逐层排查：

1. **物理层/数据链路层**：
   - 检查节点网络接口状态
   - 检查 CNI 插件是否正常运行
   - 检查 Pod 网卡是否创建成功

2. **网络层**：
   - 检查 Pod IP 是否分配成功
   - 检查路由表是否正确
   - 使用 `ping` 测试 Pod 间连通性

3. **传输层**：
   - 检查端口是否监听
   - 使用 `telnet` 测试端口连通性
   - 检查防火墙规则

4. **应用层**：
   - 检查 Service 是否正确创建
   - 检查 Endpoints 是否有后端 Pod
   - 检查 DNS 解析是否正常

**排查工具**：
```bash
# Pod 网络调试
kubectl run debug --image=nicolaka/netshoot --rm -it -- bash

# 检查 DNS
nslookup kubernetes.default

# 检查连通性
curl -v http://service-name.namespace.svc.cluster.local:80
```

---

## 附录

### A. 常用命令速查

```bash
# 集群信息
kubectl cluster-info
kubectl get nodes -o wide
kubectl get pods -A

# 资源使用
kubectl top nodes
kubectl top pods -A --sort-by=cpu

# 日志查看
kubectl logs -f <pod> -n <namespace>
kubectl logs <pod> -n <namespace> --previous

# 进入 Pod
kubectl exec -it <pod> -n <namespace> -- /bin/bash

# 端口转发
kubectl port-forward <pod> 8080:80 -n <namespace>

# 资源导出
kubectl get deployment <name> -o yaml > deployment.yaml

# 强制删除 Pod
kubectl delete pod <pod> -n <namespace> --force --grace-period=0

# 节点维护
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <node>
```

### B. 端口规划

| 服务 | 端口 | 协议 | 用途 |
|------|------|------|------|
| kube-apiserver | 6443 | TCP | API 服务 |
| etcd | 2379 | TCP | etcd 客户端 |
| etcd | 2380 | TCP | etcd 集群通信 |
| kubelet | 10250 | TCP | kubelet API |
| kube-scheduler | 10259 | TCP | 调度器 |
| kube-controller-manager | 10257 | TCP | 控制器管理器 |
| Calico BGP | 179 | TCP | BGP 通信 |
| NodePort | 30000-32767 | TCP | NodePort 服务 |

### C. 参考资料

- [Kubernetes 官方文档](https://kubernetes.io/docs/)
- [kubeadm 官方文档](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/)
- [Calico 官方文档](https://docs.tigera.io/calico/latest/about/)
- [etcd 官方文档](https://etcd.io/docs/)
- [阿里云 ACK 部署最佳实践](https://help.aliyun.com/zh/ack/)

---

*作者: 小白老师 | 更新时间: 2026-05-03 | 版本: v1.0*
