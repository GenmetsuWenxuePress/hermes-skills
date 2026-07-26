---
name: incognito-mode
description: "Incognito mode v2.4.1: defense-in-depth, hermes-snap-*.sh terminal snapshots now auto-wiped."
version: 2.4.1
author: 幻灭文学出版社 + Hermes (10-round cross-audited, process audit degraded, cache+tmp+subagent gaps closed)
license: MIT
metadata:
  hermes:
    tags: [privacy, session, cleanup, ephemeral, sandbox, isolation]
---

# 无痕模式 (Incognito Mode) v2.4.1

浏览器无痕模式的 Hermes 升级版。**四层纵深防御**（Skill 策略 → Runtime 护栏 → Framework 支持 → OS 隔离），确保会话结束后无持久痕迹残留。

> **设计原则**：v1.0 是「事后擦除」→ v2.0 是「事前隔离」→ v2.1 是「事前隔离 + 事后全量反向审计」→ v2.2.1 是「稳健性硬化」→ v2.3.0 是「实战修正」→ v2.3.1 是「去重修正」→ v2.4.0 是「盲区填补：cache+tmp+subagent+python_history 四盲区关闭 + 进程审计降级」→ v2.4.1 是「终端快照泄漏修复：hermes-snap-*.sh 纳入强制覆写」。不信任 Phase 2 的隔离完美无缺——发现痕迹 → 报告 → 清除 → 二次验证。
> ⚡ **三条铁律（每次行动前自检）**：
> 1. **每条 terminal 命令** → 前缀 `HISTFILE=/dev/null HISTSIZE=0`，无一例外。**复合语句（`if`/`for`/`while`/`case`）必须用 `bash -c '...'` 包装**——直接前缀 `HISTFILE=/dev/null if ...` 会导致 bash 语法错误
> 2. **从 Phase 2 到 Phase 4** → 必须经过 Phase 3 用户确认门禁，不得跳跃。（TTL 自动销毁需 Framework L2 支持，见 §7）
> 3. **每次 delegate_task** → 允许使用，但必须透传 incognito header + 分配独立 PID/UUID 级子沙箱

---

## 免责声明（Phase 1 必须展示）

激活时 Agent 必须向用户展示：

> 🔒 **无痕模式 v2.4.1 已激活**
>
> **本技能保护范围**：不写持久记忆、所有临时文件在 PID 隔离沙箱中、Shell History 被拦截、禁止 `/compress` 会话摘要落盘（仅保留在内存）、退出时安全覆写擦除。
>
> **不在控制范围（请知悉）**：
> - OS 级：Swap 交换分区、Core Dump、Syslog/Journald、auditd 审计日志、文件系统 atime
> - 网络级：企业防火墙/代理日志、DNS 查询记录、目标网站 IP 访问日志
> - 云端：LLM API 提供商的请求日志与 KV Cache（OpenAI/Anthropic/Google 默认保留 30 天）
> - 第三方：IDE 文件监视器、Git watcher、外部已存在服务自身日志
>
> 敏感场景建议配合私有化部署模型（Ollama）使用。

---

## 1. 能力许可矩阵 (Capability Matrix)

### ❌ 绝对禁止

| 工具 / 行为 | 禁止范围 | 原因 |
|------------|---------|------|
| `memory` | 所有 action（add/replace/remove） | 防止写入持久记忆 |
| `skill_manage` | `create`, `edit` | 防止创建/修改技能文件 |
| `cronjob` | `create` | 防止创建定时任务 |
| `kanban` | `create`, `update`, `comment`, `link` | 防止在看板留下任务记录 |
| 修改 `~/.bashrc` / `~/.gitconfig` / `~/.profile` 等 | 任何写入 | 防止污染全局配置 |
| **创建新服务/容器** | 执行 `docker run` / `docker-compose up` / `systemctl start\|enable` / `nohup` / `tmux` / `screen` / `&` 后台守护进程 | 防止派生沙箱外持久进程和日志 |

### ⚠️ 严格受限

| 工具 / 行为 | 限制规则 |
|------------|---------|
| `write_file` / `patch` | **仅限** Phase 1 创建的随机沙箱目录 `$INCOGNITO_TMP_DIR`。严禁写入 HOME、项目目录、`~/.hermes/`、`/var/log/`。新建 SQLite `.db` 必须登记在销毁清单中 |
| `terminal` | 每条命令**必须**前缀 `HISTFILE=/dev/null HISTSIZE=0`。禁止 `pip install`（除非 `--target` 指向沙箱）、`git config --global`。禁止写入 `/var/log/` 或调用 `logger`。**精准服务界定**：允许连接已存在的服务端口（如 `127.0.0.1:6379`）或使用嵌入式数据库（SQLite），严禁创建新守护进程 |
| `delegate_task` | **允许使用**，但必须在子代理 Prompt 首行注入无痕协议头（见 §3），分配 PID/UUID 级独立子沙箱，主代理负责递归清理 |
| `web_search` / `web_extract` | 底层必须使用无痕 Context（禁止持久化 Cookie/LocalStorage/HTTP Cache/DNS Cache） |
| `/compress` | **禁止落盘**。无痕期间允许在内存中压缩上下文，但严禁将摘要持久化写入磁盘文件 |

### ✅ 自由使用（只读，不触发持久化）

`read_file`、`search_files`、`session_search`（仅读取历史，不索引当前会话）、`skill_view`、`clarify`

> ⚠️ `session_search` 在无痕模式下**禁止**触发后台 RAG/向量 Embedding 索引——Agent 不得将当前无痕会话内容提交给索引管道。

---

## 2. 五阶段生命周期 (5-Phase Lifecycle)

```
Phase 1: 嵌套幂等校验 + 时间戳TTL孤儿清理 + 沙箱初始化
   ↓
Phase 2: 无痕隔离执行（事前防线，禁止新服务/禁止/compress落盘）
   ↓
Phase 3: 用户确认门禁 / TTL (Framework L2) (解耦自杀死锁)
   ↓  ← 用户选择「确认销毁」/「/incognito abort」
Phase 4: 反向全量审计 + 逐项清除 (TRY→FALLBACK) + 二次验证（事后防线）
   ├ 4.1 文件系统审计 + /tmp/根目录  ├ 4.2 Shell History 审计
   ├ 4.3 Memory 审计 (SHA-256 哈希比对)├ 4.4 Skill/Cron 审计
   ├ 4.5 进程/python_history/子代理审计 ├ 4.6 环境变量/配置审计
   ├ 4.7 Git/项目审计               ├ 4.8 沙箱 Python 覆写安全擦除
   ├ 4.9 Session 容器销毁            └ 4.10 二次验证
   ↓
Phase 5: 全量审计报告 + 最终确认回执
```

> 🛡️ **双重保险架构**：
> - **防线 1（逐项审计）**：4.1-4.8 逐条检查跨 Session 的持久化路径——文件、Shell History、Memory、Skill、进程、配置、Git。发现一条清一条。
> - **防线 2（容器销毁）**：4.9 `hermes sessions delete` 将整个 Session 连同 transcript、向量索引、会话日志一并销毁——无论防线 1 是否有遗漏，这一击覆盖所有 Session 级痕迹。
> - 两层互补：逐项审计覆盖跨 Session 泄漏，容器销毁覆盖 Session 内积累。

---

### Phase 1: 预检、幂等判定与沙箱初始化

> ⚠️ **架构约束**：Hermes `terminal()` 每次调用启动独立 bash 进程，命令执行完立即退出。PID 锁的「跨调用存活父进程」假设不成立——`ps -p <PID>` 对已退出进程永远返回非零。Phase 1 步骤 3 的孤儿清理改用**时间戳 TTL**（24 小时）替代 PID 校验。

Agent **第一条回复**必须执行：

1. **嵌套幂等判定（P0-1）**：
   - 首先检查当前上下文是否已有 `INCOGNITO_MODE_ACTIVE=true`。
   - 若已激活：输出 `🔒 当前会话已在无痕模式中（沙箱：$INCOGNITO_TMP_DIR），跳过二次初始化`，保持原有环境，直接进入 Phase 2。
   - 若未激活：导出标记 `export INCOGNITO_MODE_ACTIVE=true`。

2. **沙箱目录与时间戳标记（P0-2）**：
   - 生成符合命名规范的沙箱路径：`/tmp/hermes-incognito-<PID>-<UUID>`（PID 仅用于命名区分，不用于存活校验——见上方架构约束）。创建 `pid.lock` 记录创建时间戳供 Phase 4.1 `-newer` 引用：
   ```bash
   export PID=$$  # 导出至子进程环境（注意：此 PID 在命令退出后立即失效，跨调用仅作命名标记）
   UUID=$(python3 -c "import uuid; print(uuid.uuid4().hex[:8])" 2>/dev/null || date +%s)
   export INCOGNITO_TMP_DIR="/tmp/hermes-incognito-${PID}-${UUID}"
   mkdir -p "$INCOGNITO_TMP_DIR" && chmod 700 "$INCOGNITO_TMP_DIR"
   echo "created=$(date +%s) session=${HERMES_SESSION_ID:-unknown}" > "$INCOGNITO_TMP_DIR/pid.lock"
   ```

3. **孤儿沙箱清理（时间戳 TTL，P0-2 修订）**：
   由于 Hermes `terminal()` 每次调用是独立 bash（命令退出后 PID 立即失效，`ps -p` 永远返回"不存活"），PID 锁校验在此运行时模型下不可用。改用**目录修改时间（mtime）**判定：删除 24 小时以上未修改的残留沙箱（正常无痕会话不会超过此 TTL；超时的必然是崩溃残留）。严禁递归扫描 `/mnt/`：
   ```bash
   python3 -c '
   import os, glob, shutil, time

   CUTOFF = time.time() - 86400  # 24 小时前

   for d in glob.glob("/tmp/hermes-incognito-*"):
       if not os.path.isdir(d):
           continue
       try:
           mtime = os.path.getmtime(d)
           if mtime < CUTOFF:
               shutil.rmtree(d)
               print(f"[Orphan Cleanup] 已删除 24h+ 残留沙箱: {os.path.basename(d)}")
       except Exception:
           pass
   ' 2>/dev/null || true
   ```

4. **声明 `/compress` 禁止落盘（P1-5）**：
   - 显式声明：无痕模式下禁用 `/compress` 会话摘要落盘，所有上下文压缩摘要仅保留在内存中。

5. **展示免责声明**（见上方 §免责声明）。

6. **Memory 基线快照（P1-7，供 Phase 4.3 比对）**：
   记录 `~/.hermes/memories/MEMORY.md` 和 `USER.md` 的 SHA-256 哈希值，作为后续审计的比对基线：
   ```bash
   python3 -c '
   import os, hashlib
   mem_dir = os.path.expanduser("~/.hermes/memories")
   with open(os.path.join(os.environ.get("INCOGNITO_TMP_DIR","/tmp"), "memory_baseline.txt"), "w") as out:
       for fname in ["MEMORY.md", "USER.md"]:
           fp = os.path.join(mem_dir, fname)
           if os.path.isfile(fp):
               h = hashlib.sha256()
               with open(fp, "rb") as f_in:
                   while True:
                       chunk = f_in.read(65536)
                       if not chunk: break
                       h.update(chunk)
               out.write(f"{fname}={h.hexdigest()}\n")
   ' 2>/dev/null || true
   ```

7. **进程基线快照（P1-8，供 Phase 4.5 比对）**：
   由于 Hermes `terminal()` 每次调用是独立 bash 进程（父进程立即退出），`ps --ppid` 不可用。改用**前后快照比对**：记录当前用户进程 PID 清单，Phase 4 时 diff 检测新增进程。
   ```bash
   ps -u "$USER" -o pid= --no-headers 2>/dev/null | sort > "$INCOGNITO_TMP_DIR/process_baseline.txt"
   ```

---

### Phase 2: 无痕隔离执行

- 所有文件写入必须在 `$INCOGNITO_TMP_DIR` 内。
- 所有终端命令按以下格式包装：
  - **简单命令**（单行、无控制流）→ 直接前缀：
    ```bash
    HISTFILE=/dev/null HISTSIZE=0 TMPDIR="$INCOGNITO_TMP_DIR" <COMMAND>
    ```
  - **复合语句**（`if`/`for`/`while`/`case`）→ 用 `bash -c` 包装，环境变量移入引号内分号分隔：
    ```bash
    bash -c 'HISTFILE=/dev/null HISTSIZE=0; if [ ... ]; then ...; fi'
    ```
  > ⚠️ **为什么不能直接前缀？** Bash 的 `VAR=val cmd` 语法仅对简单命令生效。`HISTFILE=/dev/null if ...` 会被解析器视为畸形简单命令，抛出 `syntax error near unexpected token 'then'`。复合语句必须通过 `bash -c` 或子 shell `(export VAR; if ...)` 间接设置环境变量。
  > ⚠️ **单引号 Python 脚本**：形如 `python3 -c '...'` 的命令使用单引号包裹 Python 代码。**切勿**对此类命令套 `bash -c '...'` 外层——外层单引号会与脚本内单引号冲突，导致字符串提前终结。此类命令是简单命令，直接前缀 `HISTFILE=/dev/null HISTSIZE=0` 即可。
  > ⚠️ **禁止过批**：不要将多个独立步骤塞进一个 `bash -c` 块。每个步骤使用独立的 `terminal()` 调用——嵌套引号在 Hermes `eval` 层会经历双重解析（见 §8.5），过批是引号逃逸地狱的最短路径。经验法则：一个 `bash -c` 块做一件事。
- **精准服务界定**：
  - 允许连接已存在的服务端口（例如 `127.0.0.1:6379`）。
  - 允许使用嵌入式数据库（SQLite），但创建的 `.db` 文件必须存放在沙箱内或登记至销毁清单。
  - **绝对禁止**启动新的守护进程/服务容器（如 `docker run`, `systemctl start`, `nohup` / `&` 后台进程）。
- 涉及网络的命令（curl/wget）不得写入 `~/.netrc` 或持久化凭据文件。
- 不得执行 `export VAR=val >> ~/.bashrc` 等持久化环境变量操作。

---

### Phase 3: 用户审计 / 转正 / 确认门禁

> 🚫 **Phase 2 → Phase 4 直接跳跃 = 违规。Agent 不得以"用户说完成了"为由跳过 Phase 3。**

1. **成果摘要与交互决策**：
   Agent 输出：
   > 📋 **无痕会话成果摘要**
   > - 临时沙箱：`$INCOGNITO_TMP_DIR`
   > - 产生文件：`[文件列表 + 大小]`
   > - 子代理：`[数量]`
   >
   > 请选择：
   > - 回复 `确认销毁` → 立即擦除所有痕迹
   > - 回复 `/incognito export <路径>` → 将指定成果转存到持久目录后擦除
   > - 回复 `/incognito abort` → 紧急跳过确认，立即强制销毁

2. **15 分钟 TTL 无交互自动超时销毁（P1-4, 降级为 Framework L2）**：
   - > ⚠️ **架构限制**：LLM Agent 为被动响应式模型，无法在纯 Skill 层实现后台倒计时。此 TTL 机制标记为 `[Framework L2]` 需求（见 §7），需 Hermes 宿主框架提供外部 Timer 支持。在框架支持之前，Agent 在 Phase 3 应主动提醒用户：「15 分钟内无响应，请手动执行 `/incognito abort` 销毁会话」。

---

### Phase 4: 反向全量审计 + 逐项清除 + 二次验证

> 🎯 **错误降级路径规范（P1-6）**：
> 每一个审计步骤显式遵循：
> `TRY`（尝试首选命令） → `ON FAILURE`（捕获失败） → `FALLBACK`（备用方案） → `REPORT`（记录 ⚠️/❌ 状态） → `CONTINUE`（**绝不阻塞后续步骤**）。

> 💡 **设计动机**：不信任 Phase 2 的隔离完美无缺——在用户发出结束指令后，主动检查每一条可能的持久化路径，发现痕迹 → 报告 → 清除 → 二次验证，形成完整闭环。

Agent 依次执行以下 10 步流水线：

> 📌 **包装规则速查**：以下命令块中——
> - 已含 `bash -c` 的 → 直接复制执行（4.2、4.5b、4.5c、4.6、Phase 5；Phase 4 环境恢复块）
> - 未含 `bash -c` 的 → 简单命令，执行时**直接前缀** `HISTFILE=/dev/null HISTSIZE=0` 即可，**切勿**额外套 `bash -c '...'` 外层（会与脚本内的单引号冲突）
> - ⚠️ **特别注意 4.3 和 4.8**：Python 脚本使用单引号包裹（`python3 -c '...'`），若被 Agent 误套 `bash -c '...'` 外层会导致引号冲突——外层单引号会提前终结。

> ⚠️ **Phase 4 环境前提（v2.4.1）**：所有审计命令依赖 `$INCOGNITO_TMP_DIR`。Phase 1 虽已 `export`，但长时间会话中变量可能丢失。Agent 在执行 Phase 4 **第一条 `terminal()` 命令**时必须先恢复变量——通过 pid.lock 中记录的 `session=<HERMES_SESSION_ID>` 精确定位本会话沙箱（无竞态、无需外部哨兵文件）。

```bash
bash -c 'HISTFILE=/dev/null HISTSIZE=0;
if [ -z "$INCOGNITO_TMP_DIR" ] || [ ! -d "$INCOGNITO_TMP_DIR" ]; then
  export INCOGNITO_TMP_DIR=$(grep -l "session=${HERMES_SESSION_ID}" /tmp/hermes-incognito-*/pid.lock 2>/dev/null | head -1 | xargs dirname 2>/dev/null)
fi'
```

> 后续每条命令正常前缀 `HISTFILE=/dev/null HISTSIZE=0`，`$INCOGNITO_TMP_DIR` 已在上一步导出到环境中。

#### 4.1 文件系统审计 — 非沙箱写入检测（限制 `-maxdepth 5` 平衡性能与覆盖）
检查 HOME 及项目目录中本会话期间新增/修改的文件（限制 `-maxdepth 5` 以平衡性能与覆盖；深度写入 `~/.config/<app>/deep/nested/` 可能漏检，属已知权衡）。**Hermes 框架自身的运行文件（state DB、日志、cron output、gbrain、openclaw 等）已预排除，避免噪音命中：**
```bash
find ~ -maxdepth 5 -newer "$INCOGNITO_TMP_DIR/pid.lock" \
  -not -path "*/\\.cache/*" -not -path "*/\\.local/share/*" \
  -not -path "$INCOGNITO_TMP_DIR/*" \
  -not -path "*/\\.bash_history" -not -path "*/\\.wget-hsts" \
  -not -path "*/\.hermes/state*" -not -path "*/\.hermes/logs/*" \
  -not -path "*/\.hermes/cron/output/*" -not -path "*/\.hermes/cron/jobs.json" \
  -not -path "*/\.hermes/cron/ticker_*" -not -path "*/\.hermes/cron/.tick.lock" \
  -not -path "*/\.hermes/cron/executions.db" \
  -not -path "*/\.hermes/state.db*" -not -path "*/\.hermes/skills/.usage.json" \
  -not -path "*/\.hermes/*gateway*" -not -path "*/\.hermes/channel_directory.json" \
  -not -path "*/\.hermes/.update_check" -not -path "*/\.hermes/models_dev_cache.json" \
  -not -path "*/\\.hermes/desktop/*" -not -path "*/\\.hermes/hermes-agent/*" \
  -not -path "*/\\.hermes/weixin/*" \
  -not -path "*/\\.hermes/cache/*" \
  -not -path "*/.local/state/tirith/*" \
  -not -path "*/\\.openclaw/*" -not -path "*/\\.gbrain/*" \
  -type f 2>/dev/null | head -50
```
- `TRY`: **先分类**：若命中文件为 Hermes/系统运行文件（state.db、logs、cron output、gbrain、openclaw 等）→ 标记为系统文件，**不删除**。仅对用户数据文件（文档、项目源码、配置等）执行 Python 覆写 + 删除。> ⚠️ 此分类依赖 Agent 推理判断，非脚本自动化。**主要防线是上方 `find` 的 `-not -path` 预排除列表**（已覆盖绝大多数已知系统路径）。Agent 分类仅作为二次安全网，处理预排除列表未覆盖的极少数漏网路径。
- `ON FAILURE`: 捕获权限错误或文件锁定。
- `FALLBACK`: 执行 `rm -f <file>`（仅限已确认的用户数据文件）。
- `REPORT`: 记录已清除或残留路径，区分「系统文件（已跳过）」和「用户数据（已清除）」。
- `CONTINUE`: 继续下一步。

##### 4.1b `/tmp/` 根目录审计 — 沙箱外写入检测
Phase 2 规则要求所有文件写入必须在 `$INCOGNITO_TMP_DIR` 内，但 Phase 4 的设计原则是**不信任 Phase 2**。此补充审计扫描 `/tmp/` 根目录（非沙箱路径），检测是否有绕过沙箱的泄漏写入：
```bash
find /tmp -maxdepth 1 -newer "$INCOGNITO_TMP_DIR/pid.lock" \
  -not -path "$INCOGNITO_TMP_DIR" -not -path "$INCOGNITO_TMP_DIR/*" \
  -not -name "hermes-incognito-*" \
  -type f 2>/dev/null | head -20
```
- `TRY`: 列出发现的非沙箱 `/tmp/` 文件，Python 覆写 + 删除。⚠️ **特别注意**：`hermes-snap-*.sh` 包含本会话全部命令明文（含环境变量中的 API key 等敏感信息），**不视为可跳过的系统文件**——必须与其他用户数据文件同等对待，执行完整覆写擦除。Agent 不得以"Hermes 系统文件"为由跳过。
- `ON FAILURE`: 权限错误则跳过。
- `FALLBACK`: `rm -f <file>`。
- `REPORT`: 记录已清除路径，区分「hermes-snap（已覆写）」和「其他泄漏文件」。
- `CONTINUE`: 继续下一步。

#### 4.2 Shell History 审计
检查并清除本会话期间的命令历史记录。（注：`sed` 仅清理含 `hermes-incognito` 字面量的行——覆盖范围受限于标记字符串。真正的防线是 Phase 2 每条命令强制 `HISTFILE=/dev/null` 前缀，确保命令不进入 history；此处 `sed` 为冗余安全网。）
```bash
bash -c 'HISTFILE=/dev/null HISTSIZE=0;
history -c 2>/dev/null || true
[ -f ~/.bash_history ] && sed -i "/^#.*hermes-incognito/d" ~/.bash_history 2>/dev/null || true
[ -f ~/.zsh_history ] && sed -i "/hermes-incognito/d" ~/.zsh_history 2>/dev/null || true
'
```
- `TRY`: `history -c` + `sed` 清理历史文件中的无痕会话命令。
- `FALLBACK`: 提示用户手动执行 `history -c`。
- `REPORT` & `CONTINUE`

#### 4.3 Memory 存储审计（Markdown 哈希比对）
Memory 存储后端为 Markdown 文件（`~/.hermes/memories/MEMORY.md` + `USER.md`），通过比对 Phase 1 记录的 SHA-256 基线哈希检测本次会话是否发生意外写入：
```bash
python3 -c '
import os, hashlib

tmp_dir = os.environ.get("INCOGNITO_TMP_DIR", "/tmp")
baseline_file = os.path.join(tmp_dir, "memory_baseline.txt")
baseline = {}
if os.path.isfile(baseline_file):
    with open(baseline_file) as f:
        for line in f:
            line = line.strip()
            if "=" in line:
                k, v = line.split("=", 1)
                baseline[k] = v

mem_dir = os.path.expanduser("~/.hermes/memories")
modified = []
for fname in ["MEMORY.md", "USER.md"]:
    fp = os.path.join(mem_dir, fname)
    if os.path.isfile(fp):
        h = hashlib.sha256()
        with open(fp, "rb") as f_in:
            while True:
                chunk = f_in.read(65536)
                if not chunk: break
                h.update(chunk)
        cur = h.hexdigest()
        old = baseline.get(fname)
        if old and cur != old:
            modified.append(f"{fname} (旧: {old[:8]}... → 新: {cur[:8]}...)")
        elif old:
            print(f"[Memory Audit] {fname} 未变更")
        elif os.path.isfile(fp):
            print(f"[Memory Audit] {fname} 无基线快照（首次运行？），跳过比对")

if modified:
    print("⚠️ Memory 文件已变更: " + "; ".join(modified))
    print("建议手动审查变更: cat ~/.hermes/memories/MEMORY.md")
else:
    print("✅ Memory 文件未变更，未检测到意外持久化写入")
' 2>/dev/null || echo "⚠️ Memory 审计脚本执行失败，请手动检查 ~/.hermes/memories/"
```

> ⚠️ **包装规则**：此 Python 脚本使用单引号包裹（`python3 -c '...'`）。执行时**直接前缀** `HISTFILE=/dev/null HISTSIZE=0` 即可，**切勿**套 `bash -c '...'` 外层——外层单引号会与脚本内单引号冲突导致语法错误。

- `TRY`: 比对 SHA-256 哈希，检测 Markdown 文件变更。
- `REPORT` & `CONTINUE`

#### 4.4 Skill / Cron 审计（限制 `-maxdepth 5` 平衡性能与覆盖）
检查无痕会话期间是否意外创建了技能文件或定时任务：
```bash
find ~/.hermes/skills/ -maxdepth 5 -newer "$INCOGNITO_TMP_DIR/pid.lock" -type f 2>/dev/null | grep -v "incognito-mode" | head -20
HISTFILE=/dev/null HISTSIZE=0 hermes cron list 2>/dev/null
```
- `TRY`: 对发现的新增 Skill 文件执行 `rm -f <file>`；对新建 Cron 执行 `hermes cron remove <job_id>`。
- `FALLBACK`: 列出需手动移除的文件路径和 Cron ID。
- `REPORT` & `CONTINUE`

#### 4.5 进程审计 — 已知限制 + `.python_history` 补充审计 + 子代理验证

> ⚠️ **设计失效声明（v2.4.0）**：原 PID 快照 diff 方案在此运行模型下**不可用**——Hermes `terminal()` 每次调用独立 bash 进程，Phase 4 执行 `ps` 快照时只能抓到 Phase 4 自身瞬间存在的并行终端调用，它们在你读到输出前就已退出。这不是 bug，是架构层面的设计失效。
>
> **替代策略**：真正的进程防线是 Phase 2「禁止启动新守护进程/服务」规则（`docker run`/`systemctl start`/`nohup`/`&`） + Phase 4.9 Session 容器销毁。本步骤降级为三项手工/补充检查：

##### 4.5a 手工进程提示
Agent 输出以下提示（不执行自动化扫描）：
> ℹ️ 自动进程检测在此运行模型下不可用。如本会话中你手动启动了后台进程、或使用过 Python REPL 交互，请执行 `ps -u $USER` 审查。真正的进程防线是 Phase 2 规则 + 4.9 Session 销毁。

##### 4.5b `.python_history` 审计
检查 Python REPL 历史文件是否在无痕会话期间被写入（此前为盲区）：
```bash
bash -c 'HISTFILE=/dev/null HISTSIZE=0;
if [ -f ~/.python_history ]; then
  ls -la ~/.python_history 2>/dev/null
  echo "⚠️ .python_history 存在——如本会话写入则需清理：rm -f ~/.python_history"
else
  echo "✅ 无 .python_history"
fi'
```
- `TRY`: 检查文件存在性。
- `FALLBACK`: 标记为已知盲区，提示手动清理命令。
- `REPORT` & `CONTINUE`

##### 4.5c 子代理沙箱验证
检查 Phase 2 期间委派的子代理是否在正确路径下创建了沙箱，确认将被 4.8 递归擦除覆盖：
```bash
bash -c 'HISTFILE=/dev/null HISTSIZE=0;
if [ -d "$INCOGNITO_TMP_DIR" ]; then
  SUB_COUNT=$(find "$INCOGNITO_TMP_DIR" -maxdepth 1 -type d -name "subagent_*" 2>/dev/null | wc -l)
  if [ "$SUB_COUNT" -gt 0 ]; then
    echo "📋 检测到 $SUB_COUNT 个子代理沙箱："
    find "$INCOGNITO_TMP_DIR" -maxdepth 1 -type d -name "subagent_*" -exec du -sh {} \; 2>/dev/null
    echo "✅ 以上子沙箱将被 4.8 递归安全擦除覆盖"
  else
    echo "✅ 无子代理沙箱，跳过"
  fi
else
  echo "⚠️ 沙箱目录不存在，跳过子代理审计"
fi'
```
- `TRY`: 列出子代理沙箱清单及其大小，确认在 `$INCOGNITO_TMP_DIR` 范围内。
- `ON FAILURE（安全扫描拦截）`: 标记 ⚠️ 跳过，4.8 的递归擦除仍会覆盖。
- `FALLBACK`: 沙箱缺失时跳过。
- `REPORT`: 记录子代理数量和路径。
- `CONTINUE`

#### 4.6 环境变量与配置审计
检查本会话是否意外修改了全局配置文件：
```bash
bash -c 'HISTFILE=/dev/null HISTSIZE=0;
grep -n "INCOGNITO\|hermes-incognito" ~/.bashrc ~/.zshrc ~/.profile ~/.env 2>/dev/null || echo "无配置变更"
git config --global --list 2>/dev/null | head -10
'
```
- `TRY`: 对发现的变更行执行 `sed -i '/INCOGNITO/d' <file>` 回滚。
- `FALLBACK`: 报告变更文件和行号，提示手动编辑。
- `REPORT` & `CONTINUE`

> ⚠️ 完成后额外执行：`unset INCOGNITO_MODE_ACTIVE INCOGNITO_TMP_DIR PID 2>/dev/null` 清除持久 Shell 环境变量残留。

#### 4.7 Git / 项目目录审计
在项目根目录（若有）检查未提交变更。若当前不在 Git 仓库中，此步自动跳过。
```bash
cd "${PROJECT_DIR:-$HOME}" 2>/dev/null && git status --short 2>/dev/null | head -20
```
- `TRY`: 收集未追踪及已修改文件清单。
- `FALLBACK`: 输出变更文件供用户确认（不自动不可逆回滚）。
- `REPORT` & `CONTINUE`

#### 4.8 沙箱物理安全擦除 (P0-3, Python 覆写安全擦除)
放弃依赖 Shell `shred`（防 WSL/NTFS 失效），优先采用 Python `os.urandom` 覆盖文件内容后再行解挂与删除：
```bash
python3 -c '
import os, sys

CHUNK_SIZE = 64 * 1024  # 64KB 分块，防大文件 OOM

def secure_wipe_file(file_path):
    """安全覆写单个文件：symlink直接unlink，正则文件分块覆写+fsync+truncate"""
    try:
        if os.path.islink(file_path):   # 符号链接直接删除，不追溯目标
            os.unlink(file_path)
            return
        if not os.path.isfile(file_path):
            if os.path.lexists(file_path):
                os.unlink(file_path)
            return
        os.chmod(file_path, 0o600)
        file_size = os.path.getsize(file_path)
        if file_size > 0:
            with open(file_path, "rb+", buffering=0) as f:
                fd = f.fileno()
                f.seek(0)
                remaining = file_size
                while remaining > 0:
                    chunk = min(CHUNK_SIZE, remaining)
                    written = f.write(os.urandom(chunk))
                    if written == 0:
                        break
                    remaining -= written
                f.flush()
                os.fsync(fd)
                f.seek(0)
                f.truncate(0)
                f.flush()
                os.fsync(fd)
        os.unlink(file_path)
    except Exception:
        if os.path.lexists(file_path):
            try: os.unlink(file_path)
            except Exception: pass

tmp_dir = os.environ.get("INCOGNITO_TMP_DIR")
if tmp_dir and os.path.exists(tmp_dir) and tmp_dir.startswith("/tmp/hermes-incognito-"):
    for root, dirs, files in os.walk(tmp_dir, topdown=False):
        for file in files:
            secure_wipe_file(os.path.join(root, file))
        for d in dirs:
            dp = os.path.join(root, d)
            try:
                if os.path.islink(dp): os.unlink(dp)
                else: os.rmdir(dp)
            except Exception: pass
    try:
        os.rmdir(tmp_dir)
        print("✅ 沙箱安全覆写并物理擦除")
    except Exception as e:
        print(f"⚠️ 沙箱目录删除失败: {e}")
' 2>/dev/null || (find "$INCOGNITO_TMP_DIR" -type f -exec shred -u -n 1 {} + 2>/dev/null; rm -rf "$INCOGNITO_TMP_DIR")
```

> ⚠️ **包装规则**：此 Python 脚本使用单引号包裹（`python3 -c '...'`）。执行时**直接前缀** `HISTFILE=/dev/null HISTSIZE=0` 即可，**切勿**套 `bash -c '...'` 外层——外层单引号会与脚本内单引号冲突导致语法错误。

- `TRY`: Python `os.urandom` 覆写 + `rmdir`。
- `FALLBACK`: Shell `find -type f -exec shred` 降级擦除 + `rm -rf`。

#### 4.9 Session 容器销毁（延后步骤）— 防线 2 最后一击

> ⚠️ **此步骤为延后执行标记**：实际 `hermes sessions delete` 命令在 Phase 5 审计报告输出完毕后执行（见 Phase 5「最后一步」），不在此处运行。此处仅作占位，确保 Agent 知晓此步骤存在。
>
> ⚠️ 4.10 二次验证仅覆盖 4.1-4.8（文件/History/Memory/Skill/进程/配置/Git/沙箱），不验证 4.9 的 Session 删除（因为此时尚未执行，命令延后至 Phase 5）。

- `TRY`: Phase 5 报告后执行（命令见 Phase 5「最后一步」）。
- `FALLBACK`: 提示手动删除命令。
- `REPORT`: Phase 5 报告中记录执行结果。
- `CONTINUE`

#### 4.10 二次验证 — 逐项复查
对 4.1-4.9 中标记为 ⚠️ 的项目再次运行校验逻辑。
- `TRY`: 确认无残留。
- `FALLBACK`: 标记 ❌ 告警。
- `REPORT` & `CONTINUE`

---

### Phase 5: 全量审计报告 + 最终确认回执

Agent 汇总 Phase 4 的 10 步审计结果：

> 🧹 **无痕模式结束 — 全量审计报告 (v2.4.0)**
>
> | 审计项 | 状态 | 详情 |
> |------|:--:|------|
> | 4.1 文件系统（HOME + /tmp/） | ✅/⚠️/❌ | [发现 N 个文件，已清除/残留路径] |
> | 4.2 Shell History | ✅/⚠️/❌ | [增量行数，已清除/残留行号] |
> | 4.3 Memory 存储 | ✅/⚠️/❌ | [SHA-256 哈希比对，文件未变更/已变更需审查] |
> | 4.4 Skill / Cron | ✅/⚠️/❌ | [新增/修改数量，已删除/需手动] |
> | 4.5 进程/python_history/子代理 | ✅/⚠️/❌ | [手工提示已输出/.python_history状态/子代理N个] |
> | 4.6 环境变量 / 配置 | ✅/⚠️/❌ | [被修改的文件，已回滚/需手动] |
> | 4.7 Git / 项目目录 | ✅/⚠️/❌ | [未追踪变更列出，未自动回滚] |
> | 4.8 沙箱物理安全擦除 | ✅/❌ | [Python os.urandom 覆写 + 物理删除完成] |
> | 4.9 Session 容器销毁 | ✅/⚠️/❌ | [会话已删除/需手动执行 hermes sessions delete] |
> | 4.10 二次验证 | ✅/❌ | [N 项复查通过/M 项仍有残留] |
>
> **综合判定**：
> - 🟢 全部通过 → 🔒 无痕会话已关闭，零痕迹残留
> - 🟡 有 ⚠️ 项（已清除）→ 🔒 已清除，会话关闭
> - 🔴 有 ❌ 项（无法自动清除）→ ⚠️ 以下残留需手动处理：[详细路径和命令]

**Agent 执行指令（4.9 实际执行）**：以上报告输出完毕后，Agent 必须**立即执行**以下命令销毁 Session 容器——这是自动化步骤，不是输出给用户的模板文本：

```bash
bash -c 'HISTFILE=/dev/null HISTSIZE=0; if [ -n "$HERMES_SESSION_ID" ]; then
  hermes sessions delete --yes "$HERMES_SESSION_ID" 2>/dev/null \
    && echo "✅ Session 容器已销毁" \
    || echo "⚠️ Session 删除失败（可手动执行 hermes sessions delete）"
else
  echo "ℹ️ 无 HERMES_SESSION_ID，跳过容器销毁"
fi'
```


---

## 3. 子代理无痕传染协议 (P2-7)

**允许使用 `delegate_task`**。在子代理调用前，必须严格执行以下隔离规程：

### 3.1 强制注入无痕元指令

子代理 Prompt **首行**必须注入（复制此模板，替换 `{PLACEHOLDER}`）：

```
[CRITICAL INCOGNITO INHERITANCE — DO NOT REMOVE]
INCOGNITO_MODE: ACTIVE
SANDBOX_DIR: {INCOGNITO_TMP_DIR}/subagent_{PID}_{SUB_UUID}/
FORBIDDEN_TOOLS: memory, skill_manage(create/edit), cronjob(create), kanban(create/update)
TERMINAL_PREFIX: HISTFILE=/dev/null HISTSIZE=0
RULES:
- Do NOT call memory tool (any action).
- Do NOT create/edit skills or cron jobs.
- ALL file writes MUST go to SANDBOX_DIR.
- Do NOT write to HOME, project dirs, or /var/log.
- Prefix ALL shell commands with TERMINAL_PREFIX.
- Do NOT start new background services/containers (nohup, systemctl, docker).
- Do NOT write /compress summary to disk.
- Your task context and temp files will be DESTROYED after this session.
[/CRITICAL INCOGNITO INHERITANCE]
```

### 3.2 分配 PID/UUID 级独立子沙箱

```bash
SUB_UUID=$(python3 -c "import uuid; print(uuid.uuid4().hex[:8])" 2>/dev/null || date +%s)
SUB_DIR="$INCOGNITO_TMP_DIR/subagent_${PID}_${SUB_UUID}"
mkdir -p "$SUB_DIR" && chmod 700 "$SUB_DIR"
echo "created=$(date +%s) session=${HERMES_SESSION_ID:-unknown}" > "$SUB_DIR/pid.lock"
```

### 3.3 主代理递归清理

Phase 4 物理擦除时，主代理通过递归遍历擦除 `$INCOGNITO_TMP_DIR` 下包含所有子代理沙箱在内的所有子目录。

---

## 4. 运行时交互指令

在无痕模式激活期间，Agent 必须响应以下中途命令：

| 指令 | 行为 |
|------|------|
| `/incognito status` | 列出当前沙箱路径、占用空间、子代理数量、已产生的文件清单 |
| `/incognito audit` | 同 status，额外列出近期 terminal 命令摘要和 web 请求记录 |
| `/incognito export <path>` | 将沙箱中指定文件/目录安全复制到用户指定的持久路径，二次确认后放行 |
| `/incognito abort` | 立即中断当前任务，跳过 Phase 3 确认，直接进入 Phase 4 强制擦除 |

---

## 5. 已知泄漏边界与 OS 内核级盲区（Agent 无法控制）

以下路径**不在本技能控制范围内**，Agent 必须在 Phase 1 免责声明中告知用户：

### OS 内核级

| 盲区 | 泄漏机制 | 用户侧缓解措施 |
|------|---------|--------------|
| **Swap 分区** | OOM 时内存页写入 `/swapfile`，包含明文敏感数据 | `swapoff -a`（需 root），或确保足够 RAM |
| **Core Dump** | 进程崩溃时内存快照写入 `/var/coredump/` | `ulimit -c 0` |
| **Syslog / Journald** | `logger` 命令、系统调用被 `auditd` 记录 | 不调用 `logger`；auditd 需 root 配置 |
| **文件系统 atime** | `read_file` 更新文件访问时间戳 | `mount -o noatime`（需 root） |
| **IDE / Git Watcher**| VSCode LSP、Git fsmonitor 捕获文件变动 | 关闭 IDE 文件自动监视 |
| **已存在服务日志**| Redis/PostgreSQL/Docker 自身连接日志 | 不在无痕模式下发送敏感业务日志 |

### 网络/云端级

| 盲区 | 泄漏机制 |
|------|---------|
| **LLM API 厂商日志** | OpenAI/Anthropic/Google 默认保留 30 天 Prompt/Completion 日志 |
| **DNS 查询** | 即使 HTTPS，DNS 解析记录在递归解析器和 ISP |
| **企业代理/防火墙** | 公司网络层面的流量日志和 TLS 拦截 |

---

## 6. 反模式黑名单 (Anti-Patterns Blacklist)

以下行为在无痕模式下**绝对禁止**——Agent 不得以任何理由破例：

- ❌ **启动守护进程/服务容器**：通过 `docker run`, `nohup`, `&`, `systemctl start` 启动新后台服务
- ❌ **持久化 `/compress` 摘要**：将压缩摘要落盘写入文件
- ❌ **事后擦除侥幸**：把文件写到 `~` 或项目目录，认为"Phase 4 会删"
- ❌ **破例存记忆**：调用 `memory` 工具记录敏感信息
- ❌ **绕过沙箱**：写到 `/tmp/` 根目录或 `/var/tmp/` 等非 `$INCOGNITO_TMP_DIR` 路径
- ❌ **环境变量泄密**：通过 `export` 或修改 `~/.bashrc` 持久化敏感值
- ❌ **第三方 Webhook**：通过外部 API 将数据转发至外部
- ❌ **创建临时 Skill**：调用 `skill_manage create`
- ❌ **创建 Cron 清理任务**：通过 `cronjob create` 创建延迟清理任务
- ❌ **系统剪贴板**：通过 `xclip`/`pbcopy` 写入共享剪贴板

---

## 7. 框架层 TODO（需 Hermes 版本升级支持）

以下需求标记为 `[Framework L2]`—— Agent 无法在 Skill 层面实现，需底层支持：

- [ ] **`[Framework L2]` TTL Auto-Destroy Timer**：宿主框架提供外部 Timer，Phase 3 超时后自动触发 Phase 4 销毁（LLM Agent 不具备自发唤醒能力）
- [ ] **`[Framework L2]` Ephemeral Session Store**：内存型 Session 引擎，绕过 SQLite WAL 日志和 `.sqlite` 物理文件
- [ ] **`[Framework L2]` Auto-mount tmpfs**：启动无痕模式时自动挂载 `/dev/shm/hermes-session-<ID>/`，OS 关机/进程终止时物理消失
- [ ] **`[Framework L2]` CLI Flag `hermes --incognito`**：框架原生无痕模式，自动禁用 Checkpoint、Transcript 归档、RAG 向量索引
- [ ] **`[Framework L2]` ZDR Header Injection**：向 LLM API 请求自动附加 `X-Zero-Data-Retention: true`
- [ ] **`[Framework L2]` Subagent Inheritance**：子代理自动继承父代理的无痕模式标记，无需手动注入 header

---

## 8. 注意事项与错误降级

1. **显式错误降级 (TRY-FALLBACK)**：所有审计与清理动作均具备容错机制。单步（如锁文件或 WSL 缺失 shred）失败时触发 FALLBACK，记录报告并 CONTINUE 执行后续清理与 Session 销毁。
2. **会话删除 best-effort**：若 Session 容器销毁受阻，Phase 5 输出具体的 `hermes sessions delete <ID>` 提示人工执行。
3. **15 分钟 TTL 无交互提醒**：Phase 3 停滞 15 分钟无回复，Agent 应提醒用户手动执行 `/incognito abort`。自动销毁需 `[Framework L2]` 外部 Timer 支持（见 §7）。
4. **时间戳 TTL 孤儿清理**：Phase 1 步骤 3 使用目录 mtime + 24h TTL 替代 PID 锁校验（PID 在 Hermes `terminal()` 独立 bash 模型下不可用）。此机制保守且安全——正常无痕会话不会超过 24 小时，超时必然是崩溃残留。
5. **嵌套脚本引号策略**：当命令包含三层嵌套（bash → Python → 字符串）时，统一使用 `bash -c '...python3 -c "..."'` 模式。外层单引号保护 `$` 和反斜杠不被 bash 展开，内层 Python 双引号内的 `\\\"` 还原为字面引号。禁止混用单双引号导致字符串截断或语法错误。示例：
   ```bash
   bash -c 'python3 -c "
   import os
   path = os.environ.get(\"VAR_NAME\", \"default\")
   print(f\"value={path}\")
   "'
   ```

   > ⚠️ **Hermes `terminal()` eval 双重解析陷阱（v2.4.1）**：Hermes 的 `terminal()` 底层使用 `eval` 执行命令字符串，在传给 bash 之前会**额外剥一层引号**。这导致一个隐蔽的坑：
   >
   > ```bash
   > bash -c 'python3 -c "print(f'"'"'x'"'"')"'
   > #                         ↑ eval 把这里的 ' 当成 bash -c 字符串结束符
   > #                           → bash 收到截断的命令 → 语法错误
   > ```
   >
   > **Python f-string 中禁止使用单引号**（`f'...'`）——在 `bash -c '...'` 外层单引号 + Hermes `eval` 的双重解析下，`f'` 的 `'` 会提前终结外层字符串。必须使用 `f"..."` 双引号。

   > ⚠️ **禁止过批（Anti-Batching）**：不要将多个独立步骤塞进一个 `bash -c` 块。每个 Phase 1 步骤应使用**独立的 `terminal()` 调用**——这不仅避免了嵌套引号逃逸的地狱，还让每个步骤的错误隔离、可独立重试。经验法则：只要一个 `bash -c` 块内同时包含 Phase 1 的步骤 2 + 步骤 3 + 步骤 6，就已经过批了。
