# 本 fork 的改动（相对上游 superdady666/agent2api）

个人自建分支。改动集中在一处动机：**安全事件要能溯源**。

上游链路：`aimod-cc/agent2api`（原项目，作者账号已封禁）→ `weiang12345/agent2api`
→ `superdady666/agent2api`（当前维护版，本 fork 的同步来源）→ `mosenwoek/agent2api`（本仓库）。

## 改动 1：拒绝未授权的请求时记录来源 IP 与 UA

### 背景

网关在鉴权失败时只记一行 `❌ 拒绝未授权的请求: GET /.env`。面板「系统事件」页因此
只能看到「谁被拒了」，看不到「**是谁**被拒的」—— 一条扫描日志里没有任何可归因
信息，排查误报与溯源都做不了。

### 改动内容

**后端** `desktop-tauri/src-tauri/server/src/server/http.rs`：

- 新增 `client_ip(&Request)`：取真实客户端 IP。
  - 优先 `X-Forwarded-For` 的**最左**一项（越靠左越靠近原始客户端；每一跳都把自己
    追加到右侧，所以最右那项是最靠近服务端的一跳，没有溯源价值）
  - 其次 `X-Real-IP`（nginx 的 `$remote_addr`）
  - 都没有时退回 axum 的对端地址（裸机 / 直连部署时这就是真实来源）
- 新增 `client_user_agent(&HeaderMap)`：取 UA，截断到 200 字符。
- 鉴权拒绝的日志改为 `logging::log_event` 单次成型，`data` 里带
  `ip` / `ua` / `method` / `path`。控制台另打一行（`log_event` 按设计不重复打印，
  而 `docker logs` 是排障时最直接的现场）。

**为什么用 `log_event` 而不是 `log`**：`log()` 与 `log_event()` 各写一次会在面板里
出现两行重复记录。`log_event` 一次成型，控制台用 `console_line` 补。

**前端** `desktop-tauri/ui-islands/src/islands/logs-panel.tsx`：

- `LogEntry['data']` 类型补上 `ip` / `ua` / `method` / `path`
- `extraText()` 末尾追加 `data.ip` 与 `data.ua`

只改后端是不够的：前端的 `extraText` 只认 `from` / `to` / `model` / `resetAtText` /
`status` 五个字段，多出来的键会被静默丢掉，面板上看不到。

### 安全性说明

`X-Forwarded-For` 的最左一项**可以被客户端伪造** —— 直接对公网暴露、前面没有可信
反代时，攻击者可以随便写。这里仍然取它，因为本项目的典型部署是「nginx 反代 →
容器」，nginx 用 `proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for` **追加**
而不是覆盖，最左一项保留的是原始来源。

完全裸暴露的场景下，日志里的 IP 只能当**线索**看，不能当证据。

### 效果

面板「系统事件」页的安全事件行长这样：

```
❌ 拒绝未授权的请求: GET /.env
1.2.3.4 · Mozilla/5.0 (Windows NT 10.0; Win64; x64)
```

## 改动 2：CI 推自己的 Docker Hub 镜像

`.github/workflows/docker.yml`：

- `IMAGE` 从 `aimodcc/agent2api` 改为 `nbmosen/agent2api`
- 新增 `push` 触发：推 `main` 分支也构建并推 `:latest` + `:dev`
  （个人 fork 的 main 就是「当前在用的那一份」，VPS 拉 `:latest` 就能拿到改动）
- `:latest` 的移动规则：main 分支或正式版 tag（不含 `-`）；beta tag 不动 `:latest`

`docker-compose.yml` 的镜像名改为 `nbmosen/agent2api`（这是**自用**配置，
要换成带 IP/UA 改动的镜像）。**README 各语言版保持上游的 `aimodcc/agent2api`
不动** —— 那是对外发布的部署说明，指向官方镜像是对的。

## 部署

VPS 上：

```bash
# 1. 确认 compose 指向自建镜像
grep image: docker-compose.yml     # 应为 nbmosen/agent2api:latest

# 2. 拉取并重启
docker compose pull
docker compose up -d
docker logs -f agent2api
```

数据卷 `./data` 不动 —— 升级后账号、Key、日志、报表全部保留。

**nginx 反代要带 `X-Forwarded-For`**，否则日志里记的是容器网关地址（172.17.x.1）
而不是真实来源：

```nginx
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Real-IP $remote_addr;
```

## 上游同步

上游维护版仓库（`superdady666/agent2api`）的改动可以随时合进来：

```bash
git remote add upstream https://github.com/superdady666/agent2api.git
git fetch upstream
git merge upstream/main
```

改动 1 落在 `http.rs` 的鉴权中间件与 `logs-panel.tsx` 的 `extraText` 两处，
上游若也动这两处，合并时需要人工确认。
