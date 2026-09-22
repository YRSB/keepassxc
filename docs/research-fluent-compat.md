# Fluent 编译兼容实证（Qt6.2.4 / MSVC）

> 票据：gh #2（Part of #1）。分支：`research/fluent-compat`（基于 `f77f4f16`）。
> 性质：只读研究，本机无 Qt/MSVC 工具链（`qmake`/`cl` 均不可用），**未做真实编译**；
> 以下全部来自上游源码与 Qt 一手文档，结论标出了待真机验证项（→ #6 前置）。

## 上游对象

- 仓库：[XHY-ChuJian/FluentUIStyle](https://github.com/XHY-ChuJian/FluentUIStyle)（`master`，MIT，
  210 stars；旧 `.pri` 原型见 [HIllya51/Window11Style](https://github.com/HIllya51/Window11Style)，只做背景参考）。
- 本质：`FluentUI3Style : public QProxyStyle`，移植自 **Qt 6.10 自带 Windows 11 样式**，
  默认基样式 `windowsvista`、缺失时回退 `fusion`（见 §4）。
- KeePassXC 侧约束（本地 `CMakeLists.txt`）：`CMAKE_CXX_STANDARD 20`（第 166 行）、
  `Qt6Core_VERSION >= 6.2.4` 否则 FATAL（第 459–461 行）。

## 1. 顶层 CMakeLists 实测值

来源：[CMakeLists.txt](https://raw.githubusercontent.com/XHY-ChuJian/FluentUIStyle/master/CMakeLists.txt)

| 项 | 值 |
|---|---|
| `cmake_minimum_required` | **3.16**（KeePassXC 侧更高，无冲突） |
| `CMAKE_CXX_STANDARD` | **17**（`REQUIRED ON`）；KeePassXC 为 20，见 §6 |
| `find_package` | `QT NAMES Qt6 Qt5`，组件 `Core Gui Widgets Svg` |
| `BUILD_LIBRARY` | 默认 **ON**（`add_subdirectory(fluentui3style)`，SHARED 库） |
| `BUILD_PLUGIN` | 默认 **ON**，但 `if(NOT WIN32) set(BUILD_PLUGIN OFF)`——**Windows 才有插件** |
| `BUILD_FRAMELESS` | 默认 **ON**，引入 `3rd/qwindowkit` 子模块（QWindowKit）；`ANDROID` 下强制 OFF |
| `BUILD_EXAMPLES` | 默认 **ON**（Gallery + 示例） |
| `BUILD_QTCREATOR_PLUGIN` | 默认 **OFF** |
| Qt Private 包 | **仅 Qt ≥ 6.10 才 `find_package(Qt6 … CorePrivate GuiPrivate WidgetsPrivate)`**；6.2 不需要（见 §3） |
| 默认安装前缀 | 当前 Kit 的 Qt 安装目录（`qmake -query QT_INSTALL_PREFIX`），`cmake --install` 进同一 Kit |

## 2. `FLUENTUI3STYLE_COPY_TO_QT_DIR` 真值

来源：[fluentui3style/plugin/CMakeLists.txt](https://raw.githubusercontent.com/XHY-ChuJian/FluentUIStyle/master/fluentui3style/plugin/CMakeLists.txt)（该 `option` **定义在此文件，不在顶层**）；
说明见 [USAGE.md](https://raw.githubusercontent.com/XHY-ChuJian/FluentUIStyle/master/USAGE.md)。

- 默认 **ON**：`POST_BUILD` 把插件拷到 `QT_INSTALL_PLUGINS/styles`，
  并把 `fluentui3styleproperties.h` 拷到 `QT_INSTALL_HEADERS/FluentUI3Style/`。
- 需要**写 Qt 安装目录的权限**（USAGE 原话：`Program Files` 下需管理员权限）。
- 上游 CI 显式设 **OFF**（[build.yml](https://raw.githubusercontent.com/XHY-ChuJian/FluentUIStyle/master/.github/workflows/build.yml)），
  我们的超级构建/CI **必须同样显式 `-DFLUENTUI3STYLE_COPY_TO_QT_DIR=OFF`**，改走 `cmake --install` + 自有部署。
- 插件元数据：`Q_PLUGIN_METADATA(IID "org.qt-project.Qt.QStyleFactoryInterface" FILE "fluentui3styleplugin.json")`，
  json 内容仅 `{ "Keys": [ "FluentUI3" ] }` → 业务侧 `app.setStyle("FluentUI3")` 即可，无需链接库。

## 3. Qt Private 头：6.2 下不缺也不用

- 顶层 CMake 只在 `QT_VERSION >= 6.10.0` 时拉 `CorePrivate/GuiPrivate/WidgetsPrivate`；**6.2 分支不拉**。
- 对 `fluentui3style/{fluentui3style.h, fluentui3style.cpp, qstylehelper_p.h, qstyleanimation_p.h,
  thememanager.h, thememanager.cpp, plugin/fluentui3styleplugin.cpp}` 逐文件 grep，
  **零条** `private/`、`CorePrivate`、`GuiPrivate`、`WidgetsPrivate` 引用。
- `qstylehelper_p.h` / `qstyleanimation_p.h` 是**自行 vendored 的 Qt 源码拷贝**
  （头顶 `Copyright (C) 2016 The Qt Company Ltd.` + "not part of the Qt API" 警告），
  只 include 公开模块头（`QtCore/qstring.h`、`QtGui/qpainter.h`、`QtWidgets/qwidget.h` 等）。
- 结论：**无 Qt Private 头依赖，6.2.4 不存在“缺 private 包”问题**。

## 4. 基样式 `windowsvista` 在 Qt6.2.4/Windows 可用

来源：[fluentui3style.cpp](https://raw.githubusercontent.com/XHY-ChuJian/FluentUIStyle/master/fluentui3style/fluentui3style.cpp)（`createBaseStyle`，约 1546–1564 行）：

```cpp
static QStyle* createBaseStyle( QStyle* style )
{
    if ( style ) return style;                       // 显式传入为准
    const QString preferredKey = QStringLiteral( "windowsvista" );
    const QString fallbackKey  = QStringLiteral( "fusion" );
    if ( QStyle* base = QStyleFactory::create( preferredKey ) ) return base;
    return QStyleFactory::create( fallbackKey );     // 非 Windows/缺插件时兜底
}
FluentUI3Style::FluentUI3Style( QStyle* style )
    : QProxyStyle( createBaseStyle( style ) ) { … }
```

Qt 一手文档佐证：

- [QStyleFactory](https://doc.qt.io/qt-6/qstylefactory.html)：合法 key 通常含 `"windows"`/`"fusion`，
  **"Depending on the platform, `windowsvista` and `macos` may be available"**——Windows 下可用。
- [Qt for Windows – Deployment](https://doc.qt.io/qt-6/windows-deployment.html) 的部署清单明确列出
  `styles\qwindowsvistastyle.dll`：vista 基样式在 Qt6 下是 **styles 插件 DLL**，
  随包与否决定走 `windowsvista` 还是 `fusion` 兜底（视觉有细微差，README 已声明）。

## 5. Qt6.2.4 可编译性：上游写法已做版本门控，但无 6.2 实证

有利证据（`fluentui3style.cpp` 内散布的版本门）：

| 门 | 用途 | 6.2.4 落点 |
|---|---|---|
| `#if QT_VERSION >= QT_VERSION_CHECK(6, 2, 0)` | `QFlags::testAnyFlags`（[Qt 文档标注 since 6.2](https://doc.qt.io/qt-6/qflags.html)） | 走新 API，门槛恰好 6.2 ✓ |
| `#if QT_VERSION < QT_VERSION_CHECK(6, 3, 0)` | 6.3 前 `PE_IndicatorTabClose` 兼容绘制 | 命中兼容分支 ✓ |
| `#if QT_VERSION >= QT_VERSION_CHECK(6, 5, 0)` | `QStyleHints::colorScheme` 系统深浅 | 6.2 走 `fluentuiappearance.h`/`palettemanager.h` shim（`#if QT_VERSION <= 6.8.0` 引入，见文件头约 74 行） |
| 6.0 / 6.6 / 6.9 等门 | 各控件细节 | 均有对应分支 |

不利（缺口）实证：

- [README 兼容表](https://github.com/XHY-ChuJian/FluentUIStyle)只声明
  Qt **5.12 / 5.14.2 / 5.15.2 / 6.5.3 / 6.6.3 / 6.8+ / 6.10**（MSVC/MinGW），**无 6.2.x**。
- [CI 矩阵](https://raw.githubusercontent.com/XHY-ChuJian/FluentUIStyle/master/.github/workflows/build.yml)
  只有 **Windows 5.15.2 + Windows 6.8.3 + Linux×2** 四格，**无 6.2 任务**。
- 结论：**“能编过 6.2.4”目前是高置信推断（门控齐全），不是实证**；
  #6 动工前必须在 Qt6.2.4 + MSVC（v143） CI/本机上真编一次（`BUILD_EXAMPLES=OFF BUILD_FRAMELESS=OFF` 起步）。

## 6. C++17 库 × C++20 工程混编

- 上游库 `CMAKE_CXX_STANDARD 17`，KeePassXC 可执行文件 C++20。
  同一 MSVC 工具链内 C++ 标准版本不同**可以链接**（语言标准是编译期约束，不改变 MSVC ABI/CRT），
  前提：**同一编译器大版本 + 同一 CRT + 同一配置（Debug/Release 不可混）**。
  上游库/插件均设 `DEBUG_POSTFIX d`，已防 Debug/Release DLL 重名覆盖。[INFERENCE：待 #6 真编时用同一 Kit 复核]
- 额外：上游顶层对 MSVC 加 `/utf-8 /wd4273`（Release `/Z7 + /DEBUG:FULL`），与 KeePassXC 的全局 flags 各管各的
  target，一般无冲突；接入时若出现警告升级为错，优先关上游 `BUILD_EXAMPLES` 而不是动全局 flags。

## 7. windeployqt 随包方式

KeePassXC 已有部署链：`src/CMakeLists.txt`（约 559–575 行）`install(CODE)` 调 `${DEPLOYQT_EXE}`，
`cmake/KPXCWindowsDeployQt.cmake` + `tests/cmake/test_windows_deployqt_host_triplet.cmake` 负责找
`windeployqt.exe`。Qt 文档（[Deployment](https://doc.qt.io/qt-6/windows-deployment.html)）要点：

- `windeployqt` 只认 **Qt 自家插件/DLL**；`FluentUI3StylePlugin`（第三方 style 插件）**不会被自动收录**，
  必须在 `windeployqt` 之后**手动把 `plugins/styles/FluentUI3*.dll`（及 `d` 后缀的 Debug 版）拷进包内
  `styles/`**，或写一条 `install(FILES … DESTINATION styles)`。
- 同理 `qwindowsvistastyle.dll`（基样式）是否被收录要在打出的包里复核——缺了会自动降级 `fusion`，
  功能不坏但像素级效果漂移。
- 走**源码链接**（`new FluentUI3Style` + 链 `FluentUI3Style` SHARED 库）则改为：DLL 随 exe 旁部署
  （`RUNTIME DESTINATION bindir` 那套），外加 `dwmapi`（系统库，`target_link_libraries … PRIVATE dwmapi`，无需部署）。
- `Frameless` 是**静态库**（无 `Frameless.dll`），但会拖入 QWindowKit 的 `QWKCore/QWKWidgets` DLL——
  首批建议 `BUILD_FRAMELESS=OFF`，把无边框留到 #5 之后（`USAGE.md` 亦称其可选）。

## 8. 可编译的最小 CMake 片段（给 #6 用，尚未验证）

```cmake
# 以 submodule 引入：3rd/FluentUIStyle（先 OFF 最小集合，编过再逐个开）
set(BUILD_EXAMPLES OFF CACHE BOOL "" FORCE)
set(BUILD_FRAMELESS OFF CACHE BOOL "" FORCE)          # 首批不要 QWindowKit
set(BUILD_QTCREATOR_PLUGIN OFF CACHE BOOL "" FORCE)
set(FLUENTUI3STYLE_COPY_TO_QT_DIR OFF CACHE BOOL "" FORCE)  # 绝不写 Qt 安装目录
set(EXWIDGETS_BUILD_TESTS OFF CACHE BOOL "" FORCE)
add_subdirectory(3rd/FluentUIStyle)

# 方式 A（推荐首批）：插件式，业务工程不链接
#   部署时把 $<TARGET_FILE:FluentUI3StylePlugin> 拷到包内 styles/
#   app.setStyle("FluentUI3");

# 方式 B：源码链接式（Linux 同理，Windows 亦可）
# target_link_libraries(keepassxc PRIVATE FluentUI3Style)
#   app.setStyle(new FluentUI3Style);
```

对应上游 CI 的等价最小配置（除 Qt 版本外）：`build.yml` 的 Configure 行即
`-DBUILD_LIBRARY=ON -DBUILD_PLUGIN=ON -DBUILD_EXAMPLES=ON -DBUILD_FRAMELESS=OFF
-DFLUENTUI3STYLE_COPY_TO_QT_DIR=OFF`（Windows 格）。

## 9. 风险清单

- **R1（最大）：无 Qt6.2.4 真机实证。** README/CI 均无 6.2；#6 第一动作就是加一个
  Qt6.2.4+MSVC 的 configure+build 任务，编不过则钉死上游 commit 并打 patch。
- **R2：`FLUENTUI3STYLE_COPY_TO_QT_DIR` 默认 ON 会写 Qt 目录。** 超级构建/CI 必须显式 OFF；
  开发者本机若曾 ON 编过，Qt 目录残留旧插件会干扰排障（`QStyleFactory::keys()` 能验）。
- **R3：基样式依赖 `qwindowsvistastyle.dll` 随包。** 打包后无此 DLL 即静默降级 fusion；
  验收清单需加“包内 `styles/` 内容核对”。
- **R4：`BUILD_FRAMELESS=ON` 默认拖入 QWindowKit 子模块。** 首批 OFF，否则 submodule 与构建面都变大。
- **R5：C++17/20 混编与 Debug/Release 混用。** 同一 MSVC 大版本+同配置是前提；`d` 后缀已防重名，
  但 CI 必须同 job 编同配置。
- **R6：本机无工具链。** 本票结论全系静态实证；本机 `qmake`/`cl` 缺失已确认，未编译。
- **R7：6.2 的 auto 深浅走 shim。** `colorScheme`（6.5+）不可用，`fluentuiappearance` 行为差
  留给 #4 决策（classic/高对比保留策略相关）。
- **R8：MinGW 菜单弹出需特殊处理**（README 原话）。KeePassXC Windows 主力 MSVC，影响小，
  但 MinGW 构建流要回归。

## 来源

1. <https://github.com/XHY-ChuJian/FluentUIStyle>（README：兼容表、windowsvista/fusion 基、BUILD_* 默认值、CI 徽章）
2. <https://raw.githubusercontent.com/XHY-ChuJian/FluentUIStyle/master/CMakeLists.txt>（3.16 / C++17 / 选项 / Private≥6.10 / 非 Win 关插件）
3. <https://raw.githubusercontent.com/XHY-ChuJian/FluentUIStyle/master/fluentui3style/plugin/CMakeLists.txt>
   （`FLUENTUI3STYLE_COPY_TO_QT_DIR` 默认 ON、POST_BUILD 拷贝、dwmapi、DEBUG_POSTFIX d）
4. <https://raw.githubusercontent.com/XHY-ChuJian/FluentUIStyle/master/fluentui3style/CMakeLists.txt>（SHARED 库 + dwmapi）
5. <https://raw.githubusercontent.com/XHY-ChuJian/FluentUIStyle/master/USAGE.md>（COPY 语义/管理员权限/两种加载/ExWidgets-Frameless 部署）
6. <https://raw.githubusercontent.com/XHY-ChuJian/FluentUIStyle/master/.github/workflows/build.yml>
   （矩阵 5.15.2/6.8.3、无 6.2；CI 用 `COPY_TO_QT_DIR=OFF FRAMELESS=OFF`）
7. <https://raw.githubusercontent.com/XHY-ChuJian/FluentUIStyle/master/fluentui3style/fluentui3style.cpp>
   （`createBaseStyle` windowsvista→fusion；QT_VERSION 门 6.0/6.2/6.3/6.5/6.6/6.8/6.9；`fluentuiappearance` shim）
8. <https://raw.githubusercontent.com/XHY-ChuJian/FluentUIStyle/master/fluentui3style/fluentui3style.h>
   （`FluentUI3Style(QStyle* = nullptr)` 公开构造）
9. <https://doc.qt.io/qt-6/qstylefactory.html>（windowsvista 平台相关可用）
10. <https://doc.qt.io/qt-6/windows-deployment.html>（windeployqt 行为、`styles\qwindowsvistastyle.dll` 随包模型）
11. <https://doc.qt.io/qt-6/qflags.html>（`testAnyFlags` since 6.2 → 6.2 门正确）
12. 本地 `CMakeLists.txt:166`（C++20）、`:459–461`（Qt≥6.2.4）、`src/CMakeLists.txt:559–575`
    （windeployqt install 集成）、`cmake/KPXCWindowsDeployQt.cmake`（deployqt 定位）
