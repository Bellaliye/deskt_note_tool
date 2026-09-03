# macOS 桌面速查工具产品设计

日期：2026-09-03
状态：待用户复核

## 1. 产品目标

为本机所有者提供一个常驻后台的 macOS 速查工具。用户在写代码或使用 Mac 时，可以快速查看容易忘记的 macOS 快捷键和 Git 指令，不再中断当前工作去浏览器搜索。

工具像贴在屏幕右侧的标签：平时隐藏，需要时从右侧唤出。面板内不显示产品标题。

## 2. 第一版范围

第一版只有两个固定标签：

- `macOS`：预置完整、可直接参考的系统快捷键。
- `Git`：预置完整、可直接照着输入终端的 Git 指令。

两个标签内部均支持：

- 新增条目。
- 修改条目的“用途名称”和“快捷键或指令”。
- 删除已经记住的条目。
- 拖动排序，把最不熟悉的内容放在顶部。
- 自动保存，应用重启后保持内容和顺序。

第一版不提供新增、重命名或删除标签。界面中不显示不可用的“＋”按钮。

## 3. 成功标准

- 在电脑已解锁的任意状态下，3 秒内唤出面板。
- 面板打开后，10 秒内找到目标快捷键或指令。
- 在桌面、普通 App、不同桌面空间和全屏 App 中均可使用。
- 两个标签的条目新增、修改、删除和排序在应用重启后保持不变。
- 第一次启动即包含本设计第 6 节列出的全部 macOS 和 Git 内容，不需要用户手工录入。

## 4. 唤出、收起与标签切换

### 4.1 唤出与收起

- 鼠标进入当前屏幕右侧中间 160px 高的隐形触发区，并持续停留 0.5 秒，面板从右侧展开。
- 全局快捷键默认为 `Control + Option + Space`，用户可在设置中修改。
- 再次按全局快捷键、按 `Esc` 或点击面板外部时收起。
- 鼠标移出面板不会立即收起，避免用户移动鼠标操作时面板闪退。
- 面板显示在鼠标当前所在的屏幕。
- 应用以菜单栏形式常驻，不显示 Dock 图标；登录后默认自动启动，设置中可关闭。

### 4.2 标签切换

- 面板打开时恢复上次使用的标签。
- 点击顶部 `macOS` 或 `Git` 切换标签。
- 按左、右方向键切换标签；因为只有两个标签，任一方向都会切到另一个标签。

## 5. 查看与编辑

### 5.1 查看状态

- 默认进入查看状态。
- 每条内容整行左对齐，显示为“用途名称 · 快捷键或指令”。
- 不显示产品标题、说明文字、复制按钮、删除按钮或其他辅助操作。
- 普通条目不提供复制功能。

示例：

```text
推送当前分支 · git push
切换 App · ⌘ Tab
```

### 5.2 编辑状态

- 点击标签栏右侧铅笔按钮进入编辑状态。
- 编辑状态隐藏查看态内容，只显示拖动、修改和删除三个行内操作。
- 标签底部提供“新增条目”，只向当前标签添加内容。
- 修改或新增时只填写“用途名称”和“快捷键或指令”。
- 删除前显示一次轻量确认。
- 点击“完成”回到查看状态。
- 修改立即自动保存，不提供额外保存按钮。

### 5.3 自适应尺寸

- 面板宽度由当前标签最长单行内容决定，但不得超过当前屏幕可用宽度的 40%。
- 面板高度优先包裹全部条目。
- 高度超过屏幕可用高度 85% 时，先减小行距和内边距。
- 字号最低为 12px；仍放不下时固定为屏幕可用高度 85%，内容区域内部滚动。

## 6. 第一版预置内容

预置内容采用本地 TypeScript 常量，只在首次没有用户数据时写入。用户修改后，升级或重启不得重新覆盖。

### 6.1 macOS 标签

| 用途名称 | 快捷键或指令 |
|---|---|
| 切换到下一个 App | `⌘ Tab` |
| 反向切换 App | `⇧ ⌘ Tab` |
| 切换当前 App 的窗口 | <code>⌘ &#96;</code> |
| 打开 Spotlight | `⌘ Space` |
| 打开强制退出窗口 | `⌥ ⌘ Esc` |
| 锁定屏幕 | `⌃ ⌘ Q` |
| 打开或退出全屏 | `⌃ ⌘ F` |
| 打开调度中心 | `⌃ ↑` |
| 显示当前 App 的所有窗口 | `⌃ ↓` |
| 切换到左侧桌面空间 | `⌃ ←` |
| 切换到右侧桌面空间 | `⌃ →` |
| 显示桌面 | `⌘ Mission Control（F3）` |
| 截取整个屏幕 | `⇧ ⌘ 3` |
| 截取选定区域 | `⇧ ⌘ 4` |
| 打开截图与录屏工具 | `⇧ ⌘ 5` |
| 关闭当前窗口 | `⌘ W` |
| 最小化当前窗口 | `⌘ M` |
| 隐藏当前 App | `⌘ H` |
| 退出当前 App | `⌘ Q` |
| 打开 App 设置 | `⌘ ,` |
| Finder 前往文件夹 | `⇧ ⌘ G` |
| Finder 显示或隐藏隐藏文件 | `⇧ ⌘ .` |
| 快速预览所选文件 | `Space` |
| 打开表情与符号 | `⌃ ⌘ Space` |
| 向前删除字符 | `Fn Delete` |

macOS 快捷键依据 Apple 官方《Mac keyboard shortcuts》整理；只摘取快捷键事实并重新编写中文用途名称。

### 6.2 Git 标签

| 用途名称 | 快捷键或指令 |
|---|---|
| 查看当前状态 | `git status` |
| 简洁查看状态 | `git status --short` |
| 暂存指定文件 | `git add <file>` |
| 暂存当前目录全部改动 | `git add .` |
| 提交暂存内容 | `git commit -m "<message>"` |
| 推送当前分支 | `git push` |
| 首次推送并建立上游 | `git push -u origin <branch>` |
| 拉取并变基当前工作 | `git pull --rebase origin <branch>` |
| 获取远端更新但不合并 | `git fetch origin` |
| 查看当前分支名称 | `git branch --show-current` |
| 创建并切换新分支 | `git switch -c <branch>` |
| 切换已有分支 | `git switch <branch>` |
| 安全删除已合并本地分支 | `git branch -d <branch>` |
| 合并指定分支 | `git merge <branch>` |
| 把当前分支变基到 main | `git rebase main` |
| 整理最近三次提交 | `git rebase -i HEAD~3` |
| 继续处理变基 | `git rebase --continue` |
| 放弃本次变基 | `git rebase --abort` |
| 放弃本次合并 | `git merge --abort` |
| 取消暂存指定文件 | `git restore --staged <file>` |
| 放弃指定文件未暂存改动 | `git restore <file>` |
| 查看未暂存差异 | `git diff` |
| 查看已暂存差异 | `git diff --staged` |
| 查看精简图形日志 | `git log --oneline --graph --decorate --all` |
| 查看指定提交 | `git show <commit>` |
| 临时保存当前改动 | `git stash push -m "<message>"` |
| 查看临时保存列表 | `git stash list` |
| 恢复并移除最近暂存 | `git stash pop` |
| 把指定提交应用到当前分支 | `git cherry-pick <commit>` |
| 创建一个撤销指定提交的新提交 | `git revert <commit>` |
| 修改最近一次提交 | `git commit --amend` |
| 谨慎推送重写后的历史 | `git push --force-with-lease` |
| 查看引用变动记录 | `git reflog` |
| 查看远端仓库 | `git remote -v` |
| 添加 origin 远端 | `git remote add origin <url>` |
| 克隆远端仓库 | `git clone <url>` |
| 初始化本地仓库 | `git init` |

Git 指令依据 Git 官方 Cheat Sheet、命令文档和 Pro Git 整理。第一版不预置 `git reset --hard`、`git clean -f` 等容易直接丢失本地工作的命令。

## 7. 数据设计

```json
{
  "version": 1,
  "lastSelectedLabelId": "macos",
  "labels": [
    {
      "id": "macos",
      "name": "macOS",
      "order": 0,
      "items": [
        {
          "id": "本地生成的 UUID",
          "name": "切换到下一个 App",
          "content": "⌘ Tab",
          "order": 0
        }
      ]
    }
  ]
}
```

- 两个标签 ID 固定为 `macos` 和 `git`；第一版不提供标签级变更操作。
- 条目 ID 使用本地生成的 UUID。
- `order` 使用从 0 开始的连续整数，排序完成后重新编号。
- 不保存 `note` 或其他不展示的说明字段。
- 数据文件位于 Electron `app.getPath('userData')` 指向的目录。
- 每次写入采用同目录临时文件加原子替换，成功前保留上一份有效数据作为备份。

## 8. 开源调研与实现策略

### 8.1 调研结论

- [`tuckerwales/cheetos`](https://github.com/tuckerwales/cheetos) 功能最接近，但使用 Swift 且仓库没有明确许可证，不复制其代码或内置内容。
- [`PerpetualBeta/ShortcutHUD`](https://github.com/PerpetualBeta/ShortcutHUD) 可自由复用，但使用 Swift，主要解决当前 App 菜单快捷键枚举，移植和删减成本高于实现本项目。
- [`Anze/KeyCluCask`](https://github.com/Anze/KeyCluCask) 主要提供安装与说明文件，没有可作为基础的完整应用源码。
- [`RStankov/FocusedTask`](https://github.com/RStankov/FocusedTask) 使用 Electron + React 并采用 MIT 许可证，但依赖 Electron 8、React 17 和旧 Redux 架构，且业务与本项目无关，不作为代码底座。
- [`electron-vite/electron-vite-react`](https://github.com/electron-vite/electron-vite-react) 可作为参考，但包含第一版不需要的 Tailwind、更新器和额外构建配置，不直接复制整套模板。

因此不 fork 现有产品项目。工程使用 [Electron Forge 官方 `vite-typescript` 模板](https://www.electronforge.io/templates/vite)初始化，再按官方方式加入 React；产品业务代码从零实现。

### 8.2 技术选型

- TypeScript 作为统一开发语言。
- React 构建标签、查看态和编辑态。
- Vite 构建渲染页面，不使用 Next.js。
- Electron 主进程管理菜单栏、浮动窗口、全局快捷键、鼠标位置、开机启动和本地文件。
- `BrowserWindow` 使用 macOS `panel` 类型，并设置始终置顶、所有桌面空间可见和全屏 App 上方可见。
- 主进程每 100ms 读取鼠标位置，用状态机判断右侧触发区停留时间。
- 渲染进程启用 `contextIsolation` 并关闭 `nodeIntegration`。
- preload 只暴露读取文档、修改条目、排序、修改设置和收起面板所需的具名 API。
- 不引入 Redux、Zustand、数据库、CSS 框架、AI SDK 或拖动排序库；排序使用浏览器原生拖放事件。
- 测试使用 Vitest 和 React Testing Library；打包使用 Electron Forge。

## 9. 组件与数据流

### Electron 主进程

- `main.ts`：应用启动、菜单栏与组件组装。
- `panelWindow.ts`：创建、定位、显示和收起 `BrowserWindow`。
- `triggerController.ts`：全局快捷键和右侧停留状态机。
- `labelRepository.ts`：加载、校验、修改、原子保存和备份恢复。
- `settingsRepository.ts`：全局快捷键、停留时长与登录启动设置。
- `ipc.ts`：注册渲染进程允许调用的固定 IPC 通道。

### Preload

- `preload.ts` 使用 `contextBridge` 暴露最小 API。
- 不向页面暴露 Node.js、任意文件读写或任意 IPC 调用能力。

### React 渲染进程

- `App.tsx`：两标签切换、查看/编辑模式和键盘导航。
- `LabelList.tsx`：整行左对齐的查看态。
- `LabelEditor.tsx`：新增、修改、删除和拖动排序。
- `Settings.tsx`：快捷键、边缘停留时间和登录启动。
- 页面使用 React 内置 state/context，不增加全局状态库。

### 数据流

```text
用户操作 → React → preload 具名 API → Electron IPC → LabelRepository
→ 内存更新 → 原子写入 JSON → 最新文档返回 React
```

React 不能绕过主进程直接写入本地文件；主进程对每次输入重新校验。

## 10. 异常处理

- 用途名称或内容为空：阻止保存并指出具体字段。
- 本地保存失败：内存中保留用户刚做的修改，显示目标路径和系统错误原因。
- 主数据损坏：发现有效备份时询问用户是否恢复，不自动覆盖。
- 主数据和备份都损坏：保留两个文件并显示路径，以预置内容启动。
- 全局快捷键冲突：显示失败的组合键，继续保留上一个有效快捷键。
- 当前鼠标不属于任何已知屏幕：回退到主屏幕；仍无法获得屏幕时显示错误而不崩溃。
- 登录启动注册失败：恢复开关状态并提示在“系统设置 → 通用 → 登录项”中检查。

## 11. 验收与测试

### 自动测试

- 首次启动准确生成 25 条 macOS 内容和 37 条 Git 内容。
- 用户数据存在时不重复写入或覆盖预置内容。
- 条目新增、修改、删除和排序；排序后 `order` 连续。
- 两个标签的左右切换和上次标签恢复。
- JSON 读取、版本校验、原子写入与备份恢复。
- 空字段、未知标签和未知条目 ID 快速失败并包含实际值。
- 右侧触发区、0.5 秒停留和离开后重新计时。
- 面板 40% 宽度与 85% 高度限制。
- 查看态无编辑和复制入口；编辑态只有移动、修改、删除及新增条目。
- preload 不暴露 Node.js 或任意 IPC。

### 手工验收

- 桌面、普通 App、多个桌面空间和全屏 App 中均能唤出。
- 鼠标停留和默认全局快捷键均可打开；快捷键再次按下、Esc 和外部点击均可关闭。
- 面板显示在鼠标所在屏幕，移出后不会立即关闭。
- 查看态第一眼只看到两个标签和“用途名称 · 内容”。
- 编辑、删除、拖动和新增条目后重启，内容保持不变。
- 左右方向键切换标签，重启恢复上次标签。
- 短内容、长内容和大量条目符合尺寸与滚动规则。
- 产品内没有普通条目复制、新增标签或 AI 导入入口。
- 3 秒内唤出，打开后 10 秒内找到目标内容。

## 12. 明确不做

- 不新增、重命名或删除标签。
- 不提供 AI 提示词、AI 导入、内置 AI、账号或网络服务。
- 不执行 Git 指令，不操作终端。
- 不根据当前 App 或上下文推荐内容。
- 不提供普通条目复制功能。
- 不提供搜索、收藏、分组或使用频率统计。
- 不提供云同步、多人协作或其他操作系统版本。
- 不实现自动更新或应用商店发布流程。

## 13. 设计结论

第一版聚焦两个可以立即使用的固定标签，不把新增标签和 AI 导入的未来设想带入当前代码。使用官方 Electron Forge 模板减少工程搭建成本，产品功能保持为容易阅读的 React + TypeScript；预置内容来自 Apple 和 Git 官方资料，现有开源产品只作为调研参考，不直接复制不适合或许可证不明确的代码。
