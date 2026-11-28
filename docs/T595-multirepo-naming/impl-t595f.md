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

## 已知阻塞（判据 4 幂等 —— **不是本单引入**）

同仓同角色重跑 `ensure` 会取**新槽位**（`pm → pm2 → pm3`）而不是「已在运行 ⇒ 跳过」。
根因 = T595-A 的全局占用统计把**本仓**的 `pm` 也当占用（`_ensure_pick_num` 的 `term_live_peer_names` 来源），与 rename 无关。
我试过「同仓 peer 不计数（按 `nao.repo` 过滤）」，但它**破坏 `cases/44 ⑧` 的 finished 残留清理判据** ⇒ 已回退，不夹带。
⇒ 需 PM 定方向：① 同仓 peer 不计占用（要连带修 ⑧ 的清理路径）② 重跑走 dedupe 分支（不重算槽位）③ 接受并改幂等判据。
