# 企业级安全合规与 DevSecOps

> 基于 Trivy + Falco + OPA + Vault + Cosign 的安全体系
> 参考方案：Google BeyondCorp、字节跳动安全运维体系

---

## 一、安全体系架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    企业级 DevSecOps 安全体系                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────── 代码安全 ──────────────────────────────────┐     │
│  │  SAST(静态扫描) | SCA(依赖审计) | Secrets检测          │     │
│  │  工具: SonarQube | Snyk | GitLeaks | TruffleHog        │     │
│  └──────────────────────────────────────────────────────────┘     │
│         │                                                       │
│  ┌─────────── 镜像安全 ──────────────────────────────────┐     │
│  │  漏洞扫描 | 签名验证 | SBOM生成 | 合规检查             │     │
│  │  工具: Trivy | Cosign | Syft | Grype | Harbor          │     │
│  └──────────────────────────────────────────────────────────┘     │
│         │                                                       │
│  ┌─────────── 运行时安全 ────────────────────────────────┐     │
│  │  系统调用监控 | 异常检测 | 策略执行 | 网络策略          │     │
│  │  工具: Falco | OPA/Gatekeeper | K8s NetworkPolicy      │     │
│  └──────────────────────────────────────────────────────────┘     │
│         │                                                       │
│  ┌─────────── 密钥管理 ──────────────────────────────────┐     │
│  │  密钥存储 | 动态密钥 | 加密解密 | 审计日志             │     │
│  │  工具: HashiCorp Vault | K8s Secrets | Sealed Secrets  │     │
│  └──────────────────────────────────────────────────────────┘     │
│         │                                                       │
│  ┌─────────── 合规审计 ──────────────────────────────────┐     │
│  │  基线检查 | 配置审计 | 策略执行 | 报告生成             │     │
│  │  工具: CIS Benchmark | kube-bench | kube-hunter        │     │
│  └──────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、Trivy 镜像安全扫描

### 2.1 Trivy 部署

```yaml
# trivy-server.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: trivy-server
  namespace: security
spec:
  replicas: 1
  selector:
    matchLabels:
      app: trivy-server
  template:
    metadata:
      labels:
        app: trivy-server
    spec:
      containers:
        - name: trivy-server
          image: aquasec/trivy:0.49.1
          args:
            - server
            - --listen=0.0.0.0:8080
            - --cache-dir=/data
          ports:
            - containerPort: 8080
              name: http
            - containerPort: 8443
              name: grpc
          volumeMounts:
            - name: data
              mountPath: /data
          resources:
            requests:
              cpu: 200m
              memory: 512Mi
            limits:
              cpu: "1"
              memory: 2Gi
      volumes:
        - name: data
          emptyDir: {}

---
apiVersion: v1
kind: Service
metadata:
  name: trivy-server
  namespace: security
spec:
  selector:
    app: trivy-server
  ports:
    - port: 8080
      name: http
    - port: 8443
      name: grpc
```

### 2.2 CI/CD 集成扫描

```yaml
# trivy-scan-job.yaml - 镜像扫描 Job
apiVersion: batch/v1
kind: CronJob
metadata:
  name: trivy-image-scan
  namespace: security
spec:
  schedule: "0 */6 * * *"  # 每 6 小时扫描一次
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: trivy-scan
              image: aquasec/trivy:0.49.1
              command:
                - /bin/sh
                - -c
                - |
                  # 扫描所有生产镜像
                  IMAGES=$(kubectl get pods -n production -o jsonpath='{.items[*].spec.containers[*].image}' | tr ' ' '\n' | sort -u)
                  
                  for image in ${IMRES}; do
                    echo "扫描镜像: ${image}"
                    
                    # 执行扫描
                    trivy image \
                      --exit-code 1 \
                      --severity HIGH,CRITICAL \
                      --ignore-unfixed \
                      --format json \
                      -o /tmp/report.json \
                      ${image}
                    
                    # 如果发现漏洞，发送告警
                    if [ $? -ne 0 ]; then
                      echo "发现漏洞: ${image}"
                      # 这里可以添加告警通知
                    fi
                  done
              volumeMounts:
                - name: data
                  mountPath: /tmp
          restartPolicy: OnFailure
          volumes:
            - name: data
              emptyDir: {}
```

---

## 三、Falco 运行时安全

### 3.1 Falco 部署

```yaml
# falco-values.yaml
driver:
  enabled: true
  kind: ebpf

falco:
  rules:
    - rule: Terminal shell in container
      desc: A shell was used as the entrypoint/exec for a container
      condition: >
        spawned_process and container and shell_procs and proc.tty != 0
      output: >
        Shell spawned in container
        (user=%user.name user_uid=%user.uid user_login=%user.login.name
        command=%proc.cmdline shell=%proc.name parent=%proc.pname
        terminal=%proc.tty container_id=%container.id
        container_name=%container.name k8s.ns=%k8s.ns.name
        k8s.pod=%k8s.pod.name k8s.cns=%k8s.ns.name)
      priority: WARNING
      tags: [container, shell, mit_execution]
    
    - rule: Container drift detected
      desc: New executable created in container after startup
      condition: >
        spawned_process and container and not proc.name in (falco_procs) and
        proc.pname != container.img.repository
      output: >
        Drift detected in container
        (user=%user.name command=%proc.cmdline
        container_id=%container.id container_name=%container.name
        k8s.ns=%k8s.ns.name k8s.pod=%k8s.pod.name)
      priority: WARNING
      tags: [container, drift, mit_execution]
    
    - rule: Sensitive file opened
      desc: A sensitive file was opened for reading
      condition: >
        open_read and container and
        (fd.name startswith /etc/shadow or
         fd.name startswith /etc/passwd or
         fd.name startswith /etc/sudoers or
         fd.name startswith /root/.ssh or
         fd.name startswith /home/.ssh)
      output: >
        Sensitive file opened
        (file=%fd.name user=%user.name command=%proc.cmdline
        container_id=%container.id container_name=%container.name)
      priority: WARNING
      tags: [container, filesystem, mit_credential_access]
    
    - rule: K8s Pod Created in Production
      desc: A pod was created in production namespace
      condition: >
        ka.verb=create and ka.target.resource=pods and
        ka.target.namespace=production
      output: >
        Pod created in production
        (user=%ka.user.name pod=%ka.target.name
        ns=%ka.target.namespace)
      priority: WARNING
      tags: [k8s, pod, mit_persistence]

# 告警输出
falcosidekick:
  enabled: true
  config:
    webhook:
      address: "http://alertmanager-webhook:8080/webhook"
    dingtalk:
      webhook: "https://oapi.dingtalk.com/robot/send?access_token=xxx"
```

---

## 四、OPA/Gatekeeper 策略

### 4.1 Gatekeeper 部署

```yaml
# gatekeeper-values.yaml
replicas: 3

resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

audit:
  enabled: true
  interval: 60s
  replicas: 2

webhook:
  enabled: true
  replicas: 3
```

### 4.2 策略模板

```yaml
# require-labels.yaml - 必须有标签
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels
        
        violation[{"msg": msg}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_]}
          missing := required - provided
          count(missing) > 0
          msg := sprintf("必须包含标签: %v", [missing])
        }

---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-app-labels
spec:
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment", "StatefulSet", "DaemonSet"]
    namespaces: ["production", "staging"]
  parameters:
    labels:
      - "app.kubernetes.io/name"
      - "app.kubernetes.io/version"
      - "app.kubernetes.io/environment"
      - "app.kubernetes.io/managed-by"

---
# image-policy.yaml - 镜像策略
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sallowedimages
spec:
  crd:
    spec:
      names:
        kind: K8sAllowedImages
      validation:
        openAPIV3Schema:
          type: object
          properties:
            registries:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sallowedimages
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not startswith(container.image, input.parameters.registries[_])
          msg := sprintf("镜像必须来自允许的仓库: %v", [container.image])
        }

---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedImages
metadata:
  name: require-trusted-registry
spec:
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment", "StatefulSet", "DaemonSet"]
    namespaces: ["production", "staging"]
  parameters:
    registries:
      - "harbor.example.com/"
      - "registry.aliyuncs.com/"
      - "quay.io/"

---
# resource-limits.yaml - 资源限制
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sresourcelimits
spec:
  crd:
    spec:
      names:
        kind: K8sResourceLimits
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sresourcelimits
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.cpu
          msg := sprintf("容器 %v 必须设置 CPU limit", [container.name])
        }
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.memory
          msg := sprintf("容器 %v 必须设置内存 limit", [container.name])
        }
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.requests.cpu
          msg := sprintf("容器 %v 必须设置 CPU request", [container.name])
        }
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.requests.memory
          msg := sprintf("容器 %v 必须设置内存 request", [container.name])
        }
```

---

## 五、HashiCorp Vault 密钥管理

### 5.1 Vault 部署

```yaml
# vault-values.yaml
global:
  enabled: true
  tlsDisable: false

server:
  replicas: 3
  
  auditStorage:
    enabled: true
    size: 10Gi
  
  dataStorage:
    enabled: true
    size: 50Gi
    storageClass: ssd
  
  ha:
    enabled: true
    replicas: 3
    raft:
      enabled: true
      config: |
        ui = true
        listener "tcp" {
          tls_disable = 0
          address = "[::]:8200"
          cluster_address = "[::]:8201"
          tls_cert_file = "/vault/tls/tls.crt"
          tls_key_file = "/vault/tls/tls.key"
        }
        storage "raft" {
          path = "/vault/data"
          retry_join {
            leader_api_addr = "https://vault-0.vault-internal:8200"
            leader_ca_cert_file = "/vault/tls/ca.crt"
          }
          retry_join {
            leader_api_addr = "https://vault-1.vault-internal:8200"
            leader_ca_cert_file = "/vault/tls/ca.crt"
          }
          retry_join {
            leader_api_addr = "https://vault-2.vault-internal:8200"
            leader_ca_cert_file = "/vault/tls/ca.crt"
          }
        }
        service_registration "kubernetes" {}

# 启用 KV 引擎
# vault secrets enable -path=secret kv-v2
# vault kv put secret/myapp/db username=admin password=SuperSecret

# 动态数据库密钥
# vault secrets enable database
# vault write database/config/mydb \
#   plugin_name=mysql-database-plugin \
#   connection_url="{{username}}:{{password}}@tcp(mysql:3306)/" \
#   allowed_roles="mydb-role" \
#   username="vault" \
#   password="vault-password"
```

---

## 六、安全合规检查

### 6.1 CIS Benchmark 检查

```yaml
# cis-benchmark.yaml - K8s 安全基线检查
apiVersion: batch/v1
kind: Job
metadata:
  name: kube-bench
  namespace: security
spec:
  template:
    spec:
      containers:
        - name: kube-bench
          image: aquasec/kube-bench:v0.8.1
          command: ["kube-bench", "run", "--targets", "master,node,policies"]
          volumeMounts:
            - name: kubeconfig
              mountPath: /etc/kubernetes/admin.conf
              readOnly: true
            - name: kubelet-config
              mountPath: /var/lib/kubelet/config.yaml
              readOnly: true
          securityContext:
            privileged: true
      volumes:
        - name: kubeconfig
          hostPath:
            path: /etc/kubernetes/admin.conf
        - name: kubelet-config
          hostPath:
            path: /var/lib/kubelet/config.yaml
      hostPID: true
      nodeSelector:
        node-role.kubernetes.io/master: ""
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
      restartPolicy: Never
```

### 6.2 安全检查脚本

```bash
#!/bin/bash
# security-audit.sh - 安全审计脚本

set -euo pipefail

echo "=== Kubernetes 安全审计 ==="
echo "时间: $(date '+%Y-%m-%d %H:%M:%S')"

# 1. RBAC 检查
echo -e "\n--- RBAC 检查 ---"
echo "ClusterRoleBinding 数量:"
kubectl get clusterrolebindings --no-headers | wc -l

echo "具有 cluster-admin 权限的绑定:"
kubectl get clusterrolebindings -o json | jq -r '
  .items[] |
  select(.roleRef.name == "cluster-admin") |
  "\(.metadata.name) -> \(.subjects[0].name // "unknown")"
'

# 2. 网络策略检查
echo -e "\n--- 网络策略检查 ---"
for ns in $(kubectl get ns --no-headers | awk '{print $1}'); do
  np_count=$(kubectl get networkpolicies -n ${ns} --no-headers 2>/dev/null | wc -l)
  if [ "${np_count}" -eq 0 ]; then
    echo "⚠️  命名空间 ${ns} 没有网络策略"
  fi
done

# 3. Pod 安全检查
echo -e "\n--- Pod 安全检查 ---"
echo "特权容器:"
kubectl get pods -A -o json | jq -r '
  .items[] |
  . as $pod |
  .spec.containers[]? |
  select(.securityContext.privileged == true) |
  "\($pod.metadata.namespace)/\($pod.metadata.name): \(.name)"
'

# 4. 敏感信息检查
echo -e "\n--- 敏感信息检查 ---"
echo "未加密的 Secret:"
kubectl get secrets -A -o json | jq -r '
  .items[] |
  select(.type == "Opaque") |
  "\(.metadata.namespace)/\(.metadata.name)"
' | head -20

# 5. 镜像安全检查
echo -e "\n--- 镜像安全检查 ---"
echo "使用 latest 标签的 Pod:"
kubectl get pods -A -o json | jq -r '
  .items[] |
  . as $pod |
  .spec.containers[].image |
  select(endswith(":latest") or (contains(":") | not)) |
  "\($pod.metadata.namespace)/\($pod.metadata.name): \(.)" 2>/dev/null
' | sort -u

# 6. 资源限制检查
echo -e "\n--- 资源限制检查 ---"
echo "未设置资源限制的 Pod:"
kubectl get pods -A -o json | jq -r '
  .items[] |
  . as $pod |
  .spec.containers[] |
  select(.resources.limits == null) |
  "\($pod.metadata.namespace)/\($pod.metadata.name): \(.name)"
' | head -20

echo ""
echo "=== 审计完成 ==="
```

---

## 七、面试题与思考

### Q1: DevSecOps 的核心理念是什么？

**答：** DevSecOps 是将安全融入 DevOps 全流程：

1. **左移安全**：安全检查前移到开发阶段，而非部署后
2. **自动化安全**：将安全检查集成到 CI/CD 流水线
3. **持续监控**：运行时持续监控和威胁检测
4. **合规即代码**：将合规要求转化为可执行的策略代码
5. **安全即服务**：提供自助式安全服务，降低开发门槛

### Q2: 如何实现 Kubernetes 的零信任安全？

**答：** 零信任安全模型：

1. **身份验证**：每个请求都需要验证身份（mTLS、JWT）
2. **最小权限**：RBAC + NetworkPolicy 实现最小权限
3. **微分段**：使用 NetworkPolicy 隔离不同服务
4. **持续验证**：运行时持续验证安全策略
5. **加密通信**：所有通信都加密（TLS/mTLS）

### Q3: Vault 如何实现动态密钥？

**答：** 动态密钥是 Vault 的核心能力：

1. **数据库密钥**：Vault 可以动态创建数据库用户，每次请求生成新的临时凭据
2. **AWS IAM**：动态生成 AWS IAM 凭据
3. **PKI**：动态生成 TLS 证书
4. **租约管理**：每个密钥有租约，过期自动撤销
5. **轮转**：支持密钥自动轮转

---

*作者: 小白老师 | 更新时间: 2026-05-03 | 版本: v1.0*
