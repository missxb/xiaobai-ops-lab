# 企业级服务网格(Service Mesh)

> 基于 Istio + Envoy + Kiali 的微服务治理平台
> 参考方案：蚂蚁金服 MOSN、字节跳动 Service Mesh

---

## 一、Service Mesh 架构

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
│  │  └──────────┘  └──────────┘  └──────────┘          │     │
│  │                                                       │     │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐          │     │
│  │  │  Kiali   │  │Prometheus│  │  Jaeger  │          │     │
│  │  │  (UI)    │  │ (Metrics)│  │  (Traces)│          │     │
│  │  └──────────┘  └──────────┘  └──────────┘          │     │
│  └──────────────────────────────────────────────────────┘     │
│                         │                                       │
│  ┌──────────────────────▼──────────────────────────────┐     │
│  │                    数据平面                           │     │
│  │                                                       │     │
│  │  ┌──────────────────────────────────────────────┐   │     │
│  │  │  Service A  ◄── Envoy Sidecar ──▶ Service B  │   │     │
│  │  │             ◄── Envoy Sidecar ──▶ Service C  │   │     │
│  │  └──────────────────────────────────────────────┘   │     │
│  │                                                       │     │
│  │  每个 Pod 注入 Envoy Sidecar 代理                     │     │
│  │  所有流量经过 Envoy 代理处理                           │     │
│  └──────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────┘
```

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
    accessLogFile: /dev/stdout
    accessLogEncoding: JSON
    defaultConfig:
      tracing:
        sampling: 100.0
      proxyMetadata:
        ISTIO_META_DNS_CAPTURE: "true"
        ISTIO_META_DNS_AUTO_ALLOCATE: "true"
    
    enableAutoMtls: true
    
    outboundTrafficPolicy:
      mode: REGISTRY_ONLY
    
    enableTracing: true
    
    defaultProviders:
      tracing:
        - "zipkin"
      metrics:
        - "prometheus"
  
  components:
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

---

## 三、流量管理

### 3.1 VirtualService 配置

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
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: app
            subset: canary
          weight: 100
    - route:
        - destination:
            host: app
            subset: stable
          weight: 90
        - destination:
            host: app
            subset: canary
          weight: 10
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
    connectionPool:
      tcp:
        maxConnections: 1000
        connectTimeout: 30ms
      http:
        http1MaxPendingRequests: 1000
        http2MaxRequests: 1000
        maxRequestsPerConnection: 100
        maxRetries: 3
    loadBalancer:
      simple: LEAST_CONN
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

### 3.2 流量镜像

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
      mirror:
        host: app
        subset: canary
      mirrorPercentage:
        value: 10.0
```

---

## 四、安全策略

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

---
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

---

## 五、Kiali 仪表盘

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
        auth:
          type: bearer
          token: ""
      
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

---

## 六、面试题与思考

### Q1: Sidecar 模式和边车模式有什么区别？

**答：** 在 Istio 中，Sidecar 是指每个 Pod 旁边运行的 Envoy 代理：

1. **流量拦截**：通过 iptables 规则拦截所有入站和出站流量
2. **透明代理**：应用无需感知代理的存在
3. **统一治理**：所有流量都经过代理处理
4. **零侵入**：不需要修改应用代码

### Q2: Istio 如何实现 mTLS？

**答：** Istio 通过以下步骤实现 mTLS：

1. **证书生成**：Citadel 为每个服务生成 TLS 证书
2. **证书分发**：通过 SDS (Secret Discovery Service) 分发证书
3. **流量加密**：所有服务间流量都加密
4. **身份验证**：双向验证服务身份
5. **策略控制**：基于身份实施访问控制

### Q3: 如何监控 Service Mesh 的性能？

**答：** 监控维度：

1. **延迟**：请求延迟、P99 延迟、重试延迟
2. **流量**：QPS、错误率、流量分布
3. **错误**：5xx 错误、超时、熔断
4. **饱和度**：连接数、队列长度、资源使用
5. **控制平面**：Istiod 配置分发延迟、代理同步状态

---

*作者: 小白老师 | 更新时间: 2026-05-03 | 版本: v1.0*
