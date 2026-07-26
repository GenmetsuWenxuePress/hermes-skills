# incognito-mode 已知缺陷与规避方案

> incognito-mode 当前为 hub-installed 保护状态，以下问题需在每次使用时手动规避，直到上游修复。

## Bug 1 (P0): 4.6b agent.log 清洗在 Python 3.14 下静默失败

- **症状**：Phase 4.6b 输出 `bad escape \x at position 2`，搜索查询明文未被脱敏
- **根因**：脚本使用 `\x27` hex escape，Python 3.14 起为硬错误
- **规避**：执行 4.6b 时，将所有 `\x27` 替换为 `chr(39)`
- **验证**：执行后检查 `grep -c "REDACTED_INCOGNITO" ~/.hermes/logs/agent.log`

## Bug 2 (P1): Memory 基线可能写入 /tmp/ 裸路径

- **症状**：Phase 4.3 输出"无基线快照"，无法验证 Memory 是否被意外修改
- **根因**：Phase 1 step 6 脚本 `os.environ.get("INCOGNITO_TMP_DIR","/tmp")` 的 fallback 指向裸 /tmp/
- **规避**：Phase 1 step 6 执行时显式传入 `INCOGNITO_TMP_DIR`，不依赖 environ fallback
- **注意**：v2.5.1 的 export 修复降低了此问题发生率，但仍有边界情况

## Bug 3 (P2): Phase 4 自身产生新 hermes-snap 需二次清除

- **症状**：Phase 4.10 二次验证时发现残留 snap 文件
- **根因**：Phase 4 各步骤的 terminal() 调用产生新 snap，4.1b 仅清除之前的
- **规避**：在 Phase 4.10 中或紧接 4.8 之后，追加一轮 snap 清除：
  ```bash
  HISTFILE=/dev/null HISTSIZE=0 python3 -c '
  import os, glob
  for f in glob.glob("/tmp/hermes-snap-*.sh"):
      try: os.unlink(f)
      except: pass
  '
  ```

## 非 Bug：web cache 仅在 >15000 字符时落盘

- web_extract 结果 ≤15000 字符 → 内联返回，不写入 `~/.hermes/cache/web/`
- 仅超限页面触发 `_store_full_text` → 此时符号链接重定向生效
- 验证方法：用 Wikipedia 等大页面测试，小页面（example.com）不产生缓存
