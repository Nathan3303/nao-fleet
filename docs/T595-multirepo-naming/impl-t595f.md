# T595-F 实现说明 · tab 条显示（rd-infra）

- 任务：T595-F（Issue #290 · 方案 F 第 3 步）· 分支 `feat/290-tab-rename` · 更新：2026-11-28
- 指针：设计 `docs/T595-multirepo-naming/design.md §4`（实测：app 唯一能改 tab 条文字的是 `herdr-app do tab.rename`）

## 做了什么

- **显示串单一构造点** `term_tab_label <名> <仓基键> [组前缀]` → `[<组>/]<名> · <仓基键>`（两后端同一串）。
- **app**：spawn 成功后 `_term_app_tab_rename <tab_id> <串>`（`herdr-app do tab.rename --id … --name …`，带 `HC_SOCKET`）；
  **失败只 warn**（`tab 条改名失败…（只影响显示，spawn 仍成功）`）—— 不影响 spawn 成败与 rc。
- **TUI**：`tab create --label` 与 `_term_herdr_layout_converge` 的期望标签**同一构造点** ——
  否则每次 ensure 都会把 tab 改名回旧串（幂等性被打破）。`repo_key` 由 `ensure` 相位 1 算定后传入（spawn 内零重算）。
- 夹具：`tests/t4/fixtures/herdr-app`（记 `T4_HERDR_APP_LOG`；`HERDR_APP_FAIL=1` 只对 `do tab.rename` 注入失败）。
- 用例：`cases/44` ⑰ 段（显示串 / 两仓可区分 / 失败降级 / rename 后 status+close 仍可用）；`cases/33`/`cases/40` 的 tab 条断言随格式更新。

## 判据 5（真正的风险点）实测结论：**rename 不破坏匹配**

`status` / `close` 的 tab 关联走 **tab id**（`herdr.tab` label 存 id；`_term_app_mint_tab` 也是确定性 id），不读 tab 的 name/where；
`⑰⑤` 两条（rename 后 `status` 仍判在线、`close` 仍能回收）实跑 PASS。

## 判据 4（幂等）：PM 已裁**方向 ②（dedupe 先于槽位分配）**，实现已落、**仍未命中（blocked）**

**已实现**：`ensure` 目标循环里、`wsid` 之后、槽位计算**之前**新增 dedupe 分支 —— 判据 = 同仓基键（`nao.repo`）+ 同角色 + `term_liveness` 的 `verdict == online`；命中 ⇒ `warn 已在运行` + `SPAWN_SKIP++` + `continue`（effect 0），并调用 `_term_app_tab_converge_name` 做**收敛式幂等**（tab 串不一致才 rename）。`--force` 旁路。**未改 `_ensure_pick_num`**（那是方向 ① 的活，已被否）。

**实测仍不命中**：`cases/44 ⑰④` 四条红（重跑仍新建 `pm2`、无「已在运行」行、多发一次 tab.rename）。已确认前置成立：spawn **确实写了** `nao.repo = 仓基键`（⑰④ 前置断言 PASS），⑧ finished 残留清理仍 PASS。
**下一个排查点（按成本排序）**：
1. ensure 子进程里 `term_repo_label_rows`（facts 第 14 列）是否真的非空 —— 直接加一条 `-v` 打印即可定位（我最后一次调试没拿到输出，怀疑 `()` 子进程里 facts 缓存路径变量不同）。
2. `(repo, role)` 匹配：`base_label` 与 `nao.repo` 是否逐字相等（`place_key_label` vs 写入值）。
3. `term_liveness "$peer" "$repo"` 在 ensure 语境是否 `online`（可能被 `_TERM_PLACE_CACHE` 影响）。
