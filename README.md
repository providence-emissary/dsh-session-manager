# dsh-session-manager

![License](https://img.shields.io/badge/license-MIT-green)
![Node](https://img.shields.io/badge/node-%3E%3D22.13-blue)
![Platform](https://img.shields.io/badge/platform-Windows%20%7C%20macOS%20%7C%20Linux-lightgrey)

DeepSeek Harness 会话管理插件：跨工作区**移动**、**删除（回收站）**、**点击行跳转**。
全部本地运行，零外部依赖，自带安装器。

> Session manager for DeepSeek Harness: move sessions across workspaces, delete to the
> OS Recycle Bin, click a row to open the session. Fully local, zero dependencies,
> ships its own installer.

## 功能

- **会话清单**：按工作区分组，标题后台扫描渐进填充（面板秒开）；当前会话置顶
  展示；一键隐藏子代理会话（默认开）。
- **点击行跳转**：点击会话行（标题 / id / 整行）直接在主界面打开该会话，与侧栏
  点击行为一致（主界面切换 + 面板收起）。归档会话不可直接打开（提示先取消归档）。
- **跨工作区移动（真·移动）**：改写会话日志 header 的 `cwd` → 把工件目录搬到目标
  `projectKey` 下 → 旧工作区解挂 → 重建索引 → 新工作区挂接（目标目录没有工作区
  登记时自动创建）。
- **删除会话**：工件移入 Windows 回收站（其他平台进插件 trash 目录），工作区归属
  与归档/置顶标记一并清除。删除弹窗按 id 末 6 位二次确认，支持**「一键填入」**。
- **一键删除归档会话**：批量删除全部已归档会话，逐个走完整删除管线，运行中的
  自动跳过，逐条明细回报。
- **分组自愈**：「修复分组」按钮按 `header.cwd` 对齐工作区登记（幂等，可重跑），
  修复移动后不显示 / 分组错乱；投影缓存身份失配自动检测并恢复标题显示。

## 系统要求

- DeepSeek Harness 桌面版，且至少成功启动过一次（生成 profile 目录）
- Node.js ≥ 22.13（宿主自带运行时已满足）
- Windows 10+ 可用回收站；macOS / Linux 自动降级为插件 trash 目录
- 系统自带 `tar` 命令（仅安装时使用）

## 安装

先把 profile 目录准备好（`~/.dsh/profiles/desktop/`，默认 DSH_HOME；自定义
`DSH_HOME` 的机器无需额外参数——安装器与宿主同口径解析）。

### 方式 A：自带安装器（推荐）

```sh
# 直接装 tarball：
node install.mjs dsh-session-manager-0.3.1.tgz

# 或先解开再装（效果相同）：
tar -xzf dsh-session-manager-0.3.1.tgz
cd package
node install.mjs <tarball 路径>

# 非默认 profile：
node install.mjs <tarball|目录> <profile 目录路径>
```

装完**重启 DSH**，右侧栏底部出现「🗂 会话管理」即成功。

### 方式 B：插件管理器 GUI

侧栏「插件」页 → 安装 → 选择本地 tarball。GUI 走宿主包管理器流程，若报依赖
解析错误，那是 profile 既有状态所致（本插件零依赖），改用方式 A 即可。

### 安装器做了什么（三步，幂等可重跑）

1. `package.json`：`dependencies.dsh-session-manager` 指向 `vendor/` 里的 tarball
   （其他依赖原样保留），`dsh.profile.bundles` 追加 `dsh-session-manager`；
2. `cordis.patch.yml`：追加 `- insert` 条目（已有则跳过）；
3. `node_modules/dsh-session-manager/`：包内容原位展开。

**不改动** `pnpm-lock.yaml` 与任何其他插件。

## 使用

| 操作 | 怎么做 |
| --- | --- |
| 打开面板 | 右侧栏底部「🗂 会话管理」按钮 |
| 打开会话 | 点击会话行（归档会话会提示先取消归档） |
| 移动 | 行内「移动」→ 选已有工作区或输入目录（目录必须已存在） |
| 删除 | 行内「删除」→ 输入 id 末 6 位或点「一键填入」→「删除」 |
| 批量删除归档 | 「删除归档会话（N）」→ 勾选确认 |
| 修复分组 | 「修复分组」按钮（幂等，随时可点） |
| 搜索过滤 | 顶部输入框按 id / 标题 / 工作区过滤 |

## 安全与数据保护

- 全部操作本地执行，**不发起任何网络请求**；标题缓存与备份都在
  `$DSH_HOME/dsh-session-manager/` 下。
- 移动 / 删除前自动备份原日志到 `backups/<tag>/`。
- 删除进 Windows 回收站可找回；其他平台进插件 trash 目录。
- 日志重写遵循宿主 zstd 帧布局契约（`[header帧][事件帧]`），不会产生被宿主
  静默剔除的"幽灵会话"。
- 删除完成后即时通知客户端清行，不留"点击报 not found"的滞留条目。
- 修复类操作全部幂等，可安全重跑。

## API 面（面板路由）

| 路由 | 请求 | 说明 |
| --- | --- | --- |
| `POST /api-session-mgr/sessions` | `{}` | 会话清单（标题可能带 `titlePending: true`） |
| `POST /api-session-mgr/titles` | `{ since, waitMs? }` | 标题长轮询（等下一批扫描结果） |
| `POST /api-session-mgr/workspaces` | `{}` | 工作区列表 |
| `POST /api-session-mgr/projcheck` | `{}` | 投影缓存身份失配检测 |
| `POST /api-session-mgr/archived` | `{}` | 已归档会话清单 |
| `POST /api-session-mgr/move` | `{ sessionId, targetDir }` | 跨工作区移动 |
| `POST /api-session-mgr/delete` | `{ sessionId, confirm: true }` | 删除单个会话 |
| `POST /api-session-mgr/delete-archived` | `{ confirm: true }` | 删除全部归档会话 |
| `POST /api-session-mgr/repair` | `{ confirm: true }` | 全库成员对齐 + 帧布局修复 |

## 硬约束（宿主契约决定）

- 运行中的会话、当前正在对话的会话不能移动 / 删除。
- 移动目标目录必须真实存在（不代建目录，与宿主 `workspaceRegistry.create`
  同口径）。
- 移动 = 改 header + 搬目录 + 重挂工作区三步，缺一步都会变幽灵会话。

## 仓库结构

```
lib/index.mjs         入口：组合层，经 webServer 可选服务注册面板路由
lib/sessions.mjs      核心：枚举 / 工件定位 / 守卫 / 索引重建 / 回收站（零宿主依赖）
lib/ops.mjs           编排：moveSession / deleteSession / 批量删除归档
lib/panel.mjs         路由：POST /api-session-mgr/*
lib/titles.mjs        标题后台扫描（增量 + 落盘缓存）
lib/client.js         Web 面板 bundle（右侧栏 tab，纯前端）
lib/vendor/fzstd.mjs  内置 zstd 解码（MIT，见 lib/vendor/fzstd.LICENSE）
install.mjs           自包含安装器
```

## 卸载

删除 `node_modules/dsh-session-manager/` 与 `vendor/dsh-session-manager-*.tgz`，
并从 `package.json` 的 `dependencies` / `dsh.profile.bundles` 与 `cordis.patch.yml`
移除对应条目。标题缓存与备份在 `$DSH_HOME/dsh-session-manager/`，按需清理。

## 更新记录

- **0.3.1** 删除确认弹窗新增「一键填入」。
- **0.3.0** 点击会话行跳转主界面（归档门禁与宿主一致）。
- **0.2.9** 修复删除后侧栏幽灵行（删除后补发 `api-session/removed`，客户端
  `refreshList` 兜底）。
- **0.2.8** 修复一键删除归档会话：前端 TDZ 白屏 + 后端缺失导入。
- **0.2.7** 隐藏子代理开关 + 一键删除归档会话。
- **0.2.6** 面板空白修复（渲染期 TDZ 回归）。
- **0.2.5** 移动后投影缓存失配自愈（左栏无标题 / 搜索不到）。
- **0.2.4** 当前会话置顶区。
- **0.2.3** 日志帧布局契约根修（宿主同款 `[header帧][事件帧]`）+ 全库帧布局修复。
- **0.2.2** 成员登记修复（对所有工作区无条件幂等 detach）。
- **0.2.1** 弹窗对比度修复（深色主题可读）。
- **0.2.0** 面板秒开（标题后台扫描 + 落盘缓存）+ 自包含安装器。

## License

MIT（`lib/vendor/fzstd.mjs` 为 MIT，见 `lib/vendor/fzstd.LICENSE`）
