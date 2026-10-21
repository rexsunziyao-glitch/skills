---
name: settings-feature-integration
description: >-
  Use when adding, changing, refactoring, or reviewing Folia (folia-major)
  user-facing settings, especially visual settings, appearance settings,
  visualizer tuning, background/theme/typography controls,
  playback/integration/storage/desktop/lab settings, or any setting that should
  be importable/exportable or invokable from the command palette.
---
# Settings Feature Integration

Source: [chthollyphile/folia-major `skills/settings-feature-integration`](https://github.com/chthollyphile/folia-major/tree/main/skills/settings-feature-integration)

这个 skill 用于新增或调整 Folia 设置功能时，避免设置只在 UI 上“看得到”，但没有接入备份、恢复、命令面板或统一入口。

## First Decision

先把设置分成两类：

- 视觉相关设置：影响主题、歌词动画、背景、透明度、字体、字号、visualizer 模式、visualizer tuning、封面/Monet 背景、预览表现。
- 功能性设置：影响播放行为、搜索/导航、集成、存储、桌面端、实验功能、面板行为，或本质上是一个可执行动作。

如果一个设置同时有视觉和功能属性，同时执行两类接入规则。

## Visual Settings Must Join Import / Export

视觉相关设置（例如主题、字重 `fontWeight`、歌词动画、背景、透明度、字号、visualizer tuning）必须接入外观页的视觉配置导入导出：

接入点分在两个文件，别只看其中一个：

- UI 与装配：`src/components/modal/settings/AppearanceSettingsSubview.tsx`
  - 导出入口：`buildCurrentConfig`
  - 导入应用：`handleImportConfig`
- 编解码实现：`src/utils/appearanceCodec.ts`
  - 短码压缩：`compressConfig`
  - 短码解压：`decompressConfig`
  - JSON 白名单：`validKeys`（在 `decompressConfig` 内部）

subview 只是调用方；新增字段要同时改这两处，光改 subview 不会进短码。

新增 visualizer tuning（例如 `claddaghTuning`、`monetTuning`、`dioramaTuning`、`pendoloTuning`、`sonnetTuning` 等）或全局字重 `fontWeight` 时通常还要同步：

- `src/types.ts`：新增 tuning / 设置类型和默认值
- 对应领域的 `src/stores/use*SettingsStore.ts`（visualizer tuning 在 `useVisualizerSettingsStore`，字重/字体在 `useTypographySettingsStore`）：读取、持久化、setter、resetter、draft 逻辑；清空自定义字体栈时同步清空字重
- `src/utils/fontStacks.ts`：通过 `resolveThemeFontWeight(theme, modeFallback)` 在渲染与测量中统一解析
- `src/components/visualizer/definition.ts`：把 tuning 或资源 props 加到共享契约
- 各 visualizer 模式的 `entry.tsx`：通过 registry 挂载 renderer、设置面板和 reset
- `src/components/visualizer/VisPlayground.tsx` 或相邻设置面板：透传和编辑 tuning
- `src/i18n/locales/en.ts` / `src/i18n/locales/zh-CN.ts`：补设置文案

不要只把视觉设置写进 localStorage 或 store；如果用户会把它理解成“外观配置的一部分”，它就必须能随 shortcode / JSON 一起导入导出。

已知例外（用户明确决定，不要按上面的规则补进导入导出）：

- bravais 资料库墙的「透光」偏好（`src/stores/useLibraryWallLookStore.ts` 的 `look` 与 `windowsPerBlock`）。它照常接命令面板，但不进外观短码 / JSON。
- 资料库界面（UI suite）的选择（`useLibrarySuiteStore`）是界面偏好，不是视觉调参，同样不进导入导出。

新增类似的例外时，必须是用户明确的决定，并在 store 或设置组件的注释里写明。

## Functional Settings Must Join Command Palette

功能性设置或可执行动作必须评估并注册到 command palette：

- 文件：按命令的 `group` 落到 `src/components/command-palette/commands/<Group>Commands.ts`（`search` / `playback` / `settings` / `navigation` / `panel` / `visualizer`）。不要再往 `commandRegistry.ts` 里加命令。
- 构造入口：一律经 `src/components/command-palette/commandFactories.ts` 的 `defineCommand`；优先复用 `createToggleCommand`、`createSettingsCommand`、`createPanelCommand`、`createHomeTabCommand`、`createVisualizerCommand`。
- 上下文：`CommandPaletteContext` 七个命名空间；新能力同时改 `types.ts` 和 `useCommandPaletteContext.ts`。
- 平台可见性：写 `platform: ['electron' | 'win' | 'mac' | 'linux' | 'web']`，不要在过滤函数里加 id 判断。状态可见性用 `isAvailable`。
- 面板内界面：写 `surface`，组件用 `load: () => import(...)` 惰性加载。
- 复杂语法：声明 `syntax`，解析复用 `src/components/command-palette/syntax/`。
- 执行模式：危险、不可撤销、要花钱或需要确认的操作不给 `executeShortcut`。
- 命令文案：同步 `en.ts`、`zh-CN.ts` 和 `in.ts` 的 `commandPalette.commands.*`。
- 关键词：写英文与中文，**不要手写拼音**（构建期插件生成）。
- 落地列表：由 `commands/index.ts` 的 `DEFAULT_LANDING_COMMAND_IDS` 显式决定。

同步相关优先复用 `src/services/sync/syncCoordinator.ts`（`settings-r2-sync` / `sync-now`）。

新增设置子视图时，至少添加一个能打开该子视图的 settings command。开发者设置子视图刻意不提供命令入口。

`isAvailable` 必须与设置 UI 用同一个判断。

## Settings UI Placement

优先放在现有分区：

- 外观：`AppearanceSettingsSubview.tsx`
- 播放：`PlaybackSettingsSubview.tsx`
- 通用 / Bravais 墙面：`GeneralSettingsSubview.tsx`、`BravaisSettingsSection.tsx`
- 集成：`IntegrationSettingsSubview.tsx`
- 存储：`StorageSettingsSection.tsx`
- 桌面端：`DesktopSettingsSubview.tsx`
- 图形：`GraphicsSettingsSubview.tsx`
- 模组（仅桌面）：`ModsSettingsSubview.tsx`
- 实验室：`LabSettingsModal.tsx`
- 固定命令：`PinnedCommandSettings.tsx`
- visualizer 专属：模式设置面板 + registry `renderSettingsPanel`

不要继续堆回 `SettingsModal.tsx` 的大 JSX；需要新区域时先按 `file-modularization` 拆分。

## Review Checklist

- 这个设置是视觉相关、功能性，还是两者都是？
- 视觉设置是否进入 `buildCurrentConfig`、`compressConfig`、`decompressConfig`、`validKeys` 和 `handleImportConfig`？
- 功能性设置或动作是否进入对应的 `*Commands.ts`？
- 平台限定用了 `platform` 而不是 id 判断？需要界面的用了 `surface`？
- 是否明确决定过要不要给 `executeShortcut`？
- 命令是否有 en / zh-CN / in 三份 i18n、中英文关键词（拼音不手写）？
- store、localStorage key、默认值、resetter、导入恢复是否一致？
- 同步相关是否同时接入 `sync/settingsSnapshot.ts`、存储设置和命令面板？
- 新增用户可见文案是否同步中英文？
- 是否避免继续膨胀 `SettingsModal.tsx`、`VisPlayground.tsx` 或单个 visualizer 大文件？
