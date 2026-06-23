---
name: laptop-remote
description: 远程操作笔记本 xiaohaifale — SSH/WMI/MCP 连接、helix-pilot AI 操控、自动启停
---

# 远程操作笔记本 (xiaohaifale)

两台 Windows 电脑在同一局域网下，从台式机远程操作笔记本。

---

## 网络信息

```
路由器: 192.168.10.1 (ImmortalWrt)
台式机: 192.168.10.231 (benben)
笔记本: 192.168.10.121 (xiaohaifale)
  用户: XIAOHAI
  Profile: C:\Users\Administrator\  (注意：不是 C:\Users\XIAOHAI\)
WiFi SSID: fale520
```

---

## 连接方式速查

| 方式 | 命令 | 用途 |
|------|------|------|
| SSH | `ssh XIAOHAI@192.168.10.121` | 命令行操作，免密 |
| WMI | `wmic /node:192.168.10.121 /user:XIAOHAI /password:<密码> ...` | 备用，无需安装 |
| MCP | `http://192.168.10.121:8765/mcp` | AI 视觉操控（截图+点击+打字） |

---

## SSH（主力，免密+自启）

```powershell
ssh XIAOHAI@192.168.10.121 "你要的 cmd 命令"
```

**注意**：SSH 远程执行默认走 `cmd.exe`，`&&`/管道不可靠，复杂逻辑拆成多条调用。

### 踩坑备忘

1. Profile 目录是 `C:\Users\Administrator`，不是 `C:\Users\XIAOHAI`
2. `authorized_keys` 权限只留 `SYSTEM` + `xiaohai`，不能有 `Administrators` 组
3. 私钥必须无 passphrase，否则 `BatchMode` 失败
4. 公钥文件一行不断开，ASCII 编码，无空行无尾随空格

---

## WMI（备用）

```powershell
wmic /node:192.168.10.121 /user:XIAOHAI /password:<密码> process call create "命令"
wmic /node:192.168.10.121 /user:XIAOHAI /password:<密码> computersystem get name
wmic /node:192.168.10.121 /user:XIAOHAI /password:<密码> datafile where "name='C:\\path'" get FileSize
```

**限制**：看不到 stdout，复杂脚本用 base64 编码通过 `powershell -EncodedCommand`。

---

## MCP / AI 视觉操控（helix-pilot）

### 架构

```
台式机 Reasonix ──MCP/HTTP──→ 笔记本 helix-pilot ──→ Ollama (moondream)
                                        └──→ pyautogui (截图/点击/打字)
```

### 笔记本端

```cmd
cd C:\Users\Administrator\helix-pilot
python server_http.py   # 监听 0.0.0.0:8765
```

配置文件：`config\helix_pilot.json` → `ollama_endpoint: http://127.0.0.1:11434`

### 台式机端

MCP 注册：
```
名称:   laptop-pilot
传输:   http
URL:    http://192.168.10.121:8765/mcp
```

**注意**：笔记本重启 server 后需重新注册 MCP（session 失效）。

### 可用工具（20 个）

`screenshot` `click` `type_text` `hotkey` `scroll` `describe` `find` `verify` `auto` `browse` `status` `list_windows` `wait_stable` `click_screenshot` `resize_image` `spawn_pilot_agent` `send_pilot_agent_input` `wait_pilot_agent` `list_pilot_agents` `close_pilot_agent`

---

## 自动启停

笔记本计划任务 `AutoPilot_fale520`，WLAN 事件触发：

| 触发器 | 事件 |
|--------|------|
| WiFi 连接 | EventID 8001 |
| WiFi 断开 | EventID 8003 |
| 开机 | 后备 |

脚本：`C:\Users\Administrator\helix-pilot\auto_pilot.ps1`

---

## 笔记本环境

```
OS: Windows 10 (x64)
用户: xiaohai (Administrators)
OpenSSH: 9.5 (C:\Windows\System32\OpenSSH\)
Python: 3.12 (C:\Users\Administrator\AppData\Local\Microsoft\WindowsApps\)
Ollama: 0.30.9 (C:\Users\Administrator\AppData\Local\Programs\Ollama\)
  模型: moondream:latest (1.7GB, CPU推理)
  端点: http://127.0.0.1:11434
helix-pilot: C:\Users\Administrator\helix-pilot\
  HTTP 端口: 8765
Reasonix: C:\Users\Administrator\AppData\Local\Programs\reasonix\ (桌面 GUI 版)
Obsidian: 已安装
VSCode: 已安装
```

---

## 常见故障

| 症状 | 检查 | 修复 |
|------|------|------|
| MCP 连不上 | `Test-NetConnection 192.168.10.121 8765` | 笔记本重跑 `server_http.py` |
| Ollama 返回 407 | 系统代理拦截 localhost | 关系统代理或设 `NO_PROXY=localhost,127.0.0.1` |
| SSH 公钥被拒 | `icacls authorized_keys` 看权限 | 去掉 Administrators 组 |
| 模型下载超时 | ollama.com 不通 | 设 `HTTP_PROXY` 环境变量 |
| GUI 操作失败 | SSH 会话无桌面 | 让用户桌面 CMD 手动跑 |

---

## 快速命令

```powershell
# 检查笔记本状态
ssh XIAOHAI@192.168.10.121 "hostname; ollama list"

# 传文件
scp "本地文件" XIAOHAI@192.168.10.121:"C:\Users\Administrator\Desktop\"

# 重启 Ollama
ssh XIAOHAI@192.168.10.121 "taskkill /f /im ollama.exe"
ssh XIAOHAI@192.168.10.121 "start /b C:\Users\Administrator\AppData\Local\Programs\Ollama\ollama.exe serve"
```
