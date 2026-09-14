# 部署到 VPS JP

目标形态：**纯 Docker 内网服务**，消费者是同一台机上的 `new-api`。
不发布 host port、不接 Cloudflare Tunnel。

- 机器：`35.200.50.7`，GCP Tokyo e2-small，**x86_64** → 官方 amd64 镜像直接可用，不需要自建
- 网络：`shared-net`（`new-api-prod` 已在其上）
- 部署目录约定：`/home/docker/kiro2claude/`

> **不要在 JP 上 build 镜像。** 1.9 GB 内存扛不住，直接 `docker pull` 官方 amd64 镜像。

---

## 一、为什么不开公网

这个网关**没有任何 login / admin HTTP 端点**。全部路由只有
`/health`、`/claude/v1/*`、`/openai/v1/*`、`/api/*`、`/kiro/usage`（只读透传）。

加 Kiro 账号靠 `docker exec` + 浏览器打开 AWS SSO 的 device flow URL，
那条 URL 指向 `device.sso.<region>.amazonaws.com`，**不指向本网关**。
所以公网暴露对加账号这件事零帮助，只是多一份攻击面。

---

## 二、首次部署

### 0. 准备目录与 key

```bash
ssh vps-jp
mkdir -p /home/docker/kiro2claude && cd /home/docker/kiro2claude
# 把本目录的 docker-compose.yml / .env.example 传上来
cp .env.example .env
openssl rand -hex 32   # 填进 .env 的 KIRO2CLAUDE_API_KEY
```

### 1. 先登录，再启动 —— 顺序不能反

没有凭据时 `loadCredentialsFromEnv()` 会抛错 → `process.exit(1)`，
配上 `restart: unless-stopped` 就是**无限重启循环**。所以第一次必须先用
一次性容器完成登录，把凭据写进命名卷，再 `up -d`。

```bash
docker compose run --rm --entrypoint /bin/sh kiro2claude -c '
  unset KIRO2CLAUDE_API_KEY
  script -qec "kiro-cli login --use-device-flow --license free" /dev/null
'
```

日志里会打出 device flow URL 和 user code，在任意一台机器的浏览器打开完成认证。

**三个必须照抄的细节：**

1. **`unset KIRO2CLAUDE_API_KEY`** —— kiro-cli 自己也读这个环境变量，把它当作
   「已经用 API key 认证过」的标志，进而**拒绝执行 login**。容器 env 里有它，
   `docker exec` / `docker compose run` 都会继承。
   （网关自己 spawn kiro-cli 时由 `src/kiro/subprocess-env.ts` 剥掉，手动跑时得自己来。）
2. **`script -qec ... /dev/null`** —— kiro-cli 必须有 PTY。非 TTY 环境下交互式
   prompt 直接返回空串，region 校验会爆 `invalid host label`。
3. **`--license free` 对应 Builder ID**；用 IAM Identity Center 的话改成
   `--license pro --identity-provider https://xxx.awsapps.com/start --region us-east-1`。

### 2. 激活 profile —— 不做的话 `/kiro/usage` 用不了

device flow **只写 token，不写** `state.api.codewhisperer.profile`。
而上游 `GetUsageLimits` 严格要求 `profileArn`，缺了就是 `400 Invalid profileArn`。

```bash
docker compose run --rm --entrypoint /bin/sh kiro2claude -c '
  unset KIRO2CLAUDE_API_KEY
  script -qec "kiro-cli profile" /dev/null
'
```

TUI 起来后按 **Enter** 接受高亮的默认 profile。

> 走 `KIRO2CLAUDE_LOGIN_START_URL` 自动 bootstrap 的话这一步是自动的
> （`runBootstrapLogin` 内部会调 `activateProfile`），但那条路只对
> IAM Identity Center 有效 —— Builder ID 必须手动跑这两步。

### 3. 启动

```bash
docker compose up -d
docker compose logs -f          # 应看到「启动自检完成：凭据就绪」
```

---

## 三、接入 new-api

在 new-api 后台加一条渠道：

| 字段 | 值 |
|---|---|
| 类型 | Anthropic (Claude) |
| 代理 / Base URL | `http://kiro2claude:8080/api/claude` |
| 密钥 | `.env` 里的 `KIRO2CLAUDE_API_KEY` |

new-api 会自己补 `/v1/messages`。

**用 `/api/claude` 而不是 `/claude/v1`**：`/api/*` 是「去泄漏」镜像端点，
会剥掉 `usage.kiro_metering` / `kiro_derived` 这些插件注入的 `kiro_*` 扩展字段
（它们带后端身份信息）。new-api 只读 `usage.input_tokens` / `output_tokens`，
不会储存未知字段，所以两条路对 new-api 的计费统计没有差别。

**模型 ID**（每个 claude 都有 `-thinking` 变体）：

```
claude-opus-5            claude-sonnet-5           claude-haiku-4-5-20251001
claude-opus-4-8          claude-sonnet-4-6         claude-opus-4-7
claude-opus-4-6          claude-opus-4-5-20251101  claude-sonnet-4-5-20250929
gpt-5.6-sol              gpt-5.6-terra             gpt-5.6-luna
```

想对外暴露标准名就在渠道里配 model mapping。

**一个容器 = 一个 Kiro 账号。** `SingleTokenManager` 只读单一
`KIRO2CLAUDE_SQLITE_DB_PATH`，没有账号池。将来加第二个账号 = 复制一份
compose（换 `container_name` / volume 名）+ 在 new-api 加第二条渠道，
靠 new-api 自己的渠道分组做负载。注意 `plugin-metering` 是进程内内存累计
（`.env.example` 写明「仅适用于单实例部署」），多开之后各算各的。

---

## 四、查额度

```bash
ssh vps-jp 'docker exec kiro2claude sh -c "
  curl -s -H \"x-api-key: \$KIRO2CLAUDE_API_KEY\" localhost:8080/kiro/usage"'
```

返回的是上游 `getUsageLimits` 的透传（`userInfo` 已被 `routes/kiro.ts` 剥掉，
因为 token-manager 写死了 `isEmailRequired=true`，不剥会连邮箱一起下发）。
关注这几个字段：

- `subscriptionInfo.subscriptionTitle` —— KIRO FREE / KIRO POWER
- `usageBreakdownList[0].currentUsageWithPrecision` / `usageLimitWithPrecision` —— 已用 / 上限（credits）
- `nextDateReset` —— 额度重置时间（unix 秒）
- `overageConfiguration.overageStatus` + `usageBreakdownList[0].overageRate` —— 超额开关与单价

---

## 五、升级 / 回滚

```bash
cd /home/docker/kiro2claude
# 改 docker-compose.yml 里的镜像 tag（别用 :latest，留可回滚的锚点）
docker compose pull && docker compose up -d
```

凭据在命名卷 `kiro2claude-home` 里，与镜像无关，升级不会丢登录态。
回滚把 tag 改回去再 `up -d` 即可 —— 这个服务没有数据库迁移。

---

## 六、本 fork 待打的补丁

按当前「纯内网」形态重排优先级：

- **要修**：`packages/core/src/index.ts:406` 把 API key 的**前一半**明文写进启动日志
  （`apiKey.slice(0, Math.floor(apiKey.length / 2))`），`docker logs` 就能看到。
- **顺手**：`fastify` 升到 `>=5.12.1`（清掉 `fast-uri` SSRF ×2、`find-my-way`
  HTTP/2 DDoS、fastify 自身 2 个 moderate，共 15 条 advisory，全是 transitive）。
- **顺手**：根 `package.json` 的 `engines.node` 从 `>=22` 收紧成 `>=22 <25`
  —— `better-sqlite3@11.10.0` 在 Node 26 上编译失败（v8 API 移除）。只影响本地开发，
  部署走镜像不受影响。
- 内网部署下 CORS `origin: true`、API key 无长度下限这两项影响不大，但换个强 key 是免费的。
