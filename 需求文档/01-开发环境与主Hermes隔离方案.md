# 01-开发环境与主 Hermes 隔离方案

> 编号：01
> 创建时间：2026-04-26
> 状态：已对齐，待实现
> 依赖：00-项目基础规范与总体原则

---

## 一、需求概述

开发 fork（`~/Projects/hermes-agent/`）用于调试、实验、开发新功能，必须与主 Hermes（系统安装的稳定版）完全隔离，避免互相干扰。

**核心原则：**
- 开发版完全空白启动，不继承主版任何配置
- 开发版使用独立的入口（Discord 而非微信），独立的配置目录，独立的 venv
- 主版 `hermes chat` 不受影响，开发版 `hermes-dev chat` 走自己的一套

---

## 二、隔离维度

| 维度 | 主 Hermes（稳定版） | 开发 Fork（调试版） |
|------|---------------------|---------------------|
| 启动命令 | `hermes chat` | `hermes-dev chat` |
| 虚拟环境 | 系统 Python / 全局安装 | `~/Projects/hermes-agent/.venv/`（独立） |
| 安装方式 | `pip install hermes-agent` | `pip install -e .`（可编辑安装，改代码即时生效） |
| 配置目录 | `~/.hermes/` | `~/.hermes-dev/` |
| 配置文件 | `~/.hermes/config.yaml` | `~/.hermes-dev/config.yaml` |
| 环境变量 | `~/.hermes/.env` | `~/.hermes-dev/.env` |
| 状态数据库 | `~/.hermes/state.db` | `~/.hermes-dev/state.db` |
| 会话存储 | `~/.hermes/sessions/` | `~/.hermes-dev/sessions/` |
| 日志 | `~/.hermes/logs/agent.log` | `~/.hermes-dev/logs/agent.log` |
| 技能 | `~/.hermes/skills/` | `~/.hermes-dev/skills/` |
| 记忆 | `~/.hermes/memories/` | `~/.hermes-dev/memories/` |
| 平台入口 | 微信（当前） | Discord（开发版） |

---

## 三、详细需求

### 3.1 虚拟环境隔离

**需求：**
- 在 `~/Projects/hermes-agent/` 内创建 `.venv/` 目录
- 激活 venv 后，执行 `pip install -e .` 安装开发版
- 改代码即时生效，无需重装

**举例说明：**

> **场景：修改开发版代码并测试**
>
> - **输入**：用户修改 `~/Projects/hermes-agent/cli.py` 中的某行代码
> - **操作**：保存文件，直接执行 `hermes-dev chat`
> - **预期行为**：开发版启动，加载的是修改后的 `cli.py`，无需执行任何安装命令
> - **输出**：新逻辑生效
>
> - **反例（错误行为）**：
>   - 输入：修改代码后执行 `hermes-dev chat`
>   - **错误输出**：加载的是旧代码，因为开发版指向了系统安装的稳定版路径

### 3.2 配置目录隔离

**需求：**
- 开发版使用 `HERMES_HOME=~/.hermes-dev`
- 首次启动时，如果 `~/.hermes-dev/` 不存在，自动创建空目录并走首次启动向导
- **不自动复制主版配置**，完全空白开始

**举例说明：**

> **场景：首次启动开发版**
>
> - **输入**：用户在终端执行 `hermes-dev chat`
> - **系统状态**：`~/.hermes-dev/` 目录不存在
> - **预期行为**：
>   1. 创建 `~/.hermes-dev/` 目录
>   2. 创建必要的子目录（`logs/`、`skills/`、`memories/`、`sessions/` 等）
>   3. 提示用户：「开发版首次启动，配置目录已创建于 ~/.hermes-dev/，请完成初始化配置」
>   4. 进入首次启动向导（如配置 API key、选择平台等）
> - **输出**：开发版完成初始化，与主版无任何关联
>
> - **反例（错误行为）**：
>   - 输入：`hermes-dev chat`
>   - **错误输出**：读取了 `~/.hermes/config.yaml`，导致主版配置泄漏到开发版

> **场景：开发版配置 Discord，主版配置微信**
>
> - 主版 `~/.hermes/config.yaml`：
>   ```yaml
>   platform: weixin
>   weixin_bot_id: xxx
>   ```
> - 开发版 `~/.hermes-dev/config.yaml`：
>   ```yaml
>   platform: discord
>   discord_token: yyy
>   ```
> - **预期行为**：两者同时运行互不干扰，主版监听微信消息，开发版监听 Discord 消息

### 3.3 命令隔离（Shell 别名）

**需求：**
- 在 `~/.bashrc`（或 `~/.zshrc`）中定义别名 `hermes-dev`
- 别名自动激活 venv 并设置 `HERMES_HOME`，然后调用开发版入口

**举例说明：**

> **场景：定义别名**
>
> - **别名内容**：
>   ```bash
>   alias hermes-dev='source ~/Projects/hermes-agent/.venv/bin/activate && HERMES_HOME=~/.hermes-dev ~/Projects/hermes-agent/hermes'
>   ```
> - **输入**：`hermes-dev chat`
> - **预期行为**：
>   1. 激活 `~/Projects/hermes-agent/.venv/bin/activate`
>   2. 设置环境变量 `HERMES_HOME=~/.hermes-dev`
>   3. 执行开发版入口 `~/Projects/hermes-agent/hermes chat`
> - **输出**：开发版启动，使用独立配置和代码
>
> - **边界情况**：
>   - 如果 venv 不存在，提示用户先执行初始化脚本

### 3.4 技能目录隔离

**需求：**
- 开发版使用 `~/.hermes-dev/skills/`
- 与主版技能完全独立，可自由实验、修改、删除技能
- 不影响主版已配置的技能

**举例说明：**

> **场景：开发新技能**
>
> - **输入**：用户在开发版创建新技能 `~/.hermes-dev/skills/my-test-skill/SKILL.md`
> - **操作**：`hermes-dev chat`，然后测试该技能
> - **预期行为**：技能只在开发版可用，主版 `hermes chat` 看不到该技能
> - **输出**：开发版技能列表包含 `my-test-skill`，主版不包含
>
> - **反例（错误行为）**：
>   - 输入：在开发版删除某个技能
>   - **错误输出**：主版的同名技能也被删除

---

## 四、初始化流程

开发版首次使用前，需要执行一次初始化：

```bash
cd ~/Projects/hermes-agent
python -m venv .venv
source .venv/bin/activate
pip install -e .
# 可选：复制上游 skills 到开发版
# cp -r ~/.hermes/skills ~/.hermes-dev/
```

然后在 `~/.bashrc` 中添加：

```bash
alias hermes-dev='source ~/Projects/hermes-agent/.venv/bin/activate && HERMES_HOME=~/.hermes-dev ~/Projects/hermes-agent/hermes'
```

---

## 五、边界情况与异常处理

| 场景 | 预期行为 |
|------|----------|
| venv 不存在但执行 `hermes-dev` | 提示：「开发版虚拟环境未初始化，请先执行 ~/Projects/hermes-agent/scripts/init-dev.sh」 |
| `~/.hermes-dev/` 不存在 | 自动创建空目录，走首次启动向导 |
| 开发版代码有语法错误 | 正常抛出 Python 异常，不影响主版 |
| 同时运行主版和开发版 | 两者互不干扰，各自监听各自的平台消息 |

---

*本文档定义了开发环境与主 Hermes 的隔离方案，后续实现需严格遵循。*
