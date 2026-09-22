# 旧主题 / QSS / 调色板与 Fluent 冲突清单

> 对应票据：gh issue #3（Part of #1）。分支：`research/theme-conflicts`。
> 约定：只读研究，不改 `src/`；Fluent 指 #1 目标——Windows 默认启用的 `FluentUI3Style`
> （`QProxyStyle`，Qt6.10 Win11 style 移植，基类 windowsvista、无则 fusion，MIT）。
> 本仓库 Qt 为纯 QWidget（约 72 个 `.ui`，无 QML），主题入口见下文。

## 0. 机制总览（先看这段，再看逐文件表）

- `src/gui/Application.cpp:165-202` `Application::applyTheme()` 三路分发：
  `auto` → 跟随 `osUtils->isDarkMode()`（Windows 高对比强制 `classic`，170-174）；
  `light`/`dark` → `new LightStyle/DarkStyle` + `setPalette(standardPalette())` + `setStyle(s)`；
  `classic` → 仅 `setStyleSheet(classicstyle.qss)`，**不换 style 对象**。
- `src/gui/styles/base/BaseStyle.cpp:4216-4236` `BaseStyle::polish(QApplication*)` 把
  `basestyle.qss + getAppStyleSheet()`（light/dark 各自的 qss）拼接后 `app->setStyleSheet()`。
  含义：**只要旧栈生效，全量 app 级 QSS 必然覆盖 Fluent（QProxyStyle）的对应绘制**。
  依据（Qt 官方一手文档）：《Qt Style Sheets》——已设置样式表的控件改由样式表机制绘制，
  底层 `QStyle` 的对应图元绘制被取代（https://doc.qt.io/qt-6/stylesheet.html）；
  `QProxyStyle` 只是 `QStyle` 的转发包装（https://doc.qt.io/qt-6/qproxystyle.html），同理被取代；
  `QApplication::setStyleSheet` 为 app 级、对所有 widget 生效（https://doc.qt.io/qt-6/qapplication.html#setStyleSheet）。
- 切换不对称（实测代码行为，非推测）：`classic` 分支只调 `setStyleSheet` 不调 `setStyle`，
  而 light/dark 分支只调 `setStyle` 不清 app stylesheet。
  所以 `classic → dark` 会残留 `classicstyle.qss`，`light → classic` 会残留 `LightStyle`
  自绘形状＋新 QSS 叠加——这正是 `src/gui/MainWindow.cpp:1986-1987` 规定凡进出 `classic`
  必须重启 App 的原因。**Fluent 路径（auto/light/dark）必须做到免重启热切换**，
  即：一切换即同时换 style 对象＋清/换 app stylesheet，否则重蹈 classic 覆辙。

## 1. 核心主题栈（逐文件）

### 1.1 `src/gui/Application.cpp`（`applyTheme` 165-202，`interfaceThemeChanged` 131-135）
- 冲突：`setStyle(new LightStyle/DarkStyle)`（179-185）直接顶掉 Fluent style 对象；
  `setPalette(standardPalette())` 把旧调色板写到 app 级；classic 分支 `setStyleSheet`（195-199）
  与 Fluent app QSS 互斥。
- Fluent 建议：auto/light/dark 三路改走 `new FluentUI3Style`（深浅按现有 `auto` 解析逻辑），
  classic 分支原样保留；`interfaceThemeChanged` 回调里 `!= "classic"` 即重应用的逻辑可直接复用
  （Fluent 同样需要在系统深浅变化时重应用）。

### 1.2 `src/gui/styles/base/BaseStyle.h` / `BaseStyle.cpp`（约 4860 行，`QCommonStyle` 全自绘）
- 冲突：整个 Phantom 派生自绘引擎（按钮/滚动条/tab/滑杆/复选框全部手绘，见 `drawPrimitive` /
  `drawControl` / `drawComplexControl` / `pixelMetric` / `sizeFromContents` 大量分支，
  兜底才调 `QCommonStyle::`）。Fluent 生效时**整个类必须停用**——两者是互斥的完整 style，
  不存在“部分复用”。
- 另见 `standardPalette()`（约 3070 行）直接返回 `QCommonStyle::standardPalette()`，
  旧调色全靠 Light/Dark 子类覆盖。
- Fluent 建议：Fluent 模式下永不实例化 `BaseStyle` 及其子类；删除点即 `applyTheme` 一处。

### 1.3 `src/gui/styles/base/phantomcolor.h` / `phantomcolor.cpp`
- 冲突：仅被 `BaseStyle.cpp` 使用的 HSL 颜色数学（`adjustLightness`、渐变采样等）。
- Fluent 建议：随 `BaseStyle` 一起停用；无独立迁移动作。

### 1.4 `src/gui/styles/light/LightStyle.cpp` / `LightStyle.h`
- 冲突：`standardPalette()`（40-101，Window `#F7F7F7`、Highlight `#507F1F` 等全套硬编码）
  会覆盖 Fluent palette；`polish(QWidget*)`（113-132）对 `QMainWindow/QDialog/QMenuBar/QToolBar`
  逐窗改写 `QPalette::Window` 色——会盖掉 Fluent 的窗口材质/底色（Mica/亚克力直接失效）。
- Fluent 建议：二者皆禁用。调色改由 Fluent palette 提供；如需微调只调 Fluent token，不逐窗打补丁。

### 1.5 `src/gui/styles/dark/DarkStyle.cpp` / `DarkStyle.h`
- 冲突：同 1.4（Window `#3B3B3D`、Highlight `#2D532D` 等 40-101；`polish` 113-132 逐窗改色，
  另有 macOS 深浅错位特判 118-128）。
- Fluent 建议：同 1.4，皆禁用。

### 1.6 `src/gui/styles/styles.qrc`＋四份 qss
`styles.qrc` 把四份 qss 打进 `:/styles/`（`Q_INIT_RESOURCE(styles)` 在 `BaseStyle::polish` 4222 触发）。

**`base/basestyle.qss`——部分禁用（逐条）：**

| 规则 | Fluent 下 verdict |
|---|---|
| `QPushButton:default` / `:checked`（`background: palette(highlight)`） | 禁用。QSS `background` 会抹掉 Fluent 按钮的圆角/强调填充绘制；强调语义交还 Fluent default-button |
| `QPushButton:!default:hover`（`palette(mid)`） | 禁用，同上 |
| `QSpinBox { min-width: 90px }` | 保留（纯布局，与绘制无关） |
| `QCheckBox, QRadioButton { spacing: 10px }` | 保留（纯布局） |
| `ReportsDialog/EntryAttachmentsWidget QTableView::item { padding: 4px }` | 保留（密度语义） |
| `DatabaseWidget, #groupView, #tagView { background: palette(window); border: none }` | 拆分：`border: none` 保留（布局）；`background-color` 删除（底色归 Fluent，另见 light/dark qss 的 `:!active/:disabled` 变体亦删） |
| `EntryPreviewWidget *[blendIn=true]` 同上 | 同上拆分处理 |
| `DatabaseOpenWidget #centralStack, #publicSummaryLabel`（1px `palette(mid)` 边框） | 删除 QSS 版。注意它与 `DatabaseOpenWidget.cpp:263` 的 C++ 验证态边框（`4px solid <color>`）是**同一属性两处写**，内联样式赢；Fluent 下只保留 C++ 验证态（颜色建议改走 `StateColorPalette::Error`，见 2.4） |
| `QGroupBox { bold }` / `::title` 负 margin 定位 | `font-weight` 保留；`::title` 定位与 Fluent GroupBox 自绘可能打架→首批截图验证（欢迎页/设置页） |
| `QToolTip { border: none; padding: 3px }` | 禁用（Fluent tooltip 自绘；先全禁，截图若 padding 失衡再单加回 padding） |
| `#SearchBanner, #KeeShareBanner`（highlight 底＋dark 边框） | 语义保留、实现待定：`1px solid palette(dark)` 在 Fluent 下偏生硬，候选映射为 Fluent 信息横幅样式，截图后定 |
| `QPlainTextEdit, QTextEdit { background: palette(base); padding-left: 4px }` | 拆分：padding 保留；`background-color` 删除（Fluent 输入框底色自管） |
| `*[title=true] { bold }` | 保留（语义加粗，无绘制冲突） |

**`light/lightstyle.qss`——Fluent 下整体禁用**（逐条皆有硬编码 hex，不跟随 Fluent token）：
`DatabaseWidget:!active #FCFCFC` / `:disabled #EDEDED`（含 group/tag/preview 三处）、
`QPushButton:default:hover #568821`、`QToolTip`（`#F9F9F9/#4D7F1A`）、`QGroupBox::title #4B7B19`。
非激活态区分如仍需要，由 Fluent 层（调色/透明度）实现，不移植 hex。

**`dark/darkstyle.qss`——Fluent 下整体禁用**：`:!active #404042` / `:disabled #424242`、
`QPushButton:!default:hover #252528` / `:default:hover #2E582E`、`QToolTip #BFBFBF/#2D532D`、
`QMenu { border: 1px solid #56565A }`＋`QMenu::separator`（QSS 边框会破坏 Fluent 菜单圆角/阴影，
重点删除）、`QGroupBox { background: palette(light) }`。

**`base/classicstyle.qss`——完整保留**：classic 路径（原生风格＋Windows 高对比强制 classic，
`Application.cpp:170-174`）Fluent 不接管；其 `centralStack` 2px groove、`QLineEdit` padding、
SearchBanner 等规则随 classic 一起留用。

### 1.7 `src/gui/styles/StateColorPalette.h` / `.cpp`
- 结论：**必须保留**，它是全仓语义色的唯一出口（错误/警告/提示/未完成、密码健康度五档、
  True/False），明暗两套默认值（light：Error `#FF7D7D`、Warning `#FFD30F`…；dark：
  Error `#802D2D`、Warning `#73682E`…）。
- Fluent 建议：保留构造时读 `kpxcApp->isDarkTheme()` 的双版机制（`StateColorPalette.cpp:22-29`），
  仅重映射色值——Error 对齐 Fluent 错误色，Health/True/False 对齐 Fluent 文本/强调色系；
  Fluent 明暗切换时调用方多为即时构造（`PasswordWidget.cpp:211` 等），无需缓存，随切随取即可。

## 2. 散落各处的 `setStyleSheet` / 调色硬编码（逐文件）

### 2.1 语义色直通（保留，只换色值源＝`StateColorPalette`，Fluent 下不动结构）
- `src/gui/PasswordWidget.cpp:202,217,223`（`QLineEdit { background: %1 }` 重复密码校验态）
- `src/gui/URLEdit.cpp:55-60`（同模板的 URL 错误态）
- `src/gui/PasswordGeneratorWidget.cpp:121`（词表警告 label 着色）、`301-319`
  （熵/强度进度条四档 Health 色）
- `src/gui/ApplicationSettingsWidget.cpp:189-200`（全局快捷键冲突时 `QLineEdit` 染 Error 色）
- `src/browser/BrowserSettingsWidget.cpp:191-200`（代理地址不存在时同上）
- `src/gui/dbsettings/DatabaseSettingsWidgetGeneral.cpp:192-196`、
  `src/gui/entry/EditEntryWidget.cpp:1793-1798`（取色按钮 `background-color` 回显用户自选色——
  用户数据色，任何主题下都保留）
- `src/gui/entry/EntryView.cpp:571-572`（拖拽提示 `QListWidget` 用 `palette(highlight/dark)` 引用——
  写的是 role 引用而非 hex，跟随 Fluent palette，保留）
- `src/gui/WelcomeWidget.cpp:41-45`（`text-align:center`，纯文本布局，保留）
- `src/gui/Icons.cpp:137-145`（单色图标按 `ButtonText/HighlightedText/WindowText` 重染——
  role 引用，自动跟随 Fluent，保留；前提是 Fluent palette 这几个 role 有合理值，切换后抽查图标）
- `src/gui/tag/TagsEdit.cpp:426-427`（选中补全项用 `Highlight/HighlightedText` brush，保留）
- `src/gui/EntryPreviewWidget.cpp:61-64`（只读文本区 `setBackgroundRole(Window)`，保留）

### 2.2 `src/gui/KMessageWidget.cpp`（248-320）——除旧栈外冲突最直接的一处
- 现状：Positive/Information/Warning/Error 四色硬编码 RGB
 （`(37,163,83)/(24,187,42)/(252,193,57)/(198,69,21)`，Warning 配深色字 266-267），
  外加 `qlineargradient` 三段渐变＋1px 边框＋`pixelMetric(PM_DefaultFrameWidth)` 动态 margin，
  全部经 `d->content->setStyleSheet` 直写；关闭按钮另有一段 `QToolButton` 透明＋hover 边框 QSS（292-297）。
- Fluent 建议：四色语义保留、**渐变删除**（Fluent 信息条为纯色/云母质感，渐变必违和）；
  边框/margin 改对齐 Fluent 卡片规范，截图后微调。

### 2.3 `src/gui/DatabaseOpenWidget.cpp:262-266`
- 现状：`centralStack` 验证态 `4px solid <color>` 内联样式，与 `basestyle.qss` 的 1px 版同属性冲突
  （内联赢，QSS 版实际只在无验证色时可见）。
- Fluent 建议：见 1.6 表格——删 QSS 版，C++ 版保留并把颜色来源收敛到 `StateColorPalette`。

### 2.4 `src/gui/CategoryListWidget.cpp`（`IconSelectionCorrectedStyle` 159-205）
- 现状：唯一的 `QProxyStyle` 自定义点，为补 Windows/Fusion 系“选中只画文本包围盒”而全宽
  `fillRect(Highlight)`（172-176）＋吞掉原生焦点框（177-178），另有仅 Windows 的
  `CE_ItemViewItem` 文字色 hack（186-204）。
- Fluent 建议：**Fluent 下停用该 proxy**（条件挂载：仅非 Fluent 时 `setStyle` 到对应 list）。
  Fluent 列表自带全宽选中＋焦点环，留着它必定双重绘制；`CategoryListWidgetDelegate` 本体
  （ paint 207 起，图标＋文字排布）保留，截图验证。

### 2.5 自定义 `QStyledItemDelegate` 清单（其余皆保留、无需重写）
- `src/gui/entry/EntryView.cpp:38-63` `PasswordStrengthItemDelegate`——强度色块描边取
  `QPalette::Shadow`（50-53）；保留，切换后确认 Fluent `Shadow` role 可见即可（看不清则改取 Fluent 边框 token，小改）。
- `src/gui/entry/EntryURLModel.h:28-39` `URLModelIconDelegate`——仅 `decorationPosition = Right`；保留。
- `src/gui/entry/EditEntryWidget.cpp:73-90` `AttributeKeyDelegate`——仅给属性名加 completer；保留。
- `src/fdosecrets/widgets/RowButtonHelper.cpp:25-34` `WidgetItemDelegate`——行内嵌按钮；保留。
- `src/gui/tag/TagView.cpp:31-42` `TagItemDelegate`——先调基类 paint 再画底线标记；保留。
- `src/gui/CategoryListWidget.h:71` `CategoryListWidgetDelegate`——见 2.4，本体保留。
- 另有 `src/gui/entry/EntryView.cpp:452-455`：非 classic 才给表头排序列加 18px 宽——Fluent 表头
  排序指示器宽度未知，**先保留，首批截图验证**（主窗口条目表头）。

### 2.6 向导与主窗口杂项
- `src/gui/wizard/NewDatabaseWizard.cpp:53-63`、`ImportWizard.cpp:46-56`：取默认 palette
  Window/Base 加 alpha 153 再 `lighter(120)` 做半透明页框——在 Fluent/Mica 上叠加易出脏色；
  Fluent 下建议跳过该 tint（待截图验证）。
- `src/gui/MainWindow.cpp:267`：`toolbarSeparator` 仅非 classic 显示——Fluent 视为非 classic，
  逻辑沿用，无改动。
- `src/gui/MainWindow.cpp:1963-1991` 主题菜单（auto/light/dark/classic＋进出 classic 重启）：
  auto/light/dark 改走 Fluent 后应**免重启热切换**（见 §0）；classic 项＋重启逻辑保留。
  菜单是否新增 “Fluent” 显式项为 #1 目标票决策点，本票仅记录。
- `src/gui/osutils/nixutils/NixUtils.cpp:112` 以 `standardPalette().Window` 明度判深浅——
  Fluent 下 `qApp->style()->standardPalette()` 语义变化，Linux 深浅跟随逻辑需复查（Windows 首批不阻塞，记录）。

## 3. Fluent 迁移决策总表

| 类别 | 处理 |
|---|---|
| `BaseStyle` 自绘引擎＋`phantomcolor` | 停用（Fluent 模式永不实例化） |
| `LightStyle/DarkStyle` 全套 palette＋逐窗 `polish` 改色 | 禁用，改由 Fluent palette |
| `basestyle.qss` | 按 §1.6 表拆分：布局/padding/语义加粗保留，凡 `background/border` 绘制类删除或交 Fluent |
| `light/darkstyle.qss` | 整体禁用（含 `QMenu` 边框、`QToolTip` 硬色、`!active/:disabled` hex） |
| `classicstyle.qss`＋高对比强制 classic | 完整保留，Fluent 不碰 |
| `StateColorPalette`＋§2.1 各控件语义 QSS | 保留结构，仅重映射色值到 Fluent 色板 |
| `KMessageWidget` 渐变＋硬编码 RGB | 语义色保留，渐变删除，对齐 Fluent 卡片 |
| `IconSelectionCorrectedStyle`（QProxyStyle） | Fluent 下停用，delegate 本体保留 |
| 其余 delegate（强度/URL/属性/行按钮/标签） | 保留，无需重写 |
| 向导半透明 tint | Fluent 下跳过（待截图确认） |
| 主题切换重启逻辑 | 仅 classic 保留重启；Fluent 三路必须热切换 |

## 4. 一手来源

- 本地源码（行号以本分支为准）：`src/gui/Application.cpp:131-135,165-224`；
  `src/gui/styles/base/BaseStyle.h:29-89`，`BaseStyle.cpp:4216-4236,3070-3086`；
  `src/gui/styles/{base,light,dark}/*.qss` 四份全文（`basestyle.qss` 77 行、
  `darkstyle.qss` 39 行、`lightstyle.qss` 25 行、`classicstyle.qss` 24 行）；
  `src/gui/styles/{dark/DarkStyle,light/LightStyle}.cpp:39-132`；
  `src/gui/styles/StateColorPalette.{h,cpp}` 全文；`src/gui/styles/styles.qrc`；
  `src/gui/CategoryListWidget.cpp:148-205`；`src/gui/entry/EntryView.cpp:38-80,452-455,569-572`；
  `src/gui/KMessageWidget.cpp:253-320`；`src/gui/MainWindow.cpp:267,1963-1991`；
  `src/core/Config.{h,cpp}`（`GUI_ApplicationTheme` 默认 `"auto"`）。
- Qt 官方文档：Qt Style Sheets（样式表接管绘制）
  https://doc.qt.io/qt-6/stylesheet.html ；
  QProxyStyle https://doc.qt.io/qt-6/qproxystyle.html ；
  QApplication::setStyleSheet https://doc.qt.io/qt-6/qapplication.html#setStyleSheet ；
  QStyle https://doc.qt.io/qt-6/qstyle.html 。
- 上游背景：PhantomStyle（`BaseStyle.h:22` 头注，https://github.com/randrew/phantomstyle）；
  Fluent 目标见 #1（`FluentUI3Style`，Qt6.10 Win11 style 移植）——本仓尚无该 submodule，
  以上冲突点按“QProxyStyle＋全量 QSS 互斥”机制判定，不依赖其源码。
