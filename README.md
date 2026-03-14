# 🤗 HuggingFace Space Keep-Alive

> 通过 GitHub Actions 定时任务，稳定保活一个或多个 HuggingFace Space 项目。

[![Keep Alive](https://github.com/YOUR_USERNAME/hf-space-keeper/actions/workflows/keep-alive.yml/badge.svg)](https://github.com/YOUR_USERNAME/hf-space-keeper/actions/workflows/keep-alive.yml)

-----

## 原理

HuggingFace 免费 Space 会在无活动后进入休眠（约 1~2 小时无请求）。本项目利用 **GitHub Actions 定时任务**，每 **25 分钟**自动：

1. 通过 HuggingFace API 检查 Space 当前状态
1. 若 Space 已休眠/暂停，发送唤醒请求（需要 `HF_TOKEN`）
1. 等待 Space 启动后，向其公开 URL 发送 HTTP 请求，确保产生真实活动
1. 输出每个 Space 的处理结果摘要

-----

## 快速开始

### 第一步：Fork 或使用此仓库

点击右上角 **Fork**，或直接 **Use this template** 创建你自己的仓库。

-----

### 第二步：获取 HuggingFace Token

1. 登录 [huggingface.co](https://huggingface.co)
1. 进入 **Settings → Access Tokens**
1. 创建一个新 Token，权限选择 **Write**（需要写权限才能唤醒 Space）
1. 复制 Token 备用

-----

### 第三步：配置 GitHub 仓库

#### 添加 Secret（存放 HF Token）

> **Settings → Secrets and variables → Actions → Secrets → New repository secret**

|Secret 名称 |值             |说明                      |
|----------|--------------|------------------------|
|`HF_TOKEN`|`hf_xxxxxxxxx`|你的 HuggingFace API Token|

#### 添加 Variable（配置 Space 列表）

> **Settings → Secrets and variables → Actions → Variables → New repository variable**

|Variable 名称|示例值                                    |说明               |
|-----------|---------------------------------------|-----------------|
|`SPACE_IDS`|`myuser/my-app,anotheruser/their-space`|逗号分隔的 Space ID 列表|

**Space ID 格式**：`用户名/Space名称`，即 HuggingFace URL 中 `spaces/` 后面的部分。

例如：`https://huggingface.co/spaces/gradio/image-classifier` → Space ID 为 `gradio/image-classifier`

-----

### 第四步：启用 GitHub Actions

1. 进入仓库的 **Actions** 标签页
1. 如果看到提示，点击 **I understand my workflows, go ahead and enable them**
1. 工作流将自动按计划运行（每 25 分钟一次）

-----

### 第五步：手动测试

配置完成后，可以手动触发一次验证：

1. 进入 **Actions → 🤗 HuggingFace Space Keep-Alive**
1. 点击 **Run workflow**
1. 查看运行日志，确认所有 Space 均 ✅ 成功

-----

## 配置说明

### SPACE_IDS 格式

```
# 单个 Space
myusername/my-gradio-app

# 多个 Space（英文逗号分隔，空格可选）
myusername/app-one, myusername/app-two, anotherusername/shared-space
```

### 调整保活频率

编辑 `.github/workflows/keep-alive.yml` 中的 cron 表达式：

```yaml
schedule:
  - cron: "0/25 * * * *"   # 每 25 分钟（默认，推荐）
  # - cron: "0/30 * * * *" # 每 30 分钟
  # - cron: "0 * * * *"    # 每小时（适合不那么频繁休眠的 Space）
```

> ⚠️ GitHub Actions 免费版每月有 2000 分钟限制。每 25 分钟运行一次，每次约 30~60 秒，月消耗约 **1000~2000 分钟**。建议确认你的 Actions 使用量。

-----

## 运行日志示例

```
============================================================
  HuggingFace Space Keep-Alive
  Time: 2024-01-15T08:25:00+00:00
============================================================
Spaces to keep alive (2): myuser/my-app, myuser/another-app

━━━ Processing: myuser/my-app ━━━
  ℹ  Current status: RUNNING
  ✓  Pinged https://myuser-my-app.hf.space → HTTP 200
  ✅  Done — woken=False, pinged=True

━━━ Processing: myuser/another-app ━━━
  ℹ  Current status: SLEEPING
  ↺  Sending wake-up request to myuser/another-app ...
  ✓  Wake-up sent. Waiting 60s for space to boot...
  ✓  Pinged https://myuser-another-app.hf.space → HTTP 200
  ✅  Done — woken=True, pinged=True

============================================================
  SUMMARY
============================================================
  ✅  myuser/my-app  (status=RUNNING)
  ✅  myuser/another-app  (status=SLEEPING, woken=yes)

  Total: 2 | Success: 2 | Failed: 0
============================================================
```

-----

## 常见问题

**Q: 不提供 `HF_TOKEN` 可以用吗？**
A: 可以。没有 Token 时，脚本会跳过状态检查和唤醒步骤，仅通过 HTTP 请求 ping Space 的公开 URL。对于已在运行的 Space 有效，但无法唤醒已休眠的 Space。

**Q: 支持私有 Space 吗？**
A: 可以唤醒，但无法通过公开 URL ping（私有 Space 需要认证）。建议将需要保活的 Space 设置为 Public。

**Q: GitHub Actions 免费额度够用吗？**
A: 公开仓库的 Actions 完全免费无限制。私有仓库每月有 2000 分钟免费额度，本项目月耗约 1000~2000 分钟，通常够用。

**Q: 为什么选择 25 分钟而不是更长？**
A: HuggingFace 免费 Space 的实际休眠策略在 1~2 小时之间，25 分钟保证了足够的安全边际。GitHub Actions 的 cron 实际执行可能有几分钟延迟，所以不选 30 分钟正好卡点。

-----

## 文件结构

```
hf-space-keeper/
├── .github/
│   └── workflows/
│       └── keep-alive.yml    # GitHub Actions 工作流
├── keep_alive.py             # 核心保活脚本（无外部依赖）
└── README.md                 # 本文件
```

-----

## License

MIT