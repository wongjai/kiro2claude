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

`.env` 里还要填 IdC 的 `KIRO2CLAUDE_LOGIN_START_URL`（`https://d-xxx.awsapps.com/start`）
和 `KIRO2CLAUDE_LOGIN_REGION`。这两个刻意放 `.env` 不放 compose —— start URL
含 IdC 目录 ID，而 compose 是要进 git 的。

### 1. 启动并完成 device flow（IAM Identity Center）

本账号走 IdC，所以 `.env` 里填了 `KIRO2CLAUDE_LOGIN_START_URL` 之后，登录是
**容器自动完成**的 —— `index.ts` 在加载凭据之前先跑 `runBootstrapLogin`，它会：

1. `kiro-cli whoami` 判断是否已登录（比任何文件大小启发式都可靠：全新 schema
   的空 DB 也有 28 KB）
2. 没登录就 spawn `kiro-cli login --use-device-flow --license pro
   --identity-provider <START_URL> --region <LOGIN_REGION>`，把 device flow URL
   实时转发到日志
3. **登录成功后自动跑 `kiro-cli profile`** 写入 profileArn

```bash
docker compose up -d
docker compose logs -f
```

日志里会打出 device flow URL 和 user code，在任意一台机器的浏览器打开完成认证。
成功后应看到「启动自检完成：凭据就绪」。

**盯着日志做这一步。** device flow 超时（默认 10 分钟）会让启动失败退出，
`restart: unless-stopped` 会重来一轮 —— 不是死循环，但会浪费一轮等待。

后续重启这条路径是幂等的：`whoami` 通过 → 只重跑一次 profile 激活（几秒）。
凭据被清掉时它还会自己重新 bootstrap，算是自愈。

### 2. profile 激活：IdC 自动，Builder ID 手动

device flow **只写 token，不写** `state.api.codewhisperer.profile`，而上游
`GetUsageLimits` 严格要求 `profileArn`，缺了就是 `400 Invalid profileArn` ——
`/kiro/usage` 直接不可用。走 IdC 自动 bootstrap 的话 `runBootstrapLogin` 已经
替你跑了，不用管。

**只有 Builder ID（`--license free`）用户需要手动两步**：`index.ts` 的 bootstrap
入口以 `config.loginStartUrl` 为开关，而 `bootstrap-login.ts` 一定会传
`--identity-provider`，所以 Builder ID 根本进不了这条路径：

```bash
# 仅 Builder ID 需要
docker compose run --rm --entrypoint /bin/sh kiro2claude -c '
  unset KIRO2CLAUDE_API_KEY
  script -qec "kiro-cli login --use-device-flow --license free" /dev/null
  script -qec "kiro-cli profile" /dev/null       # TUI 起来后按 Enter
'
```

两个必须照抄的细节（手动跑才会遇到）：

1. **`unset KIRO2CLAUDE_API_KEY`** —— kiro-cli 自己也读这个环境变量，把它当作
   「已经用 API key 认证过」的标志，进而**拒绝执行 login**。容器 env 里有它，
   `docker exec` / `docker compose run` 都会继承。
   （网关自己 spawn kiro-cli 时由 `src/kiro/subprocess-env.ts` 剥掉，手动跑时得自己来。）
2. **`script -qec ... /dev/null`** —— kiro-cli 必须有 PTY。非 TTY 环境下交互式
   prompt 直接返回空串，region 校验会爆 `invalid host label`。

手动登录必须在 `up -d` **之前**：没有凭据时 `loadCredentialsFromEnv()` 抛错 →
`process.exit(1)`，配上 `restart: unless-stopped` 就是无限重启循环。

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
