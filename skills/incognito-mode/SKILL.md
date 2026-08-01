---
name: incognito-mode
description: "Incognito v2.6.0: sandboxed privacy, log redaction, audit."
version: 2.6.0
author: 幻灭文学出版社 + Hermes (18-round cross-audited, process audit degraded, cache+tmp+subagent gaps closed)
license: MIT
platforms: [linux, macos]
metadata:
  hermes:
    tags: [privacy, session, cleanup, ephemeral, sandbox, isolation]
---

# 无痕模式 (Incognito Mode) v2.6.0

浏览器无痕模式的 Hermes 升级版。**四层纵深防御**（Skill 策略 → Runtime 护栏 → Framework 支持 → OS 隔离），确保会话结束后无持久痕迹残留。

> **设计原则**：v1.0「事后擦除」→ v2.0「事前隔离」→ v2.1「事前隔离 + 事后全量反向审计」→ v2.2+「稳健性硬化」→ v2.5+「对照真实架构深度审计修复（R4-R10：时间戳窗口日志清洗、delegation/async_delegations 审计、向量索引哨兵、RedactingFilter 插件、bash 单引号安全）」→ v2.6.0「R11 双缺陷修复（4.9r 异常关闭补救协议 + serve 后端插件加载核心 patch）」。完整版本 Changelog 见仓库 README。**不信任 Phase 2 的隔离完美无缺——发现痕迹 → 报告 → 清除 → 二次验证。**
> ⚡ **三条铁律（每次行动前自检）**：
> 1. **每条 terminal 命令** → 前缀 `HISTFILE=/dev/null HISTSIZE=0`，无一例外。**复合语句（`if`/`for`/`while`/`case`）必须用 `bash -c '...'` 包装**——直接前缀 `HISTFILE=/dev/null if ...` 会导致 bash 语法错误
> 2. **从 Phase 2 到 Phase 4** → 必须经过 Phase 3 用户确认门禁，不得跳跃。（TTL 自动销毁需 Framework L2 支持，见 §7）
> 3. **每次 delegate_task** → 允许使用，但必须透传 incognito header + 分配独立 PID/UUID 级子沙箱

---

## 免责声明（Phase 1 必须展示）

激活时 Agent 必须向用户展示：

> 🔒 **无痕模式 v2.6.0 已激活**
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
| **brain 写入类**（brain add/update/ingest 等 gbrain 写操作） | 所有写入 action | gbrain 是持久知识库（`~/.gbrain/brain.pglite`），无痕内容一旦写入永久残留且无审计路径（v2.5.3，R4 审计 P1-3） |
| `terminal(background=true)` | 任何后台进程 | 进程在会话结束后继续运行，残留无法随 Phase 4 清理（v2.5.3） |
| **创建新服务/容器** | 执行 `docker run` / `docker-compose up` / `systemctl start\|enable` / `nohup` / `tmux` / `screen` / `&` 后台守护进程 | 防止派生沙箱外持久进程和日志 |

### ⚠️ 严格受限

| 工具 / 行为 | 限制规则 |
|------------|---------|
| `write_file` / `patch` | **仅限** Phase 1 创建的随机沙箱目录 `$INCOGNITO_TMP_DIR`。严禁写入 HOME、项目目录、`~/.hermes/`、`/var/log/`。新建 SQLite `.db` 必须登记在销毁清单中 |
| `terminal` | 每条命令**必须**前缀 `HISTFILE=/dev/null HISTSIZE=0`。禁止 `pip install`（除非 `--target` 指向沙箱）、`git config --global`。禁止写入 `/var/log/` 或调用 `logger`。**精准服务界定**：允许连接已存在的服务端口（如 `127.0.0.1:6379`）或使用嵌入式数据库（SQLite），严禁创建新守护进程 |
| `delegate_task` | **允许使用**，但必须在子代理 Prompt 首行注入无痕协议头（见 §3），分配 PID/UUID 级独立子沙箱，主代理负责递归清理。⚠️ 子代理 live transcripts 由 4.1a 专项审计覆写（v2.5.3） |
| `web_search` / `web_extract` | 允许使用。Hermes 框架会自动缓存结果至 `~/.hermes/cache/web/`——Skill 层无法拦截。Phase 1 已将 `cache/web/`、`screenshots/`、`videos/`、`images/`、`audio/`、`documents/`、`vision/` 通过符号链接重定向至沙箱，Phase 4.7b 恢复后由 4.8 递归擦除。web_search 查询明文会在 agent.log 中留存，Phase 4.6b 按**行首时间戳窗口**（pid.lock `created=` 之后）清洗脱敏为 `[REDACTED_INCOGNITO_QUERY]`（v2.5.3 修正：日志行无 session ID，不再锚定 sid）。 |
| `agy` CLI | **严格受限**：仅允许只读分析（`-p` 无写盘场景）。禁止 `--add-dir` 指向真实数据目录做写入类任务。agy 的对话记录与 OAuth token 明文位于 `~/.gemini/antigravity-cli/`（4.1 已排除防误删，但无痕期间应避免调用 agy 写盘）（v2.5.3） |
| `execute_code` | **严格受限**：仅允许临时计算（不写盘）。禁止写 `~`、项目目录、`$INCOGNITO_HERMES_HOME` 之外路径（v2.5.3） |
| `/compress` | **禁止落盘**。无痕期间允许在内存中压缩上下文，但严禁将摘要持久化写入磁盘文件 |

### ✅ 自由使用（只读，不触发持久化）

`read_file`、`search_files`、`session_search`（仅读取历史，不索引当前会话）、`skill_view`、`clarify`

> ⚠️ **session_search 说明修正（v2.5.3）**：Hermes 的 `session_search` 是 **SQLite FTS5 本地查询**（SessionDB），本身不触发 RAG/向量 Embedding。向量索引由独立 cron（`index_all.py`）控制——无痕会话通过**哨兵文件**（Phase 1 步骤 3.5 创建 `/tmp/.hermes-incognito-active`，Phase 5 删除；v2.5.5 起 index_all.py 只跳过 active_sessions 源，4.9 删 session 即隐式释放）排除。

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
   ├ 4.1 文件系统审计 + /tmp/根目录  ├ 4.1a delegation transcripts 审计
   ├ 4.1c interrupted_turns 审计
   ├ 4.2 Shell History 审计
   ├ 4.3 Memory 审计 (SHA-256 哈希比对)├ 4.4 Skill/Cron 审计
   ├ 4.5 进程/python_history/子代理审计 ├ 4.6 环境变量/配置审计
   ├ 4.6b agent.log 清洗           ├ 4.6c hermes-snap 二次清除
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

> ⚠️ **环境变量持久化约束（v2.5.1）**：Hermes `terminal()` 的持久化环境变量仅在**简单命令**（不套 `bash -c`）中设置的 `export` 才能跨调用保留。`bash -c '...'` 是子 shell，其内部的 `export` 退出即失。因此步骤 1 必须拆分为两次 `terminal()` 调用：
> 1. **先**用简单命令链 `export` 变量（不加 `bash -c`，用 `;` 串联多个简单命令）
> 2. **再**用 `bash -c` 执行幂等校验 + 目录创建（此时 `$INCOGNITO_TMP_DIR` 已在父环境中可访问）

1. **变量导出与幂等校验（P0-1，拆分为两次 terminal 调用）**：

   **Terminal 调用 1**：先导出变量（简单命令链，不加 `bash -c`，确保持久化）：
   ```bash
   HISTFILE=/dev/null HISTSIZE=0; export INCOGNITO_MODE_ACTIVE=true; export PID=$$; export UUID=$(python3 -c "import uuid; print(uuid.uuid4().hex[:8])" 2>/dev/null || date +%s); export INCOGNITO_TMP_DIR="/tmp/hermes-incognito-${PID}-${UUID}"; export INCOGNITO_HERMES_HOME="${HERMES_HOME:-$HOME/.hermes}"; echo "INCOGNITO_TMP_DIR=$INCOGNITO_TMP_DIR"; echo "INCOGNITO_HERMES_HOME=$INCOGNITO_HERMES_HOME"
   ```
   > 使用 `;` 串联多个**简单命令**——每条都是独立简单命令，`export` 在同一个 shell 进程内生效，变量持久化到后续 `terminal()` 调用。

   **Terminal 调用 2**：幂等校验 + 沙箱创建（用 `bash -c` 包裹 `if` 判断）：
   ```bash
   HISTFILE=/dev/null HISTSIZE=0 bash -c '
   # 幂等校验：通过 pid.lock 中的 session= 字段检测本会话是否已有沙箱
   EXISTING=$(grep -l "session=${HERMES_SESSION_ID:-unknown}" /tmp/hermes-incognito-*/pid.lock 2>/dev/null | head -1 | xargs dirname 2>/dev/null)
   if [ -n "$EXISTING" ] && [ -d "$EXISTING" ]; then
     echo "NESTED_IDEMPOTENT: 已存在沙箱=$EXISTING"
     exit 0
   fi
   # 创建沙箱目录（$INCOGNITO_TMP_DIR 由 Terminal 调用 1 持久化到父环境，子 shell 继承）
   mkdir -p "$INCOGNITO_TMP_DIR" && chmod 700 "$INCOGNITO_TMP_DIR"
   echo "created=$(date +%s) session=${HERMES_SESSION_ID:-unknown}" > "$INCOGNITO_TMP_DIR/pid.lock"
   echo "沙箱已创建: $INCOGNITO_TMP_DIR"
   echo "SESSION_ID=${HERMES_SESSION_ID:-unknown}"
   echo "READY"
   '
   ```
   > 幂等校验改用 pid.lock 中的 `session=` 字段匹配（替代原先的环境变量 `INCOGNITO_MODE_ACTIVE` 检查），因为后者在 `bash -c` 子 shell 中本就不可靠。这也是 Phase 4 环境恢复使用的同一机制。

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

   # v2.5.3：同步清理超龄哨兵（防崩溃残留哨兵锁死向量索引 cron）
   SENTINEL = "/tmp/.hermes-incognito-active"
   if os.path.isfile(SENTINEL) and os.path.getmtime(SENTINEL) < CUTOFF:
       try:
           os.unlink(SENTINEL)
           print("[Orphan Cleanup] 已删除 24h+ 残留哨兵")
       except Exception:
           pass
   ' 2>/dev/null || true
   ```

3.5. **向量索引哨兵创建（v2.5.5 语义重构，R6）**：
   创建 `/tmp/.hermes-incognito-active` 哨兵，**防止无痕会话内容被嵌入持久向量库**（`~/.hermes/vector_store.db` + `vector_index.faiss`）。
   > ⚠️ **v2.5.5 语义（采纳用户思路）**：哨兵不再"一刀切阻塞"向量任务——index_all.py 检测到哨兵时**只跳过 `active_sessions` 索引源**（无痕内容唯一可能进入的源），memory/skills/jsonl 源照常。且**4.9 删除 session 即隐式释放哨兵**（index_all.py 检查 state.db 中无痕 session 是否存在：已删 → 内容已销毁 → 自动 unlink 哨兵并正常索引；还在 → 无痕内容可能活跃 → 跳过）。Phase 5 显式清理保留（最佳实践）；崩溃残留由步骤 3 的 24h TTL 兜底。
   ```bash
   HISTFILE=/dev/null HISTSIZE=0; echo "session=${HERMES_SESSION_ID:-unknown} created=$(date +%s)" > /tmp/.hermes-incognito-active
   ```

3.6. **Logging RedactingFilter 插件检查（v2.6.0 环境感知升级，R11 根因实证）**：
   确认 `$INCOGNITO_HERMES_HOME/plugins/incognito-log-filter/` 存在且启用——它是"事前拦截"第一层防线（哨兵门控 + 内存脱敏，明文不落盘），缺失时仅剩 4.6b 事后擦除。**软警告不阻塞**（Phase 4 照常执行）。
   > 🔴 **R11 实证（2026-08-01）**：`serve` 后端（desktop 的 backend）启动时 `_plugin_cli_discovery_needed()` 对内置子命令返回 False → `discover_plugins()` 从不执行 → **用户插件在 serve/gateway 模式完全不加载**（filter 永不挂载，无痕期间明文照常落盘）。Hermes 核心修复前，本检查必须同时判定**运行环境**：
   ```bash
   HISTFILE=/dev/null HISTSIZE=0 bash -c '
   # 环境感知：检测当前进程是否为 serve/gateway 后端（此类进程不加载用户插件）
   PROC_ENV=$(ps -eo args | grep -E "hermes_cli\.main (serve|gateway)" | grep -v grep | head -1)
   if [ -n "$PROC_ENV" ]; then
     echo "⚠️ 当前环境为 serve/gateway 后端——用户插件不加载（Hermes main.py _plugin_cli_discovery_needed 设计取舍），filter 未生效，本次会话仅 4.6b 事后擦除兜底。核心修复见 §7"
   elif [ -f "$INCOGNITO_HERMES_HOME/plugins/incognito-log-filter/plugin.yaml" ]; then
     STATUS=$(hermes plugins list 2>/dev/null | grep "incognito-log-filter" | grep -c "enabled")
     if [ "$STATUS" -gt 0 ]; then
       echo "✅ incognito-log-filter 已启用（CLI/TUI 环境，register 在进程启动时执行）"
     else
       echo "⚠️ incognito-log-filter 已安装但未启用 → 执行: hermes plugins enable incognito-log-filter"
     fi
   else
     echo "⚠️ incognito-log-filter 未安装 → 建议安装（v2.5.6 起的事前拦截防线，插件仓库路径见 §7）"
   fi'
   ```
   > ⚠️ **生效条件**：插件 `register()` 在 Hermes 进程启动时执行——若当前进程未重启，filter 尚未生效，本次会话仅 4.6b 事后擦除兜底（声明即可，不阻塞）。

4. **声明 `/compress` 禁止落盘（P1-5）**：
   - 显式声明：无痕模式下禁用 `/compress` 会话摘要落盘，所有上下文压缩摘要仅保留在内存中。

5. **展示免责声明**（见上方 §免责声明）。

6. **Memory 基线快照（P1-7，供 Phase 4.3 比对）**：
   记录 `~/.hermes/memories/MEMORY.md` 和 `USER.md` 的 SHA-256 哈希值，作为后续审计的比对基线：
   ```bash
   python3 -c '
   import os, hashlib
   mem_dir = os.path.join(os.environ.get("INCOGNITO_HERMES_HOME", os.path.expanduser("~/.hermes")), "memories")
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

8. **用户数据缓存重定向至沙箱（P1-9）**：
   将 `~/.hermes/cache/` 下的用户数据子目录（`web/`、`screenshots/`、`videos/`、`images/`、`audio/`、`documents/`、`vision/`）通过**符号链接**重定向至沙箱。无痕期间 Hermes 框架的所有 `web_search`/`web_extract` 缓存、浏览器截图、图片/音频/视频/文档缓存将自动落入 `$INCOGNITO_TMP_DIR`，随 Phase 4.8 递归擦除一并销毁——无需单独审计比对。含自愈逻辑处理上次崩溃残留与悬空链接。

   > ⚠️ **v2.5.3 扩展（R4 审计 P1-1）**：原版只覆盖 web/screenshots/videos 3 目录，遗漏 images/audio/documents/vision（image_gen、TTS、文档处理缓存）。`delegation/` 因 Hermes 远程 backend 挂载只读语义**不重定向**——改由 4.1 专项审计覆盖（见 4.1 附加步骤）。并发无痕会话会导致 `.incognito-bak` 同名冲突——**不支持并发无痕会话**。

   ```bash
   # P1-9: 用户数据缓存重定向至沙箱（符号链接隔离，v2.5.3 扩展 7 目录）
   for dir in web screenshots videos images audio documents vision; do
       CACHE_DIR="$INCOGNITO_HERMES_HOME/cache/$dir"
       BAK_DIR="$INCOGNITO_HERMES_HOME/cache/${dir}.incognito-bak"
       
       # 自愈 1：恢复上次无痕会话崩溃残留的符号链接（BAK 存在）
       if [ -L "$CACHE_DIR" ] && [ -d "$BAK_DIR" ]; then
           rm "$CACHE_DIR"
           mv "$BAK_DIR" "$CACHE_DIR"
       fi
       # 自愈 2：悬空符号链接（崩溃 + 沙箱已被 24h TTL 清理，BAK 不存在）→ 直接删除
       if [ -L "$CACHE_DIR" ] && [ ! -e "$CACHE_DIR" ]; then
           rm "$CACHE_DIR"
       fi
       
       # 重定向：将真实目录备份，替换为指向沙箱的符号链接
       # v2.5.3 修正：仅当 BAK 不存在才 mv——防止重复 Phase 1 产生 BAK/CACHE_DIR 嵌套
       if [ -d "$CACHE_DIR" ] && [ ! -L "$CACHE_DIR" ] && [ ! -e "$BAK_DIR" ]; then
           mv "$CACHE_DIR" "$BAK_DIR"
           mkdir -p "$INCOGNITO_TMP_DIR/cache_$dir"
           ln -s "$INCOGNITO_TMP_DIR/cache_$dir" "$CACHE_DIR"
       elif [ -d "$CACHE_DIR" ] && [ ! -L "$CACHE_DIR" ] && [ -e "$BAK_DIR" ]; then
           echo "⚠️ 异常：$dir 真实目录与 BAK 并存，需人工处理（并发无痕会话？）"
       fi
   done
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

> ⚠️ **Phase 4 环境前提（v2.5.3 修正，R4 审计 P0-4）**：长时间会话中变量可能因 Hermes 进程重启等原因丢失。Agent 在执行 Phase 4 **第一条 `terminal()` 命令**时必须先恢复变量。**注意：`bash -c` 子 shell 内的 `export` 不持久**（Hermes 只持久化**简单命令**的 export——实测确认），因此恢复必须**拆两次调用**：

**调用 A**（bash -c 计算沙箱路径，输出**纯路径**，禁止附加任何其他文本）：
```bash
HISTFILE=/dev/null HISTSIZE=0 bash -c '
if [ -n "$INCOGNITO_TMP_DIR" ] && [ -d "$INCOGNITO_TMP_DIR" ]; then
  echo "$INCOGNITO_TMP_DIR"
else
  grep -l "session=${HERMES_SESSION_ID:-unknown}" /tmp/hermes-incognito-*/pid.lock 2>/dev/null | head -1 | xargs dirname 2>/dev/null
fi'
```

**Agent 校验**：调用 A 的输出必须**非空且匹配 `^/tmp/hermes-incognito-`**（防空赋值、防 `session=unknown` 误配残留沙箱）。校验失败 → **中断 Phase 4**，明确报错「沙箱路径无法恢复，请手动检查 /tmp/hermes-incognito-*」。

**调用 B**（简单命令 export，持久化——**禁止套 `bash -c`**；同时恢复 `INCOGNITO_HERMES_HOME`，v2.5.4）：
```bash
HISTFILE=/dev/null HISTSIZE=0; export INCOGNITO_TMP_DIR=<调用A的校验后输出>; export INCOGNITO_HERMES_HOME="${HERMES_HOME:-$HOME/.hermes}"
```

> 后续每条命令正常前缀 `HISTFILE=/dev/null HISTSIZE=0`，`$INCOGNITO_TMP_DIR` 已持久化到环境中。若变量未丢失，调用 A 直接输出当前值，调用 B 原样回写。

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
  -not -path "*/.openclaw/*" -not -path "*/.gbrain/*" \
  -not -path "*/.hermes/sessions/*" -not -path "*/.hermes/memories/*" \
  -not -path "*/.hermes/config.yaml" -not -path "*/.hermes/.env" \
  -not -path "*/.hermes/auth.json" \
  -not -path "*/.hermes/vector_store.db" -not -path "*/.hermes/vector_index.faiss" \
  -not -path "*/.hermes/.index_tracker.json" \
  -not -path "*/.hermes/audio_cache/*" \
  -not -path "*/.gemini/*" -not -path "*/.npm/*" \
  -type f 2>/dev/null | head -50
```
- `TRY`: **先分类**：若命中文件为 Hermes/系统运行文件（state.db、logs、cron output、gbrain、openclaw 等）→ 标记为系统文件，**不删除**。仅对用户数据文件（文档、项目源码、配置等）执行 Python 覆写 + 删除。> ⚠️ 此分类依赖 Agent 推理判断，非脚本自动化。**主要防线是上方 `find` 的 `-not -path` 预排除列表**（已覆盖绝大多数已知系统路径）。Agent 分类仅作为二次安全网，处理预排除列表未覆盖的极少数漏网路径。
- `ON FAILURE`: 捕获权限错误或文件锁定。
- `FALLBACK`: 执行 `rm -f <file>`（仅限已确认的用户数据文件）。
- `REPORT`: 记录已清除或残留路径，区分「系统文件（已跳过）」和「用户数据（已清除）」。
- `CONTINUE`: 继续下一步。

##### 4.1a delegation live transcripts 专项审计（v2.5.3，R4 审计 P0-2）
`delegate_task` 子代理的 live transcripts 位于 `$INCOGNITO_HERMES_HOME/cache/delegation/live/<delegation_id>/task-*.log`——含子代理**全部对话、思考、工具调用明文**，默认保留 7 天（`delegation_live_log.py` `LIVE_RETENTION_DAYS=7`）。该目录**不在** 4.1 排除列表（`cache/` 整体排除），且 P1-9 不重定向 `delegation/`（远程 backend 挂载语义）——**必须专项审计**：
```bash
find "$INCOGNITO_HERMES_HOME/cache/delegation/live" -maxdepth 2 -type f -newer "$INCOGNITO_TMP_DIR/pid.lock" 2>/dev/null
```
- `TRY`: 命中文件与 4.1b 的 `hermes-snap-*.sh` 同等对待——**不视为可跳过系统文件**，执行 Python 覆写 + 删除（无痕会话产生的子代理内容必须销毁）。
- `ON FAILURE`: 远程 backend 挂载只读导致删除失败 → 标记 ⚠️，列出路径供手动清理。
- `FALLBACK`: `rm -f <file>`。
- `REPORT`: 记录已清除的 delegation 文件数与 delegation_id。
- `CONTINUE`: 继续下一步（本步骤必须在 4.8 之前完成——不依赖沙箱，但保持一致顺序）。

**async_delegations 表清理（v2.5.8，R9 现场发现）**：`delegate_task` 的任务书/结果全文还会持久化进 `state.db` 的 `async_delegations` 表（`origin_session`/`origin_session_id` 字段可精确匹配）——**4.9 的 `hermes sessions delete` 不清理该表**（实测确认），无痕会话 delegate 的子代理任务书明文会留表。按无痕 sid 精确删除：
```bash
python3 -c '
import os, sqlite3

sid = os.environ.get("HERMES_SESSION_ID", "")
hermes_home = os.environ.get("INCOGNITO_HERMES_HOME", os.path.expanduser("~/.hermes"))
if not sid:
    print("⚠️ 无 HERMES_SESSION_ID，无法精确匹配——跳过 async_delegations 清理")
    exit(0)
db = os.path.join(hermes_home, "state.db")
if not os.path.isfile(db):
    print("✅ 无 state.db，跳过")
    exit(0)
try:
    con = sqlite3.connect(db, timeout=5)
    cur = con.execute(
        "DELETE FROM async_delegations WHERE origin_session = ? OR origin_session_id = ?",
        (sid, sid),
    )
    con.commit()
    print("✅ async_delegations 已清理 " + str(cur.rowcount) + " 条（session=" + sid + "）")
    con.close()
except Exception as e:
    print("⚠️ async_delegations 清理失败: " + str(e))
'
```
- `TRY`: 按 `origin_session`/`origin_session_id` 精确删除（零误伤——只删无痕会话派生的 delegation 记录）。
- `ON FAILURE`: DB 锁/只读 → 标记 ⚠️，报告。
- `REPORT` & `CONTINUE`

##### 4.1c interrupted_turns.json 审计（v2.5.6，R7）
desktop/TUI 的自动续跑标记（`tui_gateway/turn_marker.py`）：回合开始写 `{session_id, prompt}` 明文，回合正常结束清除，**进程死亡才残留**。无痕会话崩溃时 prompt 明文留此文件（`$INCOGNITO_HERMES_HOME/desktop/interrupted_turns.json`，24h 过期）。按 sid 精确清理无痕会话条目：
```bash
python3 -c '
import os, json

sid = os.environ.get("HERMES_SESSION_ID", "")
hermes_home = os.environ.get("INCOGNITO_HERMES_HOME", os.path.expanduser("~/.hermes"))
marker = os.path.join(hermes_home, "desktop", "interrupted_turns.json")
if not os.path.isfile(marker):
    print("\u2705 \u65e0 interrupted_turns.json\uff0c\u8df3\u8fc7")
    exit(0)
with open(marker, encoding="utf-8") as f:
    data = json.load(f)
if not isinstance(data, dict):
    print("\u26a0\ufe0f marker \u683c\u5f0f\u5f02\u5e38\uff0c\u8df3\u8fc7")
    exit(0)
if sid and sid in data:
    del data[sid]
    with open(marker, "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=2)
    print("\u2705 \u5df2\u5220\u9664\u65e0\u75d5\u4f1a\u8bdd\u6761\u76ee: " + sid)
elif sid:
    print("\u2705 marker \u4e2d\u65e0\u672c\u4f1a\u8bdd\u6761\u76ee\uff0c\u8df3\u8fc7")
else:
    print("\u26a0\ufe0f \u65e0 HERMES_SESSION_ID\uff0c\u65e0\u6cd5\u7cbe\u786e\u5339\u914d\u2014\u2014\u5982\u5d29\u6e83\u6b8b\u7559\u8bf7\u624b\u52a8\u68c0\u67e5 " + marker)
'
```
- `TRY`: 按 sid 删除条目（key 即 session_id，零误伤）。
- `ON FAILURE`: 文件被占用/格式异常 → 跳过，报告。
- `REPORT` & `CONTINUE`

##### 4.1b `/tmp/` 根目录审计 — 沙箱外写入检测
Phase 2 规则要求所有文件写入必须在 `$INCOGNITO_TMP_DIR` 内，但 Phase 4 的设计原则是**不信任 Phase 2**。此补充审计扫描 `/tmp/` 根目录（非沙箱路径），检测是否有绕过沙箱的泄漏写入：
```bash
find /tmp -maxdepth 1 -newer "$INCOGNITO_TMP_DIR/pid.lock" \
  -not -path "$INCOGNITO_TMP_DIR" -not -path "$INCOGNITO_TMP_DIR/*" \
  -not -name "hermes-incognito-*" -not -name ".hermes-incognito-active" \
  -type f 2>/dev/null | head -20
```
- `TRY`: 列出发现的非沙箱 `/tmp/` 文件，Python 覆写 + 删除。⚠️ **特别注意**：`hermes-snap-*.sh` 包含本会话全部命令明文（含环境变量中的 API key 等敏感信息），**不视为可跳过的系统文件**——必须与其他用户数据文件同等对待，执行完整覆写擦除。Agent 不得以"Hermes 系统文件"为由跳过。⚠️ **Windows/其他平台（v2.5.3）**：snap 可能位于 `$INCOGNITO_HERMES_HOME/cache/terminal/`（非 `/tmp`）——同步扫描 `[ -d "$INCOGNITO_HERMES_HOME/cache/terminal" ] && find "$INCOGNITO_HERMES_HOME/cache/terminal" -maxdepth 1 -name "hermes-snap-*.sh"`。
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

mem_dir = os.path.join(os.environ.get("INCOGNITO_HERMES_HOME", os.path.expanduser("~/.hermes")), "memories")
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

- `TRY`: 比对 SHA-256 哈希，检测 Markdown 文件变更。**若超时或执行失败，执行重试策略（最多 3 次，间隔递增）**：
  1. 第 1 次失败 → 等待 5 秒，重试
  2. 第 2 次失败 → 等待 10 秒，重试
  3. 第 3 次仍失败 → 不再重试，标记 ⚠️，输出手动检查命令：`diff <(sha256sum ~/.hermes/memories/MEMORY.md) <(cat $INCOGNITO_TMP_DIR/memory_baseline.txt)`
- `REPORT`: 记录哈希比对结果或失败原因。失败时向用户明确说明"Memory 审计未完成，请手动验证"。
- `CONTINUE`: 无论成功或三次重试失败，均继续下一步（不阻塞 Phase 4 流水线）。

#### 4.4 Skill / Cron 审计（限制 `-maxdepth 5` 平衡性能与覆盖）
检查无痕会话期间是否意外创建了技能文件或定时任务：
```bash
find "$INCOGNITO_HERMES_HOME/skills/" -maxdepth 5 -newer "$INCOGNITO_TMP_DIR/pid.lock" -type f -not -name ".curator_state" -not -name ".bundled_manifest" -not -name ".usage.json" 2>/dev/null | grep -v "incognito-mode" | head -20
HISTFILE=/dev/null HISTSIZE=0 hermes cron list 2>/dev/null
```
- `TRY`: 对 `find` 命中的 **SKILL.md** 逐条分类（元数据文件 `.curator_state`/`.bundled_manifest`/`.usage.json` 已被 `-not -name` 排除，不应出现），**不得自行删除，必须向用户汇报**：
  1. 列出每个命中 skill 的名称、简短内容摘要（10 字以内）
  2. 标注分类：🔴 **本会话相关**（与本次任务直接关联）/ ⚪ **Curator 自动生成**（通用知识）
  3. **对每个标 🔴 的 skill 给出删除建议**，逐条写明理由（如"此 skill 文档化了本次下载 anyflip 翻页书的全流程"）
  4. **等待用户确认后再执行删除**
  > ⚠️ 判断标准（给用户看，不是给自己看）：**问自己"如果没做本次会话的任务，这个 skill 还会被创建吗？"**——会 → curator 噪音，跳过；不会 → 会话残留，建议删除。
- `FALLBACK`: 列出需手动移除的文件路径和 Cron ID。
- `REPORT`: 记录汇报结果和用户决策。
- `CONTINUE`

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

#### 4.6c hermes-snap 二次清除（v2.5.3，R4 审计 P1-4）
> ⚠️ **再生悖论**：Hermes 每次 `terminal()` 执行后都会重写 `/tmp/hermes-snap-<env_id>.sh`（含环境变量明文，可能含 API key 等敏感凭据）——4.1b 清除的存量会在后续命令执行时**再次产生**。此步骤在 4.6（unset 敏感变量）之后、Phase 5 报告前执行，尽量缩小暴露窗口。
```bash
find /tmp -maxdepth 1 -name "hermes-snap-*.sh" -newer "$INCOGNITO_TMP_DIR/pid.lock" 2>/dev/null | while read f; do python3 -c '
import os, sys
def wipe(fp):
    try:
        if os.path.islink(fp): os.unlink(fp); return
        os.chmod(fp, 0o600)
        sz = os.path.getsize(fp)
        if sz > 0:
            with open(fp, "rb+", buffering=0) as fh:
                import os as _os
                fd = fh.fileno(); fh.seek(0); remaining = sz
                while remaining > 0:
                    c = min(65536, remaining); w = fh.write(_os.urandom(c))
                    if w == 0: break
                    remaining -= w
                fh.flush(); _os.fsync(fd); fh.seek(0); fh.truncate(0); fh.flush(); _os.fsync(fd)
        os.unlink(fp)
    except Exception:
        try: os.unlink(fp)
        except Exception: pass
wipe(sys.argv[1])
' "$f"; done 2>/dev/null || true
```
- `TRY`: 覆写 + 删除 Phase 4 期间产生的 snap。
- `REPORT`: 记录清除数量。⚠️ **已知降级**：本清除命令自身执行后 Hermes 会再写一次 snap——此残留 snap 内容 = 清除时刻环境（此时 INCOGNITO 变量已 unset、Phase 4 审计已结束），暴露窗口最小化。**根治需框架支持 `[Framework L2]`**（execute 后不重写快照）。
- `CONTINUE`

#### 4.6b agent.log 搜索查询清洗
利用 Linux `O_APPEND` 特性：Hermes 以追加模式打开 agent.log，外部进程在会话结束后按行清洗本会话之后的 `web_search` 日志行，原地替换查询明文为 `[REDACTED_INCOGNITO_QUERY]` 后 `truncate`。主进程后续写入自动对齐新 EOF——不会产生空洞或文件损坏。同步清洗轮转日志 `agent.log.1` 至 `agent.log.3`。

> ⚠️ **v2.5.5 扩展（R6 实测验证）**：v2.5.3 只清洗 `Web search via` 行——真实无痕会话暴露两个盲区：① firecrawl 等 provider 层自打 `Firecrawl search: '...' (limit=N)` 日志（格式不同，旧正则不匹配）；② `agent.turn_context` 把**用户消息明文**写入 `conversation turn: ... msg='...'`。v2.5.5 扩展：查询行覆盖 7 种 provider 格式（Web search via / Firecrawl / Exa / Tavily / Parallel / SearXNG / Brave），`msg=` 用户消息行锚定 sid 清洗（兼容 repr 双引号包裹），时间窗口内 URL 明文一并脱敏。**根治方向见 §7 logging RedactingFilter（L2）**。
```bash
python3 -c '
import os, re
from datetime import datetime

# 从本会话 pid.lock 读无痕会话开始时间（时间窗口过滤，不动历史行）
tmp_dir = os.environ.get("INCOGNITO_TMP_DIR", "")
created = 0.0
lock = os.path.join(tmp_dir, "pid.lock") if tmp_dir else ""
if lock and os.path.isfile(lock):
    with open(lock) as f:
        for line in f:
            if line.startswith("created="):
                try:
                    created = float(line.split("=", 1)[1].split()[0])
                except (ValueError, IndexError):
                    pass

sid = os.environ.get("HERMES_SESSION_ID", "")
q = chr(39)  # 单引号 ASCII 39，全版本兼容
dq = chr(34)  # 双引号 ASCII 34（turn_re 字符类拼接用，避免裸单引号）
prefix_re = re.compile(r"^(\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2},\d{3}) ")

# v2.5.5: 查询行覆盖 7 种 provider 格式；贪婪 (.*) + 尾部锚定，兼容 query 内含单引号
# 尾缀两种：(limit: N) 冒号+空格 / (limit=N) 等号无空格 / : N results（SearXNG/Brave 单独处理）
query_re = re.compile(
    r"^(.*?(?:Web search via [^:]+: |Firecrawl search: |Exa search: |Tavily search: |Parallel search: ))"
    + q + r"(.*)" + q + r"(?: \(limit[=:] ?\d+\)|: \d+ results)$"
)
query_repl = r"\1" + q + "[REDACTED_INCOGNITO_QUERY]" + q + " (limit: N)"

# v2.5.5: SearXNG / Brave 格式：search <xxx>: N results（无 limit 尾缀）
sb_re = re.compile(
    r"^(.*?(?:SearXNG search |Brave Search ))"
    + q + r"(.*)" + q + r": \d+ results$"
)
sb_repl = r"\1" + q + "[REDACTED_INCOGNITO_QUERY]" + q

# v2.5.5: conversation turn 用户消息行——repr 遇单引号改用双引号包裹，兼容两种引号
# 注意：字符类用 dq+q 拼接，禁止裸单引号（会破坏 bash 单引号包裹——R10 复检发现）
turn_re = None
if sid:
    turn_re = re.compile(
        r"^(\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2},\d{3} .*?\[" + re.escape(sid) + r"\].*?msg=)([" + dq + q + r"])(.*)\2$"
    )
turn_repl = r"\1" + q + "[REDACTED_INCOGNITO_QUERY]" + q

# v2.5.5: 通用 URL 明文（时间窗口内，如 scraping/truncated/Blocked URL 行）
url_re = re.compile(q + r"((?:https?://)[^" + q + r"]*)" + q)
url_repl = q + "[REDACTED_INCOGNITO_URL]" + q

def clean_log(path):
    if not os.path.isfile(path):
        return 0
    hits = 0
    with open(path, "r+", encoding="utf-8", errors="ignore") as f:
        lines = f.readlines()
        for i, ln in enumerate(lines):
            ts_m = prefix_re.match(ln)
            if not ts_m:
                continue
            try:
                ts = datetime.strptime(ts_m.group(1), "%Y-%m-%d %H:%M:%S,%f").timestamp()
            except ValueError:
                continue
            if ts < created:
                continue
            q_m = query_re.match(ln)
            sb_m = sb_re.match(ln)
            t_m = turn_re.match(ln) if turn_re else None
            if q_m:
                lines[i] = query_re.sub(query_repl, ln)
                hits += 1
            elif sb_m:
                lines[i] = sb_re.sub(sb_repl, ln)
                hits += 1
            elif t_m:
                lines[i] = turn_re.sub(turn_repl, ln)
                hits += 1
            elif url_re.search(ln):
                lines[i] = url_re.sub(url_repl, ln)
                hits += 1
        if hits:
            f.seek(0)
            f.writelines(lines)
            f.truncate()
    return hits

for name in ["agent.log", "agent.log.1", "agent.log.2", "agent.log.3"]:
    try:
        hermes_home = os.environ.get("INCOGNITO_HERMES_HOME", os.path.expanduser("~/.hermes"))
        n = clean_log(os.path.join(hermes_home, "logs", name))
        print("\u2705 " + name + " \u5df2\u6e05\u6d17 " + str(n) + " \u6761\u641c\u7d22\u67e5\u8be2")
    except Exception as e:
        print("\u26a0\ufe0f " + name + " \u6e05\u6d17\u5931\u8d25: " + str(e))
'
```
- `TRY`: 行首时间戳窗口 + `Web search via` 宽松匹配 → 正则替换查询明文为 `[REDACTED_INCOGNITO_QUERY]` → 同步处理轮转日志 `.1/.2/.3`。
- `ON FAILURE`: 权限/编码错误记录。
- `FALLBACK`: 报告未清洗的查询条数，提示手动审查 `agent.log*`。
- `REPORT`: 记录清洗条数（含全部轮转文件）。
- `CONTINUE`

#### 4.7 Git / 项目目录审计
在项目根目录（若有）检查未提交变更。若当前不在 Git 仓库中，此步自动跳过。
```bash
cd "${PROJECT_DIR:-$HOME}" 2>/dev/null && git status --short 2>/dev/null | head -20
```
- `TRY`: 收集未追踪及已修改文件清单。
- `FALLBACK`: 输出变更文件供用户确认（不自动不可逆回滚）。
- `REPORT` & `CONTINUE`

#### 4.7b 用户数据缓存恢复（在 4.8 之前——符号链接目标即将被销毁）
将 Phase 1 重定向至沙箱的符号链接恢复为原始目录。**必须在 4.8 之前执行**——4.8 递归销毁沙箱后符号链接目标消失，恢复会失败。
```bash
# 4.7b: 用户数据缓存重定向恢复（v2.5.3 扩展 7 目录，与 P1-9 对齐）
for dir in web screenshots videos images audio documents vision; do
    CACHE_DIR="$INCOGNITO_HERMES_HOME/cache/$dir"
    BAK_DIR="$INCOGNITO_HERMES_HOME/cache/${dir}.incognito-bak"
    
    if [ -L "$CACHE_DIR" ] && [ -d "$BAK_DIR" ]; then
        rm "$CACHE_DIR"
        mv "$BAK_DIR" "$CACHE_DIR"
    fi
done
```
- `TRY`: 遍历 web/screenshots/videos/images/audio/documents/vision，检测符号链接 → 删除 → 恢复备份目录。
- `ON FAILURE`: 符号链接已在 4.8 中被销毁 → 标记 ⚠️，提示手动执行 `mv ~/.hermes/cache/<dir>.incognito-bak ~/.hermes/cache/<dir>`。
- `FALLBACK`: 报告残留的 `.incognito-bak` 备份路径供用户手动恢复。
- `REPORT`: 记录恢复状态（成功/残留备份路径）。
- `CONTINUE`

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

> ⚠️ **unset 时机（v2.5.7 修正，R9 现场发现）**：**必须在此（4.8 完成后）才执行** `unset INCOGNITO_MODE_ACTIVE INCOGNITO_TMP_DIR PID 2>/dev/null` 清除持久 Shell 环境变量残留——4.6b/4.6c/4.7b/4.8 仍依赖 `INCOGNITO_TMP_DIR`/`INCOGNITO_HERMES_HOME`，若在 4.6 后立即 unset 会导致后续步骤变量为空静默失效（R9 实测确认）。4.9/4.10 不依赖这些变量，可安全 unset。

#### 4.9 Session 容器销毁（延后步骤）— 防线 2 最后一击

> ⚠️ **此步骤为延后执行标记**：实际 `hermes sessions delete` 命令在 Phase 5 审计报告输出完毕后执行（见 Phase 5「最后一步」），不在此处运行。此处仅作占位，确保 Agent 知晓此步骤存在。
>
> ⚠️ 4.10 二次验证仅覆盖 4.1-4.8（文件/4.1a delegation/History/Memory/Skill/进程/配置/4.6b 日志/4.6c snap/Git/沙箱），不验证 4.9 的 Session 删除（因为此时尚未执行，命令延后至 Phase 5）。
>
> ⚠️ **已知盲区（v2.5.3，R4 审计 FIX-10）**：Phase 5 报告后若 Agent 崩溃/网络中断，4.9 删除命令可能永不执行。缓解：Phase 4 末尾输出「将删除 Session 集」预检清单（`hermes sessions list | grep <sid>`），用户可手动执行 `hermes sessions delete --yes <sid>` 兜底。

---

#### 4.9r 异常关闭补救协议（Recovery Protocol，v2.6.0，R11 现场实证）

> **R11 实证（2026-08-01）**：真实无痕会话异常关闭（`session scope close failed`，4.9 删 session 后主进程 FK 写入失败）→ 4.6b 未执行 → **agent.log 残留 16 条搜索/用户消息明文**。事后用本协议手动兜底清洗成功（幂等可重入）。

**触发条件**：哨兵 `/tmp/.hermes-incognito-active` 存在 **且** 对应 session 已不在 state.db（= 会话异常结束，Phase 4/5 未完成）。

**执行步骤（幂等，可随时重跑）**：
```bash
# 1. 读哨兵（session + created）
SID=$(grep -o "session=[^ ]*" /tmp/.hermes-incognito-active | cut -d= -f2)
CREATED=$(grep -o "created=[0-9]*" /tmp/.hermes-incognito-active | cut -d= -f2)
echo "恢复对象: session=$SID created=$CREATED"

# 2. 构造假沙箱 pid.lock（created 复用哨兵值 → 4.6b 时间窗口对齐）
mkdir -p /tmp/hermes-incognito-recovery && echo "created=$CREATED session=$SID" > /tmp/hermes-incognito-recovery/pid.lock

# 3. 以无痕 sid 身份重跑 4.6b（幂等——重复清洗无害）
INCOGNITO_TMP_DIR=/tmp/hermes-incognito-recovery INCOGNITO_HERMES_HOME="${HERMES_HOME:-$HOME/.hermes}" HERMES_SESSION_ID=$SID bash -c '<4.6b 代码块>'

# 4. 清理恢复沙箱 + 哨兵
rm -rf /tmp/hermes-incognito-recovery && rm -f /tmp/.hermes-incognito-active

# 5. 验证：关键词 0 命中
grep -a -c '<关键词>' ~/.hermes/logs/agent.log
```
- 4.6b 代码块即上文 `#### 4.6b` 的完整 bash 代码（从 SKILL.md 提取执行）。
- 若 session 未删（仍在 state.db）→ 不适用本协议（会话可能仍在运行），先确认会话状态。
- 补救后向量索引 cron 自动恢复正常（哨兵已清 + session 已删，双重释放）。

> ⚠️ **此协议与 4.9 手动兜底互补**：4.9 兜底解决"session 未删"；本协议解决"明文未洗 + 哨兵残留"。两者独立可分别执行。

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

> 🧹 **无痕模式结束 — 全量审计报告 (v2.6.0)**
>
> | 审计项 | 状态 | 详情 |
> |------|:--:|------|
> | 4.1 文件系统（HOME + /tmp/） | ✅/⚠️/❌ | [发现 N 个文件，已清除/残留路径] |
> | 4.1a delegation transcripts | ✅/⚠️/❌ | [清除 N 个文件 / 无子代理 / 需手动] |
> | 4.1c interrupted_turns | ✅/⚠️/❌ | [已删除无痕条目 / 无条目 / 需手动] |
> | 4.2 Shell History | ✅/⚠️/❌ | [增量行数，已清除/残留行号] |
> | 4.3 Memory 存储 | ✅/⚠️/❌ | [SHA-256 哈希比对，文件未变更/已变更需审查] |
> | 4.4 Skill / Cron | ✅/⚠️/❌ | [新增/修改数量，已删除/需手动] |
> | 4.5 进程/python_history/子代理 | ✅/⚠️/❌ | [手工提示已输出/.python_history状态/子代理N个] |
> | 4.6 环境变量 / 配置 | ✅/⚠️/❌ | [被修改的文件，已回滚/需手动] |
> | 4.6b agent.log 清洗 | ✅/⚠️/❌ | [清洗 N 条搜索查询 / 无记录 / M 条失败] |
> | 4.6c hermes-snap 二次清除 | ✅/⚠️/❌ | [清除 N 个 / 再生降级已声明] |
> | 4.7 Git / 项目目录 | ✅/⚠️/❌ | [未追踪变更列出，未自动回滚] |
> | 4.7b 缓存恢复 | ✅/⚠️/❌ | [已恢复/残留 .incognito-bak 备份需手动] |
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

**Agent 执行指令（哨兵清理，v2.5.5）**：4.9 执行后立即删除向量索引哨兵（正常路径主动清；**即使漏删也无碍**——index_all.py 的 session 存在性检查会检测 4.9 已删除的 session 并自动释放哨兵）：
```bash
HISTFILE=/dev/null HISTSIZE=0; rm -f /tmp/.hermes-incognito-active
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
| **Hermes Web Cache** | `web_search`/`web_extract` 结果由框架自动缓存至 `~/.hermes/cache/web/`；v2.5.0 起 Phase 1 通过符号链接重定向至沙箱，但若 Hermes 进程在 Phase 4.7b 恢复前崩溃，缓存文件在 `/tmp/` 沙箱中残留（24h TTL 孤儿清理兜底） | 无痕会话中避免访问高度敏感 URL |
| **agent.log 搜索查询** | web_search 查询明文写入 `~/.hermes/logs/agent.log`；v2.5.3 起 Phase 4.6b 按**行首时间戳窗口**（pid.lock `created=` 之后）清洗脱敏为 `[REDACTED_INCOGNITO_QUERY]`（不再锚定 session ID——`tools.web_tools` 日志行无 sid，旧正则 0 命中）。若 Hermes 在 Phase 4 前崩溃则查询明文永久残留 | Phase 4.6b 自动清洗；崩溃残留需手动审查 agent.log |

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
- ❌ **无痕会话中依赖事后擦除**：web_search 查询明文和 web_extract 缓存文件在 Phase 4 清洗前仍以原始形式存在于 agent.log 和沙箱中。若 Hermes 在 Phase 4 前崩溃，这些明文永久残留

---

## 7. 框架层 TODO（需 Hermes 版本升级支持）

以下需求标记为 `[Framework L2]`—— Agent 无法在 Skill 层面实现，需底层支持：

- [ ] **`[Framework L2]` TTL Auto-Destroy Timer**：宿主框架提供外部 Timer，Phase 3 超时后自动触发 Phase 4 销毁（LLM Agent 不具备自发唤醒能力）
- [ ] **`[Framework L2]` Execute No-Snapshot Flag**：`terminal()` 支持跳过命令后重写环境快照（`/tmp/hermes-snap-*.sh`），从根上消除 4.6c 的再生降级（v2.5.3 新增）
- [ ] **`[Framework L2]` Logging RedactingFilter 原生支持**（v2.5.6 更新）：**已通过用户插件落地**（`~/.hermes/plugins/incognito-log-filter/`，哨兵门控 + 内存脱敏，明文不落盘）。框架原生支持（无插件依赖、无需哨兵文件轮询）仍列为可选优化；当前插件方案与 skill 4.6b 事后擦除构成双层防御
- [ ] **`[Framework L2]` serve/dashboard 后端加载用户插件**（v2.6.0 新增，R11 根因）：`_plugin_cli_discovery_needed()` 对内置子命令返回 False → serve/gateway 从不加载用户插件（hooks/tools/filter 全失效）。**已本地 patch**（`hermes_cli/main.py` cmd_dashboard 显式 `discover_plugins()`，16 行）——待上游合并；`hermes update` 后需重新应用或依赖上游版本
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

   > ⚠️ **Python hex escape 兼容性（v2.5.1）**：`\x27` 等 hex escape 在 Python 3.12+ 触发 `DeprecationWarning`，3.14+ 升级为硬错误 `SyntaxError: bad escape`。Skill 内嵌 Python 代码应使用 `chr(39)` 等 `chr()` 函数替代 hex escape——在所有 Python 3.x 版本下行为一致。

   > ⚠️ **禁止过批（Anti-Batching）**：不要将多个独立步骤塞进一个 `bash -c` 块。每个 Phase 1 步骤应使用**独立的 `terminal()` 调用**——这不仅避免了嵌套引号逃逸的地狱，还让每个步骤的错误隔离、可独立重试。经验法则：只要一个 `bash -c` 块内同时包含 Phase 1 的步骤 2 + 步骤 3 + 步骤 6，就已经过批了。
