# Implementation Plan: rwm — rill-ed 的 Rust 移植（无动画）

## Overview

将 rill-ed（Zig，约 2700 行）移植为 Rust 单二进制 river 窗口管理器：功能对齐（滚动/浮动布局、10 workspace/输出、overview、多显示器迁移、TTY 韧性、热重载、键位/指针绑定、窗口规则、边框、会话锁、spawn），**移除全部动画与 kwim 热插拔**。沿用仓库现有 `wayland-client` 0.31 阻塞式脚手架，配置 ZON→TOML，许可证 MIT。

## Architecture Decisions

- **事件循环**：`wayland-client` 阻塞式 `EventQueue`（现有脚手架，已验证），不用 calloop；动画与 kwim 删除后**无 timer**。
- **协议绑定**：`wayland-scanner` 宏编译期从 XML 生成；`mod river` 从 `main.rs` 抽到 `src/river.rs`；补 `river-layer-shell-v1.xml`（layer-shell 焦点跟踪用），不引 `river-input-management-v1`（kwim 已砍）。
- **状态模型**：`AppData` 持有 `WindowManager`。Zig 的「裸指针 + 模块级全局 `pending_windows`」→ Rust 字段；Wayland proxy 是 `Clone` 句柄，事件经 `Dispatch` trait 以 `&mut self` 传递，无裸指针/全局可变状态。
- **布局快照**：无动画 → `layout.apply` 直接 `set_position`（对齐 rill-ed `-Danimation=false` 的 `snapToFinish` 路径），无 `Spring`/`finish`/`is_animating`。
- **配置**：`serde` + `toml`，`KeybindingAction` 用 `#[serde(tag="action", content="arg", rename_all="snake_case")]` 对齐 ZON 形状。
- **构建策略（保持绿）**：增量重写。T2–T16 为新模块，编译 + 单测持续通过，`main.rs` 仍跑现有最小 WM；T17 用新模块树替换 `main.rs` 内联实现并删除旧代码。未接线模块临时 `#[allow(dead_code)]`，T17 移除。

## Task List（索引 — 详情与验收见 `tasks/todo.md`）

### Phase 1: Foundation
- T1 协议代码生成模块化 + river-layer-shell 绑定
- T2 `types`/`actions` 数据模型（无动画字段）
- T3 `config` TOML 加载/热重载

### Phase 2: 布局几何（纯函数 + 单测）
- T4 `layout/common` 矩形助手
- T5 `layout/scroller` 滚动列
- T6 `layout/floating` 浮动布局

### Phase 3: 状态与布局协调
- T7 `wm` WindowManager 状态 + 索引逻辑
- T8 `layout` 协调器（apply/update/边框/焦点，快照）

### Phase 4: 生命周期与 IO
- T9 `window` 窗口生命周期
- T10 `output` 输出管理 + `detached_outputs` 保存/恢复
- T11 `seat` 指针绑定 + layer-shell 焦点跟踪
- T12 `spawn` setsid + 双重 fork

### Phase 5: overview + 键位
- T13 `overview` 网格模式
- T14 `keybinding` keysym 解析 + 默认表 + 绑定建立/路由
- T15 `keybinding` action 分发 — 窗口/workspace 动作
- T16 `keybinding` action 分发 — 输出/overview/session/spawn 动作

### Phase 6: 装配与收尾
- T17 `main`/`app` 装配（manage 状态机、session lock、删旧代码）
- T18 默认配置示例 + README + 手动验收清单

### Checkpoints
- **Checkpoint 1**（T1–T3 后）：`cargo build --release` + `cargo test` 绿；数据模型/配置单测通过。
- **Checkpoint 2**（T4–T6 后）：几何单测全绿（rectangle/center/resize/move 边界、scroller 比例、floating）。
- **Checkpoint 3**（T7–T8 后）：`move_window_to_workspace` 索引语义、布局快照单测绿。
- **Checkpoint 4**（T13–T16 后）：全量 action 移植完毕，无 no-op 存根；`cargo clippy -- -D warnings` 绿。
- **Complete**（T17–T18 后）：功能对齐表全满足，手动验收清单通过。

## Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| action 分发器体量大（Zig `keybindingPressed` 436 行） | 高 | 拆成 per-category helper（对齐 rill-ed 架构分析 #9），T15/T16 各管一类 |
| serde tagged enum 形状与 ZON 语义不一致 | 中 | T2/T3 单测锁定 TOML↔action 反序列化，含带参 action |
| `detached_outputs`（TTY/热插拔）生命周期难推理 | 中 | T7/T10 用纯函数单测 restore 语义，T18 手动验收 TTY 切换 |
| 新旧代码并存期 clippy 警告 | 低 | 增量重写 + 临时 `allow(dead_code)`，T17 移除并强制 `-D warnings` |
| `xkbcommon` C 依赖在目标机缺失 | 低 | 记录在 README 依赖；keysym 名解析仅 `xkb::keysym_from_name` 一处 |
