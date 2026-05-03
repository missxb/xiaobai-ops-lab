# 企业级安全合规与 DevSecOps

> 基于 Trivy + Falco + OPA + Vault + Cosign 的安全体系
> 参考方案：Google BeyondCorp、字节跳动安全运维体系、阿里云安全中心

---

## 一、安全体系架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    企业级 DevSecOps 安全体系                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────── 代码安全 (Shift Left) ───────────────────┐     │
│  │  SAST(静态扫描) | SCA(依赖审计) | Secrets检测          │     │
│  │  工具: SonarQube | Snyk | GitLeaks | TruffleHog        │     │
│  │  阶段: 开发 → 提交 → PR                                 │     │
│  └──────────────────────────────────────────────────────┘     │
│         │                                                       │
│  ┌─────────── 镜像安全 (Build Time) ───────────────────┐     │
│  │  漏洞扫描 | 签名验证 | SBOM生成 | 合规检查             │     │
│  │  工具: Trivy | Cosign | Syft | Grype | Harbor          │     │
│  │  阶段: 构建 → 推送 → 部署                               │     │
│  └──────────────────────────────────────────────────────┘     │
│         │                                                       │
│  ┌─────────── 运行时安全 (Runtime) ────────────────────┐     │
│  │  系统调用监控 | 异常检测 | 策略执行 | 网络策略          │     │
│  │  工具: Falco | OPA/Gatekeeper | K8s NetworkPolicy      │     │
│  │  阶段: 运行时持续监控                                   │     │
│  └──────────────────────────────────────────────────────┘     │
│         │                                                       │
│  ┌─────────── 密钥管理 (Secrets) ─────────────────────┐     │
│  │  密钥存储 | 动态密钥 | 加密解密 | 审计日志             │     │
│  │  工具: HashiCorp Vault | K8s Secrets | Sealed Secrets  │     │
│  │  阶段: 全生命周期                                       │     │
│  └──────────────────────────────────────────────────────┘     │
│         │                                                       │
│  ┌─────────── 合规审计 (Compliance) ───────────────────┐     │
│  │  基线检查 | 配置审计 | 策略执行 | 报告生成             │     │
│  │  工具: CIS Benchmark | kube-bench | kube-hunter        │     │
│  │  阶段: 持续合规                                         │     │
│  └──────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、代码安全 - GitLeaks + SonarQube

### 2.1 GitLeaks 配置

```yaml
# .gitleaks.toml - Secrets 检测配置
[allowlist]
  description = "允许列表"
  paths = [
    '''vendor/''',
    '''node_modules/''',
    '''\.gitleaks\.toml$''',
  ]
  regexes = [
    '''example\.com''',
    '''test_password = "test.*"''',
  ]

[[rules]]
  id = "generic-api-key"
  description = "检测通用 API Key"
  regex = '''(?i)(api[_-]?key|apikey|api_secret)[\"'\s:=]{1,10}[\"']?([a-zA-Z0-9\-_]{20,})'''
  tags = ["key", "API"]

[[rules]]
  id = "private-key"
  description = "检测私钥"
  regex = '''-----BEGIN (RSA |EC |DSA )?PRIVATE KEY-----'''
  tags = ["key", "private"]

[[rules]]
  id = "password"
  description = "检测密码"
  regex = '''(?i)(password|passwd|pwd)[\"'\s:=]{1,10}[\"']?([^\s\"']{8,})'''
  tags = ["password"]

[[rules]]
  id = "aws-access-key"
  description = "检测 AWS Access Key"
  regex = '''(?i)(AKIA|ABIA|ACCA|ASIA)[A-Z0-9]{16}'''
  tags = ["aws", "key"]

[[rules]]
  id = "github-token"
  description = "检测 GitHub Token"
  regex = '''ghp_[a-zA-Z0-9]{36}'''
  tags = ["github", "token"]

[[rules]]
  id = "docker-password"
  description = "检测 Docker 密码"
  regex = '''(?i)(docker[_-]?password|docker[_-]?pass)[\"'\s:=]{1,10}[\"']?([^\s\"']{8,})'''
  tags = ["docker", "password"]
```

### 2.2 GitLeaks CI/CD 集成

```yaml
# .gitlab-ci.yml - GitLeaks 扫描
gitleaks-scan:
  stage: security
  image: zricethezav/gitleaks:8.18.1
  script:
    - gitleaks detect --source . --report-format json --report-path gitleaks-report.json
    - gitleaks detect --source . --verbose
  artifacts:
    reports:
      sast: gitleaks-report.json
    paths:
      - gitleaks-report.json
    expire_in: 30 days
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'
```

### 2.3 SonarQube 配置

```yaml
# sonar-project.properties - SonarQube 项目配置
sonar.projectKey=myapp
sonar.projectName=My Application
sonar.projectVersion=1.0

# 源码目录
sonar.sources=src
sonar.tests=tests

# 编码
sonar.sourceEncoding=UTF-8

# 语言
sonar.language=java

# 排除文件
sonar.exclusions=**/vendor/**,**/node_modules/**,**/test/**,**/*Test.java

# 覆盖率报告
sonar.jacoco.reportPaths=target/jacoco.exec
sonar.junit.reportPaths=target/surefire-reports

# 质量门禁
sonar.qualitygate.wait=true
sonar.qualitygate.timeout=300

# 忽略问题
sonar.issue.ignore.multicriteria=e1
sonar.issue.ignore.multicriteria.e1.ruleKey=java:S1135
sonar.issue.ignore.multicriteria.e1.resourceKey=**/generated/**
```

---

## 三、镜像安全 - Trivy 全流程

### 3.1 Trivy 部署

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
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 30
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

```yaml
# trivy-pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: trivy-pdb
  namespace: security
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: trivy-server
```

### 3.2 Trivy 完整扫描脚本

```bash
#!/bin/bash
# trivy-full-scan.sh - Trivy 完整安全扫描

set -euo pipefail

TRIVY_SERVER="trivy-server.security:8080"
HARBOR_URL="https://harbor.example.com"
HARBOR_USER="admin"
HARBOR_PASS="${HARBOR_PASSWORD}"  # 从环境变量读取
REPORT_DIR="/data/reports/trivy"
DATE=$(date +%Y%m%d)

mkdir -p ${REPORT_DIR}

log() { echo "[$(date '+%H:%M:%S')] $1"; }

# 扫描镜像
scan_image() {
  local image=$1
  local output="${REPORT_DIR}/$(echo ${image} | tr '/:' '_')-${DATE}.json"
  
  log "扫描镜像: ${image}"
  
  # 执行扫描
  trivy image \
    --server ${TRIVY_SERVER} \
    --exit-code 1 \
    --severity HIGH,CRITICAL \
    --ignore-unfixed \
    --format json \
    -o ${output} \
    ${image}
  
  # 检查结果
  if [ $? -ne 0 ]; then
    log "⚠️ 发现漏洞: ${image}"
    # 生成摘要
    jq '.Results[] | select(.Vulnerabilities != null) | .Vulnerabilities[] | {Severity, VulnerabilityID, PkgName, InstalledVersion, FixedVersion}' ${output} | \
      jq -s 'group_by(.Severity) | map({severity: .[0].Severity, count: length})' 
    return 1
  else
    log "✅ 无漏洞: ${image}"
    return 0
  fi
}

# 扫描文件系统
scan_fs() {
  local path=$1
  local output="${REPORT_DIR}/fs-scan-${DATE}.json"
  
  log "扫描文件系统: ${path}"
  
  trivy fs \
    --server ${TRIVY_SERVER} \
    --exit-code 1 \
    --severity HIGH,CRITICAL \
    --scanners vuln,misconfig,secret \
    --format json \
    -o ${output} \
    ${path}
}

# 扫描 Helm Chart
scan_helm() {
  local chart_path=$1
  local output="${REPORT_DIR}/helm-scan-${DATE}.json"
  
  log "扫描 Helm Chart: ${chart_path}"
  
  trivy config \
    --server ${TRIVY_SERVER} \
    --exit-code 1 \
    --severity HIGH,CRITICAL \
    --format json \
    -o ${output} \
    ${chart_path}
}

# 生成扫描报告
generate_report() {
  local report_file="${REPORT_DIR}/scan-report-${DATE}.html"
  
  log "生成扫描报告..."
  
  cat > ${report_file} <<'EOF'
<!DOCTYPE html>
<html>
<head>
  <title>安全扫描报告</title>
  <style>
    body { font-family: Arial; margin: 20px; }
    .summary { display: flex; gap: 20px; margin: 20px 0; }
    .card { padding: 20px; border-radius: 5px; text-align: center; }
    .critical { background: #f8d7da; color: #721c24; }
    .high { background: #fff3cd; color: #856404; }
    .medium { background: #d1ecf1; color: #0c5460; }
    table { width: 100%; border-collapse: collapse; }
    th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
  </style>
</head>
<body>
  <h1>安全扫描报告</h1>
  <p>生成时间: DATE_PLACEHOLDER</p>
EOF
  
  # 统计结果
  if [ -d "${REPORT_DIR}" ]; then
    total_files=$(ls ${REPORT_DIR}/*.json 2>/dev/null | wc -l)
    echo "<p>扫描文件数: ${total_files}</p>" >> ${report_file}
  fi
  
  cat >> ${report_file} <<'EOF'
</body>
</html>
EOF
  
  sed -i "s/DATE_PLACEHOLDER/$(date '+%Y-%m-%d %H:%M:%S')/" ${report_file}
  
  log "报告已生成: ${report_file}"
}

# 主函数
main() {
  local action=${1:-"all"}
  
  case ${action} in
    image)
      scan_image ${2:?需要镜像名称}
      ;;
    fs)
      scan_fs ${2:-.}
      ;;
    helm)
      scan_helm ${2:-.}
      ;;
    report)
      generate_report
      ;;
    all)
      # 扫描所有生产镜像
      log "=== 开始全量安全扫描 ==="
      kubectl get pods -n production -o jsonpath='{.items[*].spec.containers[*].image}' | \
        tr ' ' '\n' | sort -u | while read image; do
        scan_image ${image} || true
      done
      generate_report
      log "=== 扫描完成 ==="
      ;;
  esac
}

main "$@"
```

---

## 四、运行时安全 - Falco

### 4.1 Falco 完整规则库

```yaml
# falco-rules.yaml - 完整 Falco 规则库
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
    k8s.pod=%k8s.pod.name)
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

- rule: Unexpected outbound connection
  desc: Unexpected outbound network connection
  condition: >
    outbound and container and
    not proc.name in (allowed_outbound_procs) and
    not container.image.repository in (allowed_outbound_images)
  output: >
    Unexpected outbound connection
    (command=%proc.cmdline connection=%fd.name
    container_id=%container.id container_name=%container.name)
  priority: NOTICE
  tags: [container, network, mit_command_and_control]

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

- rule: K8s ConfigMap Modified
  desc: A ConfigMap was modified
  condition: >
    ka.verb in (update, patch) and ka.target.resource=configmaps
  output: >
    ConfigMap modified
    (user=%ka.user.name configmap=%ka.target.name
    ns=%ka.target.namespace)
  priority: NOTICE
  tags: [k8s, configmap, mit_persistence]

- rule: Container with root privileges
  desc: Container running with root privileges
  condition: >
    container and container.privileged=true
  output: >
    Container with root privileges
    (container_id=%container.id container_name=%container.name
    k8s.ns=%k8s.ns.name k8s.pod=%k8s.pod.name)
  priority: WARNING
  tags: [container, privilege, mit_privilege_escalation]

- rule: Container with host network
  desc: Container using host network
  condition: >
    container and container.net.host=true
  output: >
    Container with host network
    (container_id=%container.id container_name=%container.name
    k8s.ns=%k8s.ns.name k8s.pod=%k8s.pod.name)
  priority: WARNING
  tags: [container, network, mit_privilege_escalation]

- rule: K8s Service Account Token Accessed
  desc: Service account token accessed
  condition: >
    open_read and container and
    fd.name startswith /var/run/secrets/kubernetes.io/serviceaccount/token
  output: >
    Service account token accessed
    (file=%fd.name user=%user.name command=%proc.cmdline
    container_id=%container.id container_name=%container.name)
  priority: WARNING
  tags: [container, k8s, mit_credential_access]
```

### 4.2 Falco 部署

```yaml
# falco-values.yaml
driver:
  enabled: true
  kind: ebpf
  ebpf:
    least_privileged: false

falco:
  rules:
    - /etc/falco/falco_rules.yaml
    - /etc/falco/custom_rules.yaml
  
  # 性能优化
  buffered_outputs: true
  output_rate: 1
  output_keep_alive: 1
  
  # JSON 输出
  json_output: true
  json_include_output_property: true
  json_include_tags_property: true

# 告警输出
falcosidekick:
  enabled: true
  config:
    webhook:
      address: "http://auto-remediation:8080/webhook"
    dingtalk:
      webhook: "https://oapi.dingtalk.com/robot/send?access_token=xxx"
    slack:
      webhookurl: "https://hooks.slack.com/services/xxx"
    pagerduty:
      routingkey: "xxx"

# 资源配置
resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

---

## 五、策略管理 - OPA/Gatekeeper

### 5.1 Gatekeeper 部署

```yaml
# gatekeeper-values.yaml
replicas: 3

resources:
  requests:
    cpu: 200m
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

### 5.2 安全策略模板

```yaml
# require-security-context.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8ssecuritycontext
spec:
  crd:
    spec:
      names:
        kind: K8sSecurityContext
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8ssecuritycontext
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.securityContext.runAsNonRoot
          msg := sprintf("容器 %v 必须设置 runAsNonRoot: true", [container.name])
        }
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          container.securityContext.privileged == true
          msg := sprintf("容器 %v 不能以特权模式运行", [container.name])
        }
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.securityContext.readOnlyRootFilesystem
          msg := sprintf("容器 %v 必须使用只读根文件系统", [container.name])
        }
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.securityContext.allowPrivilegeEscalation == false
          msg := sprintf("容器 %v 必须禁用特权提升", [container.name])
        }

---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sSecurityContext
metadata:
  name: require-security-context
spec:
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment", "StatefulSet", "DaemonSet"]
    namespaces: ["production", "staging"]

---
# require-resource-limits.yaml
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
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          container.resources.limits.memory
          container.resources.requests.memory
          to_number(container.resources.limits.memory) < to_number(container.resources.requests.memory) * 1.5
          msg := sprintf("容器 %v 的内存 limit 应至少为 request 的 1.5 倍", [container.name])
        }
```

---

## 六、密钥管理 - Vault

### 6.1 Vault 完整部署

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
```

### 6.2 Vault 初始化脚本

```bash
#!/bin/bash
# vault-init.sh - Vault 初始化和配置

set -euo pipefail

VAULT_ADDR="https://vault.example.com:8200"
export VAULT_ADDR

log() { echo "[$(date '+%H:%M:%S')] $1"; }

# 初始化 Vault
init_vault() {
  log "=== 初始化 Vault ==="
  
  # 检查是否已初始化
  vault status 2>/dev/null
  if [ $? -eq 2 ]; then
    log "Vault 未初始化，开始初始化..."
    
    # 初始化（5 个密钥，3 个解封阈值）
    vault operator init \
      -key-shares=5 \
      -key-threshold=3 \
      -format=json > /opt/vault/init-keys.json
    
    log "初始化完成，密钥已保存到 /opt/vault/init-keys.json"
    log "请妥善保管密钥！"
  else
    log "Vault 已初始化"
  fi
}

# 解封 Vault
unseal_vault() {
  log "=== 解封 Vault ==="
  
  # 从文件读取密钥
  keys=$(jq -r '.unseal_keys_b64[]' /opt/vault/init-keys.json | head -3)
  
  for key in ${keys}; do
    vault operator unseal ${key}
  done
  
  log "Vault 已解封"
}

# 配置认证
configure_auth() {
  log "=== 配置认证 ==="
  
  # 启用 Kubernetes 认证
  vault auth enable kubernetes
  
  # 配置 Kubernetes 认证
  vault write auth/kubernetes/config \
    kubernetes_host="https://kubernetes.default.svc" \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
    token_reviewer_jwt=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
  
  # 创建策略
  vault policy write myapp-policy - <<EOF
    path "secret/data/myapp/*" {
      capabilities = ["read"]
    }
    path "secret/data/myapp/config" {
      capabilities = ["read", "list"]
    }
    path "database/creds/myapp-role" {
      capabilities = ["read"]
    }
EOF
  
  # 创建角色
  vault write auth/kubernetes/role/myapp \
    bound_service_account_names=myapp \
    bound_service_account_namespaces=production \
    policies=myapp-policy \
    ttl=1h
}

# 配置密钥引擎
configure_secrets_engines() {
  log "=== 配置密钥引擎 ==="
  
  # 启用 KV v2
  vault secrets enable -path=secret kv-v2
  
  # 启用数据库密钥引擎
  vault secrets enable database
  
  # 配置 MySQL 连接
  vault write database/config/mydb \
    plugin_name=mysql-database-plugin \
    connection_url="{{username}}:{{password}}@tcp(mysql-primary:3306)/" \
    allowed_roles="mydb-role" \
    username="vault" \
    password="${VAULT_DB_PASSWORD}"
  
  # 创建数据库角色
  vault write database/roles/mydb-role \
    db_name=mydb \
    creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}'; GRANT SELECT, INSERT, UPDATE, DELETE ON *.* TO '{{name}}'@'%';" \
    default_ttl="1h" \
    max_ttl="24h"
  
  # 启用 PKI 引擎
  vault secrets enable pki
  
  # 配置 PKI
  vault write pki/config/urls \
    issuing_certificates="https://vault.example.com:8200/v1/pki/ca" \
    crl_distribution_points="https://vault.example.com:8200/v1/pki/crl"
  
  # 创建根证书
  vault write pki/root/generate/internal \
    common_name="Example CA" \
    ttl=87600h
  
  # 创建角色
  vault write pki/roles/example-dot-com \
    allowed_domains="example.com" \
    allow_subdomains=true \
    max_ttl=720h
}

# 主函数
main() {
  local action=${1:-"init"}
  
  case ${action} in
    init)
      init_vault
      unseal_vault
      configure_auth
      configure_secrets_engines
      ;;
    unseal)
      unseal_vault
      ;;
    auth)
      configure_auth
      ;;
    secrets)
      configure_secrets_engines
      ;;
  esac
}

main "$@"
```

---

## 七、合规检查

### 7.1 CIS Benchmark 检查

```yaml
# cis-benchmark-job.yaml
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

### 7.2 安全审计脚本

```bash
#!/bin/bash
# security-audit.sh - 综合安全审计

set -euo pipefail

REPORT_DIR="/data/reports/security"
DATE=$(date +%Y%m%d)
REPORT_FILE="${REPORT_DIR}/security-audit-${DATE}.html"

mkdir -p ${REPORT_DIR}

log() { echo "[$(date '+%H:%M:%S')] $1"; }

# 1. RBAC 检查
check_rbac() {
  log "--- RBAC 检查 ---"
  
  echo "ClusterRoleBinding 数量:"
  kubectl get clusterrolebindings --no-headers | wc -l
  
  echo "具有 cluster-admin 权限的绑定:"
  kubectl get clusterrolebindings -o json | jq -r '
    .items[] |
    select(.roleRef.name == "cluster-admin") |
    "\(.metadata.name) -> \(.subjects[0].name // "unknown")"
  '
  
  echo "具有过高权限的 ServiceAccount:"
  kubectl get clusterrolebindings -o json | jq -r '
    .items[] |
    select(.roleRef.name == "cluster-admin") |
    .subjects[]? |
    select(.kind == "ServiceAccount") |
    "\(.namespace)/\(.name)"
  ' 2>/dev/null | head -10
}

# 2. 网络策略检查
check_network_policies() {
  log "--- 网络策略检查 ---"
  
  for ns in $(kubectl get ns --no-headers | awk '{print $1}'); do
    np_count=$(kubectl get networkpolicies -n ${ns} --no-headers 2>/dev/null | wc -l)
    if [ "${np_count}" -eq 0 ]; then
      echo "⚠️  命名空间 ${ns} 没有网络策略"
    fi
  done
}

# 3. Pod 安全检查
check_pod_security() {
  log "--- Pod 安全检查 ---"
  
  echo "特权容器:"
  kubectl get pods -A -o json | jq -r '
    .items[] |
    . as $pod |
    .spec.containers[]? |
    select(.securityContext.privileged == true) |
    "\($pod.metadata.namespace)/\($pod.metadata.name): \(.name)"
  ' 2>/dev/null | head -10
  
  echo ""
  echo "使用 hostNetwork 的 Pod:"
  kubectl get pods -A -o json | jq -r '
    .items[] |
    select(.spec.hostNetwork == true) |
    "\(.metadata.namespace)/\(.metadata.name)"
  ' 2>/dev/null | head -10
}

# 4. 镜像安全检查
check_image_security() {
  log "--- 镜像安全检查 ---"
  
  echo "使用 latest 标签的 Pod:"
  kubectl get pods -A -o json | jq -r '
    .items[] |
    . as $pod |
    .spec.containers[].image |
    select(endswith(":latest") or (contains(":") | not)) |
    "\($pod.metadata.namespace)/\($pod.metadata.name): \(.)" 2>/dev/null
  ' | sort -u | head -10
  
  echo ""
  echo "使用非信任仓库的 Pod:"
  kubectl get pods -A -o json | jq -r '
    .items[] |
    . as $pod |
    .spec.containers[].image |
    select(startswith("harbor.example.com/") | not) |
    select(startswith("registry.aliyuncs.com/") | not) |
    select(startswith("quay.io/") | not) |
    "\($pod.metadata.namespace)/\($pod.metadata.name): \(.)" 2>/dev/null
  ' | sort -u | head -10
}

# 5. 资源限制检查
check_resource_limits() {
  log "--- 资源限制检查 ---"
  
  echo "未设置资源限制的 Pod:"
  kubectl get pods -A -o json | jq -r '
    .items[] |
    . as $pod |
    .spec.containers[] |
    select(.resources.limits == null) |
    "\($pod.metadata.namespace)/\($pod.metadata.name): \(.name)"
  ' 2>/dev/null | head -10
}

# 6. 敏感信息检查
check_secrets() {
  log "--- 敏感信息检查 ---"
  
  echo "未加密的 Secret:"
  kubectl get secrets -A -o json | jq -r '
    .items[] |
    select(.type == "Opaque") |
    "\(.metadata.namespace)/\(.metadata.name)"
  ' 2>/dev/null | head -20
}

# 生成报告
generate_report() {
  log "=== 生成安全审计报告 ==="
  
  cat > ${REPORT_FILE} <<EOF
<!DOCTYPE html>
<html>
<head>
  <title>安全审计报告 - $(date '+%Y-%m-%d')</title>
  <style>
    body { font-family: Arial; margin: 20px; }
    .header { background: #f5f5f5; padding: 20px; border-radius: 5px; }
    .section { margin: 20px 0; padding: 15px; border: 1px solid #ddd; border-radius: 5px; }
    .warning { background: #fff3cd; border-color: #ffc107; }
    .danger { background: #f8d7da; border-color: #dc3545; }
    pre { background: #f8f9fa; padding: 10px; border-radius: 3px; overflow-x: auto; }
  </style>
</head>
<body>
  <div class="header">
    <h1>安全审计报告</h1>
    <p>生成时间: $(date '+%Y-%m-%d %H:%M:%S')</p>
  </div>
  
  <div class="section">
    <h2>1. RBAC 检查</h2>
    <pre>$(check_rbac 2>&1)</pre>
  </div>
  
  <div class="section">
    <h2>2. 网络策略检查</h2>
    <pre>$(check_network_policies 2>&1)</pre>
  </div>
  
  <div class="section">
    <h2>3. Pod 安全检查</h2>
    <pre>$(check_pod_security 2>&1)</pre>
  </div>
  
  <div class="section">
    <h2>4. 镜像安全检查</h2>
    <pre>$(check_image_security 2>&1)</pre>
  </div>
  
  <div class="section">
    <h2>5. 资源限制检查</h2>
    <pre>$(check_resource_limits 2>&1)</pre>
  </div>
  
  <div class="section">
    <h2>6. 敏感信息检查</h2>
    <pre>$(check_secrets 2>&1)</pre>
  </div>
</body>
</html>
EOF
  
  log "报告已生成: ${REPORT_FILE}"
}

# 主函数
main() {
  log "=== 安全审计开始 ==="
  
  check_rbac > /tmp/rbac.txt 2>&1
  check_network_policies > /tmp/network.txt 2>&1
  check_pod_security > /tmp/pod.txt 2>&1
  check_image_security > /tmp/image.txt 2>&1
  check_resource_limits > /tmp/resource.txt 2>&1
  check_secrets > /tmp/secrets.txt 2>&1
  
  generate_report
  
  log "=== 安全审计完成 ==="
}

main "$@"
```

---

## 八、面试题与思考

### Q1: DevSecOps 的核心理念是什么？

**答：** DevSecOps 是将安全融入 DevOps 全流程：

1. **左移安全 (Shift Left)**：安全检查前移到开发阶段，而非部署后
2. **自动化安全**：将安全检查集成到 CI/CD 流水线
3. **持续监控**：运行时持续监控和威胁检测
4. **合规即代码 (Compliance as Code)**：将合规要求转化为可执行的策略代码
5. **安全即服务**：提供自助式安全服务，降低开发门槛

### Q2: 如何实现 Kubernetes 的零信任安全？

**答：** 零信任安全模型：

1. **身份验证**：每个请求都需要验证身份（mTLS、JWT）
2. **最小权限**：RBAC + NetworkPolicy 实现最小权限
3. **微分段**：使用 NetworkPolicy 隔离不同服务
4. **持续验证**：运行时持续验证安全策略
5. **加密通信**：所有通信都加密（TLS/mTLS）
6. **审计日志**：记录所有操作，便于事后审计

### Q3: Vault 如何实现动态密钥？

**答：** 动态密钥是 Vault 的核心能力：

1. **数据库密钥**：Vault 可以动态创建数据库用户，每次请求生成新的临时凭据
2. **AWS IAM**：动态生成 AWS IAM 凭据
3. **PKI**：动态生成 TLS 证书
4. **租约管理**：每个密钥有租约，过期自动撤销
5. **轮转**：支持密钥自动轮转

**优势：**
- 凭据不持久化，减少泄露风险
- 自动轮转，无需人工干预
- 细粒度权限控制
- 完整的审计日志

---

## 附录

### A. 安全工具版本推荐

| 工具 | 推荐版本 | 用途 |
|------|---------|------|
| Trivy | 0.49.x | 镜像漏洞扫描 |
| Falco | 0.37.x | 运行时监控 |
| OPA/Gatekeeper | 3.14.x | 策略执行 |
| Vault | 1.15.x | 密钥管理 |
| Cosign | 2.2.x | 镜像签名 |
| GitLeaks | 8.18.x | Secrets 检测 |
| SonarQube | 10.x | 代码质量 |

### B. 参考资料

- [Trivy 官方文档](https://trivy.dev/)
- [Falco 官方文档](https://falco.org/docs/)
- [OPA 官方文档](https://www.openpolicyagent.org/docs/)
- [Vault 官方文档](https://developer.hashicorp.com/vault/docs)
- [Kubernetes Security Guide](https://kubernetes.io/docs/concepts/security/)
- [CIS Kubernetes Benchmark](https://www.cisecurity.org/benchmark/kubernetes)

---

*作者: 小白老师 | 更新时间: 2026-05-03 | 版本: v2.0*
