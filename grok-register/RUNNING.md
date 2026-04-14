# Grok 注册机运行指南

> 本文档汇总运行 `grok.py` 所需的最低配置、运行步骤、离开后报错处理以及 `tmux` 后台运行方案。

---

## 一、最低 API-Key 配置

### 1. 默认方案（gptmail）——只需 1 个 Key

| 环境变量 | 说明 | 是否必填 |
|---|---|---|
| `YESCAPTCHA_KEY` | YesCaptcha 平台 API Key，用于自动过 x.ai 的 Turnstile 验证码 | **必填** |

默认邮箱提供商 `gptmail` 通过爬取第三方临时邮箱页面获取地址，**不需要额外 API Key**。

**`.env` 最低配置示例：**

```env
YESCAPTCHA_KEY="你的_yescaptcha_key"
EMAIL_PROVIDER="gptmail"
```

### 2. 备选方案（luckmail）——需要 2 个 Key

若 `gptmail` 不稳定，改用 `luckmail` 时的最低配置：

```env
YESCAPTCHA_KEY="你的_yescaptcha_key"
EMAIL_PROVIDER="luckmail"
LUCKMAIL_API_KEY="你的_luckmail_api_key"
```

> `LUCKMAIL_BASE_URL` 已内置为 `https://mails.luckyous.com`，通常无需额外设置。

---

## 二、运行方法与步骤

```bash
# 1. 进入项目目录
cd grok-register

# 2. 安装依赖（Python 3.10+）
uv sync

# 3. 创建输出目录
mkdir -p keys

# 4. 复制并编辑环境变量文件
cp .env.example .env
# 编辑 .env，填入 YESCAPTCHA_KEY（以及 luckmail 相关 key）

# 5. 运行脚本
uv run python grok.py
```

**显式指定参数运行：**

```bash
# 使用 luckmail
uv run python grok.py --email-provider luckmail

# 指定并发线程数
uv run python grok.py --email-provider luckmail --threads 8
```

**成功输出位置：**

- `keys/grok.txt`：SSO token 列表
- `keys/accounts.txt`：`email:password:SSO`

---

## 三、离开后报错/中断如何处理？

`grok.py` 是前台 CLI 脚本，关闭终端或 SSH 会话会直接停止进程。离开后若进程中断，可通过以下方案解决：

### 方案 A：`nohup` + 自动重启循环（最简单）

创建启动脚本 `run.sh`：

```bash
cat > run.sh << 'EOF'
#!/bin/bash
cd "$(dirname "$0")"
mkdir -p keys logs
while true; do
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] 启动 grok.py..." >> logs/run.log
    uv run python grok.py --email-provider "${EMAIL_PROVIDER:-gptmail}" --threads "${THREADS:-1}" >> logs/run.log 2>&1
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] 进程退出，5秒后自动重启..." >> logs/run.log
    sleep 5
done
EOF
chmod +x run.sh
```

后台运行：

```bash
nohup ./run.sh &
```

查看报错日志：

```bash
tail -f logs/run.log
tail -n 100 logs/run.log
```

### 方案 B：`tmux`（适合需要偶尔手动查看的场景）

#### 安装 tmux

```bash
# 检查是否已安装
which tmux

# macOS 上安装
brew install tmux
```

#### 使用步骤

```bash
# 1. 新建名为 grok 的 tmux 会话
tmux new -s grok

# 2. 在 tmux 会话内部运行脚本
cd grok-register
uv run python grok.py --email-provider gptmail --threads 1

# 3. 让它在后台继续运行：先按 Ctrl+B，松开后再按 D
#    此时脚本已在后台运行，你可以关闭终端
```

#### 常用命令

| 操作 | 命令 |
|---|---|
| 重新连回去查看 | `tmux attach -t grok` |
| 临时退出（不结束程序）| 在 tmux 里按 `Ctrl+B`，再按 `D` |
| 彻底结束会话 | 连进去后按 `Ctrl+D` 或输入 `exit` |
| 查看有哪些会话 | `tmux ls` |
| 强制结束一个会话 | `tmux kill-session -t grok` |

#### 注意事项

- **断网或关机**会导致 tmux 会话里的脚本也停止（tmux 只能保持终端会话不死，不能抗机器重启）。
- 若需**机器重启后自动恢复**，请使用 `nohup` 自动重启脚本、`supervisor` 或 `systemd`。

### 方案 C：Supervisor / systemd（适合服务器长期稳定运行）

如需长期稳定运行，推荐使用进程管理器自动拉起。示例 Supervisor 配置：

```ini
[program:grok]
directory=/Users/e/workspace/register/grok-register
command=uv run python grok.py --email-provider gptmail --threads 1
autostart=true
autorestart=true
stderr_logfile=/Users/e/workspace/register/grok-register/logs/err.log
stdout_logfile=/Users/e/workspace/register/grok-register/logs/out.log
environment=YESCAPTCHA_KEY="你的key",EMAIL_PROVIDER="gptmail"
```

---

## 四、常见报错速查

| 报错现象 | 原因 | 处理 |
|---|---|---|
| `未找到 Action ID` | x.ai 页面结构变了，或 IP/代理被风控 | 换代理 IP 后重启 |
| `缺少 YESCAPTCHA_KEY` | `.env` 未加载或 Key 为空 | 检查 `.env` 和 `YESCAPTCHA_KEY` |
| `邮箱创建返回空，可能接口挂了` | GPTMail/LuckMail 服务不稳定 | 换邮箱提供商或等恢复 |
| `验证码无效 / CAPTCHA 失败` | 第三方服务波动 | 脚本会自动重试，一般无需手动处理 |
| 线程全部卡住不输出 | 连接池死锁或代理超时 | 重启进程 |
