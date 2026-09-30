# 运维说明（本 fork 专用）

上游 `pingmike2/cline2api-workers` 只讲「怎么用」，不讲「fork 之后怎么维护」。这份文档只写后者。

## 当前部署形态

| 项 | 值 |
|---|---|
| 本 fork | `plitoy/cline2api-workers` |
| 上游 | `pingmike2/cline2api-workers` |
| Worker 名称 | `cline2api` |
| 线上地址 | `https://api.1788.dpdns.org` |
| Base URL（OpenAI 客户端填这个） | `https://api.1788.dpdns.org/v1` |
| 部署方式 | push 到 `main` → GitHub Actions 自动 `wrangler deploy` |
| workers.dev 子域 | **已关闭**，只有自定义域可用 |
| 账号池 | 2 个（见下方「账号池」） |

> ⚠️ 本 fork **没有**使用 Cloudflare Dashboard 的「Git 集成」。上游 README 里警告的是那条路，容易因入口文件/构建环境失败。
> 这里走的是独立的 GitHub Actions + `wrangler deploy`，不受该问题影响。

### workers.dev 为什么必须显式关掉

`wrangler.toml` 里写死了两行，**别删**：

```toml
workers_dev = false
routes = [
  { pattern = "api.1788.dpdns.org", custom_domain = true },
]
```

不写 `workers_dev = false` 的话，每次 `wrangler deploy` 都会把 `cline2api.plitoy.workers.dev` 重新打开。
`routes` 则让自定义域的绑定跟着代码走，这样即使 Worker 被删掉重建，域名也会自动绑回来。

## 仓库 secrets

Settings → Secrets and variables → Actions → **Secrets**：

| Secret | 用途 | 换掉它的时机 |
|---|---|---|
| `CLOUDFLARE_API_TOKEN` | 让 wrangler 有权限部署 | token 被吊销时 |
| `CLOUDFLARE_ACCOUNT_ID` | Cloudflare 账号 ID | 换 CF 账号时 |
| `CLINE_REFRESH_TOKEN` | Cline 账号长期令牌 | Cline 返回 401/403 时（见下） |
| `API_KEY` | 客户端访问本服务的 key | 想轮换客户端密钥时 |

仓库 **Variables**（非 secret）：`CUSTOM_DOMAIN` = `api.1788.dpdns.org`，仅用于部署后的健康检查拼 URL。

**本 fork 对 `.gitignore` 的增量**：额外屏蔽 `cline_refresh_token.txt`、`cline_state.json`、`step*_*.py`，防止本地换 token 时误提交。这些文件只存在于本地。

## 日常升级：同步上游

```bash
git fetch upstream
git log --oneline HEAD..upstream/main   # 先看上游有什么
git merge upstream/main                 # 有冲突就解，然后照常 push
```

push 之后 Actions 自动部署，不需要再手动操作。

冲突高发点只有两个文件：`worker.js` 和 `api/index.js`（上游要求这两份逻辑同步）。**如果你没改过它们，merge 基本不会冲突**；真出了冲突，大概率是本 fork 的 `.gitignore` 或 workflow 与上游无交集。

### 只想看上游更新、不自动合

上游的 `Update branch` 按钮（fork 页面顶部）在 `main` 落后时会出现，点它会开一个 PR，走 review 后 merge，merge 到 `main` 同样触发自动部署。

## 升级 wrangler 版本

`wrangler` 已 pin 在 `package.json` 的 `devDependencies`：

```bash
npm i -D wrangler@latest
git commit -am "chore: bump wrangler" && git push
```

会连带更新 `package-lock.json`。两个 push 合一次提交即可，只改 `package*.json` 也会触发部署。

## Cline refreshToken 过期了怎么办

症状：线上请求返回 401/403，或 `/v1/health` 的 `accounts` 变 0。

上游提供的 `cline_oauth.py` 在**本 fork 里同样存在**（没有被删），走 WorkOS 设备授权流程：浏览器登录一次即可。注意两点：

1. 脚本里的中文/emoji 输出在 **GBK 控制台**（中文 Windows 的 `cmd`/PowerShell 默认）会抛 `UnicodeEncodeError`。先设好环境变量再跑：

   ```powershell
   $env:PYTHONIOENCODING="utf-8"
   $env:PYTHONUTF8="1"
   python -u cline_oauth.py
   ```

2. **拿到新 token 后必须更新仓库 secret 并重新部署**，否则线上还在用旧的：

   ```powershell
   # 脚本打印的 refreshToken 粘进来
   "<新 refreshToken>" | gh secret set CLINE_REFRESH_TOKEN --repo plitoy/cline2api-workers
   gh workflow run "Deploy to Cloudflare Workers" --repo plitoy/cline2api-workers
   ```

### 重要：Cline 会轮换 refreshToken

`/auth/refresh` 可能返回**新的** `refreshToken`（`worker.js` 内部会保存到内存，所以 Worker 自身不受影响）。但如果你在本地手工调过 `auth/refresh` 而丢弃了新值，仓库里存的旧值可能已经 `invalid_grant`。遇到这种情况直接重跑 `cline_oauth.py` 换一个新的，别试图抢救旧的。

## 账号池

`CLINE_REFRESH_TOKEN` 机密变量支持**一行一个 token**，多行即账号池。

### 当前账号

| # | 邮箱 | 加入时间 |
|---|---|---|
| 1 | `hynize@gmail.com` | 2026-09-30 03:35 |
| 2 | `wfu.lee@gmail.com` | 2026-09-30 04:42 |

本地副本（含明文 token，注意别提交）：
`C:\Users\wfule\AppData\Local\Temp\opencode\cline_pool.txt`
`cline_refresh_token.txt` = 账号 1，`cline_refresh_token_2.txt` = 账号 2。

### 上限

代码侧无上限（`worker.js:192` 只做 `split("\n")` + `filter(len > 8)`）。真正的天花板是
**单个机密变量 5 KB**：当前 token 为 25 字符 + 换行 = 26 字节，`5120 / 26 ≈ 196` 个。
但真正的约束是运维摩擦 —— 换任意一个账号的 token 都要重写整个多行 secret 并重新部署。

### 加账号的完整流程

```powershell
# 1. 取新 token（浏览器登录新账号并授权）
$env:PYTHONIOENCODING="utf-8"; $env:PYTHONUTF8="1"
python -u step1_device.py                          # 打印 AUTH_URL
start chrome $AUTH_URL                              # 浏览器里授权
python -u step2_register.py 240 cline_refresh_token_N.txt

# 2. 追加进本地池（注意是追加，不要覆盖整个文件）
Add-Content C:\Users\wfule\AppData\Local\Temp\opencode\cline_pool.txt "<新 token>"

# 3. 整池写回 secret + 重新部署
Get-Content C:\Users\wfule\AppData\Local\Temp\opencode\cline_pool.txt -Raw |
  gh secret set CLINE_REFRESH_TOKEN --repo plitoy/cline2api-workers
gh workflow run "Deploy to Cloudflare Workers" --repo plitoy/cline2api-workers

# 4. 核对
curl -s https://api.1788.dpdns.org/v1/health          # accounts 应等于账号数
```

### 两个坑

1. **`filter(len > 8)` 会静默丢弃长度 ≤ 8 的行**。粘贴截断、空行、表头都会被无声吃掉，
   账号直接从池子消失，**不报错**。所以每次改完 secret 一定要核对 `/v1/health` 的
   `accounts` 数字，别只看部署是否成功。
2. **round-robin 不是全局的**。`accountIndex` 是模块级变量，Cloudflare 会同时跑多个
   isolate，每个 isolate 各有一份计数器。所以是「大致均摊」而非严格交替，
   没有任何响应头能反映当次用了哪个账号。整体上 N 个账号 ≈ N 倍日配额。

### 关于 refreshToken 轮换

实测（Cline 官方 API，2026-09-30 逐个验活）：**`/auth/refresh` 不返回新 refreshToken**，
两个账号都是「有效、未轮换」。所以仓库 secret 里存的原始 token 可以一直用。

`worker.js` 里保留了轮换处理（万一将来上游改成轮换也不会炸），但你**不需要**为此做任何事。


## 验证部署成功

```bash
curl https://api.1788.dpdns.org/v1/health
```

期望：`{"ok":true,"version":"...","authenticated":true,"accounts":2,"model":"..."}`

- `authenticated: true` = `API_KEY` 变量已生效
- `accounts: 2` = `CLINE_REFRESH_TOKEN` 里解析出 2 个账号（**这只数行数，不验活**）

workflow 末尾已内置这一步冒烟检查，Actions 绿了基本就是好的。

## 已知坑

1. **非浏览器 UA 可能被网关拦（错误码 1010）**。客户端若不好改请求头，自定义一个浏览器 UA 即可：
   ```
   User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36
   ```
   与 API Key 正确与否无关，先排 UA。

2. **`/v1/models` 和 `/v1/health` 不鉴权**，这是 worker 原设计。只有 `/v1/chat/completions` 和 `/v1/messages` 校验 `API_KEY`。介意模型列表外露的话需要改 `worker.js`。

3. **Actions 日志出现 Node.js 20 弃用警告**（来自 `actions/checkout@v4` / `actions/setup-node@v4`）。目前只是 warning，不影响运行；上游升到 v5 后会自动消失。

4. **`concurrency` 设为不取消进行中的任务**。连续 push 时后一次会等前一次跑完，避免两次 `secret put` 交叉覆盖。
