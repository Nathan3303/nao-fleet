# T595-B 实现说明 · label 承载归属（rd-infra）

- 任务：T595-B（Issue #273 的 B 步）· 分支 `feat/595b-label-attribution` · 更新：2026-11-28
- 指针：设计 `docs/T595-multirepo-naming/design.md` · A 步实现见归档 `docs/tasks-archive.md#T595`

## 做了什么

- **写**：`ensure` spawn 时一次写全 `nao.repo`（= 仓基键 `place_key_label`，≤16、禁绝对路径）· `nao.role` · `nao.slot`；app 走 `hc run --label`，herdr 走 sidecar `repo=`。仓基键由 `ensure` 相位 1 算定后作为 `term_spawn` 第 8 参传入 —— spawn 内**零重算**（`cases/28` C3④ 判据仍绿）。
- **读**：`status` 的归属裁决 = `nao.repo` → pane cwd → 名字(legacy)；**一致才采信，冲突 ⇒ `unknown`**（不猜、不误杀）；唯一例外：label 在而 cwd 不可读（pane 死）⇒ 采信 label。
- **呈现**：冲突行 `? <名>（归属不可判定：nao.repo=… 与 pane cwd 冲突）` + 一行处置（`hc get` 核对 / `ensure --force` 重开）；`status --json` 增 `repoSource: label|cwd|conflict`（schema 仍 1，纯增量）。
- **不迁移不回填**：老会话无 `nao.repo` ⇒ 走 cwd，行为与 A 步之后完全一致。`where` 全程未进解析（app 回写的派生字段）。

## 已知边界（本单刻意不扩 —— QA 复核时点出）

**归属冲突的文本行只在「健康终端」（有 hook / status）上出现**：缺 hook 的终端在 `_term_app_liveness` 里先落 `integration-missing` 分支，走不到冲突渲染。
- **为什么不在这单解**：两者是**不同的病** —— 冲突是「两份归属证据打架」，集成缺失是「证据根本不在场」。合并处理会让「缺 hook」被误读成「归属冲突」，反而掩盖真正的集成问题（T597 刚把这两种信号分开）。
- **后续**：若要覆盖，应在「集成缺失」行里补一句「若该终端有 nao.repo，先核 label 与 cwd」（等于让两条信号各自带上下文），而不是把 unknown 折叠进去。
