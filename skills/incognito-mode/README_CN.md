# 🔒 Hermes 无痕模式 v2.6.3

> ⚠️ **正式使用前必测。** 本技能执行破坏性操作（安全覆写、会话销毁）。正式用于敏感或真实数据前，请先用非隐私的信息和数据测试，确认其在你的环境中能正常运作。

## 这是什么？

[Hermes Agent](https://hermes-agent.nousresearch.com/) 的隐私保护 Skill，通过 **四层纵深防御架构** 确保会话完全无痕：

1. **Skill 策略层** — 能力矩阵拦截持久化写入（记忆、技能、定时任务、配置文件）
2. **Runtime 护栏层** — Shell 历史抑制、复合命令包装（`bash -c`）、时间戳 TTL 沙箱
3. **Framework 支持层** — 会话级隔离标记、子代理传染协议
4. **事后审计层** — 10+ 步反向审计流水线 + 安全擦除（Python `os.urandom` 覆写 → `fsync` → `truncate` → `unlink`），含 `/tmp/` 根目录和 `hermes-snap-*.sh` 终端快照

## 架构

```
Phase 1: 嵌套幂等校验 + 时间戳TTL沙箱初始化 + 孤儿清理
   ↓
Phase 2: 无痕隔离执行（事前防线）
   ↓
Phase 3: 用户确认门禁 / 15分钟 TTL
   ↓
Phase 4: 全量反向审计（10+步）+ 安全擦除 + 会话容器销毁
   ↓
Phase 5: 审计报告 + 最终确认回执
```

### 保护范围

| 层面 | 保护措施 |
|------|---------|
| 文件系统 | 所有写入限定在时间戳 TTL 沙箱内；`/tmp/` 根目录扫描检测绕过写入；非沙箱写入事后检测并擦除 |
| Shell 历史 | 每条命令包装 `HISTFILE=/dev/null HISTSIZE=0`；复合语句（`if`/`for`/`while`）通过 `bash -c '...'` 执行 |
| 终端快照 | `hermes-snap-*.sh` 文件（含命令明文+环境变量）纳入强制安全擦除 |
| Web 缓存 | `~/.hermes/cache/` 下 7 个用户数据目录（web/screenshots/videos/images/audio/documents/vision）符号链接重定向至沙箱——崩溃安全 |
| 日志脱敏 | `agent.log` 查询/用户消息明文按**时间戳窗口**清洗为 `[REDACTED_INCOGNITO_QUERY]`，覆盖 7 种搜索 provider 格式；v2.5.6 起由 **Logging RedactingFilter 插件**在内存中事前拦截（明文不落盘），4.6b 事后擦除兜底 |
| 向量索引 | 哨兵文件（`/tmp/.hermes-incognito-active`）精细跳过 `active_sessions` 索引源——无痕内容不入持久向量库；4.9 删 session 即隐式释放 |
| Memory 记忆 | SHA-256 哈希比对基线快照，检测意外持久化写入 |
| Skill/Cron | 检测会话期间未授权的技能/定时任务创建 |
| 进程 | 快照 diff 检测孤儿进程，含 `.python_history` |
| 子代理 | 残留子代理沙箱 + live transcripts + `async_delegations` 表在清理阶段审计 |
| 崩溃残留 | `interrupted_turns.json`（desktop 自动续跑标记）按 sid 清理 |
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

## 依赖（v2.5.6+）

无痕模式的**事前日志拦截**依赖两个可选组件（缺失时降级为 4.6b 事后擦除，功能不中断）：

1. **incognito-log-filter 插件**（推荐）— 哨兵激活时在内存中脱敏查询/URL/用户消息 preview，明文不落盘：
   ```bash
   # 复制插件到 Hermes 用户插件目录
   cp -r incognito-log-filter ~/.hermes/plugins/
   hermes plugins enable incognito-log-filter
   ```
   > 插件 `register()` 在 Hermes 进程启动时执行——启用后需重启 Hermes 生效。Phase 1 步骤 3.6 会自动检查插件状态（软警告不阻塞）。

2. **index_all.py 哨兵配合**（可选）— 若使用向量索引 cron（`~/.hermes/scripts/index_all.py`），需升级其 `incognito_active()` 为 v2.5.5 语义（精细跳过 + session 存在性检查）。无痕会话会自动创建/释放哨兵，向量任务不被阻塞。

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
1. 10+ 步反向全量审计（Phase 4）
2. 安全覆写擦除所有临时文件，包括终端快照（`hermes-snap-*.sh`）
3. 输出最终审计报告（Phase 5）
4. 销毁会话容器

> 💡 **15 分钟 TTL 提醒**：如果你中途离开，Agent 会在 15 分钟无交互后提示你结束会话。自动销毁需 `[Framework L2]` 框架层支持（详见下方 §能力边界）。

结束后建议执行 `/new` 开启新会话，确保旧会话容器被彻底清除。

## 限制与替代方案

### 能力边界

本 Skill 运行在 LLM Agent 层面，以下需求需 Hermes 宿主框架升级支持（已标记 `[Framework L2]`）：

- **TTL 自动销毁 Timer**：Phase 3 超时后自动触发 Phase 4（当前靠用户手动 `/incognito abort`）
- **Execute No-Snapshot Flag**：`terminal()` 支持跳过命令后重写环境快照（`hermes-snap-*.sh`）——当前靠 4.6c 二次清除兜底
- **Logging RedactingFilter 原生支持**：已通过**用户插件**落地（`incognito-log-filter`，无需核心改动）；框架原生支持（无插件依赖）为可选优化
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

## 更新日志

### v2.6.3（R14 质疑复检）
- **4.6b query/turn 与 URL 清洗完全链式化** — 链式替换覆盖 query 与 turn 行（原只处理 URL 内部互斥）
- **能力矩阵/流程总览同步** — 对齐 v2.6.2 哨兵语义（index_all 跳过 sessions 源）
- **filter 挂载时机实证** — 确认 `setup_logging` 先于 `register` 执行
- **register 幂等实证** — 重复注册不再产生副作用

### v2.6.2（R13 双轨复检 — agy 独立审查，10 项修复）
- **LogFilter 挂载根因修复** — 原挂在 root logger 自身（Filter 不随日志传播，子 logger 全量静默失效），改为挂到 root 的所有 handlers
- **4.9r 去 bash -c 嵌套** — 修复嵌套引号冲突
- **URL 链式替换** — 混合引号行全量清洗
- **index_all 哨兵跳过 sessions 源** — 不只 active_sessions，sessions 源一并跳过
- **4.6b-2 占位符检测** — 防止清洗后占位符残留
- **章节编号修正** — 4.1c/4.6c 顺序调整
- **R11 根因重定位** — 确认 serve 后端原本就加载用户插件，删除多余 patch

### v2.6.1（R12 实测缺陷修复）
- **裸 URL 正则** — Firecrawl scraping / tools.web_tools 行裸 URL 明文（旧正则 0 命中）
- **4.6b-2 敏感域全文件清扫** — 不依赖时间戳行，对 agent.log 全文+轮转文件幂等清洗
- **并行进程 skill 固化审计** — 防止并行进程/curator 在会话中把知识固化进持久 skill
- **Phase 5 窗口外残留提示** — 提示时间窗口之外的残留可能性

### v2.6.0（R11 双缺陷修复）
- **4.9r 异常关闭补救协议** — 会话非正常关闭时的补救销毁流程
- **serve 后端插件加载核心 patch**（后被 R13 重定位为误判，见 v2.6.2）

### v2.5.9
- **P0 修复：bash 单引号安全** — 4.6b 代码内裸单引号（注释/正则 `(["'])`）会破坏 `python3 -c '...'` 的 bash 包裹，真实终端执行静默失败——改为 chr 拼接；30/30 bash 块全量验证通过
- **发布合规** — description ≤60 字符、platforms 字段、通用化示例措辞

### v2.5.8
- **async_delegations 表审计** — delegate_task 任务书/结果持久化进 state.db（4.9 不清理），按 sid 精确删除
- **unset 时机修正** — 移至 4.8 之后，避免 4.6b/4.7b/4.8 变量丢失静默失效

### v2.5.7
- **插件依赖检查（步骤 3.6）** — Phase 1 显式检查 incognito-log-filter 是否启用，防静默失效

### v2.5.6
- **Logging RedactingFilter 插件** — 哨兵门控 + 内存脱敏，事前拦截替代事后擦除，与 4.6b 双层防御
- **interrupted_turns.json 审计（4.1c）** — 防崩溃残留 prompt 明文

### v2.5.5
- **agent.log 双盲区修复（R6 实测）** — 覆盖 firecrawl 等 7 种 provider 日志格式 + `conversation turn` 用户消息明文（repr 双引号兼容）+ URL 行
- **哨兵语义重构** — 精细跳过（只跳过 active_sessions 源）+ 4.9 删 session 隐式释放

### v2.5.4
- **profile-safe 完整落地** — 4 处运行路径改 `$INCOGNITO_HERMES_HOME`；恢复块补恢复该变量

### v2.5.3
- **R4 深度审计修复** — 4.6b 清洗改时间戳窗口（原正则 0 命中静默失效）、4.1a delegation live transcripts 审计、向量索引哨兵、Phase 4 恢复块拆分、cache 重定向扩展 7 目录、4.1 排除列表补充、能力矩阵补 gbrain/agy/execute_code、4.6c snap 二次清除

### v2.5.2
- **Skill 审计降噪** — 4.4 find 排除 `.curator_state`/`.bundled_manifest`/`.usage.json` 元数据

### v2.5.1
- **Export 持久化修复** — Phase 1 拆分为两次 `terminal()` 调用：简单命令链做 `export`（跨调用持久化），再 `bash -c` 做幂等校验（子 shell 安全）。修复长会话中 `$INCOGNITO_TMP_DIR` 丢失问题。

### v2.5.0
- **Web 缓存符号链接至沙箱** — `~/.hermes/cache/web/`、`screenshots/`、`videos/` 通过符号链接重定向至沙箱。事前隔离替代事后擦除——崩溃安全，即使意外退出也无残留缓存。
- **Agent 日志关键词脱敏** — `agent.log` 中 `web_search` 查询明文在会话结束时按 session ID 精确清洗为 `[REDACTED]`。

### v2.4.1
- **终端快照自动覆写** — `hermes-snap-*.sh` 文件（含本会话全部命令明文+环境变量）纳入强制安全擦除

### v2.4.0
- **Cache + tmp + 子代理盲区关闭** — `~/.hermes/cache/`、`/tmp/` 根目录、子代理残留目录纳入审计和擦除
- **Python 历史审计** — `.python_history` 纳入进程/文件系统审计范围
- **进程审计降级** — 承认 Hermes `terminal()` 模型的固有局限
- **Phase 4 环境恢复** — 长时间会话中 `$INCOGNITO_TMP_DIR` 丢失时，通过 `pid.lock` 会话匹配自动恢复
- **预排除路径扩展** — `.hermes/cache/`、`.local/state/tirith/` 加入噪音过滤

### v2.3.1
- **Phase 4.9/5 去重** — 删除重复的 `hermes sessions delete` 指令
- **Phase 1 编号修复** — 修正 v2.3.0 新增步骤导致的编号错位
- **预排除路径泛化** — 文件系统审计中更广泛的路径过滤

### v2.3.0
- **时间戳 TTL 孤儿清理** — 替代失效的 PID 锁校验（Hermes `terminal()` 每次独立 bash，PID 退出即失）
- **复合命令安全包装** — `if`/`for`/`while` 统一用 `bash -c '...'` 包装，避免直接前缀 `HISTFILE=/dev/null` 导致语法错误
- **安全扫描器兼容** — 降低 `skills-guard-v1` 误报
- **系统路径噪音过滤** — 收窄文件系统审计范围
- **引号逃逸标准化** — 统一所有 Shell 代码块的引号模式

### v2.2.1（初始发布）
- PID 锁沙箱 + Python `os.urandom` 安全覆写
- 10 步反向审计流水线
- 子代理传染协议
- 15 分钟 TTL + 用户确认门禁

## 许可

MIT — 详见 [LICENSE](LICENSE)

## 作者

幻灭文学出版社 + Hermes（17 轮交叉审计）
