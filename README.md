# M-Team Keepalive

M-Team (馒头) 账号自动保活工具，基于 GitHub Actions + Docker 实现，无需服务器，免费运行。

基于 [CangShui/mtlogin-py](https://github.com/CangShui/mtlogin-py)（Python 重构自 [scjtqs2/mtlogin](https://github.com/scjtqs2/mtlogin)）。

## 工作原理

M-Team 要求 40 天内有活跃记录，否则账号会被封禁。本工具通过 GitHub Actions 定时执行以下操作：

1. 调用 `updateLastBrowse` 接口刷新账号的最后浏览时间
2. 使用 `actions/cache` 缓存登录 token，避免每次重新登录
3. 缓存过期时自动使用账号密码 + TOTP 重新登录

每 6 小时运行一次（UTC 0/6/12/18），单次运行约 15 秒，月消耗 GitHub Actions 免费额度约 180 分钟（总额度 2000 分钟）。

## 快速开始

### 1. Fork 本仓库

Fork 后确保仓库为 **Private**（保护 Secrets）。

### 2. 配置 Secrets

在仓库 `Settings → Secrets and variables → Actions` 中添加以下 Secrets：

| Secret | 说明 | 必填 |
|--------|------|------|
| `MT_USERNAME` | M-Team 用户名 | 是 |
| `MT_PASSWORD` | M-Team 密码 | 是 |
| `TOTPSECRET` | TOTP 密钥（从 2FA 二维码解析的 secret 字段） | 是 |
| `FEISHU_APP_ID` | 飞书应用 App ID | 否 |
| `FEISHU_APP_SECRET` | 飞书应用 App Secret | 否 |
| `FEISHU_RECEIVE_ID` | 飞书接收消息的用户 ID | 否 |

> TOTP 获取方式：将 M-Team 二次验证从邮箱验证切换为动态验证码，扫描生成的二维码，提取其中的 `secret` 字段。

### 3. 启用 Actions

首次 Fork 后需在仓库 Actions 页面手动启用 workflows。

### 4. 手动触发测试

在 Actions 页面选择 `M-Team keepalive` workflow，点击 `Run workflow` 验证是否正常运行。

## 通知

支持飞书 Bot 通知（仅在登录失败时发送）。需要在 [飞书开放平台](https://open.feishu.cn/) 创建应用，获取 `App ID` 和 `App Secret`，并配置接收消息的用户 ID。

上游项目还支持 Telegram、企业微信、QQ、钉钉、ntfy 等通知渠道，可通过环境变量配置。

## 项目结构

```
.
├── .github/workflows/
│   ├── build.yml          # 自动构建 Docker 镜像并推送到 GHCR
│   └── keepalive.yml      # 定时保活任务
├── Dockerfile             # 基于 python:3.11-slim 构建镜像
├── mtlogin.py             # 保活脚本（来自上游 + 改进）
├── requirements.txt       # Python 依赖
└── README.md
```

## 相对上游的改进

- **GitHub Actions 部署**：无需服务器/NAS，Fork 即用
- **Docker 镜像**：自动构建并推送到 GHCR，隔离运行环境
- **Token 缓存**：通过 `actions/cache` 持久化登录态，减少登录频率
- **自动重试**：Token 过期时单次运行内自动重新登录（账号+密码+TOTP），无需等待下次定时任务
- **飞书通知**：支持飞书 Open API Bot，仅在失败时通知

## 反检测机制

脚本使用 [curl_cffi](https://github.com/lexiforest/curl_cffi) 模拟 Chrome 124 的 TLS/JA3 指纹，而非 Python 原生 `requests` 库。这意味着：

- 请求的 TLS 握手特征与真实 Chrome 浏览器一致，不会被识别为 Python 脚本
- 配合随机化的 User-Agent、设备指纹（DID/visitorid）和请求签名算法（`_sgin`），进一步降低被风控检测的风险

Go 版 [scjtqs2/mtlogin](https://github.com/scjtqs2/mtlogin) 使用的是 CycleTLS（JA4 指纹伪造），原理类似但实现不同。两者在反检测能力上各有优势，本项目的 curl_cffi 方案在 GitHub Actions 环境下更轻量且易于部署。

## 环境变量

脚本支持通过环境变量配置所有参数，命令行参数优先。完整参数列表参见[上游文档](https://github.com/CangShui/mtlogin-py#环境变量)。

GitHub Actions 相关的关键变量：

| 环境变量 | 默认值 | 说明 |
|----------|--------|------|
| `USERNAME` | 空 | M-Team 用户名 |
| `PASSWORD` | 空 | M-Team 密码 |
| `TOTPSECRET` | 空 | TOTP 密钥 |
| `FEISHU_APP_ID` | 空 | 飞书 App ID |
| `FEISHU_APP_SECRET` | 空 | 飞书 App Secret |
| `FEISHU_RECEIVE_ID` | 空 | 飞书接收用户 ID |
| `DB_PATH` | `/data/cookie.db` | Token 缓存数据库路径 |

## 常见问题

### GitHub Actions 60 天无活动后暂停

GitHub 会在仓库 60 天无活动时自动暂停 scheduled workflows。解决方法：偶尔手动触发一次 workflow，或向仓库提交任意 commit。

### 如何强制重新登录

在 Actions 页面清除 cache（`Actions → Caches`），或直接修改 Secrets 中的密码触发 token 失效。

### 本地运行（非 GitHub Actions）

```bash
# 安装依赖
pip install -r requirements.txt

# 运行
python mtlogin.py \
  --username "用户名" \
  --password "密码" \
  --totpsecret "TOTP密钥"

# 带飞书通知
FEISHU_APP_ID=xxx FEISHU_APP_SECRET=xxx FEISHU_RECEIVE_ID=xxx \
  python mtlogin.py --username "用户名" --password "密码" --totpsecret "TOTP密钥"
```

## 安全建议

- **务必使用 Private 仓库**，避免 Secrets 泄露
- 不要将密码、TOTP 密钥提交到代码中
- 上游项目的数据库文件和日志应加入 `.gitignore`
