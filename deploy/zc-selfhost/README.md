# zc-selfhost: 自托管构建(基于本 fork 源码)

单容器自建镜像,与上游 `deploy/coolify` 同构,但安装的是**本 fork 构建出的 npm 包**,
包含我们的两个修复(见分支 `fix/large-store-deploy`):

1. `entrypoint.sh` CORS `allowed_origins` 流式单行(修复 v0.9.30 CLI `setManagedCorsOrigins`
   整行替换与块式模板冲突导致的非法 YAML / 引擎 crash loop)
2. `AGENTMEMORY_WORKER_READY_TIMEOUT_MS` 可配置就绪超时(默认仍 15000ms)

## 构建步骤

```bash
# 1. 本机(fork 根目录)构建 + 打包
npm install --legacy-peer-deps --no-fund --no-audit
npm run build
npm pack --pack-destination /tmp
mv /tmp/agentmemory-agentmemory-0.9.30.tgz /tmp/agentmemory-0.9.30-zc.tgz

# 2. 组装构建目录(服务器上)
scp /tmp/agentmemory-0.9.30-zc.tgz root@192.168.89.144:/root/am-build-zc/
scp deploy/coolify/entrypoint.sh root@192.168.89.144:/root/am-build-zc/
scp deploy/zc-selfhost/Dockerfile  root@192.168.89.144:/root/am-build-zc/

# 3. 服务器上构建 + 换容器(见 upgrade.sh)
```

## 与上游 coolify 模板的差异

- `npm install` 装本地 tgz(而非 registry 的 @agentmemory/agentmemory)
- `npm install` 加 `--legacy-peer-deps`(npm 10.9.4 arborist peer-set bug 规避)
- healthcheck `--start-period=900s`(大存储首次索引构建可达数分钟)
- 运行时必须带 `-e AGENTMEMORY_WORKER_READY_TIMEOUT_MS=600000`

## 升级前检查清单

1. diff 上游 `deploy/coolify/entrypoint.sh` —— 若上游已改流式,本地 entrypoint 补丁可退
2. grep 上游 `src/cli.ts` 的 `waitForAgentmemoryReady` —— 若已可配置,本 fork 可同步上游后退役
