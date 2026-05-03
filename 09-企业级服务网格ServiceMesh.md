# 企业级服务网格(Service Mesh)

> 基于 Istio + Envoy + Kiali + Jaeger 的微服务治理平台
> 参考方案：蚂蚁金服 MOSN、字节跳动 Service Mesh、Linkerd

---

## 一、Service Mesh 架构

### 1.1 架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                    企业级服务网格架构                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────── 控制平面 ─────────────────────────────┐     │
│  │                                                       │     │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐          │     │
│  │  │  Istiod  │  │  Citadel │  │  Galley  │          │     │
│  │  │  (Pilot) │  │  (CA)    │  │  (Config)│          │     │
│  │  │ 配置分发  │  │ 证书管理  │  │ 配置验证  │          │     │
│  │  └──────────┘  └──────────┘  └──────────┘          │     │
│  │                                                       │     │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐          │     │
│  │  │  Kiali   │  │Prometheus│  │  Jaeger  │          │     │
│  │  │  (UI)    │  │ (Metrics)│  │  (Traces)│          │     │
│  │  │ 服务拓扑  │  │ 指标采集  │  │ 链路追踪  │          │     │
│  │  └──────────┘  └──────────┘  └──────────┘          │     │
│  └──────────────────────────────────────────────────────┘     │
│                         │                                       │
│  ┌──────────────────────▼──────────────────────────────┐     │
│  │                    数据平面                           │     │
│  │                                                       │     │
│  │  ┌──────────────────────────────────────────────┐   │     │
│  │  │  Pod A                    Pod B               │   │     │
│  │  │  ┌────────┐  ┌────────┐  ┌────────┐        │   │     │
│  │  │  │ App A  │  │ Envoy  │  │ App B  │        │   │     │
│  │  │  │        │◄▶│Sidecar │◄▶│        │        │   │     │
│  │  │  └────────┘  └────────┘  └────────┘        │   │     │
│  │  │        ▲                           ▲         │   │     │
│  │  │        │       iptables 拦截       │         │   │     │
│  │  │        └───────────────────────────┘         │   │     │
│  │  └──────────────────────────────────────────────┘   │     │
│  │                                                       │     │
│  │  所有流量经过 Envoy Sidecar 代理处理                   │     │
│  │  应用无需感知代理的存在（透明代理）                     │     │
│  └──────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 核心组件职责

| 组件 | 职责 | 部署方式 |
|------|------|---------|
| Istiod | 配置分发、服务发现、证书管理 | Deployment (3 副本) |
| Envoy | 流量代理、负载均衡、熔断 | Sidecar (每 Pod 一个) |
| Kiali | 服务拓扑可视化、配置管理 | Deployment (1 副本) |
| Jaeger | 分布式链路追踪 | Deployment (1 副本) |
| Prometheus | 指标采集和存储 | Deployment (1 副本) |

---

## 二、Istio 部署

### 2.1 Istio Operator 配置

```yaml
# istio-operator.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: istio-control-plane
  namespace: istio-system
spec:
  profile: default
  
  meshConfig:
    # 访问日志
    accessLogFile: /dev/stdout
    accessLogEncoding: JSON
    
    # 默认配置
    defaultConfig:
      # 链路追踪
      tracing:
        sampling: 100.0
      # DNS 配置
      proxyMetadata:
        ISTIO_META_DNS_CAPTURE: "true"
        ISTIO_META_DNS_AUTO_ALLOCATE: "true"
    
    # 自动 mTLS
    enableAutoMtls: true
    
    # 出站流量策略
    outboundTrafficPolicy:
      mode: REGISTRY_ONLY
    
    # 启用追踪
    enableTracing: true
    
    # 默认提供者
    defaultProviders:
      tracing:
        - "zipkin"
      metrics:
        - "prometheus"
  
  components:
    # 控制平面
    pilot:
      k8s:
        replicaCount: 2
        resources:
          requests:
            cpu: 500m
            memory: 2Gi
          limits:
            cpu: "2"
            memory: 4Gi
        hpaSpec:
          minReplicas: 2
          maxReplicas: 5
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
                          - istiod
                  topologyKey: kubernetes.io/hostname
    
    # 入站网关
    ingressGateways:
      - name: istio-ingressgateway
        enabled: true
        k8s:
          replicaCount: 2
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              cpu: "1"
              memory: 1Gi
          hpaSpec:
            minReplicas: 2
            maxReplicas: 10
    
    # 出站网关
    egressGateways:
      - name: istio-egressgateway
        enabled: true
        k8s:
          replicaCount: 1
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
    
    # Kiali
    kiali:
      enabled: true
      k8s:
        replicaCount: 1
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
    
    # Prometheus
    prometheus:
      enabled: true
      k8s:
        replicaCount: 1
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 1Gi
    
    # 链路追踪
    tracing:
      enabled: true
      provider:
        name: zipkin
      k8s:
        replicaCount: 1
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 1Gi
```

```yaml
# istio-pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: istiod-pdb
  namespace: istio-system
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: istiod
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: istio-ingress-pdb
  namespace: istio-system
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: istio-ingressgateway
```

```yaml
# istio-network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: istio-network-policy
  namespace: istio-system
spec:
  podSelector:
    matchLabels:
      app: istiod
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: istio-system
      ports:
        - protocol: TCP
          port: 15012
        - protocol: TCP
          port: 15014
  egress:
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: TCP
          port: 443
        - protocol: TCP
          port: 15012
```

```yaml
# istio-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: istio-ingress
  namespace: istio-system
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - kiali.example.com
        - jaeger.example.com
        - grafana.example.com
      secretName: istio-tls
  rules:
    - host: kiali.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: kiali
                port:
                  number: 20001
    - host: jaeger.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: jaeger-collector
                port:
                  number: 14268
    - host: grafana.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: grafana
                port:
                  number: 3000
```

### 2.2 安装步骤

```bash
#!/bin/bash
# install-istio.sh - Istio 安装脚本

set -euo pipefail

log() { echo "[$(date '+%H:%M:%S')] $1"; }

log "=== 安装 Istio ==="

# 1. 下载 istioctl
if ! command -v istioctl &> /dev/null; then
  log "下载 istioctl..."
  curl -L https://istio.io/downloadIstio | sh -
  export PATH=$PWD/istio-*/bin:$PATH
fi

# 2. 检查版本
istioctl version --remote=false

# 3. 安装 Istio
log "安装 Istio..."
istioctl install -f istio-operator.yaml -y

# 4. 等待安装完成
log "等待 Istiod 就绪..."
kubectl wait --for=condition=Ready pods -l app=istiod -n istio-system --timeout=300s

# 5. 验证安装
log "验证安装..."
istioctl verify-install

# 6. 查看 Pod 状态
kubectl get pods -n istio-system

# 7. 启用自动注入
log "启用命名空间自动注入..."
kubectl label namespace production istio-injection=enabled --overwrite
kubectl label namespace staging istio-injection=enabled --overwrite

log "=== Istio 安装完成 ==="
```

---

## 三、Sidecar 自动注入

### 3.1 命名空间标签

```bash
# 启用自动注入
kubectl label namespace production istio-injection=enabled

# 禁用自动注入
kubectl label namespace production istio-injection=disabled

# 查看标签
kubectl get namespace -L istio-injection
```

### 3.2 手动注入

```bash
# 手动注入 Sidecar
istioctl kube-inject -f deployment.yaml | kubectl apply -f -

# 查看注入后的 YAML
istioctl kube-inject -f deployment.yaml
```

### 3.3 验证 Sidecar 注入

```bash
#!/bin/bash
# verify-sidecar.sh - 验证 Sidecar 注入

set -euo pipefail

NAMESPACE=${1:-production}

echo "=== 验证 Sidecar 注入: ${NAMESPACE} ==="

# 检查 Pod 中的容器数量
echo "Pod 容器数量检查:"
kubectl get pods -n ${NAMESPACE} -o json | jq -r '
  .items[] |
  . as $pod |
  "\($pod.metadata.name): \(.spec.containers | length) 个容器 (\(.spec.containers | map(.name) | join(", ")))"
'

echo ""
echo "Sidecar 注入状态:"
kubectl get pods -n ${NAMESPACE} -o json | jq -r '
  .items[] |
  . as $pod |
  if .metadata.annotations["sidecar.istio.io/inject"] == "true" or
     (.metadata.labels["istio-injection"] == "enabled") then
    "\($pod.metadata.name): ✅ 已注入"
  else
    "\($pod.metadata.name): ❌ 未注入"
  end
'
```

---

## 四、流量管理

### 4.1 金丝雀发布

```yaml
# canary-deployment.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: app-canary
  namespace: production
spec:
  hosts:
    - app.example.com
  gateways:
    - app-gateway
  http:
    # 金丝雀流量（10%）
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: app
            subset: canary
          weight: 100
    # 主流量（90% stable + 10% canary）
    - route:
        - destination:
            host: app
            subset: stable
          weight: 90
        - destination:
            host: app
            subset: canary
          weight: 10
      # 超时重试
      timeout: 30s
      retries:
        attempts: 3
        perTryTimeout: 10s
        retryOn: 5xx,reset,connect-failure

---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: app-destination
  namespace: production
spec:
  host: app
  trafficPolicy:
    # 连接池配置
    connectionPool:
      tcp:
        maxConnections: 1000
        connectTimeout: 30ms
      http:
        http1MaxPendingRequests: 1000
        http2MaxRequests: 1000
        maxRequestsPerConnection: 100
        maxRetries: 3
    # 负载均衡
    loadBalancer:
      simple: LEAST_CONN
    # 异常检测（熔断）
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  subsets:
    - name: stable
      labels:
        version: v1
    - name: canary
      labels:
        version: v2
```

### 4.2 流量镜像（影子流量）

```yaml
# traffic-mirroring.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: app-mirror
  namespace: production
spec:
  hosts:
    - app
  http:
    - route:
        - destination:
            host: app
            subset: stable
      # 镜像 10% 流量到 canary
      mirror:
        host: app
        subset: canary
      mirrorPercentage:
        value: 10.0
```

### 4.3 故障注入

```yaml
# fault-injection.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: app-fault-injection
  namespace: production
spec:
  hosts:
    - app
  http:
    - fault:
        delay:
          percentage:
            value: 10.0
          fixedDelay: 5s
        abort:
          percentage:
            value: 5.0
          httpStatus: 500
      route:
        - destination:
            host: app
```

### 4.4 超时重试

```yaml
# timeout-retry.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: app-timeout-retry
  namespace: production
spec:
  hosts:
    - app
  http:
    - route:
        - destination:
            host: app
      timeout: 30s
      retries:
        attempts: 3
        perTryTimeout: 10s
        retryOn: 5xx,reset,connect-failure,retriable-4xx
```

---

## 五、安全策略

### 5.1 mTLS 强制

```yaml
# peer-authentication.yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT
```

### 5.2 授权策略

```yaml
# authorization-policy.yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: app-auth
  namespace: production
spec:
  selector:
    matchLabels:
      app: app
  rules:
    # 允许 frontend 访问 backend
    - from:
        - source:
            principals:
              - "cluster.local/ns/production/sa/frontend"
      to:
        - operation:
            methods: ["GET", "POST"]
            paths: ["/api/*"]
      when:
        - key: request.auth.claims[iss]
          values: ["https://auth.example.com"]
    
    # 允许 Prometheus 访问 metrics
    - from:
        - source:
            namespaces: ["monitoring"]
      to:
        - operation:
            paths: ["/metrics", "/health"]
    
    # 拒绝所有其他访问
    - from:
        - source: {}
```

### 5.3 JWT 认证

```yaml
# jwt-authentication.yaml
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: jwt-auth
  namespace: production
spec:
  selector:
    matchLabels:
      app: app
  jwtRules:
    - issuer: "https://auth.example.com"
      jwksUri: "https://auth.example.com/.well-known/jwks.json"
      audiences:
        - "https://api.example.com"
      forwardOriginalToken: true
```

---

## 六、可观测性

### 6.1 Kiali 配置

```yaml
# kiali-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kiali
  namespace: istio-system
data:
  config.yaml: |
    server:
      port: 20001
      metrics_enabled: true
      metrics_port: 9090
      web_root: /
    
    auth:
      strategy: token
      token:
        issuer: "https://kubernetes.default.svc.cluster.local"
        audience: "https://kubernetes.default.svc.cluster.local"
    
    external_services:
      prometheus:
        url: "http://prometheus:9090"
        custom_metrics_url: "http://prometheus:9090/api/v1/query"
      
      grafana:
        url: "http://grafana:3000"
        auth:
          type: basic
          username: admin
          password: admin
      
      jaeger:
        url: "http://tracing:16686"
      
      custom_dashboards:
        enabled: true
```

### 6.2 Istio 指标告警

```yaml
# istio-alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: istio-alerts
  namespace: monitoring
spec:
  groups:
    - name: istio
      rules:
        - alert: IstioHighRequestLatency
          expr: |
            histogram_quantile(0.99, 
              sum(rate(istio_request_duration_milliseconds_bucket[5m])) by (le, destination_service)
            ) > 1000
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "Istio 请求延迟过高"
            description: "服务 {{ $labels.destination_service }} P99 延迟 {{ $value }}ms"
        
        - alert: IstioHighErrorRate
          expr: |
            sum(rate(istio_requests_total{response_code=~"5.."}[5m])) by (destination_service)
            /
            sum(rate(istio_requests_total[5m])) by (destination_service)
            > 0.05
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Istio 错误率超过 5%"
            description: "服务 {{ $labels.destination_service }} 错误率 {{ $value | humanizePercentage }}"
        
        - alert: IstioSidecarInjectionFailure
          expr: |
            sum(rate(istio_sidecar_injection_failure_total[5m])) > 0
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Istio Sidecar 注入失败"
```

---

## 七、运维脚本

### 7.1 Istio 健康检查

```bash
#!/bin/bash
# istio-health-check.sh - Istio 健康检查

set -euo pipefail

echo "=== Istio 健康检查 ==="

# 1. 控制平面状态
echo "--- 控制平面状态 ---"
kubectl get pods -n istio-system -o wide
echo ""
istioctl proxy-status

# 2. 配置验证
echo ""
echo "--- 配置验证 ---"
istioctl analyze -n production

# 3. 代理同步状态
echo ""
echo "--- 代理同步状态 ---"
kubectl get pods -n production -o json | jq -r '
  .items[] |
  . as $pod |
  .metadata.annotations["sidecar.istio.io/status"] // "not-injected" |
  "\($pod.metadata.name): \(.)"
' 2>/dev/null | head -10

# 4. 服务拓扑
echo ""
echo "--- 服务列表 ---"
kubectl get virtualservices -A
echo ""
kubectl get destinationrules -A
echo ""
kubectl get gateways -A
```

### 7.2 流量调试

```bash
#!/bin/bash
# istio-traffic-debug.sh - 流量调试

set -euo pipefail

NAMESPACE=${1:-production}

echo "=== 流量调试: ${NAMESPACE} ==="

# 1. 查看 Pod 代理配置
echo "--- 代理配置 ---"
kubectl exec -n ${NAMESPACE} $(kubectl get pod -n ${NAMESPACE} -l app=app -o jsonpath='{.items[0].metadata.name}') -c istio-proxy -- \
  pilot-agent request GET config_dump | jq '.configs[] | select(."@type" == "type.googleapis.com/envoy.admin.v3.ConfigDump")' | \
  jq '.configs[0].dynamic_active_clusters[]' | head -20

# 2. 查看路由规则
echo ""
echo "--- 路由规则 ---"
kubectl exec -n ${NAMESPACE} $(kubectl get pod -n ${NAMESPACE} -l app=app -o jsonpath='{.items[0].metadata.name}') -c istio-proxy -- \
  pilot-agent request GET routes | jq '.routes[].virtual_hosts[].name' 2>/dev/null | head -10

# 3. 查看集群信息
echo ""
echo "--- 集群信息 ---"
kubectl exec -n ${NAMESPACE} $(kubectl get pod -n ${NAMESPACE} -l app=app -o jsonpath='{.items[0].metadata.name}') -c istio-proxy -- \
  pilot-agent request GET clusters | jq '.cluster_descriptors[] | select(.type | contains("EDS"))' | \
  jq '{name: .name, endpoints: (.endpoints | length)}' 2>/dev/null | head -10
```

---

## 八、面试题与思考

### Q1: Sidecar 模式的工作原理是什么？

**答：** Sidecar 模式通过在每个 Pod 旁边运行 Envoy 代理来实现：

1. **流量拦截**：通过 iptables 规则拦截所有入站和出站流量
2. **透明代理**：应用无需感知代理的存在
3. **统一治理**：所有流量都经过代理处理
4. **零侵入**：不需要修改应用代码

**流量路径：**
```
外部请求 → kube-proxy → Pod IP → iptables → Envoy → App
App 响应 → Envoy → iptables → Pod IP → 外部
```

### Q2: Istio 如何实现 mTLS？

**答：** Istio 通过以下步骤实现 mTLS：

1. **证书生成**：Citadel 为每个服务生成 TLS 证书
2. **证书分发**：通过 SDS (Secret Discovery Service) 分发证书
3. **流量加密**：所有服务间流量都加密
4. **身份验证**：双向验证服务身份
5. **策略控制**：基于身份实施访问控制

**mTLS 模式：**
- **DISABLE**：不使用 mTLS
- **PERMISSIVE**：同时支持 mTLS 和明文
- **STRICT**：只允许 mTLS

### Q3: 如何监控 Service Mesh 的性能？

**答：** 监控维度：

1. **延迟**：请求延迟、P99 延迟、重试延迟
2. **流量**：QPS、错误率、流量分布
3. **错误**：5xx 错误、超时、熔断
4. **饱和度**：连接数、队列长度、资源使用
5. **控制平面**：Istiod 配置分发延迟、代理同步状态

**关键指标：**
```promql
# 请求延迟
histogram_quantile(0.99, rate(istio_request_duration_milliseconds_bucket[5m]))

# 错误率
sum(rate(istio_requests_total{response_code=~"5.."}[5m])) / sum(rate(istio_requests_total[5m]))

# 活跃连接数
sum(envoy_cluster_upstream_cx_active) by (cluster_name)
```

### Q4: 如何处理 Istio 的性能开销？

**答：** Sidecar 会增加约 1-2ms 的延迟，处理方式：

1. **减少转发次数**：合并多个请求
2. **调整连接池**：优化 HTTP/TCP 连接池大小
3. **启用 HTTP/2**：减少连接建立开销
4. **调整采样率**：降低追踪采样率
5. **资源限制**：为 Sidecar 设置合理的资源限制

**性能数据：**
- 延迟增加：1-2ms (P99)
- CPU 开销：约 0.5 核/1000 QPS
- 内存开销：约 50MB/Sidecar

### Q5: Istio 和 Linkerd 如何选择？

**答：**

| 特性 | Istio | Linkerd |
|------|-------|---------|
| 复杂度 | 高 | 低 |
| 功能 | 丰富 | 精简 |
| 性能 | 中等 | 高 |
| 资源消耗 | 较高 | 较低 |
| 学习曲线 | 陡峭 | 平缓 |
| 社区 | CNCF 毕业 | CNCF 毕业 |
| 适用场景 | 大规模、多功能 | 中小规模、轻量级 |

**选择建议：**
- **选 Istio**：需要丰富功能、大规模集群、多团队协作
- **选 Linkerd**：追求简单、轻量级、快速部署

---

## 附录

### A. Istio 常用命令

```bash
# 安装
istioctl install --set profile=default
istioctl install -f istio-operator.yaml

# 验证
istioctl verify-install
istioctl analyze -n <namespace>

# 代理状态
istioctl proxy-status
istioctl proxy-config cluster <pod>
istioctl proxy-config listener <pod>
istioctl proxy-config route <pod>

# 流量管理
istioctl experimental authz check <pod>
kubectl get virtualservices -A
kubectl get destinationrules -A

# 调试
istioctl dashboard kiali
istioctl dashboard grafana
istioctl dashboard jaeger
```

### B. 参考资料

- [Istio 官方文档](https://istio.io/latest/docs/)
- [Envoy 官方文档](https://www.envoyproxy.io/docs/envoy/latest/)
- [Kiali 官方文档](https://kiali.io/documentation/)
- [Jaeger 官方文档](https://www.jaegertracing.io/docs/)
- [Service Mesh 性能测试](https://github.com/istio/tools/tree/master/perf)

---

*作者: 小白老师 | 更新时间: 2026-05-03 | 版本: v2.0*
