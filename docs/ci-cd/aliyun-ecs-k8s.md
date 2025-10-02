# 基于阿里云 ECS 的 ruoyi-cloud CI/CD 部署指南

本文档面向已经在阿里云 ECS 自建 Kubernetes 集群，并完成 Harbor、Jenkins、GitLab 安装的场景，说明如何借助本仓库新增的流水线脚本与 Kubernetes 清单，实现 ruoyi-cloud 项目的自动化构建、发布与运维。

## 1. 整体架构

```
GitLab (源代码)
   │  Webhook / Jenkins GitLab 插件
   ▼
Jenkins Pipeline (deploy/ci-cd/Jenkinsfile)
   │  mvn & npm 构建 / Docker Buildx / 镜像推送
   ▼
Harbor (ruoyi 项目镜像仓库)
   │  kubectl apply -k deploy/k8s/overlays/prod
   ▼
Kubernetes (ECS 节点)
   ├─ 基础设施：MySQL、Redis、Nacos、Sentinel
   └─ 微服务：Gateway、Auth、System、File、Gen、Job、Monitor、UI
```

## 2. 前置条件

1. **Kubernetes 集群**：
   - 建议基于至少 3 台 ECS 节点（1 主 2 从）安装 Kubernetes 1.24+，或直接使用 ACK。
   - 安装 `containerd`/`docker`、`kubectl`、`helm`（可选），配置 `calico`/`flannel` 网络插件。
2. **存储/负载均衡**：
   - 示例清单使用 `emptyDir`，生产环境请为 MySQL、Nacos、Redis 配置云盘或 NAS 的 PVC，并结合阿里云 SLB / Ingress Controller 暴露服务。
3. **GitLab**：
   - 创建 `ruoyi-cloud` 仓库，导入本项目代码。
   - 在 *Settings → Webhooks* 中配置 Jenkins 回调地址（`http://jenkins.example.com/project/ruoyi-cloud`）。
4. **Harbor**：
   - 创建名为 `ruoyi` 的项目，获取一个拥有推送权限的 Robot 账号或普通用户。
5. **Jenkins**：
   - 安装插件：GitLab、Pipeline、Kubernetes CLI、NodeJS、Blue Ocean（可选）。
   - Jenkins agent 需安装：`jdk11+`、`maven`、`node 16+`、`npm`、`docker`、`kubectl`、`kustomize`。
   - 配置全局工具：Maven、NodeJS（可选）。

## 3. Jenkins 凭据约定

在 Jenkins 「凭据」中新增以下条目，与 Jenkinsfile 保持一致：

| ID | 类型 | 说明 |
| --- | --- | --- |
| `harbor-credential-id` | Username/Password | Harbor 推送账号（Robot 或普通用户） |
| `kubeconfig-ruoyi` | Secret file | 具备 `ruoyi` 命名空间管理权限的 kubeconfig |
| `gitlab-api-token`（可选） | Secret text | 若需通过 API 触发 MR 状态回调 |

> 如需使用 GitLab webhook 触发，请在 Jenkins 新建「GitLab 项目」或「多分支流水线」并关联上述凭据。

## 4. 新增 Jenkins Pipeline

1. 在 Jenkins 创建 *Pipeline* 作业，`Pipeline script from SCM` 指向 GitLab 仓库，分支如 `master/main`。
2. Jenkinsfile 路径设为 `deploy/ci-cd/Jenkinsfile`。
3. 保存后即可手动或通过 GitLab Webhook 触发流水线。

### 4.1 流水线阶段说明

| 阶段 | 动作 |
| --- | --- |
| `Checkout` | 从 GitLab 拉取代码 |
| `Initialize` | 生成 `IMAGE_TAG`（`构建号-提交ID`）、记录构建时间 |
| `Maven Build` | 执行 `mvn -B clean package -DskipTests`，构建后端 Jar |
| `Frontend Build` | 在 `ruoyi-ui` 目录运行 `npm install`、`npm run build:prod` 构建前端静态资源 |
| `Prepare Docker Context` | 调用 `docker/copy.sh`，将 Jar / 前端静态文件拷贝到 Docker 构建上下文 |
| `Build & Push Images` | 登录 Harbor，构建并推送 8 个服务镜像（含 `latest` 和构建标签） |
| `Deploy to Kubernetes` | `kubectl apply -k deploy/k8s/overlays/prod` 部署基础设施+业务服务，并使用 `kubectl set image` 滚动更新容器镜像 |

流水线失败时 Jenkins 自动标记失败；成功后会执行 `docker image prune -f` 清理构建节点残余镜像层。

## 5. Kubernetes 清单使用说明

新增的 Kubernetes 清单位于 `deploy/k8s`：

- `base/`：包含命名空间、配置、基础服务（MySQL/Redis/Nacos/Sentinel）以及 ruoyi 各微服务的 Deployment/Service/Ingress。镜像默认指向 `harbor.example.com`，实际部署前需修改为企业 Harbor 地址。
- `overlays/prod/`：示例化的生产覆写，给 Ingress 规则加上生产域名标签，可按需扩展更多环境（如 `dev`、`staging`）。

### 5.1 初始化命名空间与镜像拉取密钥

```bash
# 创建命名空间（首次部署时执行）
kubectl create namespace ruoyi

# 将 Harbor Robot 账号创建为镜像拉取密钥
kubectl -n ruoyi create secret docker-registry ruoyi-harbor \
  --docker-server=harbor.example.com \
  --docker-username='robot$ruoyi' \
  --docker-password='xxxxxxxx' \
  --docker-email='devops@example.com'
```

### 5.2 同步配置与数据库

1. 将 `sql/` 目录中的 `ry_*.sql` 导入到 Kubernetes 内部或外部 MySQL 中：

```bash
kubectl -n ruoyi exec -it statefulset/mysql -- bash -c "mysql -uroot -p$MYSQL_ROOT_PASSWORD < /docker-entrypoint-initdb.d/ry_20250523.sql"
```

2. 登录 Nacos 控制台（`http://<任意节点>:8848/nacos`，默认用户/密码 `nacos/nacos`），导入 `sql/ry_config_20250224.sql` 中的配置，或按需手动创建 `application-prod.yml` 等配置文件。
3. 根据实际环境修改 `deploy/k8s/base/configmap-ruoyi-env.yaml` 中的：
   - `NACOS_ADDR`、`SENTINEL_ADDR`（如果使用外部组件）；
   - `MYSQL_*` 参数；
   - `JAVA_OPTS`（JVM 资源限制）。
4. **生产环境建议**：为 MySQL、Nacos、Redis 配置持久化卷（替换 `emptyDir`）、开启账号复杂度、并限制 Node 选择器 / 亲和性。

### 5.3 部署

```bash
# 首次部署全部资源
git pull
kubectl apply -k deploy/k8s/overlays/prod

# 查看 Pod 状态
kubectl -n ruoyi get pods

# 排查问题
kubectl -n ruoyi logs deployment/ruoyi-gateway
```

部署完成后，即可通过 Ingress 域名访问前端与网关（示例：`https://ruoyi.prod.example.com`）。

## 6. GitLab → Jenkins 自动触发

1. 在 GitLab 项目中配置 Webhook，URL 指向 Jenkins Job 的 `Build with Parameters` 接口（或使用 GitLab 插件生成的触发 URL），勾选 `Push events`、`Merge request events`。
2. Jenkins 端安装并配置 GitLab 插件，勾选 *Build when a change is pushed to GitLab*，并填写 GitLab 服务器地址与凭据。
3. 可在 Jenkinsfile 中追加测试/扫描阶段或根据分支决定是否部署，例如：

```groovy
when {
  expression { env.BRANCH_NAME == 'main' }
}
```

## 7. 常见问题与优化建议

- **镜像构建失败**：检查 Jenkins 节点是否具有 Docker 权限、Harbor 登录凭据是否正确。
- **拉取镜像超时**：请在各节点配置 `/etc/docker/daemon.json` 镜像加速（阿里云 Registry Mirror）。
- **Pod CrashLoopBackOff**：通过 `kubectl logs`、`kubectl describe` 排查，重点关注 Nacos、数据库连接是否正常。
- **资源限制**：根据业务压力调整 Deployment 中 `resources` 配置，并结合 HPA 自动扩缩容。
- **多环境隔离**：复制 `overlays/prod` 为 `overlays/dev`，修改镜像标签、ConfigMap、Ingress 域名，即可实现同一套基础清单的多集群复用。
- **安全**：
  - 使用 `SealedSecret` 或 `ExternalSecrets` 管理敏感信息；
  - 开启 Jenkins 凭据审计，定期轮转密码；
  - 在 Harbor 中启用镜像扫描和签名策略。

## 8. 手动回滚与灰度发布

- **镜像回滚**：使用 `kubectl rollout undo deployment/ruoyi-system -n ruoyi` 回滚到上一个 ReplicaSet。
- **灰度**：可在 Jenkinsfile 中新增阶段，通过修改 Deployment 的 `image` 标签、Replica 数实现 A/B 测试；或引入 Argo Rollouts、Flagger 等工具。

## 9. 后续扩展

- 集成 SonarQube、SAST、单元测试覆盖率报告，提升质量门禁；
- 接入 Prometheus + Grafana、EFK/ELK 构建监控与日志平台；
- 使用 Helm/Argo CD/Flux 实现 GitOps 模式的持续交付。

---

如需进一步定制（多机房、弹性伸缩、自动化运维脚本等），可在上述基础上扩展。
