# macOS 屏幕边缘速查工具 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 构建一款只服务当前用户的原生 macOS 菜单栏工具，可从屏幕右侧或全局快捷键唤出，用最简界面查看、编辑和导入快捷键及命令速查内容。

**Architecture:** 使用 Swift Package Manager 管理一个纯 Swift 核心库和一个 SwiftUI/AppKit 可执行目标；核心库负责模型、排序、持久化、导入校验和可测试的布局/触发规则，应用目标负责 `NSPanel`、系统事件、菜单栏、登录项和界面。构建脚本把 release 可执行文件封装为本地签名的 `.app`，因此当前开发阶段只依赖已安装的 Apple Command Line Tools，不要求先安装完整 Xcode。

**Tech Stack:** Swift 6.1、SwiftUI、AppKit、Combine、Carbon Hot Key API、ServiceManagement、Swift Testing/XCTest、Swift Package Manager、macOS `codesign`

**Spec:** `docs/superpowers/specs/2026-09-02-desktop-cheatsheet-design.md`

## Global Constraints

- 只支持当前 Apple 芯片 Mac，目标系统为 macOS 26.5.1；第一版不做其他操作系统。
- 不引入第三方依赖；只使用 Swift 标准库与 Apple 系统框架。
- 应用以菜单栏代理形式运行，`LSUIElement` 为 `true`，不显示 Dock 图标。
- 查看态每条内容只显示“用途名称 · 快捷键或指令”，整行左对齐，不显示产品标题、说明或条目复制入口。
- 编辑态只提供拖动、修改和删除；数据修改后自动保存。
- 唯一的复制功能是“复制 AI 提示词”；普通条目永远不提供复制按钮。
- AI 只通过外部工具使用；应用不内置模型、账号、网络请求或 API 密钥。
- 本地数据保存在应用的 `Application Support` 目录，主文件原子替换并保留上一份有效备份。
- 全局快捷键默认 `Control + Option + Space`；鼠标在右侧中间 160pt 触发区停留 0.5 秒后展开。
- 面板宽度按最长内容自适应且不超过当前屏幕可用宽度 40%；高度不超过 85%，先压缩间距，最低字号 12pt，最后才内部滚动。
- 所有输入边界必须明确校验并返回包含具体值或字段位置的错误；禁止静默失败。
- 实现遵循 YAGNI、KISS、精确命名和单一职责；不为未确认的未来功能预留抽象层。

## File Structure

```text
.
├── Package.swift
├── Resources/
│   └── Info.plist                         # 应用包元数据、LSUIElement 与最低系统版本
├── Scripts/
│   ├── build-app.sh                       # 编译、组装并本机签名 .app
│   └── install-app.sh                     # 安装到用户 Applications 并启动
├── Sources/
│   ├── EdgeCheatsheetCore/
│   │   ├── Models/
│   │   │   ├── CheatsheetDocument.swift  # 文档、标签和条目模型
│   │   │   └── HotKeyCombination.swift   # 可持久化快捷键值对象
│   │   ├── Store/
│   │   │   ├── LabelStore.swift          # 标签状态、CRUD、排序、选择与保存失败状态
│   │   │   └── AppSettings.swift         # 快捷键、停留时长和登录启动偏好
│   │   ├── Storage/
│   │   │   └── DocumentRepository.swift  # JSON 读取、原子写入、备份与损坏恢复
│   │   ├── Import/
│   │   │   ├── AIPromptBuilder.swift     # 生成唯一可复制的完整 AI 提示词
│   │   │   └── ImportParser.swift        # 宽容解析、逐条校验、重复标记和导入预览
│   │   ├── Panel/
│   │   │   ├── PanelLayoutCalculator.swift # 宽高、紧凑密度和滚动规则
│   │   │   └── EdgeHoverDetector.swift   # 右侧触发区与 0.5 秒停留状态机
│   │   └── Support/
│   │       └── UserFacingError.swift      # 可展示且带上下文的错误
│   └── EdgeCheatsheetApp/
│       ├── EdgeCheatsheetApp.swift        # SwiftUI 入口与 AppDelegate 连接
│       ├── AppLifecycle.swift             # 依赖组装、菜单栏和应用生命周期
│       ├── Panel/
│       │   ├── CheatsheetPanel.swift      # 可成为 key window 的 NSPanel
│       │   └── PanelController.swift      # 显示、收起、定位与面板外点击
│       ├── Trigger/
│       │   ├── GlobalHotKeyRegistrar.swift # Carbon 快捷键注册和冲突错误
│       │   └── TriggerController.swift    # 鼠标、快捷键和 Esc 触发编排
│       ├── System/
│       │   └── LaunchAtLoginController.swift # SMAppService 登录项开关
│       └── Views/
│           ├── PanelRootView.swift        # 标签栏、模式切换、方向键导航
│           ├── LabelListView.swift        # 极简查看态
│           ├── LabelEditView.swift        # 拖动、修改、删除
│           ├── ItemEditorView.swift       # 条目新增/修改表单
│           ├── NewLabelView.swift         # 手动创建、提示词和 JSON 输入
│           ├── ImportPreviewView.swift    # 有效、重复、无效条目预览
│           └── SettingsView.swift         # 快捷键、停留时长与登录启动
├── Tests/
│   └── EdgeCheatsheetCoreTests/
│       ├── LabelStoreTests.swift
│       ├── DocumentRepositoryTests.swift
│       ├── ImportParserTests.swift
│       ├── PanelLayoutCalculatorTests.swift
│       └── EdgeHoverDetectorTests.swift
└── README.md                              # 构建、安装、运行和手工验收说明
```

---

### Task 1: 可测试的 Swift 工程与本地 `.app` 构建

**Files:**
- Create: `Package.swift`
- Create: `Resources/Info.plist`
- Create: `Scripts/build-app.sh`
- Create: `Scripts/install-app.sh`
- Create: `Sources/EdgeCheatsheetCore/Support/UserFacingError.swift`
- Create: `Sources/EdgeCheatsheetApp/EdgeCheatsheetApp.swift`

**Interfaces:**
- Consumes: 当前机器上的 Swift 6.1、macOS SDK、`codesign`。
- Produces: `swift test` 可运行的 `EdgeCheatsheetCore`；`Scripts/build-app.sh` 生成 `build/EdgeCheatsheet.app`；应用包标识为 `com.bella.edge-cheatsheet`。

- [ ] **Step 1: 创建最小包定义并写一个失败的冒烟测试**

`Package.swift` 只声明一个 library、一个 executable 和一个 test target：

```swift
// swift-tools-version: 6.0
import PackageDescription

let package = Package(
    name: "EdgeCheatsheet",
    platforms: [.macOS(.v15)],
    products: [
        .library(name: "EdgeCheatsheetCore", targets: ["EdgeCheatsheetCore"]),
        .executable(name: "EdgeCheatsheet", targets: ["EdgeCheatsheetApp"])
    ],
    targets: [
        .target(name: "EdgeCheatsheetCore"),
        .executableTarget(
            name: "EdgeCheatsheetApp",
            dependencies: ["EdgeCheatsheetCore"],
            linkerSettings: [
                .linkedFramework("AppKit"),
                .linkedFramework("Carbon"),
                .linkedFramework("ServiceManagement")
            ]
        ),
        .testTarget(name: "EdgeCheatsheetCoreTests", dependencies: ["EdgeCheatsheetCore"])
    ]
)
```

首个测试引用尚未创建的 `UserFacingError`，证明测试目标正确连线。

- [ ] **Step 2: 运行测试并确认按预期失败**

Run: `swift test`

Expected: FAIL，错误包含 `cannot find 'UserFacingError' in scope`，而不是包结构或 SDK 错误。

- [ ] **Step 3: 实现最小错误类型和可启动应用入口**

`UserFacingError` 保存标题、具体说明和可选系统错误：

```swift
public struct UserFacingError: Error, Equatable, Identifiable {
    public let id = UUID()
    public let title: String
    public let message: String

    public init(title: String, message: String) {
        self.title = title
        self.message = message
    }
}
```

应用入口先创建只含空设置场景的 `App`，后续 Task 9 接入生命周期对象。

- [ ] **Step 4: 创建固定内容的 Info.plist 与构建脚本**

`Info.plist` 必须包含 `CFBundleIdentifier=com.bella.edge-cheatsheet`、`CFBundleExecutable=EdgeCheatsheet`、`LSUIElement=true`、`NSHighResolutionCapable=true` 和 `LSMinimumSystemVersion=15.0`。

`build-app.sh` 必须：

1. 使用 `set -euo pipefail`。
2. 运行 `swift build -c release`。
3. 重建明确路径 `build/EdgeCheatsheet.app/Contents/{MacOS,Resources}`。
4. 复制 release 二进制和 `Info.plist`。
5. 执行 `codesign --force --deep --sign - build/EdgeCheatsheet.app`。
6. 用 `codesign --verify --deep --strict` 验证后才返回成功。

`install-app.sh` 只覆盖 `$USER/Applications/EdgeCheatsheet.app`，目标必须先解析为该精确路径；复制完成后用 `open` 启动。

- [ ] **Step 5: 验证包测试、release 构建和签名**

Run: `swift test && ./Scripts/build-app.sh && codesign --verify --deep --strict build/EdgeCheatsheet.app`

Expected: 全部退出码为 0，最后一个命令无错误输出。

- [ ] **Step 6: 提交工程骨架**

Run:

```bash
git add Package.swift Resources Scripts Sources Tests
git commit -m "build: bootstrap native macOS app"
```

---

### Task 2: 标签数据模型、选择状态与 CRUD

**Files:**
- Create: `Sources/EdgeCheatsheetCore/Models/CheatsheetDocument.swift`
- Create: `Sources/EdgeCheatsheetCore/Models/HotKeyCombination.swift`
- Create: `Sources/EdgeCheatsheetCore/Store/LabelStore.swift`
- Create: `Tests/EdgeCheatsheetCoreTests/LabelStoreTests.swift`

**Interfaces:**
- Consumes: `UserFacingError`。
- Produces: `CheatsheetDocument`、`CheatsheetLabel`、`CheatsheetItem`、`HotKeyCombination`、`@MainActor LabelStore`；后续存储、导入和 UI 只通过这些类型交换状态。

- [ ] **Step 1: 写模型编码与排序规范化的失败测试**

覆盖以下行为：

- 新文档 `version == 1`。
- 标签与条目排序后，`order` 都重编号为从 0 开始的连续整数。
- 删除当前标签后选择相邻标签；删除最后一个标签后 `lastSelectedLabelId == nil`。
- 左右导航首尾循环。
- 名称或内容去除首尾空白后为空时，操作抛出包含原始字段名的错误。

示例测试：

```swift
@MainActor
func testMovingItemRewritesContinuousOrder() throws {
    let store = LabelStore(document: .fixtureWithThreeItems())
    let labelID = try XCTUnwrap(store.document.labels.first?.id)

    try store.moveItems(in: labelID, fromOffsets: IndexSet(integer: 2), toOffset: 0)

    XCTAssertEqual(store.document.labels[0].items.map(\.order), [0, 1, 2])
    XCTAssertEqual(store.document.labels[0].items.map(\.name), ["third", "first", "second"])
}
```

- [ ] **Step 2: 运行指定测试并确认失败**

Run: `swift test --filter LabelStoreTests`

Expected: FAIL，缺少 `CheatsheetDocument` 或 `LabelStore`。

- [ ] **Step 3: 实现 Codable 模型和精确的 Store API**

模型保持值类型：

```swift
public struct CheatsheetDocument: Codable, Equatable, Sendable {
    public static let currentVersion = 1
    public var version: Int
    public var lastSelectedLabelId: UUID?
    public var labels: [CheatsheetLabel]
}

public struct CheatsheetLabel: Codable, Equatable, Identifiable, Sendable {
    public let id: UUID
    public var name: String
    public var order: Int
    public var items: [CheatsheetItem]
}

public struct CheatsheetItem: Codable, Equatable, Identifiable, Sendable {
    public let id: UUID
    public var name: String
    public var content: String
    public var order: Int
}
```

`LabelStore` 公开以下精确入口：

```swift
@MainActor
public final class LabelStore: ObservableObject {
    @Published public private(set) var document: CheatsheetDocument
    @Published public var presentedError: UserFacingError?

    public var selectedLabel: CheatsheetLabel? { get }
    public func selectLabel(id: UUID)
    public func selectAdjacentLabel(offset: Int)
    @discardableResult public func addLabel(name: String) throws -> UUID
    public func renameLabel(id: UUID, name: String) throws
    public func deleteLabel(id: UUID)
    @discardableResult public func addItem(to labelID: UUID, name: String, content: String) throws -> UUID
    public func updateItem(labelID: UUID, itemID: UUID, name: String, content: String) throws
    public func deleteItem(labelID: UUID, itemID: UUID)
    public func moveItems(in labelID: UUID, fromOffsets: IndexSet, toOffset: Int) throws
}
```

当前 Task 只更新内存；保存回调在 Task 3 加入。所有查找失败都包含对应 UUID，字符串边界只去除首尾空白，不改写中间内容。

- [ ] **Step 4: 运行 Store 测试并补齐边界案例**

Run: `swift test --filter LabelStoreTests`

Expected: PASS，且至少覆盖添加、修改、删除、移动、首尾导航、空输入与未知 UUID。

- [ ] **Step 5: 提交领域模型**

Run:

```bash
git add Sources/EdgeCheatsheetCore Tests/EdgeCheatsheetCoreTests/LabelStoreTests.swift
git commit -m "feat: add editable cheatsheet model"
```

---

### Task 3: 原子持久化、自动保存与损坏恢复

**Files:**
- Create: `Sources/EdgeCheatsheetCore/Storage/DocumentRepository.swift`
- Modify: `Sources/EdgeCheatsheetCore/Store/LabelStore.swift`
- Create: `Tests/EdgeCheatsheetCoreTests/DocumentRepositoryTests.swift`
- Modify: `Tests/EdgeCheatsheetCoreTests/LabelStoreTests.swift`

**Interfaces:**
- Consumes: `CheatsheetDocument`、`LabelStore`、`UserFacingError`。
- Produces: `DocumentRepository.load()`、`save(_:)`、`restoreBackup()`；每次 Store 变更后的自动保存和可观察保存错误。

- [ ] **Step 1: 写真实临时目录上的失败测试**

每个测试使用 `FileManager.default.temporaryDirectory` 下的新 UUID 目录，不 mock 文件系统。覆盖：

- 无文件时返回 `.empty`。
- 首次保存后可完整读取。
- 第二次保存前，上一份有效主文件成为备份。
- 主文件损坏、备份有效时返回 `.backupAvailable`，不自动覆盖主文件。
- 主文件与备份都损坏时返回 `.emptyWithDamage` 并保留两个文件。
- `version != 1` 时错误包含实际版本值。
- 保存失败时 `LabelStore` 保留内存修改并设置包含目标路径和系统原因的 `presentedError`。

- [ ] **Step 2: 运行存储测试并确认失败**

Run: `swift test --filter DocumentRepositoryTests`

Expected: FAIL，缺少 `DocumentRepository`。

- [ ] **Step 3: 实现加载结果和仓库**

```swift
public enum DocumentLoadResult: Equatable {
    case empty
    case loaded(CheatsheetDocument)
    case backupAvailable(document: CheatsheetDocument, damagedMainFile: URL)
    case emptyWithDamage(mainFile: URL, backupFile: URL)
}

public struct DocumentRepository {
    public let directoryURL: URL
    public var mainFileURL: URL { directoryURL.appendingPathComponent("labels.json") }
    public var backupFileURL: URL { directoryURL.appendingPathComponent("labels.backup.json") }

    public func load() -> DocumentLoadResult
    public func save(_ document: CheatsheetDocument) throws
    public func restoreBackup() throws -> CheatsheetDocument
}
```

保存顺序固定为：编码并验证新数据 → 写同目录临时文件 → 验证现有主文件有效后原子更新备份 → 原子替换主文件。任一步失败都抛出带文件路径的错误，不删除损坏文件。

- [ ] **Step 4: 给 LabelStore 注入一个窄保存闭包**

避免为单一实现建立仓库协议，只注入：

```swift
public typealias SaveDocument = @MainActor (CheatsheetDocument) throws -> Void

public init(
    document: CheatsheetDocument,
    saveDocument: @escaping SaveDocument = { _ in }
)
```

每个成功内存变更后同步调用 `saveDocument(document)`；保存失败只更新 `presentedError`，不回滚用户刚做的修改。

- [ ] **Step 5: 运行核心测试并检查 JSON 内容**

Run: `swift test`

Expected: PASS；测试生成的 JSON 可被 `JSONDecoder` 重新读取，`order` 连续，且不存在 `note` 字段。

- [ ] **Step 6: 提交持久化功能**

Run:

```bash
git add Sources/EdgeCheatsheetCore Tests/EdgeCheatsheetCoreTests
git commit -m "feat: persist cheatsheets with backup recovery"
```

---

### Task 4: AI 提示词、宽容导入解析和重复检测

**Files:**
- Create: `Sources/EdgeCheatsheetCore/Import/AIPromptBuilder.swift`
- Create: `Sources/EdgeCheatsheetCore/Import/ImportParser.swift`
- Modify: `Sources/EdgeCheatsheetCore/Store/LabelStore.swift`
- Create: `Tests/EdgeCheatsheetCoreTests/ImportParserTests.swift`

**Interfaces:**
- Consumes: 现有标签 `content` 集合、`LabelStore.addItem`。
- Produces: `AIPromptBuilder.prompt(topic:)`、`ImportParser.parse(json:existingContents:)`、`LabelStore.importSelectedRows(_:into:)`。

- [ ] **Step 1: 写提示词与导入行为的失败测试**

覆盖：

- 空主题被拒绝，错误明确提到“主题”。
- 主题 `Git` 被替换到提示词正文和 JSON 的 `label`。
- 提示词要求 10～15 条、只返回有效 JSON、字段仅为 `label/name/content`。
- 有效 JSON 产生默认选中的有效行。
- 非 JSON 错误包含解析位置。
- 缺少 `label` 使整个预览不可确认。
- `items` 中单条缺少 `name` 或 `content` 时，其余有效条目仍进入预览。
- `" git push "` 与已有 `"git push"` 重复，默认不选中。
- `"git  push"` 与 `"git push"` 不重复，因为只裁剪首尾空白。
- 导入确认前 Store 不发生变化；确认后只追加已选有效行。

- [ ] **Step 2: 运行导入测试并确认失败**

Run: `swift test --filter ImportParserTests`

Expected: FAIL，缺少 `AIPromptBuilder` 和 `ImportParser`。

- [ ] **Step 3: 实现固定提示词生成器**

```swift
public enum AIPromptBuilder {
    public static let preview = "你正在为一个桌面速查工具生成标签页内容……"
    public static func prompt(topic rawTopic: String) throws -> String
}
```

完整字符串必须逐字实现设计稿第 6.2 节；主题只裁剪首尾空白，不对用户内容做转义性改写。空主题立即抛错。

- [ ] **Step 4: 用 JSONSerialization 实现逐条校验**

不能用一次性 `Codable` 解码整个响应，否则一条坏数据会丢掉其他有效行。使用以下结果类型：

```swift
public struct ImportPreview: Equatable {
    public let label: String?
    public var rows: [ImportPreviewRow]
    public let problems: [ImportProblem]
    public var canImport: Bool { get }
}

public struct ImportPreviewRow: Equatable, Identifiable {
    public enum State: Equatable { case valid, duplicate, invalid }
    public let id: UUID
    public let name: String?
    public let content: String?
    public let state: State
    public var isSelected: Bool
    public let problems: [ImportProblem]
}

public struct ImportProblem: Equatable {
    public let location: String
    public let message: String
}

public enum ImportParser {
    public static func parse(json: String, existingContents: Set<String>) throws -> ImportPreview
}
```

`existingContents` 与候选 `content` 都仅调用 `trimmingCharacters(in: .whitespacesAndNewlines)` 后比较。无效或重复行不可被导入。

- [ ] **Step 5: 实现确认导入并运行完整测试**

`LabelStore.importSelectedRows(_:into:)` 再次验证每行状态与字段，按预览顺序追加并只保存一次；不存在的目标标签立即报出标签 UUID。

Run: `swift test`

Expected: PASS，混合有效/无效/重复用例全部通过。

- [ ] **Step 6: 提交 AI 导入核心**

Run:

```bash
git add Sources/EdgeCheatsheetCore Tests/EdgeCheatsheetCoreTests
git commit -m "feat: validate external AI imports"
```

---

### Task 5: 面板状态、极简查看态和编辑态

**Files:**
- Create: `Sources/EdgeCheatsheetApp/Views/PanelRootView.swift`
- Create: `Sources/EdgeCheatsheetApp/Views/LabelListView.swift`
- Create: `Sources/EdgeCheatsheetApp/Views/LabelEditView.swift`
- Create: `Sources/EdgeCheatsheetApp/Views/ItemEditorView.swift`
- Modify: `Sources/EdgeCheatsheetCore/Store/LabelStore.swift`

**Interfaces:**
- Consumes: `LabelStore.selectedLabel` 与 CRUD API。
- Produces: 查看、编辑、新建三种互斥面板模式；方向键标签导航；无普通条目复制入口的主界面。

- [ ] **Step 1: 给 PanelRootView 定义最小模式状态**

```swift
enum PanelMode: Equatable {
    case viewing
    case editing
    case newLabel
}
```

`PanelRootView` 只持有 `@State private var mode`、当前条目编辑表单和删除确认目标；数据本身始终来自 `@ObservedObject LabelStore`。

- [ ] **Step 2: 实现顶部标签栏与键盘导航**

顶部只显示标签名、`+` 和查看态下的铅笔/编辑态下的“完成”。使用 `.onKeyPress(.leftArrow)` 与 `.onKeyPress(.rightArrow)` 调用 `selectAdjacentLabel(offset:)`，并把新选择自动保存到文档。

禁止在根视图加入“便利贴”标题或副标题。

- [ ] **Step 3: 实现查看态的单行左对齐条目**

`LabelListView` 每行采用自然流式排列：

```swift
HStack(spacing: 6) {
    Text(item.name)
    Text("·").foregroundStyle(.secondary)
    Text(item.content).fontDesign(.monospaced)
}
.frame(maxWidth: .infinity, alignment: .leading)
```

不得添加 trailing 对齐列、复制图标、悬浮菜单、`note` 或隐藏的复制快捷键。

- [ ] **Step 4: 实现编辑态三个操作**

- 行首显示拖动把手并使用 `ForEach(...).onMove`。
- 修改按钮打开 `ItemEditorView`，只含“用途名称”和“快捷键或指令”。
- 删除按钮弹出一次轻量确认，确认后调用 `deleteItem`。
- 空标签显示一个“添加第一条”入口；已有标签底部显示“添加条目”。
- 所有操作立即写入 Store，不增加保存按钮。

- [ ] **Step 5: 编译并运行核心回归测试**

Run: `swift build && swift test`

Expected: 编译与测试均成功；源码搜索不存在普通条目剪贴板写入。

Run: `rg "NSPasteboard|copy" Sources/EdgeCheatsheetApp/Views`

Expected: 当前 Task 无匹配；唯一剪贴板写入将在 Task 6 的提示词按钮中加入。

- [ ] **Step 6: 提交主面板视图**

Run:

```bash
git add Sources/EdgeCheatsheetApp Sources/EdgeCheatsheetCore/Store/LabelStore.swift
git commit -m "feat: add minimal view and edit modes"
```

---

### Task 6: 新建标签、提示词复制、导入预览和设置界面

**Files:**
- Create: `Sources/EdgeCheatsheetCore/Store/AppSettings.swift`
- Create: `Sources/EdgeCheatsheetApp/Views/NewLabelView.swift`
- Create: `Sources/EdgeCheatsheetApp/Views/ImportPreviewView.swift`
- Create: `Sources/EdgeCheatsheetApp/Views/SettingsView.swift`
- Modify: `Sources/EdgeCheatsheetApp/Views/PanelRootView.swift`

**Interfaces:**
- Consumes: `AIPromptBuilder`、`ImportParser`、`LabelStore.importSelectedRows`、`HotKeyCombination`。
- Produces: 手动创建或外部 AI 导入完整流程；`AppSettings.edgeHoverDelay` 与 `AppSettings.hotKey` 的持久化设置。

- [ ] **Step 1: 写 AppSettings 边界测试并确认失败**

在 `LabelStoreTests.swift` 或独立测试中使用专用 `UserDefaults(suiteName:)`，覆盖：

- 默认停留时间为 `0.5` 秒。
- 默认快捷键为 Control + Option + Space。
- 停留时间限制在 `0.2...2.0` 秒，越界输入报出实际值。
- 修改后新实例可读回。

Run: `swift test --filter AppSettingsTests`

Expected: FAIL，缺少 `AppSettings`。

- [ ] **Step 2: 实现 AppSettings**

```swift
@MainActor
public final class AppSettings: ObservableObject {
    @Published public private(set) var edgeHoverDelay: TimeInterval
    @Published public private(set) var hotKey: HotKeyCombination

    public func setEdgeHoverDelay(_ seconds: TimeInterval) throws
    public func setHotKey(_ combination: HotKeyCombination) throws
}
```

只用 `UserDefaults` 保存这两个小型偏好；登录项实际状态由 Task 9 的系统控制器读取，不在这里复制一份状态。

- [ ] **Step 3: 实现 NewLabelView 的视觉层级与唯一复制入口**

页面顺序固定为：主题输入框 → 一句说明 → 显著的“复制 AI 提示词”主按钮 → 提示词开头示意 → 次要“手动创建”按钮 → AI 返回 JSON 多行输入框 → “预览导入”。

只有主按钮可以写剪贴板：

```swift
let prompt = try AIPromptBuilder.prompt(topic: topic)
NSPasteboard.general.clearContents()
NSPasteboard.general.setString(prompt, forType: .string)
```

空主题时两个创建入口均不可执行并显示具体提示。手动创建成功后选择新标签并直接进入编辑态。

- [ ] **Step 4: 实现 ImportPreviewView**

- 标签级问题显示在顶部并阻止确认。
- 有效新行默认选中，可取消。
- 重复行显示“已存在”，不可选择。
- 无效行显示精确字段位置，例如 `items[3].content`，不可选择。
- “确认导入”只在至少一条有效行选中且标签有效时启用。
- 确认后追加到目标标签末尾、选择该标签、返回查看态。

- [ ] **Step 5: 实现 SettingsView 的三项设置**

界面只包含：快捷键录制按钮、右侧停留时长、登录时启动开关。快捷键录制视图只接受同时带 Control/Option/Command 中至少一个修饰键的组合；注册冲突由 Task 8 回传并在原位显示。

- [ ] **Step 6: 运行测试、构建并核对剪贴板入口**

Run:

```bash
swift test
swift build
rg -n "NSPasteboard" Sources
```

Expected: 测试和构建成功；`NSPasteboard` 只出现在 `NewLabelView.swift` 的提示词复制动作中。

- [ ] **Step 7: 提交创建、导入和设置 UI**

Run:

```bash
git add Sources Tests
git commit -m "feat: add label creation and AI import flow"
```

---

### Task 7: 自适应布局与跨空间浮动面板

**Files:**
- Create: `Sources/EdgeCheatsheetCore/Panel/PanelLayoutCalculator.swift`
- Create: `Sources/EdgeCheatsheetApp/Panel/CheatsheetPanel.swift`
- Create: `Sources/EdgeCheatsheetApp/Panel/PanelController.swift`
- Create: `Tests/EdgeCheatsheetCoreTests/PanelLayoutCalculatorTests.swift`
- Modify: `Sources/EdgeCheatsheetApp/Views/PanelRootView.swift`

**Interfaces:**
- Consumes: 当前屏幕 `visibleFrame`、SwiftUI 内容理想尺寸、面板模式。
- Produces: `PanelLayout`；`PanelController.show(on:)`、`hide()`、`toggle(on:)`；面板外点击与 Esc 收起。

- [ ] **Step 1: 写纯布局计算的失败测试**

覆盖：

- 理想宽度低于上限时保留紧凑宽度。
- 理想宽度超过屏幕 40% 时截断到 40%。
- 理想高度低于屏幕 85% 时不滚动。
- 普通密度超高但紧凑密度可放下时选择紧凑密度。
- 紧凑密度仍超高时高度为 85% 且 `usesScrollView == true`。
- 最小字号永远为 12pt。
- 面板右边缘贴齐当前屏幕可用区域右边缘并垂直居中。

- [ ] **Step 2: 运行布局测试并确认失败**

Run: `swift test --filter PanelLayoutCalculatorTests`

Expected: FAIL，缺少布局类型。

- [ ] **Step 3: 实现无 AppKit 依赖的布局计算器**

```swift
public enum PanelDensity: Equatable { case regular, compact }

public struct PanelLayout: Equatable {
    public let frame: CGRect
    public let density: PanelDensity
    public let fontSize: CGFloat
    public let usesScrollView: Bool
}

public enum PanelLayoutCalculator {
    public static func calculate(
        screenFrame: CGRect,
        regularContentSize: CGSize,
        compactContentSize: CGSize
    ) -> PanelLayout
}
```

使用固定规则：宽上限 `screen.width * 0.40`，高上限 `screen.height * 0.85`，常规字号 14pt，紧凑字号 12pt。不要继续缩小字体。

- [ ] **Step 4: 实现可交互 NSPanel**

`CheatsheetPanel` 继承 `NSPanel` 并令 `canBecomeKey == true`，样式为 borderless/nonactivatingPanel，背景透明，阴影开启。必须设置：

```swift
panel.level = .floating
panel.collectionBehavior = [.canJoinAllSpaces, .fullScreenAuxiliary]
panel.hidesOnDeactivate = false
panel.isMovable = false
```

SwiftUI 根视图置于 `NSHostingView`。进入表单或设置时允许获取键盘焦点；查看态不强制抢走当前 App 焦点。

- [ ] **Step 5: 实现 PanelController 的显示/收起**

```swift
@MainActor
final class PanelController {
    var isVisible: Bool { get }
    func show(on screen: NSScreen)
    func hide()
    func toggle(on screen: NSScreen)
    func refreshSize()
}
```

显示时先测量当前 SwiftUI 内容，再调用布局计算器；位置使用传入屏幕而不是 `NSScreen.main`。添加本地与全局 mouse-down monitor，点击不在 panel frame 内时收起。Esc 通过 panel 的 key handling 收起。`hide()` 必须移除属于本次显示周期的事件 monitor。

- [ ] **Step 6: 运行测试、构建与静态配置检查**

Run:

```bash
swift test --filter PanelLayoutCalculatorTests
swift build
rg "canJoinAllSpaces|fullScreenAuxiliary|hidesOnDeactivate" Sources/EdgeCheatsheetApp/Panel
```

Expected: 测试/构建成功，三项窗口行为均有明确配置。

- [ ] **Step 7: 提交面板功能**

Run:

```bash
git add Sources Tests
git commit -m "feat: add adaptive cross-space panel"
```

---

### Task 8: 屏幕边缘停留与全局快捷键触发

**Files:**
- Create: `Sources/EdgeCheatsheetCore/Panel/EdgeHoverDetector.swift`
- Create: `Sources/EdgeCheatsheetApp/Trigger/GlobalHotKeyRegistrar.swift`
- Create: `Sources/EdgeCheatsheetApp/Trigger/TriggerController.swift`
- Create: `Tests/EdgeCheatsheetCoreTests/EdgeHoverDetectorTests.swift`

**Interfaces:**
- Consumes: `AppSettings.edgeHoverDelay`、`AppSettings.hotKey`、`PanelController`。
- Produces: 可重复测试的 hover 状态机；系统热键注册；当前鼠标屏幕上的面板切换。

- [ ] **Step 1: 写边缘触发状态机的失败测试**

用明确时间戳测试：

- 触发区为屏幕右边缘、垂直中心总高 160pt。
- 进入 0.49 秒不触发，达到 0.5 秒触发一次。
- 离开触发区会清空计时。
- 已触发后持续停留不重复触发；离开再进入可再次触发。
- 多屏时只返回包含鼠标点的屏幕标识。

- [ ] **Step 2: 运行 detector 测试并确认失败**

Run: `swift test --filter EdgeHoverDetectorTests`

Expected: FAIL，缺少 `EdgeHoverDetector`。

- [ ] **Step 3: 实现 EdgeHoverDetector**

```swift
public struct ScreenFrame: Equatable {
    public let id: String
    public let frame: CGRect
}

public struct EdgeHoverDetector {
    public mutating func update(
        mouseLocation: CGPoint,
        screens: [ScreenFrame],
        now: TimeInterval,
        requiredDuration: TimeInterval
    ) -> String?
}
```

Detector 不持有 timer、不引用 AppKit，只记录进入时间、当前屏幕 ID 和本轮是否已触发。

- [ ] **Step 4: 实现 Carbon 全局快捷键注册**

`GlobalHotKeyRegistrar` 封装 `RegisterEventHotKey`/`UnregisterEventHotKey`，保存唯一 `EventHotKeyRef`，换键时先尝试注册新组合：成功后注销旧组合；失败则保留旧组合并抛出包含显示组合键和 OSStatus 的错误。

```swift
@MainActor
final class GlobalHotKeyRegistrar {
    var onPressed: (() -> Void)?
    func register(_ combination: HotKeyCombination) throws
    func unregister()
}
```

- [ ] **Step 5: 编排鼠标与快捷键触发**

`TriggerController.start()` 安装一个 `NSEvent.addGlobalMonitorForEvents(matching: .mouseMoved)`；每次事件把鼠标位置、所有屏幕 frame 和单调时钟交给 detector。detector 返回 screen ID 时调用 `panelController.show(on:)`。

快捷键回调寻找 `NSEvent.mouseLocation` 所在屏幕并调用 `toggle(on:)`。再次按键收起。若没有屏幕包含鼠标点，明确回退到 `NSScreen.main`，仍不存在则报告错误而不崩溃。

- [ ] **Step 6: 验证状态机、编译和快捷键符号**

Run:

```bash
swift test --filter EdgeHoverDetectorTests
swift build
rg "RegisterEventHotKey|UnregisterEventHotKey|mouseMoved" Sources/EdgeCheatsheetApp/Trigger
```

Expected: 测试与构建成功；注册、注销、鼠标监控各有唯一实现位置。

- [ ] **Step 7: 提交触发器**

Run:

```bash
git add Sources Tests
git commit -m "feat: open panel from screen edge or hot key"
```

---

### Task 9: 菜单栏生命周期、登录项与错误恢复

**Files:**
- Create: `Sources/EdgeCheatsheetApp/AppLifecycle.swift`
- Create: `Sources/EdgeCheatsheetApp/System/LaunchAtLoginController.swift`
- Modify: `Sources/EdgeCheatsheetApp/EdgeCheatsheetApp.swift`
- Modify: `Sources/EdgeCheatsheetApp/Views/SettingsView.swift`
- Modify: `Sources/EdgeCheatsheetApp/Views/PanelRootView.swift`

**Interfaces:**
- Consumes: Repository、Store、Settings、PanelController、TriggerController。
- Produces: 完整常驻菜单栏应用；登录后启动开关；主文件损坏恢复提示；快捷键冲突和保存失败可见提示。

- [ ] **Step 1: 实现应用支持目录和依赖组装**

`AppLifecycle` 启动时按固定顺序：

1. 解析 `FileManager.default.urls(for: .applicationSupportDirectory, in: .userDomainMask)`。
2. 追加 `EdgeCheatsheet` 子目录。
3. 用 `DocumentRepository.load()` 获得启动状态。
4. 创建 `LabelStore` 与真实保存闭包。
5. 创建面板、触发器、设置与登录项控制器。
6. 建立菜单栏按钮后启动 trigger。

首次空数据写入两个默认标签：`macOS` 和 `Git`。默认内容只放经过人工核对的常用项，每个标签不超过 12 条；这批内容在同一提交内写成固定 seed 函数并由编码测试确保没有重复 `content`。

- [ ] **Step 2: 实现菜单栏入口**

使用 `NSStatusBar.system.statusItem(withLength: .square)` 和 SF Symbol 图标。菜单只包含：

- “显示/隐藏”
- “设置…”
- 分隔线
- “退出”

面板内继续不显示产品名。退出时停止鼠标 monitor、注销热键并移除面板外点击 monitor。

- [ ] **Step 3: 实现登录项开关**

```swift
@MainActor
final class LaunchAtLoginController: ObservableObject {
    @Published private(set) var isEnabled: Bool
    @Published var presentedError: UserFacingError?

    func refresh()
    func setEnabled(_ enabled: Bool)
}
```

使用 `SMAppService.mainApp.register()` / `.unregister()`，成功后从 `SMAppService.mainApp.status` 重新读取真实状态；失败时恢复 UI 开关并显示系统错误。首次启动尝试开启，若系统拒绝则提示用户前往“系统设置 → 通用 → 登录项”。

- [ ] **Step 4: 接通恢复和错误界面**

- `.backupAvailable`：启动后展示“主数据损坏”，提供“从备份恢复”和“暂不恢复”。只有用户确认才调用 `restoreBackup()`。
- `.emptyWithDamage`：保留文件，展示两个精确路径，并以空状态启动。
- Store 保存失败：面板顶部出现非遮挡错误条，可查看完整路径/系统原因并关闭。
- 热键冲突：Settings 原位显示失败组合键，旧热键继续有效。
- 所有错误都通过 `UserFacingError` 展示，不 `print` 后忽略。

- [ ] **Step 5: 接通 SwiftUI 入口并构建应用包**

`EdgeCheatsheetApp.swift` 使用 `@NSApplicationDelegateAdaptor(AppLifecycle.self)`，Settings 场景只作为系统入口备用；实际面板由 `PanelController` 管理。

Run:

```bash
swift test
./Scripts/build-app.sh
plutil -lint Resources/Info.plist
codesign --verify --deep --strict build/EdgeCheatsheet.app
```

Expected: 全部成功。

- [ ] **Step 6: 启动并检查进程形态**

Run:

```bash
open build/EdgeCheatsheet.app
sleep 2
pgrep -fl EdgeCheatsheet
```

Expected: 只看到一个运行进程；Dock 中无图标，菜单栏出现入口。若应用崩溃，先读取 `log show --last 2m --predicate 'process == "EdgeCheatsheet"' --style compact` 定位，不绕过错误。

- [ ] **Step 7: 提交生命周期集成**

Run:

```bash
git add Sources Resources Scripts Tests
git commit -m "feat: integrate menu bar app lifecycle"
```

---

### Task 10: 端到端验收、安装说明与发布前清理

**Files:**
- Create: `README.md`
- Modify: `.gitignore`
- Modify: any file proven incorrect by verification

**Interfaces:**
- Consumes: 完整应用和设计稿第 11 节。
- Produces: 可重复构建、安装和手工验收的第一版；干净 Git 工作树。

- [ ] **Step 1: 写 README 的本机操作说明**

必须包含以下 macOS 命令：

```bash
swift test
./Scripts/build-app.sh
./Scripts/install-app.sh
open "$USER/Applications/EdgeCheatsheet.app"
```

说明数据路径、备份路径、菜单栏退出方式、登录项权限和卸载方式。卸载只描述将精确应用路径移到废纸篓，不提供宽泛递归删除命令。

- [ ] **Step 2: 更新忽略文件**

`.gitignore` 增加 `.build/`、`build/` 和 `.DS_Store`，保留已有 `.superpowers/`。确认没有忽略 `docs/superpowers`、`Resources` 或 `Tests`。

- [ ] **Step 3: 运行全部自动验证**

Run:

```bash
swift test
swift build -c release
./Scripts/build-app.sh
plutil -lint Resources/Info.plist
codesign --verify --deep --strict build/EdgeCheatsheet.app
rg -n "note|复制|Copy|NSPasteboard" Sources Resources
```

Expected:

- 所有测试通过，release 构建与签名通过。
- `note` 无业务字段匹配。
- 用户可见“复制”只存在于 AI 提示词按钮及相应说明。
- `NSPasteboard` 只存在于 `NewLabelView.swift`。

- [ ] **Step 4: 逐项执行手工功能验收**

在当前 Mac 上记录每项通过/失败：

1. 桌面、普通 App、第二个桌面空间、全屏 App 中都能在 3 秒内唤出。
2. 右侧中间停留 0.5 秒打开；移出不立即关闭。
3. `Control + Option + Space` 打开，再按一次关闭。
4. 点击外部和 Esc 均关闭。
5. 面板显示在鼠标所在屏幕。
6. 查看态只见“名称 · 内容”；编辑态只见移动、修改、删除。
7. 左右方向键循环切换；重启恢复上次标签。
8. 新增、修改、删除、拖动顺序在重启后保留。
9. 短/长内容与少/大量条目符合 40%/85% 尺寸规则，字号不低于 12pt。
10. 手动创建进入编辑态；AI 提示词按钮醒目且复制内容正确。
11. 有效、重复、部分无效、完全无效 JSON 的预览和确认行为正确。
12. 产品中不存在普通条目复制入口。
13. 登录时启动能开启和关闭；冲突热键显示错误且旧热键仍有效。
14. 目标指令能在打开面板后 10 秒内找到。

- [ ] **Step 5: 做一次损坏恢复演练**

先从菜单栏退出应用，复制当前 `labels.json` 到安全临时目录，再把主文件改为无效 JSON 后启动；验证出现备份恢复选择且损坏文件仍在。测试后退出应用并把安全副本恢复到原路径，再次启动确认数据无损。

- [ ] **Step 6: 检查工作树和提交最终文档**

Run:

```bash
git status --short
git diff --check
git add README.md .gitignore
git commit -m "docs: add build and acceptance guide"
git status --short --branch
```

Expected: `git diff --check` 无输出；最终工作树干净。

- [ ] **Step 7: 推送已验证的实现**

Run: `git push origin main`

Expected: 推送成功，远端 `main` 指向本地最终提交。只有自动测试和手工验收都完成后才能执行此步。
