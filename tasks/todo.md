# Tasks: rwm

> 记录格式：Description / Acceptance / Verification / Dependencies / Files / Scope。
> 命令约定：`cargo test`、`cargo build --release`、`cargo clippy -- -D warnings`、`cargo fmt --check`。

## Checkpoint 1: 之后（T1–T3）build + test 绿

## Task 1 ✓: 协议代码生成模块化 + river-layer-shell 绑定

**Description:** 把 `mod river`（`generate_interfaces!`/`generate_client_code!` 宏）从 `main.rs` 抽到 `src/river.rs`，`main.rs` 改为 `use crate::river`。复制 rill-ed 的 `river-layer-shell-v1.xml` 到 `protocol/` 并生成 `RiverLayerShellV1`/`RiverLayerShellSeatV1` 绑定。

**Acceptance:**
- [ ] `cargo build --release` 成功，行为与现状一致
- [ ] `mod river` 只存在于 `src/river.rs`，`main.rs` 无内联代码生成
- [ ] layer-shell 绑定已生成（编译可用，尚未使用）
- [ ] 不引入 `river-input-management-v1`/`river-xkb-config-v1`（kwim 已砍）

**Verification:**
- [ ] `cargo build --release`
- [ ] `cargo clippy -- -D warnings`

**Dependencies:** None

**Files:** `src/main.rs`、`src/river.rs`（新）、`protocol/river-layer-shell-v1.xml`（复制）

**Scope:** S

## Task 2 ✓: types/actions 数据模型

**Description:** `src/types.rs`（Rectangle、Color、Border、Layout、Config、Keybinding、PointerBinding、WindowRule、Window、Workspace、Output、DetachedOutput、OverviewState、Status）+ `src/actions.rs`（Button、PointerAction、KeybindingAction，与 rill-ed `actions.zig` 一一对应，含带参变体）。**无** `Spring`/`finish`/`is_animating`/`animation_*`。Config 树加 `serde` 派生。

**Acceptance:**
- [ ] 所有类型与 rill-ed `types.zig` 对应（动画相关除外）
- [ ] `KeybindingAction` 带 `#[serde(tag="action", content="arg", rename_all="snake_case")]`
- [ ] 单测：`Rectangle::eq`、`WindowRule::matches`（glob `*`/`?`、回溯）、默认值

**Verification:**
- [ ] `cargo test types::`
- [ ] `cargo build --release`

**Dependencies:** None（可与 T1 并行）

**Files:** `src/types.rs`（新）、`src/actions.rs`（新）、`src/main.rs`（加 `mod` 声明）

**Scope:** M

## Task 3 ✓: config TOML 加载/热重载

**Description:** `src/config.rs`：`load()`（读 TOML → `Config`，失败日志 + 默认值）、`reload()`（失败保留旧配置）、`default_config()`。用 `serde` 反序列化，键名对齐 `config.zon`（去掉 `animation_*`）。modifiers 支持 `["mod4"]` 列表形式。

**Acceptance:**
- [ ] 样例 TOML 完整反序列化（含带参 action、modifiers、window_rules、spawn_at_startup）
- [ ] 缺字段回退默认值；非法配置报错不 panic
- [ ] 单测：默认值、样例解析、非法值

**Verification:**
- [ ] `cargo test config::`
- [ ] `cargo build --release`

**Dependencies:** T2

**Files:** `src/config.rs`（新）、`Cargo.toml`（加 `toml`/`serde`/`xkbcommon`）、`src/main.rs`（mod 声明）

**Scope:** M

## Checkpoint 2: 之后（T4–T6）几何单测全绿

## Task 4 ✓: layout/common 矩形助手

**Description:** `src/layout/common.rs`：`initial_rectangle`、`center_rectangle`、`resize_floating_window`、`scale_floating_window`、`move_floating_window`、`floating_move_step`（50px），clamp 到输出边界/最小尺寸（2×border）。移植 rill-ed `layout/common.zig`。

**Acceptance:**
- [ ] 函数签名/行为与 rill-ed 一致（gap、min_size clamp、边界 clamp）
- [ ] 单测：居中、缩放、移动、边界 clamp

**Verification:**
- [ ] `cargo test layout::common::`

**Dependencies:** T2

**Files:** `src/layout/mod.rs`（新，声明子模块）、`src/layout/common.rs`（新）

**Scope:** S

## Task 5 ✓: layout/scroller 滚动列

**Description:** `src/layout/scroller.rs`：滚动列布局的矩形计算——按 `proportion` 分配宽度，聚焦窗口居中/对齐，`center_focused_window`（never/always/single）语义。移植 rill-ed `layout/scroller.zig`。

**Acceptance:**
- [ ] 输出矩形计算与 rill-ed 一致（不含动画插值）
- [ ] 单测：单窗/多窗、proportion 分配、center 三种模式

**Verification:**
- [ ] `cargo test layout::scroller::`

**Dependencies:** T4

**Files:** `src/layout/scroller.rs`（新）、`src/layout/mod.rs`

**Scope:** M

## Task 6 ✓: layout/floating 浮动布局

**Description:** `src/layout/floating.rs`：浮动窗口矩形（初始居中、resize/move 后位置），全屏处理。移植 rill-ed `layout/floating.zig`。

**Acceptance:**
- [ ] 浮动窗口几何与 rill-ed 一致
- [ ] 单测：初始矩形、缩放后仍在输出内

**Verification:**
- [ ] `cargo test layout::floating::`

**Dependencies:** T4

**Files:** `src/layout/floating.rs`（新）、`src/layout/mod.rs`

**Scope:** S

## Checkpoint 3: 之后（T7–T8）索引/快照单测绿

## Task 7 ✓: wm WindowManager 状态 + 索引逻辑

**Description:** `src/wm.rs`：`WindowManager` 结构（output_list、focused_output_idx、previous_workspace、detached_outputs、pending_windows、config、overview_state、status、needs_*、last_focused_window、layer_shell_focus、session_locked、lock_focus）+ `current_workspace`/`current_focus` + `move_window_to_workspace`（含焦点索引调整语义）。无 timer 字段。

**Acceptance:**
- [ ] 字段对齐 rill-ed `WindowManager`（timer/动画相关除外）
- [ ] `move_window_to_workspace`：移除窗口后 `focused_window_idx` 正确移动；插入到目标聚焦位之后
- [ ] 单测：空/单/多窗口移动的索引语义

**Verification:**
- [ ] `cargo test wm::`

**Dependencies:** T2、T3

**Files:** `src/wm.rs`（新）、`src/types.rs`（如需微调）、`src/main.rs`（mod）

**Scope:** M

## Task 8 ✓: layout 协调器

**Description:** `src/layout/mod.rs`：`apply`/`update`——移除失效输出、应用 pending 窗口、计算布局矩形、快照 `set_position`、边框/焦点应用、浮动窗口置顶。移植 rill-ed `layout.zig` 但去掉 `animation` 分支（直接 snap）。

**Acceptance:**
- [ ] `apply` 无动画路径：布局后窗口矩形直接为最终值
- [ ] 边框/焦点应用、`raiseFloatingWindows` 一致
- [ ] 单测：对假 Output/Workspace 调用布局纯逻辑

**Verification:**
- [ ] `cargo test layout::`（含 common/scroller/floating 回归）
- [ ] `cargo clippy -- -D warnings`

**Dependencies:** T4–T7

**Files:** `src/layout/mod.rs`

**Scope:** M

## Task 9 ✓: window 窗口生命周期

**Description:** `src/window.rs`：`Dispatch<RiverWindowV1>`（dimensions → pending→workspace、app_id/title、close、pointer move/resize 请求、fullscreen 请求），window_rules 匹配（app_id/title glob → floating）。移植 rill-ed `window.zig`。

**Acceptance:**
- [ ] dimensions 到达后窗口进入 workspace；close 清理；rules 命中强制 floating
- [ ] 无 `Spring`/`finish` 动画字段写入

**Verification:**
- [ ] `cargo build --release`（真实 river 行为 T18 验收）

**Dependencies:** T7、T8

**Files:** `src/window.rs`（新）、`src/main.rs`（mod）

**Scope:** M

## Task 10 ✓: output 输出管理 + detached_outputs

**Description:** `src/output.rs`：`Dispatch<RiverOutputV1>`（removed/dimensions/position）、输出新增/移除、`detached_outputs` 按名保存/恢复、TTY 切换韧性。移植 rill-ed `output.zig` + `types.zig` 的 detached 逻辑。

**Acceptance:**
- [ ] 输出移除时 workspace/窗口存入 `detached_outputs`；同名输出重现时恢复
- [ ] 单测：detach/restore 纯逻辑（构造假 DetachedOutput）

**Verification:**
- [ ] `cargo test output::`

**Dependencies:** T7、T8

**Files:** `src/output.rs`（新）、`src/main.rs`（mod）

**Scope:** M

## Task 11 ✓: seat 指针绑定 + layer-shell 焦点

**Description:** `src/seat.rs`：`Dispatch<RiverSeatV1>`（pointer enter/leave、window interaction、op delta/release、pointer position）、指针 move/resize 绑定、layer-shell seat 焦点跟踪（none/non_exclusive/exclusive）。移植 rill-ed `seat.zig`。

**Acceptance:**
- [ ] Super+左键移动 / Super+右键 resize 可用（真实 river）
- [ ] layer-shell 焦点状态机一致；独占焦点时不抢焦点

**Verification:**
- [ ] `cargo build --release`（真实行为 T18 验收）

**Dependencies:** T7、T8

**Files:** `src/seat.rs`（新）、`src/main.rs`（mod）

**Scope:** M

## Task 12 ✓: spawn setsid + 双重 fork

**Description:** `src/spawn.rs`：`spawn_detached`——setsid + 双重 fork，子进程继承/清理环境，spawn_at_startup + 键位 spawn 共用。移植 rill-ed `spawn.zig`。

**Acceptance:**
- [ ] spawn 的进程脱离 rill 控制终端，GUI 程序行为一致
- [ ] 错误返回 `Result` 而非静默（对齐架构分析 #18）

**Verification:**
- [ ] `cargo build --release`；手动：Super+t 启动 alacritty

**Dependencies:** None

**Files:** `src/spawn.rs`（新）、`src/main.rs`（mod）

**Scope:** S

## Task 13 ✓: overview 网格模式

**Description:** `src/overview.rs`：进入时记录 origins（预分配后置平，避免半途失败）、把窗口排成跨输出/workspace 网格、方向导航（边界 clamp 到真实窗口数）、Return/点击选中、Escape 取消、`prune` 清理已关窗口。移植 rill-ed `overview.zig`。

**Acceptance:**
- [ ] enter/cancel/select/nav/prune 行为一致（快照，无动画）
- [ ] origins 预分配（对齐架构分析 #19 的 OOM 回滚修复）
- [ ] 单测：导航边界、grid 计算纯函数

**Verification:**
- [ ] `cargo test overview::`

**Dependencies:** T7、T8

**Files:** `src/overview.rs`（新）、`src/main.rs`（mod）

**Scope:** M

## Task 14 ✓: keybinding keysym 解析 + 默认表 + 绑定/路由

**Description:** `src/keybinding.rs`：`xkb::keysym_from_name` 解析、默认键位/指针表（完整移植 rill-ed `keybinding.zig` 两张表）、`setup_keybindings`（重建绑定）、`set_overview_keybinds`（overview 专属键启停）、`Dispatch<RiverXkbBindingV1>` 路由到 action 分发入口。

**Acceptance:**
- [ ] 默认键位表与 rill-ed 完全一致（含 PipeWire 音量键、overview vi 键）
- [ ] 默认表所有 keysym 可解析（单测）
- [ ] 会话锁/overview 状态下按键拦截一致

**Verification:**
- [ ] `cargo test keybinding::`（keysym 解析 + 默认表校验）

**Dependencies:** T2、T3、T7

**Files:** `src/keybinding.rs`（新）、`Cargo.toml`（`xkbcommon`，若 T3 未加）、`src/main.rs`（mod）

**Scope:** M

## Checkpoint 4: 之后（T13–T16）action 全移植、无 no-op

## Task 15 ✓: action 分发 — 窗口/workspace 动作

**Description:** `src/keybinding.rs` 的 dispatcher + 窗口/workspace 类 helper：close/fullscreen/toggle_maximize/adjust·set_width/floating 调整与移动/focus·move window/toggle floating/focus·move workspace/`move_window_to_workspace`。移植 rill-ed `keybindingPressed` 对应分支。

**Acceptance:**
- [ ] 每个窗口/workspace 动作与 rill-ed 语义一致（边界：首/末窗口、workspace 0/9、fullscreen 短路）
- [ ] dispatcher 按类别拆 helper（对齐架构分析 #9），非 436 行单体
- [ ] 单测：至少覆盖 `adjust_window_width` clamp 与 workspace 边界

**Verification:**
- [ ] `cargo test keybinding::`
- [ ] `cargo clippy -- -D warnings`

**Dependencies:** T14、T8

**Files:** `src/keybinding.rs`

**Scope:** L

## Task 16 ✓: action 分发 — 输出/overview/session/spawn 动作

**Description:** `src/keybinding.rs` 输出/overview/session/spawn 类 helper：focus·move output（方向邻接匹配 + `needs_pointer_warp`）、enter_overview、reload_config（重载 + 重建绑定 + 光标 + layout.update）、spawn、exit、overview 导航/确认/取消。移植 rill-ed `keybindingPressed` + `overviewKeyPressed` 对应分支。

**Acceptance:**
- [ ] 输出方向匹配（矩形邻接）一致；`reload_config` 失败保留旧配置
- [ ] overview 键位（含 toggle 退出、Return 选中、边界 clamp）一致
- [ ] 无 no-op 存根残留

**Verification:**
- [ ] `cargo clippy -- -D warnings`；真实 river T18 验收

**Dependencies:** T15、T13、T3、T12

**Files:** `src/keybinding.rs`

**Scope:** M

## Task 17 ✓: main/app 装配（manage 状态机 + session lock）

**Description:** `src/main.rs` + `src/app.rs`：`AppData` 持 `WindowManager`；所有 `Dispatch` impl；registry 全局绑定（含 layer-shell）；事件循环；`manage()` 状态机（layout/pointer_action/overview/setup_bindings/exit/none，**无 animation**，layout 后直接 snap 转 none）；session_locked/unlocked 焦点保存/恢复。删除 `main.rs` 内联旧实现与临时 `allow(dead_code)`。

**Acceptance:**
- [ ] 替换后 `main.rs` 只剩装配/事件循环/Dispatch 接线，无内联 WM 逻辑
- [ ] 状态机无 animation 分支；会话锁保存/恢复焦点一致
- [ ] `cargo clippy -- -D warnings` 全绿（临时 allow 已移除）

**Verification:**
- [ ] `cargo build --release`、`cargo test`、`cargo clippy -- -D warnings`、`cargo fmt --check`

**Dependencies:** T9–T16

**Files:** `src/main.rs`、`src/app.rs`（新）

**Scope:** L

## Task 18 ✓: 默认配置示例 + README + 手动验收

**Description:** 提交 `config.example.toml`（对齐 `config.zon`，去动画字段）、更新 `README.md`（构建/运行/依赖/键位表/与 rill-ed 差异）、许可证头改 MIT + rill-ed 署名。产出并跑通手动验收清单。

**Acceptance:**
- [ ] README 依赖含 `libxkbcommon`；键位表与 rill-ed 一致；说明无动画
- [ ] 许可证头 0BSD→MIT，含 rill-ed attribution
- [ ] 手动验收清单全通过

**Verification:**
- [ ] 真实 river 下逐项：布局、浮动、overview、多显示器迁移、Super+r 热重载、TTY 切换不丢状态、锁屏恢复焦点、音量键、Super+t spawn

**Dependencies:** T17

**Files:** `config.example.toml`（新）、`README.md`、各文件头 SPDX

**Scope:** M

## Status

T1–T17 已实现（2026-08-28 会话）。T18 的手动验收清单（真实 river 会话）待执行：布局、浮动、overview、多显示器迁移、Super+r 热重载、TTY 切换、锁屏恢复、音量键、Super+t。
