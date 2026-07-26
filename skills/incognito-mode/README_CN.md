# 🔒 Hermes 无痕模式 v2.2.1

> 浏览器无痕模式的 AI Agent 升级版。四层纵深防御，零痕迹残留。

## 这是什么？

[Hermes Agent](https://hermes-agent.nousresearch.com/) 的隐私保护 Skill，通过 **四层纵深防御架构** 确保会话完全无痕：

1. **Skill 策略层** — 能力矩阵拦截持久化写入（记忆、技能、定时任务、配置文件）
2. **Runtime 护栏层** — Shell 历史抑制、沙箱文件系统、PID 锁临时目录
3. **Framework 支持层** — 会话级隔离标记、子代理传染协议
4. **事后审计层** — 10 步反向审计流水线 + 安全擦除（Python `os.urandom` 覆写 → `fsync` → `truncate` → `unlink`）

## 架构

```
Phase 1: 嵌套幂等校验 + PID锁沙箱初始化 + 孤儿清理
   ↓
Phase 2: 无痕隔离执行（事前防线）
   ↓
Phase 3: 用户确认门禁 / 15分钟 TTL
   ↓
Phase 4: 全量反向审计（10步）+ 安全擦除 + 会话容器销毁
   ↓
Phase 5: 审计报告 + 最终确认回执
```

### 保护范围

| 层面 | 保护措施 |
|------|---------|
| 文件系统 | 所有写入限定在 PID 锁沙箱内；非沙箱写入事后检测并擦除 |
| Shell 历史 | 每条命令强制前缀 `HISTFILE=/dev/null HISTSIZE=0` |
| Memory 记忆 | SHA-256 哈希比对基线快照，检测意外持久化写入 |
| Skill/Cron | 检测会话期间未授权的技能/定时任务创建 |
| 进程 | 快照 diff 检测孤儿进程 |
| 会话容器 | 容器级销毁作为最后防线 |

### 已知盲区（知情同意）

- **OS 级**：Swap 交换分区、Core Dump、Syslog/Journald、文件系统 atime
- **网络级**：企业代理/防火墙日志、DNS 查询记录、LLM API 厂商日志（默认保留 30 天）
- **第三方**：IDE 文件监视器、已存在服务自身日志

> 敏感场景建议配合私有化部署模型（Ollama）使用。

## 安装

```bash
# 克隆到 Hermes skills 目录
git clone https://github.com/GenmetsuWenxuePress/hermes-incognito-mode.git ~/.hermes/skills/incognito-mode

# 或手动复制
cp SKILL.md ~/.hermes/skills/incognito-mode/
```

## 使用

在任意 Hermes 会话中激活：

```
/incognito start
```

会话中可用指令：

| 指令 | 作用 |
|------|------|
| `/incognito status` | 显示沙箱路径、PID 锁状态、文件清单 |
| `/incognito audit` | status + 近期终端/web 活动摘要 |
| `/incognito export <路径>` | 将成果转存到持久目录 |
| `/incognito abort` | 紧急跳过确认，立即强制销毁 |

## 结束无痕会话

> ⚠️ **需手动操作。** Agent 不会自动销毁——你必须显式结束会话。

完成后，对 Agent 说：

```
确认销毁
```

或紧急跳过确认，直接强制销毁：

```
/incognito abort
```

Agent 随后会执行：
1. 10 步反向全量审计（Phase 4）
2. 安全覆写擦除所有临时文件
3. 输出最终审计报告（Phase 5）
4. 销毁会话容器

> 💡 **15 分钟 TTL 提醒**：如果你中途离开，Agent 会在 15 分钟无交互后提示你结束会话。自动销毁需 `[Framework L2]` 框架层支持（详见下方 §能力边界）。

结束后建议执行 `/new` 开启新会话，确保旧会话容器被彻底清除。

## 限制与替代方案

### 能力边界

本 Skill 运行在 LLM Agent 层面，以下需求需 Hermes 宿主框架升级支持（已标记 `[Framework L2]`）：

- **TTL 自动销毁 Timer**：Phase 3 超时后自动触发 Phase 4（当前靠用户手动 `/incognito abort`）
- **Ephemeral Session Store**：纯内存 Session 引擎，绕过 SQLite WAL 日志
- **Auto-mount tmpfs**：启动时自动挂载 `/dev/shm/`，关机即物理消失
- **CLI Flag** `hermes --incognito`：框架原生无痕模式
- **ZDR Header Injection**：向 LLM API 自动附加 `X-Zero-Data-Retention: true`

### 与类似项目的对比

| 方案 | 机制 | 优势 | 局限 |
|------|------|------|------|
| **本 Skill** | Agent 层策略 + 事后审计 | 无需框架改动，即装即用 | OS 内核级盲区不可控 |
| 浏览器隐私模式 | 隔离 Cookie/Storage | 对 Web 浏览有效 | 不覆盖终端、文件系统、LLM API |
| Tails OS | 全系统 amnesia | 完整 OS 级隔离 | 需重启切换，不适合日常开发 |
| Docker ephemeral | `--rm` 容器自动销毁 | 进程级隔离 | 需额外配置，不能保护 API 日志 |
| Qubes OS | VM 级隔离 | 最强隔离性 | 硬件要求高，学习曲线陡 |

**推荐组合**：本 Skill（Agent 层）+ 私有化模型 Ollama（API 层）+ `swapoff -a`（OS 层）= 接近 Tails 级别的隐私保护，同时保持日常开发便利性。

## 许可

MIT — 详见 [LICENSE](LICENSE)

## 作者

幻灭文学出版社 + Hermes（7 轮交叉审计，Python 脚本硬化）
