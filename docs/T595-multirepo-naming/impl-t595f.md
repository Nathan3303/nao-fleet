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

## 判据 4（幂等）：**已知缺陷，已拆单 #292 / T601**

`ensure` 同仓同角色重跑会取新槽位（`pm → pm2 → pm3`）而不是「已在运行 ⇒ 跳过」。根因 = **T595-A 的全局占用统计**把本仓的 `pm` 也当占用 —— 与 tab 显示无关，本单之前就存在 ⇒ **不在本单修**，已拆 **Issue #292（T601）**；本单分支**不带**未验证代码（死路径比没有更坏）。

我在 `feat/290-tab-rename` 上按 PM 裁决的「方向 ②（dedupe 先于槽位分配）」实现过一版：位置 `wsid` 之后 / 槽位计算之前；判据 `nao.repo == base_label` ∧ 角色同 ∧ `term_liveness` verdict `online`（finished/残留不算 ⇒ 保住 `cases/44 ⑧`）；命中 runner-up 动作 = warn「已在运行 ⇒ 跳过」+ `SPAWN_SKIP++` + tab 串收敛。**实测仍不命中**（⑰④ 四条红：仍新建 `pm2`、无「已在运行」、多发一次 rename），故 **整体回退、不带进本 PR**（PM 决定）。

**三条排查线索（已同步进 #292，下一轮从这里切）**：
1. **ensure 子进程里 `term_repo_label_rows`（facts 第 14 列）是否非空** —— 我最后一次调试的 `[DBGD]` 打不出来（怀疑 `<( … )` 子进程丢了 facts 缓存路径变量，或被 `set -e` 短路吞掉），**这是最可能的断点**；
2. `base_label` 与写入的 `nao.repo` 是否**逐字相等**（`place_key_label` 归一 vs 写入值）；
3. ensure 语境下 `term_liveness "$peer" "$repo"` 是否 `online`（可能被 `_TERM_PLACE_CACHE` / 快照影响）。

> 调试方法论（PM 追加提醒，值得记）：**先把「哪一层看不到」解决** —— 需要时写文件而不是 stdout，否则会一直在猜。
