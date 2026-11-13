# MCPHub Kubernetes 部署

当前业务使用 `mcphub-db` Deployment（两个副本）和独立 PostgreSQL 数据库
`mcphub`。公网 HTTPRoute 和 Service 名称仍为 `mcphub`。

服务器、用户、分组、Bearer Key、OAuth 数据和 Prompt 由数据库持久化。
控制台修改后无需同步 ConfigMap；`mcphub-runtime-config` 只存运行环境配置，
数据库连接、JWT 和凭据加密密钥保存在 `mcphub-runtime-secrets` Secret。
不要重新生成该 Secret。旧 PVC 保留作迁移前数据副本。

## 日常滚动更新

修改 `deployment.yaml` 中的镜像后执行：

```sh
kubectl -n mcphub apply -f deployment.yaml
kubectl -n mcphub rollout status deployment/mcphub-db --timeout=1200s
kubectl -n mcphub get pods -l app=mcphub-db
curl --fail https://mcphub.magikcloud.cn/readyz
```

镜像必须能从集群访问。滚动策略为 `maxUnavailable: 0`、`maxSurge: 1`；
启动探针等待数据库就绪且所有启用上游连接成功，新 Pod 稳定就绪 15 秒后
才退出旧 Pod。集群须能容纳第三个 Pod；若启动失败，旧副本继续接流。
退出前等待 15 秒供路由收敛，终止宽限期为 90 秒。

`backendtrafficpolicy.yaml` 将同一 Authorization 请求头的请求映射到同一副本。
这只提供路由亲和性，不承担认证。旧版 MCP 会话、SSE 和进行中的工具调用
不会迁移到新进程，客户端仍须支持重新初始化；不要自动重放结果不明的写操作。
滚动更新过程中健康探针持续成功不代表所有长连接完全无感。

有数据库结构变更的版本必须先核对其迁移与旧版本兼容性；
`RollingUpdate` 不能保证任意数据库升级都不停服。`setup-job.yaml` 是独立的
schema 准备模板，不应在每个服务 Pod 的 initContainer 中执行。
执行前备份数据库，确保只运行一个 setup Job，并确认适当的维护策略。
不要把整个目录作为日常更新入口，以免误运行 schema Job 或旧版 Deployment。

## 旧文件模式

`legacy-deployment.yaml` 保留旧文件模式部署，副本数为 0，初始化命令仅在配置文件
不存在时复制 ConfigMap，防止覆盖 PVC。不要直接扩容它来回滚：迁移后新数据
写入数据库，旧 PVC 不再同步。需要回滚时应单独评估版本和数据兼容性。

`configmap.yaml` 是历史初始化模板，不再是运行数据源；修改它不会更新数据库。
