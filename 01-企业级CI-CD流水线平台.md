# 企业级 CI/CD 流水线平台

> 基于 GitLab CI + ArgoCD + Harbor + Kubernetes 的企业级持续集成/持续部署平台
> 参考方案：字节跳动 Aegle CI、Spotify Backstage、Netflix Spinnaker

---

## 一、架构设计

```
┌─────────────────────────────────────────────────────────────────┐
│                     企业级 CI/CD 平台架构                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐  │
│  │  开发者   │───▶│  GitLab  │───▶│ GitLab   │───▶│  Harbor  │  │
│  │  Git Push │    │  Server  │    │  Runner  │    │  镜像仓库 │  │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘  │
│                        │               │              │         │
│                        ▼               ▼              │         │
│                  ┌──────────┐    ┌──────────┐        │         │
│                  │ Webhook  │    │  制品库   │        │         │
│                  │ 触发器    │    │ Artifacts│        │         │
│                  └──────────┘    └──────────┘        │         │
│                                                      │         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    GitOps 控制平面                        │   │
│  │  ┌──────────┐    ┌──────────┐    ┌──────────┐          │   │
│  │  │  ArgoCD  │───▶│  K8s API │───▶│ Worker   │          │   │
│  │  │  Server  │    │  Server  │    │  Nodes   │          │   │
│  │  └──────────┘    └──────────┘    └──────────┘          │   │
│  │       │                                             │   │
│  │       ▼                                             │   │
│  │  ┌──────────┐    ┌──────────┐                       │   │
│  │  │  Git Repo│    │  Kustomize│                       │   │
│  │  │ (配置源) │    │  Helm     │                       │   │
│  │  └──────────┘    └──────────┘                       │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    监控告警层                              │   │
│  │  ┌──────────┐    ┌──────────┐    ┌──────────┐          │   │
│  │  │Prometheus│    │ Grafana  │    │ AlertMgr │          │   │
│  │  └──────────┘    └──────────┘    └──────────┘          │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘

数据流：
  开发者 git push → GitLab 触发 Pipeline → 构建/测试/扫描 → 推送镜像到 Harbor
  → ArgoCD 监测到新镜像 → 自动/手动同步到 K8s 集群 → 部署完成
```

---

## 二、Harbor 私有镜像仓库部署

### 2.1 Harbor 高可用 docker-compose 配置

```yaml
# docker-compose.yml - Harbor HA (使用外部数据库和Redis)
version: '3.8'

services:
  # ===== Nginx 反向代理 =====
  nginx:
    image: goharbor/nginx-photon:v2.11.0
    container_name: harbor-nginx
    restart: always
    ports:
      - "443:443"
      - "80:80"
    volumes:
      - ./common/config/nginx:/etc/nginx
      - /data/cert:/etc/cert:ro
      - /data/nginx:/var/log/nginx
    depends_on:
      - core
      - portal
      - registry
    networks:
      - harbor-net
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 1G

  # ===== Registry 镜像存储 =====
  registry:
    image: goharbor/registry-photon:v2.11.0
    container_name: harbor-registry
    restart: always
    volumes:
      - /data/registry:/storage
      - ./common/config/registry:/etc/registry
    environment:
      - CORE_SECRET=${CORE_SECRET}
      - JOBSERVICE_SECRET=${JOBSERVICE_SECRET}
      - REGISTRY_HTTP_SECRET=${REGISTRY_HTTP_SECRET}
    networks:
      - harbor-net

  # ===== Core 核心服务 =====
  core:
    image: goharbor/core-photon:v2.11.0
    container_name: harbor-core
    restart: always
    environment:
      - CORE_URL=http://core:8080
      - REGISTRY_URL=http://registry:5000
      - TOKEN_SERVICE_URL=http://core:8080/service/token
      - JOBSERVICE_URL=http://jobservice:8080
      - DATABASE_TYPE=postgresql
      - DATABASE_HOST=postgresql
      - DATABASE_PORT=5432
      - DATABASE_USERNAME=${DB_USER}
      - DATABASE_PASSWORD=${DB_PASS}
      - DATABASE_DBNAME=registry
      - REDIS_URL=redis:6379
      - REDIS_PASSWORD=${REDIS_PASS}
    volumes:
      - /data/ca_download:/etc/core/ca
      - /data/core:/data
    networks:
      - harbor-net

  # ===== Job Service 任务服务 =====
  jobservice:
    image: goharbor/jobservice-photon:v2.11.0
    container_name: harbor-jobservice
    restart: always
    environment:
      - CORE_URL=http://core:8080
      - REGISTRY_URL=http://registry:5000
      - TOKEN_SERVICE_URL=http://core:8080/service/token
      - DATABASE_TYPE=postgresql
      - DATABASE_HOST=postgresql
      - DATABASE_PORT=5432
      - DATABASE_USERNAME=${DB_USER}
      - DATABASE_PASSWORD=${DB_PASS}
      - DATABASE_DBNAME=registry
      - REDIS_URL=redis:6379
      - REDIS_PASSWORD=${REDIS_PASS}
    volumes:
      - /data/job_logs:/var/log/jobs
    networks:
      - harbor-net

  # ===== Portal 前端 =====
  portal:
    image: goharbor/portal-photon:v2.11.0
    container_name: harbor-portal
    restart: always
    networks:
      - harbor-net

  # ===== Registryctl 管理控制 =====
  registryctl:
    image: goharbor/registryctl-photon:v2.11.0
    container_name: harbor-registryctl
    restart: always
    volumes:
      - /data/registry:/storage
      - ./common/config/registry:/etc/registry
    networks:
      - harbor-net

  # ===== PostgreSQL 数据库 =====
  postgresql:
    image: goharbor/harbor-db:v2.11.0
    container_name: harbor-db
    restart: always
    environment:
      - POSTGRES_PASSWORD=${DB_PASS}
      - POSTGRES_DB=registry
    volumes:
      - /data/database:/var/lib/postgresql/data
    networks:
      - harbor-net

  # ===== Redis 缓存 =====
  redis:
    image: goharbor/redis-photon:v2.11.0
    container_name: harbor-redis
    restart: always
    command: redis-server --requirepass ${REDIS_PASS} --maxmemory 2g --maxmemory-policy allkeys-lru
    volumes:
      - /data/redis:/var/lib/redis
    networks:
      - harbor-net

  # ===== Trivy 漏洞扫描 =====
  trivy-adapter:
    image: goharbor/trivy-adapter-photon:v2.11.0
    container_name: harbor-trivy
    restart: always
    environment:
      - SCANNER_REDIS_URL=redis://:${REDIS_PASS}@redis:6379
      - TRIVY_SERVER_URL=http://trivy-server:8080
    volumes:
      - /data/trivy-adapter:/home/scanner/.cache
    networks:
      - harbor-net

  # ===== Chartmuseum 制品仓库 =====
  chartmuseum:
    image: goharbor/chartmuseum-photon:v2.11.0
    container_name: harbor-chartmuseum
    restart: always
    environment:
      - STORAGE=local
      - STORAGE_LOCAL_ROOTDIR=/chart_storage
    volumes:
      - /data/chart_storage:/chart_storage
    networks:
      - harbor-net

  # ===== Notary 签名服务 =====
  notary-server:
    image: goharbor/notary-server-photon:v2.11.0
    container_name: harbor-notary-server
    restart: always
    volumes:
      - ./common/config/notary:/etc/notary
    networks:
      - harbor-net

  notary-signer:
    image: goharbor/notary-signer-photon:v2.11.0
    container_name: harbor-notary-signer
    restart: always
    volumes:
      - ./common/config/notary:/etc/notary
    networks:
      - harbor-net

networks:
  harbor-net:
    driver: bridge
```

### 2.2 Harbor 环境变量配置

```bash
# harbor.env - 生产环境配置
# 数据库
DB_USER=harbor
DB_PASS=HarborDB@2026!Secure
DB_PASS.Escape=$$

# Redis
REDIS_PASS=HarborRedis@2026!

# Core
CORE_SECRET=HarborCoreSecret2026!
JOBSERVICE_SECRET=HarborJobServiceSecret2026!
REGISTRY_HTTP_SECRET=HarborRegistryHTTPSecret2026!

# 证书
CERT_PATH=/data/cert/server.crt
KEY_PATH=/data/cert/server.key
```

### 2.3 Harbor 证书配置

```bash
#!/bin/bash
# setup-harbor-certs.sh - 生成Harbor自签名证书

set -euo pipefail

CERT_DIR="/data/cert"
DOMAIN="harbor.example.com"
DAYS=3650
NAMESPACE="harbor"

echo "=== 生成 Harbor TLS 证书 ==="

mkdir -p ${CERT_DIR}

# 生成 CA 证书
openssl genrsa -out ${CERT_DIR}/ca.key 4096
openssl req -x509 -new -nodes -key ${CERT_DIR}/ca.key \
  -sha256 -days ${DAYS} \
  -out ${CERT_DIR}/ca.crt \
  -subj "/C=CN/ST=Beijing/L=Beijing/O=Example/CN=Harbor-CA"

# 生成服务端证书
openssl genrsa -out ${CERT_DIR}/server.key 4096
openssl req -new -key ${CERT_DIR}/server.key \
  -out ${CERT_DIR}/server.csr \
  -subj "/C=CN/ST=Beijing/L=Beijing/O=Example/CN=${DOMAIN}"

cat > ${CERT_DIR}/v3.ext <<EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = ${DOMAIN}
DNS.2 = *.${DOMAIN}
DNS.3 = localhost
IP.1 = 127.0.0.1
EOF

openssl x509 -req -in ${CERT_DIR}/server.csr \
  -CA ${CERT_DIR}/ca.crt -CAkey ${CERT_DIR}/ca.key \
  -CAcreateserial -out ${CERT_DIR}/server.crt \
  -days ${DAYS} -sha256 -extfile ${CERT_DIR}/v3.ext

echo "=== 证书生成完成 ==="
echo "CA:   ${CERT_DIR}/ca.crt"
echo "Key:  ${CERT_DIR}/server.key"
echo "Cert: ${CERT_DIR}/server.crt"
```

### 2.4 Harbor 垃圾回收策略

```bash
#!/bin/bash
# harbor-gc.sh - Harbor 镜像垃圾回收定时任务

set -euo pipefail

HARBOR_URL="https://harbor.example.com"
HARBOR_USER="admin"
HARBOR_PASS="${HARBOR_PASSWORD}"
LOG_FILE="/var/log/harbor/gc-$(date +%Y%m%d).log"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a ${LOG_FILE}; }

# 获取项目列表
PROJECTS=$(curl -sk -u "${HARBOR_USER}:${HARBOR_PASS}" \
  "${HARBOR_URL}/api/v2.0/projects" | jq -r '.[].name')

for project in ${PROJECTS}; do
  log "开始回收项目: ${project}"
  
  # 触发 GC
  RESPONSE=$(curl -sk -u "${HARBOR_USER}:${HARBOR_PASS}" \
    -X POST "${HARBOR_URL}/api/v2.0/projects/${project}/artifacts" \
    -H "Content-Type: application/json" \
    -d '{"delete": true, "with_chart": true}' \
    -w "%{http_code}")
  
  if [ "${RESPONSE}" = "200" ] || [ "${RESPONSE}" = "202" ]; then
    log "项目 ${project} GC 触发成功"
  else
    log "项目 ${project} GC 触发失败 (HTTP ${RESPONSE})"
  fi
  
  # 避免过载
  sleep 5
done

# 删除未打标签的镜像
for project in ${PROJECTS}; do
  log "清理项目 ${project} 未打标签的镜像"
  curl -sk -u "${HARBOR_USER}:${HARBOR_PASS}" \
    -X DELETE "${HARBOR_URL}/api/v2.0/projects/${project}/artifacts" \
    -H "Content-Type: application/json" \
    -d '{"with_tag": false}'
done

log "=== 垃圾回收完成 ==="
```

---

## 三、GitLab CI/CD 完整配置

### 3.1 .gitlab-ci.yml - 全流程流水线

```yaml
# .gitlab-ci.yml - 企业级CI/CD流水线
# 参考：字节跳动 Aegle CI 最佳实践

# ===== 全局变量 =====
variables:
  # 镜像仓库
  HARBOR_URL: "https://harbor.example.com"
  HARBOR_USER: "${HARBOR_USER}"
  HARBOR_PASSWORD: "${HARBOR_PASSWORD}"
  HARBOR_PROJECT: "production"
  DOCKER_REGISTRY: "${HARBOR_URL}"
  IMAGE_NAME: "${HARBOR_URL}/${HARBOR_PROJECT}/${CI_PROJECT_NAME}"
  
  # K8s 部署
  KUBE_NAMESPACE_DEV: "dev"
  KUBE_NAMESPACE_STAGING: "staging"
  KUBE_NAMESPACE_PROD: "production"
  ARGOCD_SERVER: "argocd.example.com"
  
  # 安全扫描
  TRIVY_SERVER: "http://trivy-server:8080"
  
  # 缓存
  PIP_CACHE_DIR: "$CI_PROJECT_DIR/.cache/pip"
  NPM_CACHE_DIR: "$CI_PROJECT_DIR/.cache/npm"

# ===== 阶段定义 =====
stages:
  - setup
  - build
  - test
  - security-scan
  - image-build
  - image-push
  - deploy-dev
  - integration-test
  - deploy-staging
  - performance-test
  - approval
  - deploy-prod
  - post-deploy

# ===== 全局缓存 =====
cache:
  key: "${CI_COMMIT_REF_SLUG}"
  paths:
    - .cache/
    - node_modules/
    - vendor/

# ===== 模板定义 =====
.deploy_template: &deploy_template
  image: bitnami/kubectl:1.28.0
  before_script:
    - kubectl config use-context ${KUBE_CONTEXT}
    - kubectl config set-context --current --namespace=${KUBE_NAMESPACE}
  script:
    - echo "部署版本: ${IMAGE_TAG}"
    - kubectl set image deployment/${DEPLOY_NAME}
      ${DEPLOY_NAME}=${IMAGE_NAME}:${IMAGE_TAG}
      -n ${KUBE_NAMESPACE}
    - kubectl rollout status deployment/${DEPLOY_NAME}
      -n ${KUBE_NAMESPACE} --timeout=300s

.scan_template: &scan_template
  image: aquasec/trivy:0.49.1
  before_script:
    - trivy image --download-db-only
  script:
    - trivy image --exit-code 1 --severity HIGH,CRITICAL
      --ignore-unfixed --format json
      -o trivy-report.json ${IMAGE_NAME}:${IMAGE_TAG}
    - trivy image --exit-code 1 --severity HIGH,CRITICAL
      --ignore-unfixed ${IMAGE_NAME}:${IMAGE_TAG}

# ===== Stage 1: 环境检查 =====
setup:env-check:
  stage: setup
  image: alpine:3.19
  script:
    - echo "=== 环境检查 ==="
    - echo "分支: ${CI_COMMIT_BRANCH}"
    - echo "提交: ${CI_COMMIT_SHORT_SHA}"
    - echo "触发者: ${GITLAB_USER_LOGIN}"
    - echo "项目: ${CI_PROJECT_PATH}"
    - echo "Pipeline ID: ${CI_PIPELINE_ID}"
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_COMMIT_BRANCH == "develop"'

# ===== Stage 2: 构建 =====
build:app:
  stage: build
  image: node:20-alpine
  script:
    - echo "=== 安装依赖 ==="
    - npm ci --cache .cache/npm --prefer-offline
    - echo "=== 构建应用 ==="
    - npm run build
    - echo "=== 构建完成 ==="
  artifacts:
    paths:
      - dist/
      - build/
    expire_in: 1 hour
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_COMMIT_BRANCH == "develop"'
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'

# ===== Stage 3: 测试 =====
test:unit:
  stage: test
  image: node:20-alpine
  services:
    - name: redis:7-alpine
      alias: redis
    - name: postgres:15-alpine
      alias: postgres
  variables:
    REDIS_URL: "redis://redis:6379"
    DATABASE_URL: "postgresql://postgres:postgres@postgres:5432/test"
    NODE_ENV: "test"
  script:
    - npm ci --cache .cache/npm --prefer-offline
    - npm run test:unit -- --coverage
    - npm run test:e2e
  artifacts:
    paths:
      - coverage/
      - test-results/
    reports:
      junit: test-results/junit.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura.xml
    expire_in: 7 days
  coverage: '/Lines\s*:\s*(\d+\.\d+%)/'
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_COMMIT_BRANCH == "develop"'
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'

test:lint:
  stage: test
  image: node:20-alpine
  script:
    - npm ci --cache .cache/npm --prefer-offline
    - npm run lint
    - npm run type-check
  allow_failure: true
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_COMMIT_BRANCH == "develop"'
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'

# ===== Stage 4: 安全扫描 =====
security:trivy-image:
  stage: security-scan
  <<: *scan_template
  variables:
    IMAGE_TAG: "${CI_COMMIT_SHORT_SHA}"
  artifacts:
    paths:
      - trivy-report.json
    reports:
      container_scanning: trivy-report.json
    expire_in: 30 days
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_COMMIT_BRANCH == "develop"'

security:code-scan:
  stage: security-scan
  image: aquasec/trivy:0.49.1
  script:
    - trivy fs --exit-code 1 --severity HIGH,CRITICAL
      --scanners vuln,misconfig,secret .
    - echo "代码安全扫描通过"
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_COMMIT_BRANCH == "develop"'

# ===== Stage 5: 构建镜像 =====
image:build:
  stage: image-build
  image: docker:24
  services:
    - docker:24-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
    IMAGE_TAG: "${CI_COMMIT_SHORT_SHA}"
  before_script:
    - docker login -u ${HARBOR_USER} -p ${HARBOR_PASSWORD} ${HARBOR_URL}
  script:
    - echo "=== 构建 Docker 镜像 ==="
    - docker build
      --build-arg BUILD_DATE="$(date -u +%Y-%m-%dT%H:%M:%SZ)"
      --build-arg VCS_REF="${CI_COMMIT_SHORT_SHA}"
      --cache-from ${IMAGE_NAME}:latest
      --tag ${IMAGE_NAME}:${IMAGE_TAG}
      --tag ${IMAGE_NAME}:latest
      --label "git.commit=${CI_COMMIT_SHA}"
      --label "git.branch=${CI_COMMIT_BRANCH}"
      --label "git.pipeline=${CI_PIPELINE_ID}"
      .
    - echo "=== 推送镜像 ==="
    - docker push ${IMAGE_NAME}:${IMAGE_TAG}
    - docker push ${IMAGE_NAME}:latest
    - echo "=== 镜像推送完成 ==="
    - docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_COMMIT_BRANCH == "develop"'
      when: on_success

# ===== Stage 6: 部署到 Dev =====
deploy:dev:
  stage: deploy-dev
  <<: *deploy_template
  variables:
    KUBE_CONTEXT: "dev-cluster"
    KUBE_NAMESPACE: "${KUBE_NAMESPACE_DEV}"
    DEPLOY_NAME: "${CI_PROJECT_NAME}"
    IMAGE_TAG: "${CI_COMMIT_SHORT_SHA}"
  environment:
    name: development
    url: https://dev.${CI_PROJECT_NAME}.example.com
    on_stop: stop:dev
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'

# ===== Stage 7: 集成测试 =====
test:integration:
  stage: integration-test
  image: curlimages/curl:8.5.0
  variables:
    TARGET_URL: "https://dev.${CI_PROJECT_NAME}.example.com"
  script:
    - |
      echo "=== 集成测试 ==="
      # 健康检查
      for i in $(seq 1 30); do
        STATUS=$(curl -s -o /dev/null -w "%{http_code}" ${TARGET_URL}/health)
        if [ "$STATUS" = "200" ]; then
          echo "健康检查通过"
          break
        fi
        echo "等待服务启动... (${i}/30)"
        sleep 10
      done
      if [ "$STATUS" != "200" ]; then
        echo "健康检查失败"
        exit 1
      fi
      # API 测试
      curl -sf ${TARGET_URL}/api/v1/status | jq .
    - echo "=== 集成测试完成 ==="
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'

# ===== Stage 8: 部署到 Staging =====
deploy:staging:
  stage: deploy-staging
  <<: *deploy_template
  variables:
    KUBE_CONTEXT: "staging-cluster"
    KUBE_NAMESPACE: "${KUBE_NAMESPACE_STAGING}"
    DEPLOY_NAME: "${CI_PROJECT_NAME}"
    IMAGE_TAG: "${CI_COMMIT_SHORT_SHA}"
  environment:
    name: staging
    url: https://staging.${CI_PROJECT_NAME}.example.com
    on_stop: stop:staging
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

# ===== Stage 9: 性能测试 =====
test:performance:
  stage: performance-test
  image: grafana/k6:0.49.0
  variables:
    TARGET_URL: "https://staging.${CI_PROJECT_NAME}.example.com"
  script:
    - |
      cat > /tmp/perf-test.js << 'TESTEOF'
      import http from 'k6/http';
      import { check, sleep } from 'k6';

      export let options = {
        stages: [
          { duration: '2m', target: 100 },   // 预热
          { duration: '5m', target: 100 },   // 基准负载
          { duration: '2m', target: 200 },   // 峰值
          { duration: '5m', target: 200 },   // 峰值持续
          { duration: '2m', target: 0 },     // 缓降
        ],
        thresholds: {
          http_req_duration: ['p(95)<500'],  // P95 < 500ms
          http_req_failed: ['rate<0.01'],    // 错误率 < 1%
        },
      };

      export default function () {
        let res = http.get(`${__ENV.TARGET_URL}/api/v1/status`);
        check(res, {
          'status is 200': (r) => r.status === 200,
          'response time < 500ms': (r) => r.timings.duration < 500,
        });
        sleep(1);
      }
      TESTEOF
    - k6 run /tmp/perf-test.js
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

# ===== Stage 10: 审批 =====
approval:deploy-prod:
  stage: approval
  script:
    - echo "=== 等待部署审批 ==="
    - echo "版本: ${IMAGE_TAG}"
    - echo "请在 GitLab UI 中确认部署"
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
      allow_failure: false

# ===== Stage 11: 部署到生产 =====
deploy:production:
  stage: deploy-prod
  <<: *deploy_template
  variables:
    KUBE_CONTEXT: "prod-cluster"
    KUBE_NAMESPACE: "${KUBE_NAMESPACE_PROD}"
    DEPLOY_NAME: "${CI_PROJECT_NAME}"
    IMAGE_TAG: "${CI_COMMIT_SHORT_SHA}"
  environment:
    name: production
    url: https://${CI_PROJECT_NAME}.example.com
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: on_success

# ===== Stage 12: 部署后验证 =====
post:verify:
  stage: post-deploy
  image: curlimages/curl:8.5.0
  script:
    - |
      echo "=== 生产环境验证 ==="
      STATUS=$(curl -s -o /dev/null -w "%{http_code}" https://${CI_PROJECT_NAME}.example.com/health)
      if [ "$STATUS" = "200" ]; then
        echo "✅ 生产环境健康检查通过"
      else
        echo "❌ 生产环境健康检查失败 (HTTP ${STATUS})"
        echo "执行回滚..."
        # 自动回滚到上一个版本
        exit 1
      fi
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

# ===== 清理环境 =====
stop:dev:
  stage: deploy-dev
  image: bitnami/kubectl:1.28.0
  variables:
    KUBE_CONTEXT: "dev-cluster"
    KUBE_NAMESPACE: "${KUBE_NAMESPACE_DEV}"
  script:
    - kubectl delete deployment ${CI_PROJECT_NAME} -n ${KUBE_NAMESPACE} --ignore-not-found
  when: manual
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'
```

---

## 四、ArgoCD GitOps 配置

### 4.1 ArgoCD Application 配置

```yaml
# argocd-app.yaml - ArgoCD Application 定义
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ${CI_PROJECT_NAME}-${ENVIRONMENT}
  namespace: argocd
  labels:
    app.kubernetes.io/name: ${CI_PROJECT_NAME}
    app.kubernetes.io/environment: ${ENVIRONMENT}
    app.kubernetes.io/managed-by: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: ${GIT_REPO_URL}
    targetRevision: ${GIT_BRANCH}
    path: k8s/${ENVIRONMENT}
    helm:
      valueFiles:
        - values.yaml
        - values-${ENVIRONMENT}.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: ${KUBE_NAMESPACE}
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
      - ApplyOutOfSyncOnly=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
```

### 4.2 ApplicationSet 多环境配置

```yaml
# appset.yaml - ApplicationSet 多环境自动化
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: ${CI_PROJECT_NAME}
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - env: dev
            namespace: dev
            server: https://kubernetes-dev.example.com
            revision: develop
          - env: staging
            namespace: staging
            server: https://kubernetes-staging.example.com
            revision: main
          - env: production
            namespace: production
            server: https://kubernetes-prod.example.com
            revision: main
  template:
    metadata:
      name: '${CI_PROJECT_NAME}-{{env}}'
      namespace: argocd
      labels:
        app.kubernetes.io/name: ${CI_PROJECT_NAME}
        app.kubernetes.io/environment: '{{env}}'
      finalizers:
        - resources-finalizer.argocd.argoproj.io
    spec:
      project: default
      source:
        repoURL: ${GIT_REPO_URL}
        targetRevision: '{{revision}}'
        path: 'k8s/{{env}}'
        helm:
          valueFiles:
            - values.yaml
            - 'values-{{env}}.yaml'
      destination:
        server: '{{server}}'
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

### 4.3 ArgoCD RBAC 配置

```yaml
# argocd-rbac-cm.yaml - ArgoCD RBAC 权限控制
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.default: role:readonly
  policy.csv: |
    # 开发者角色 - 只能查看和同步 dev 环境
    p, role:developer, applications, get, */*, allow
    p, role:developer, applications, sync, dev/*, allow
    p, role:developer, applications, action/*, dev/*, allow
    p, role:developer, logs, get, */*, allow
    p, role:developer, exec, create, dev/*, allow

    # 运维角色 - 管理所有环境
    p, role:operator, applications, *, */*, allow
    p, role:operator, repositories, *, *, allow
    p, role:operator, clusters, *, *, allow
    p, role:operator, logs, get, */*, allow
    p, role:operator, exec, create, */*, allow
    p, role:operator, projects, *, */*, allow

    # 只读角色
    p, role:viewer, applications, get, */*, allow
    p, role:viewer, logs, get, */*, allow

    # 用户绑定
    g, developer-team, role:developer
    g, ops-team, role:operator
    g, qa-team, role:viewer
```

### 4.4 ArgoCD 通知配置

```yaml
# argocd-notifications-cm.yaml - 部署通知
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  # 触发器
  trigger.on-sync-succeeded: |
    - when: app.status.operationState.phase in ['Succeeded']
      send:
        - app-sync-succeeded
  trigger.on-sync-failed: |
    - when: app.status.operationState.phase in ['Error', 'Failed']
      send:
        - app-sync-failed
  trigger.on-health-degraded: |
    - when: app.status.health.status == 'Degraded'
      send:
        - app-health-degraded

  # 通知模板
  template.app-sync-succeeded: |
    message: |
      ✅ 应用 {{.app.metadata.name}} 同步成功
      环境: {{.app.spec.destination.namespace}}
      版本: {{.app.status.sync.revision}}
      时间: {{.app.status.operationState.finishedAt}}
  template.app-sync-failed: |
    message: |
      ❌ 应用 {{.app.metadata.name}} 同步失败
      环境: {{.app.spec.destination.namespace}}
      错误: {{.app.status.operationState.message}}
  template.app-health-degraded: |
    message: |
      ⚠️ 应用 {{.app.metadata.name}} 健康状态降级
      当前状态: {{.app.status.health.status}}

  # 服务（钉钉）
  service.webhook.dingtalk: |
    url: https://oapi.dingtalk.com/robot/send?access_token=${DINGTALK_TOKEN}
    headers:
      - name: Content-Type
        value: application/json

  # 订阅
  subscriptions: |
    - appRegexp: .*
      triggers:
        - on-sync-succeeded
        - on-sync-failed
        - on-health-degraded
      destinations:
        - service: webhook.dingtalk
```

---

## 五、Kubernetes 部署清单

### 5.1 Deployment 配置

```yaml
# k8s/base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${APP_NAME}
  namespace: ${NAMESPACE}
  labels:
    app.kubernetes.io/name: ${APP_NAME}
    app.kubernetes.io/version: "${IMAGE_TAG}"
    app.kubernetes.io/managed-by: argocd
    app.kubernetes.io/part-of: ${APP_NAME}
spec:
  replicas: 3
  revisionHistoryLimit: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app.kubernetes.io/name: ${APP_NAME}
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ${APP_NAME}
        app.kubernetes.io/version: "${IMAGE_TAG}"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: ${APP_NAME}
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      terminationGracePeriodSeconds: 30
      containers:
        - name: ${APP_NAME}
          image: "${IMAGE_NAME}:${IMAGE_TAG}"
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
            - name: metrics
              containerPort: 9090
              protocol: TCP
          env:
            - name: APP_ENV
              value: "${ENVIRONMENT}"
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: ${APP_NAME}-secrets
                  key: db-password
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          livenessProbe:
            httpGet:
              path: /health/live
              port: http
            initialDelaySeconds: 15
            periodSeconds: 20
            timeoutSeconds: 5
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /health/ready
              port: http
            initialDelaySeconds: 5
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3
          startupProbe:
            httpGet:
              path: /health/startup
              port: http
            initialDelaySeconds: 10
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 30
          volumeMounts:
            - name: config
              mountPath: /etc/app/config
              readOnly: true
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: config
          configMap:
            name: ${APP_NAME}-config
        - name: tmp
          emptyDir:
            sizeLimit: 100Mi
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: ${APP_NAME}
```

### 5.2 HPA 自动扩缩

```yaml
# k8s/base/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ${APP_NAME}
  namespace: ${NAMESPACE}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ${APP_NAME}
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 4
          periodSeconds: 60
        - type: Percent
          value: 100
          periodSeconds: 60
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 2
          periodSeconds: 120
      selectPolicy: Min
```

### 5.3 PodDisruptionBudget

```yaml
# k8s/base/pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: ${APP_NAME}
  namespace: ${NAMESPACE}
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: ${APP_NAME}
```

### 5.4 Ingress 配置

```yaml
# k8s/base/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ${APP_NAME}
  namespace: ${NAMESPACE}
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Frame-Options: DENY";
      more_set_headers "X-Content-Type-Options: nosniff";
      more_set_headers "X-XSS-Protection: 1; mode=block";
      more_set_headers "Referrer-Policy: strict-origin-when-cross-origin";
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - ${APP_NAME}.example.com
      secretName: ${APP_NAME}-tls
  rules:
    - host: ${APP_NAME}.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: ${APP_NAME}
                port:
                  number: 80
```

### 5.5 NetworkPolicy

```yaml
# k8s/base/networkpolicy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ${APP_NAME}
  namespace: ${NAMESPACE}
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: ${APP_NAME}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - protocol: TCP
          port: 8080
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: prometheus
      ports:
        - protocol: TCP
          port: 9090
  egress:
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: TCP
          port: 443
        - protocol: TCP
          port: 80
    - to:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: postgresql
      ports:
        - protocol: TCP
          port: 5432
    - to:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: redis
      ports:
        - protocol: TCP
          port: 6379
```

---

## 六、自动化脚本

### 6.1 一键回滚脚本

```bash
#!/bin/bash
# rollback.sh - 一键回滚到上一个版本
# 用法: ./rollback.sh <namespace> <deployment> [revision]

set -euo pipefail

NAMESPACE=${1:?用法: $0 <namespace> <deployment> [revision]}
DEPLOYMENT=${2:?用法: $0 <namespace> <deployment> [revision]}
REVISION=${3:-""}

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

log() { echo -e "${GREEN}[$(date '+%H:%M:%S')]${NC} $1"; }
warn() { echo -e "${YELLOW}[$(date '+%H:%M:%S')] ⚠️ $1${NC}"; }
error() { echo -e "${RED}[$(date '+%H:%M:%S')] ❌ $1${NC}"; exit 1; }

# 获取当前版本
CURRENT_IMAGE=$(kubectl get deployment ${DEPLOYMENT} -n ${NAMESPACE} \
  -o jsonpath='{.spec.template.spec.containers[0].image}')
log "当前镜像: ${CURRENT_IMAGE}"

# 获取历史版本
if [ -z "${REVISION}" ]; then
  log "获取部署历史..."
  REVISIONS=$(kubectl rollout history deployment/${DEPLOYMENT} -n ${NAMESPACE} | grep -v "^REVISION" | tail -n +2)
  echo "${REVISIONS}"
  
  # 自动选择上一个版本
  REVISION=$(kubectl rollout history deployment/${DEPLOYMENT} -n ${NAMESPACE} \
    | grep -v "^REVISION" | tail -n +2 | head -n 1 | awk '{print $1}')
  warn "自动选择回滚版本: ${REVISION}"
fi

# 确认回滚
read -p "确认回滚到版本 ${REVISION}? (y/N): " CONFIRM
if [[ ! "${CONFIRM}" =~ ^[Yy]$ ]]; then
  log "已取消"
  exit 0
fi

# 执行回滚
log "开始回滚..."
kubectl rollout undo deployment/${DEPLOYMENT} -n ${NAMESPACE} --to-revision=${REVISION}

# 等待回滚完成
log "等待回滚完成..."
kubectl rollout status deployment/${DEPLOYMENT} -n ${NAMESPACE} --timeout=300s

# 验证
NEW_IMAGE=$(kubectl get deployment ${DEPLOYMENT} -n ${NAMESPACE} \
  -o jsonpath='{.spec.template.spec.containers[0].image}')
log "回滚完成！"
log "新镜像: ${NEW_IMAGE}"

# 检查 Pod 状态
log "检查 Pod 状态..."
kubectl get pods -n ${NAMESPACE} -l app.kubernetes.io/name=${DEPLOYMENT} -o wide
```

### 6.2 环境同步脚本

```bash
#!/bin/bash
# sync-env.sh - 将配置从 staging 同步到 production
# 用法: ./sync-env.sh <configmap|secret> <name> <source-ns> <target-ns>

set -euo pipefail

RESOURCE_TYPE=${1:?用法: $0 <configmap|secret> <name> <source-ns> <target-ns>}
RESOURCE_NAME=${2:?}
SOURCE_NS=${3:?}
TARGET_NS=${4:?}

RED='\033[0;31m'
GREEN='\033[0;32m'
NC='\033[0m'

log() { echo -e "${GREEN}[$(date '+%H:%M:%S')]${NC} $1"; }
error() { echo -e "${RED}[$(date '+%H:%M:%S')] ❌ $1${NC}"; exit 1; }

# 导出资源
log "从 ${SOURCE_NS} 导出 ${RESOURCE_TYPE}/${RESOURCE_NAME}..."
kubectl get ${RESOURCE_TYPE} ${RESOURCE_NAME} -n ${SOURCE_NS} -o yaml \
  > /tmp/sync-${RESOURCE_NAME}.yaml

# 修改 metadata
sed -i "/namespace: ${SOURCE_NS}/c\\  namespace: ${TARGET_NS}" /tmp/sync-${RESOURCE_NAME}.yaml
sed -i '/resourceVersion:/d' /tmp/sync-${RESOURCE_NAME}.yaml
sed -i '/uid:/d' /tmp/sync-${RESOURCE_NAME}.yaml
sed -i '/creationTimestamp:/d' /tmp/sync-${RESOURCE_NAME}.yaml

# 检查目标是否存在
if kubectl get ${RESOURCE_TYPE} ${RESOURCE_NAME} -n ${TARGET_NS} &>/dev/null; then
  log "目标资源已存在，执行更新..."
  kubectl apply -f /tmp/sync-${RESOURCE_NAME}.yaml -n ${TARGET_NS}
else
  log "目标资源不存在，创建新资源..."
  kubectl create -f /tmp/sync-${RESOURCE_NAME}.yaml -n ${TARGET_NS}
fi

# 清理
rm -f /tmp/sync-${RESOURCE_NAME}.yaml

log "同步完成！"
```

---

## 七、监控与告警

### 7.1 CI/CD 流水线监控

```yaml
# ci-cd-metrics.yaml - CI/CD 指标采集配置
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: gitlab-runner
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: gitlab-runner
  podMetricsEndpoints:
    - port: metrics
      path: /metrics

---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: ci-cd-alerts
  namespace: monitoring
spec:
  groups:
    - name: cicd
      rules:
        # 流水线失败率
        - alert: PipelineFailureRateHigh
          expr: |
            sum(rate(pipeline_failed_total[1h]))
            / sum(rate(pipeline_total[1h])) > 0.1
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "流水线失败率超过 10%"
            description: "当前失败率: {{ $value | humanizePercentage }}"

        # 构建时间过长
        - alert: BuildDurationTooLong
          expr: |
            histogram_quantile(0.95, sum(rate(build_duration_seconds_bucket[1h])) by (le)) > 600
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "构建时间 P95 超过 10 分钟"

        # 部署失败
        - alert: DeploymentFailed
          expr: |
            sum(rate(deployment_failed_total[1h])) > 0
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "部署失败"
            description: "项目: {{ $labels.project }}, 环境: {{ $labels.env }}"
```

---

## 八、安全加固

### 8.1 镜像签名配置

```bash
#!/bin/bash
# setup-cosign.sh - 安装和配置 Cosign 镜像签名

set -euo pipefail

echo "=== 安装 Cosign ==="
# 安装 Cosign
go install github.com/sigstore/cosign/v2/cmd/cosign@latest

# 生成密钥对
cosign generate-key-pair

# 设置环境变量
export COSIGN_PASSWORD="your-password"

# 签名镜像
cosign sign --key cosign.key ${IMAGE_NAME}:${IMAGE_TAG}

# 验证签名
cosign verify --key cosign.pub ${IMAGE_NAME}:${IMAGE_TAG}

echo "=== Cosign 配置完成 ==="
```

### 8.2 SBOM 生成

```bash
#!/bin/bash
# generate-sbom.sh - 生成软件物料清单

set -euo pipefail

IMAGE_NAME=${1:?用法: $0 <image:tag>}

echo "=== 生成 SBOM ==="

# 使用 Syft 生成 SBOM
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  anchore/syft:latest \
  ${IMAGE_NAME} -o spdx-json > sbom.json

# 使用 Grype 扫描漏洞
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  anchore/grype:latest \
  ${IMAGE_NAME} --fail-on high

# 附加 SBOM 到镜像
cosign attach sbom --sbom sbom.json ${IMAGE_NAME}

echo "=== SBOM 生成完成 ==="
```

### 8.3 RBAC 最佳实践

```yaml
# rbac.yaml - GitLab CI/CD RBAC 权限模型
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gitlab-deployer
  namespace: production
  labels:
    app: gitlab-deployer

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: gitlab-deployer
  namespace: production
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "update", "patch"]
  - apiGroups: [""]
    resources: ["services", "configmaps", "secrets"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: ["autoscaling"]
    resources: ["horizontalpodautoscalers"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: ["policy"]
    resources: ["poddisruptionbudgets"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: gitlab-deployer
  namespace: production
subjects:
  - kind: ServiceAccount
    name: gitlab-deployer
    namespace: production
roleRef:
  kind: Role
  name: gitlab-deployer
  apiGroup: rbac.authorization.k8s.io
```

---

## 九、运维手册

### 9.1 日常操作清单

```bash
#!/bin/bash
# daily-check.sh - CI/CD 平台日常巡检

set -euo pipefail

echo "=== CI/CD 平台日常巡检 ==="
echo "时间: $(date '+%Y-%m-%d %H:%M:%S')"

# 1. 检查 Harbor 状态
echo -e "\n--- Harbor 状态 ---"
curl -sk https://harbor.example.com/api/v2.0/systeminfo | jq .

# 2. 检查 GitLab Runner 状态
echo -e "\n--- GitLab Runner 状态 ---"
gitlab-runner list

# 3. 检查 ArgoCD 状态
echo -e "\n--- ArgoCD 状态 ---"
argocd app list -o wide

# 4. 检查 K8s 部署状态
echo -e "\n--- 部署状态 ---"
kubectl get deployments -A | grep -v "kube-system"

# 5. 检查证书有效期
echo -e "\n--- 证书有效期 ---"
echo | openssl s_client -connect harbor.example.com:443 2>/dev/null \
  | openssl x509 -noout -dates

echo "=== 巡检完成 ==="
```

### 9.2 故障排查手册

```bash
#!/bin/bash
# troubleshoot.sh - CI/CD 故障排查工具

set -euo pipefail

echo "=== CI/CD 故障排查 ==="

# 检查 GitLab Runner 日志
check_gitlab_runner() {
  echo "--- GitLab Runner 日志 ---"
  systemctl status gitlab-runner --no-pager -l
  journalctl -u gitlab-runner --since "1 hour ago" --no-pager
}

# 检查 ArgoCD 状态
check_argocd() {
  echo "--- ArgoCD 状态 ---"
  kubectl get pods -n argocd
  kubectl logs -n argocd -l app.kubernetes.io/name=argocd-server --tail=50
}

# 检查 Harbor 状态
check_harbor() {
  echo "--- Harbor 状态 ---"
  docker ps | grep harbor
  docker logs harbor-core --tail=50
}

# 检查 K8s 集群
check_k8s() {
  echo "--- K8s 集群状态 ---"
  kubectl get nodes
  kubectl top nodes
  kubectl get events --sort-by='.lastTimestamp' | tail -20
}

# 主菜单
echo "选择排查项:"
echo "1. GitLab Runner"
echo "2. ArgoCD"
echo "3. Harbor"
echo "4. K8s 集群"
echo "5. 全部检查"

read -p "选择: " choice
case $choice in
  1) check_gitlab_runner ;;
  2) check_argocd ;;
  3) check_harbor ;;
  4) check_k8s ;;
  5) check_gitlab_runner; check_argocd; check_harbor; check_k8s ;;
  *) echo "无效选择" ;;
esac
```

---

## 十、面试题与思考

### Q1: GitLab CI 和 ArgoCD 如何配合实现 GitOps？

**答：** 整个流程分为两个阶段：

1. **CI 阶段（GitLab CI）**：代码提交后触发流水线，完成构建、测试、镜像构建推送到 Harbor。关键是镜像标签使用 Git commit SHA，保证可追溯。

2. **CD 阶段（ArgoCD）**：ArgoCD 监听 Git 仓库中的 Kubernetes 清单文件（Kustomize/Helm）。CI 流水线完成后，自动更新 Git 仓库中的镜像版本（通过 CI 脚本修改 YAML 或 Kustomize overlay），ArgoCD 检测到变化后自动同步到 K8s 集群。

**核心理念**：Git 是唯一事实来源（Single Source of Truth），所有变更通过 Git 提交，ArgoCD 负责将 Git 中的期望状态同步到集群实际状态。

### Q2: 如何实现蓝绿部署和金丝雀发布？

**答：**

**蓝绿部署**：维护两套完整环境，通过 Service selector 切流量。新版本部署到"绿"环境，测试通过后切换 Service 指向"绿"，"蓝"作为回滚备用。

**金丝雀发布**：使用 Istio VirtualService 按权重分配流量。先将 5% 流量导入新版本，监控指标无异常后逐步增加比例。

```yaml
# Istio 金丝雀示例
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
spec:
  http:
    - route:
        - destination:
            host: app
            subset: stable
          weight: 95
        - destination:
            host: app
            subset: canary
          weight: 5
```

### Q3: 如何保证 CI/CD 流水线的安全性？

**答：** 从五个层面防护：

1. **代码安全**：代码扫描（SonarQube）、依赖漏洞扫描（Trivy/Snyk）
2. **镜像安全**：构建时扫描（Trivy）、签名（Cosign）、SBOM 生成
3. **运行时安全**：PodSecurityPolicy/Standards、NetworkPolicy、只读根文件系统
4. **凭证安全**：Vault 管理密钥、GitLab CI/CD Variables 加密、最小权限原则
5. **审计追溯**：所有操作通过 Git 提交、ArgoCD 操作日志、K8s 审计日志

### Q4: ArgoCD 的 Self-Heal 机制是什么？

**答：** Self-Heal（自愈）是 ArgoCD 的核心能力。当集群中的实际状态与 Git 中的期望状态不一致时（比如有人手动修改了 Deployment 的副本数），ArgoCD 会自动检测到这种偏差（OutOfSync），并自动将集群状态恢复到 Git 中定义的期望状态。

这保证了：
- **防止配置漂移**：任何人手动修改都会被自动恢复
- **声明式管理**：Git 中的配置是唯一事实来源
- **故障自愈**：意外变更会被自动纠正

### Q5: 多集群 CI/CD 如何实现？

**答：** 使用 ArgoCD 的多集群管理能力：

1. **注册多个集群**：将 dev/staging/prod 集群注册到 ArgoCD
2. **ApplicationSet**：使用 List/Matrix/Git generator 自动生成多环境 Application
3. **项目隔离**：通过 ArgoCD Project 隔离不同团队/业务
4. **权限控制**：RBAC 控制不同团队只能操作对应环境
5. **同步策略**：dev 自动同步，staging 半自动，prod 需要审批

---

## 附录

### A. 环境变量清单

| 变量名 | 说明 | 示例值 |
|--------|------|--------|
| CI_REGISTRY | 镜像仓库地址 | harbor.example.com |
| CI_REGISTRY_USER | 仓库用户名 | admin |
| CI_REGISTRY_PASSWORD | 仓库密码 | **** |
| CI_REGISTRY_IMAGE | 镜像完整路径 | harbor.example.com/prod/app |
| KUBE_CONFIG | K8s 配置 | base64 encoded |
| ARGOCD_TOKEN | ArgoCD API Token | **** |
| DINGTALK_TOKEN | 钉钉机器人 Token | **** |

### B. 端口规划

| 服务 | 端口 | 说明 |
|------|------|------|
| GitLab | 80/443 | Web 访问 |
| GitLab Runner | - | 内部通信 |
| Harbor | 443 | 镜像仓库 |
| ArgoCD | 443 | GitOps UI |
| ArgoCD API | 8080 | API 接口 |
| Trivy | 8080 | 漏洞扫描 |
| Prometheus | 9090 | 指标采集 |
| Grafana | 3000 | 监控看板 |

### C. 参考资料

- [ArgoCD 官方文档](https://argo-cd.readthedocs.io/)
- [GitLab CI/CD 文档](https://docs.gitlab.com/ee/ci/)
- [Harbor 官方文档](https://goharbor.io/docs/)
- [Google SRE Workbook](https://sre.google/workbook/table-of-contents/)
- [字节跳动 Aegle CI 最佳实践](https://tech.bytedance.com/)

---

*作者: 小白老师 | 更新时间: 2026-05-03 | 版本: v1.0*
