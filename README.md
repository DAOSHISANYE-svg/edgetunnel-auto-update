# edgetunnel 云端自动更新

edgetunnel（Cloudflare Pages 项目 `warpzhuanyong`）的云端自动更新方案。

## 为什么需要它

你的 Pages 项目 **Git Provider = No**（手动上传部署），所以：

> 上游 `cmliu/edgetunnel` 仓库更新了 → **站点不会自动更新**

上游自带 `sync.yml`（每日同步代码到 fork），但那只同步仓库内容，**不会触发 Cloudflare 部署**。本工作流补上这一环。

## 工作流程

| 步骤 | 说明 |
|---|---|
| 1. 定时触发 | 每天 `05:23 UTC`（北京时间 13:23），与 CFnew 的 04:17 错开 |
| 2. 拉上游 | 下载 `cmliu/edgetunnel/main/_worker.js`（约 5.1 MB） |
| 3. 完整性校验 | **文件必须 >100 KB**，否则判定为下载不完整并中止 |
| 4. 读兼容日期 | 从上游 `wrangler.toml` 自动提取 |
| 5. 算 hash 比对 | `sha256sum` 与 `deployed-hash.txt` 对比 |
| 6. 部署 | `wrangler pages deploy dist --project-name=warpzhuanyong` |
| 7. 冒烟验证 | 主地址 HTTP 非 200 即 fail |
| 8. 记录 hash | 提交 `deployed-hash.txt` |

## 与 CFnew 方案的关键差异

| | CFnew | edgetunnel |
|---|---|---|
| 更新源 | GitHub **Release** → `Pages.zip` | 直接拉 `_worker.js`（5.1 MB） |
| 版本判断 | 版本号字符串 | **sha256 内容 hash** |
| 完整性校验 | 查包内有无 `_worker.js` | **查大小 >100 KB** |
| 兼容日期 | 硬编码 `2026-01-20` | **从上游 wrangler.toml 读** |

## 需要你配置的东西（一次性）

### A. Cloudflare API Token

1. 打开 https://dash.cloudflare.com/profile/api-tokens
2. **Create Custom Token**
   - Permission：`Account / Cloudflare Pages / Edit`
   - Account Resources：选 `Junjiang412@gmail.com's Account`
3. 复制 Token

### B. GitHub Secrets

仓库 → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**

| 名称 | 值 |
|---|---|
| `CLOUDFLARE_API_TOKEN` | 上一步的 Token |
| `CLOUDFLARE_ACCOUNT_ID` | `071c36c227211679dd1537626856116f` |

## 手动触发

Actions → **edgetunnel Auto Update** → **Run workflow**
- 直接点 → 只在 hash 变化时部署
- 输入 `force=true` → 强制重新部署

## ⚠️ 首次运行说明

`deployed-hash.txt` 初始为空 → 首次运行会**触发一次全量部署**（正常行为，用来建立 hash 基线）。

## 失败排查

| 现象 | 原因 |
|---|---|
| 报「文件仅 x 字节，明显不完整」 | GitHub raw 下载被截断，检查网络 |
| 主地址 404/5xx | 部署破坏了线上服务，用 Actions 历史版本回滚 |
| Action 报 auth 失败 | `CLOUDFLARE_API_TOKEN` 无效或过期 |

## 回滚

GitHub Actions 页面 → 点对应的那次运行 → 右上角 **Re-run jobs**
或 Cloudflare 控制台 Pages 项目 → **Deployments** → 选旧版本 **Rollback to this deployment**

## 相关链接

- 上游仓库：https://github.com/cmliu/edgetunnel
- 线上地址：https://warpzhuanyong.pages.dev