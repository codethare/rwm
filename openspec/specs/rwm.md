# Spec: rwm — rill-ed 的 Rust 移植版（无动画）

## Objective

将 [rill-ed](https://github.com/codethare/rill-ed/)（一个基于 river 的极简滚动式窗口管理器，约 2700 行 Zig）移植为 Rust，功能对齐，但**移除全部动画能力**。产物是单个二进制，运行在 river 上，实现 `river-window-management-v1` 协议，供用户作为 river 的 `-c` 命令启动。

用户：使用 river + Wayland 的极简主义桌面用户。成功 = 与 rill-ed 相同的人机交互（键位、滚动/浮动布局、overview、多显示器、TTY 切换不丢状态、热重载配置），只是布局变化是**瞬时跳变**而非弹簧插值动画。

## 假设（ASSUMPTIONS — 请现在纠正，否则我按此推进）

1. **Wayland 栈沿用现有脚手架**：`wayland-client` 0.31 阻塞式 `EventQueue`（非 calloop），`wayland-scanner` 宏从 `protocol/*.xml` 编译期生成绑定。这是仓库里已经跑通的选择，不重造。
2. **配置格式**：ZON（Zig 专有）→ **TOML**（`toml` + `serde`）。键名/值尽量与 `config.zon` 一一对应，`animation_*` 字段删除。
3. **keysym 名解析**用 `xkbcommon` crate（`xkb::keysym_from_name`），对齐 Zig 的 `xkbcommon.Keysym.fromName`。
4. **kwim 热插拔集成整体砍掉**（kwim 是 Zig 输入法，Rust 无对应物；且它依赖 `river-input-management-v1`/`river-libinput-config-v1` 等协议）。
5. **无定时器**：动画删除后，rill-ed 里唯一的 timer 用途（kwim 热插拔防抖）也随之消失，阻塞式事件循环不需要 timer。
6. **单一 crate**（不拆 workspace）：~3k 行规模，一个 crate + 模块目录足够。
7. **许可证：MIT**（已定）。文件头 SPDX 从 0BSD 改为 MIT，并保留 rill-ed 原版权署名（© Zhijian Li）作为 attribution（NOTICE 或文件头）。
8. **默认终端保持 `alacritty`**（Super+t）、音量键保持 `wpctl` spawn，与 rill-ed 一致。

## 功能对齐表（保留 / 移除）

### 保留（忠实移植）

| 功能 | 说明 |
|---|---|
| 每 workspace 布局 | `scroller`（滚动列）/ `floating`（浮动）两种，每 workspace 独立 |
| 10 个 workspace/输出 | 上下/编号切换，`previous_workspace` 记忆 |
| 浮动窗口 | 居中、鼠标拖拽移动、鼠标 resize、键盘调整尺寸/移动（50px 步进） |
| 多显示器 + 窗口迁移 | 方向键切换/迁移窗口到相邻输出，`former_output_name` 记录来源 |
| TTY/显示器热插拔韧性 | `detached_outputs`（按输出名保存 workspace，输出重现时恢复） |
| Overview 模式 | 跨输出/workspace 的网格视图、vi/方向键导航、Return/点击选中、Escape 取消 |
| 配置热重载 | Super+r 重载 TOML、重建绑定、重排布局，失败保留旧配置 |
| 键位 + 指针绑定 | 完整默认键位表（含 `adjust_window_width` 等带参 action）、指针 move/resize |
| 窗口规则 | `app_id` 精确 + `title` glob（`*`/`?`）匹配，命中强制 `floating` |
| 边框渲染 | 聚焦/非聚焦颜色与宽度 |
| 会话锁 | 锁屏保存/恢复焦点（ext-session-lock） |
| layer-shell 焦点跟踪 | `none`/`non_exclusive`/`exclusive` 状态 |
| spawn 外部程序 | `setsid` + 双重 fork 分离（对齐 Zig 的 `spawnDetached`） |
| PipeWire 音量键 | XF86Audio* → `wpctl` spawn |

### 移除

| 项 | 原因 |
|---|---|
| **全部动画**：`animation.zig`、`Spring`、`finish` 矩形、帧插值、`is_animating`、`snapToFinish` | 用户明确要求 |
| 配置字段 `animation_duration` / `animation_stiffness` / `animation_damping_ratio` | 随动画删除 |
| kwim 热插拔（`kwim_hotplug.zig` + `river-input-management-v1` 等协议） | Zig 专有，见假设 4 |
| ZON 配置格式 | Zig 专有，改 TOML |

## Tech Stack

| 依赖 | 版本 | 用途 |
|---|---|---|
| Rust | edition 2024, stable | 语言 |
| `wayland-client` | 0.31 | Wayland 客户端协议 |
| `wayland-backend` | 0.3 | `wayland-client` 底层 |
| `wayland-scanner` | 0.31 | 编译期从 XML 生成协议绑定 |
| `bitflags` | 2 | `Modifiers` / `Edges` 位标志 |
| `toml` + `serde` | 最新稳定 | TOML 配置解析 |
| `xkbcommon` | 最新稳定 | keysym 名 → u32 |

协议 XML 需补齐：`river-layer-shell-v1.xml`（layer-shell 焦点跟踪）需加入 `protocol/` 并生成；现有 `river-window-management-v1.xml`、`river-xkb-bindings-v1.xml` 已就绪。

## Commands

```sh
cargo build --release        # 构建
cargo test                   # 单元测试
cargo clippy -- -D warnings  # lint
cargo fmt --check            # 格式
river -c ./target/release/rwm   # 运行（river 的 init 中）
```

## Project Structure

```
rwm/
  Cargo.toml
  protocol/                  # river 协议 XML（wayland-scanner 输入）
  src/
    main.rs                  # 入口：连接、registry 全局绑定、事件循环
    river.rs                 # 协议代码生成模块（从 main.rs 抽出）
    app.rs                   # AppData：Dispatch 状态根，持有 WindowManager
    wm.rs                    # WindowManager：Output/Workspace/Window 状态 + manage 周期
    types.rs                 # Config、Rectangle、Border、Color、枚举
    actions.rs               # KeybindingAction、PointerAction、Button
    config.rs                # TOML 加载/热重载、默认值
    layout/
      mod.rs                 # 布局协调器 apply/update
      scroller.rs            # 滚动列布局
      floating.rs            # 浮动布局
      common.rs              # 矩形几何助手（initial/center/resize/move）
    window.rs                # 窗口生命周期 + Dispatch<RiverWindowV1>
    output.rs                # 输出管理 + Dispatch<RiverOutputV1>
    seat.rs                  # seat + 指针绑定 + Dispatch<RiverSeatV1>
    keybinding.rs            # keysym 解析 + 绑定建立 + action 分发
    overview.rs              # overview 网格模式
    spawn.rs                 # setsid + 双重 fork spawn
  openspec/specs/rwm.md      # 本 spec
  tasks/                     # plan.md + todo.md（Phase 2/3 生成）
```

对应 rill-ed 的 `src/*.zig` 布局；`animation.zig`、`kwim_hotplug.zig` 不存在。

## Code Style

命名：模块 snake_case、类型 CamelCase、字段 snake_case。`rustfmt` 默认 + clippy 严格。文件头保留 SPDX 头（与现有脚手架一致）。

关键差异：Zig 的「单 `WindowManager` 全局 + 裸指针 + 模块级 `pending_windows` 全局变量」在 Rust 里改为 `AppData` 持有 `WindowManager`，Wayland proxy 是 `Clone` 句柄，事件经 `Dispatch` trait 以 `&mut self` 传递，无裸指针/全局可变状态（`pending_windows` 成为 `WindowManager` 的字段 —— 恰好是 rill-ed 架构分析里建议的改进方向）。

Action 枚举（serde tagged union，形状对齐 Zig 的 `.close_window` / `.{ .adjust_window_width = -0.1 }`）：

```rust
#[derive(Deserialize, Serialize, Clone, Debug)]
#[serde(tag = "action", content = "arg", rename_all = "snake_case")]
enum KeybindingAction {
    CloseWindow,
    ToggleFullscreen,
    AdjustWindowWidth(f32),        // TOML: action = { adjust_window_width = -0.1 }
    SetWindowWidth(f32),
    FocusWorkspaceNumber(usize),   // TOML: action = { focus_workspace_number = 1 }
    MoveWindowLeft,
    // ...其余对齐 actions.zig 的 KeybindingAction
    Spawn(Vec<String>),            // TOML: action = { spawn = ["alacritty"] }
}
```

TOML 配置片段：

```toml
vertical_gap = 9
horizontal_gap = 9
default_window_width = 0.5
center_focused_window = "never"   # never | always | single
no_csd = true

[border]
width = 3
focused_color = { r = 141, g = 214, b = 0, a = 1.0 }
unfocused_color = { r = 160, g = 160, b = 160, a = 1.0 }

[[keybindings]]
key = "q"
modifiers = ["mod4"]
action = "close_window"

[[keybindings]]
key = "minus"
modifiers = ["mod4"]
action = { adjust_window_width = -0.1 }

[[window_rules]]
app_id = "footclient"
floating = true
```

## Testing Strategy

`cargo test` 覆盖**纯函数**，不 mock Wayland（对齐 rill-ed 架构分析第 13/17 条建议 —— 几何与索引逻辑是回归高发区）：

- `layout/common.rs`：`initial_rectangle` / `center_rectangle` / resize / move 边界（gap、最小尺寸 clamp）
- `layout/scroller.rs`：滚动列宽度分配、`proportion` clamp
- `config.rs`：TOML 反序列化（含 action 枚举、modifiers、window_rules）、缺省值、非法配置
- `keybinding.rs`：glob 匹配（`*`/`?`，含回溯）、keysym 名解析（默认键位表全部可解析）
- `wm.rs` / `keybinding.rs`：`move_window_to_workspace` 的焦点索引调整（窗口移除后 `focused_window_idx` 的移动语义）

真实 river 下的集成行为（布局、overview、TTY 切换）在 v1 用手动验收清单验证，不写自动化集成测试。

## Boundaries

- **Always**：`cargo fmt` + `cargo clippy -D warnings` + `cargo test` 通过后才算完成；模块划分对齐上面的目录；SPDX 头保留。
- **Ask first**：新增依赖；改协议版本 pin；改默认键位表/默认配置值；换配置格式。
- **Never**：提交密钥；删除 SPDX 头；手改 `river.rs` 生成的代码；用 `unwrap()`/`expect()` 处理可能来自 compositor 的运行时输入（事件里的 Option 要显式处理）。

## Success Criteria

1. `cargo build --release` 零警告；`cargo test`、`cargo clippy -- -D warnings`、`cargo fmt --check` 全绿。
2. 功能对齐表「保留」列全部实现，「移除」列全部不出现（`animation*`、`Spring`、`is_animating` 等在代码库中不存在）。
3. 在真实 river 下：默认键位表逐条可用；浮动/滚动布局、overview、多显示器迁移、`Super+r` 热重载、TTY 切换后 workspace/窗口不丢失。
4. 配置：`config.zon` 的每个非动画字段在 TOML 中有对应项，默认行为与 rill-ed 一致（不含动画）。
5. 纯函数有 `cargo test` 覆盖（见 Testing Strategy）。

## Open Questions（已全部解决）

1. 许可证 → **MIT**（含 rill-ed 署名）。
2. 配置格式 → **TOML**。
3. kwim 热插拔 → **砍掉**（未来要再加：补 `river-input-management-v1` 协议 + 重新引入 timer）。
4. 默认终端 `alacritty` / 音量 `wpctl` → **保持 rill-ed 默认**。
