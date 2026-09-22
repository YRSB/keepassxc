# Handoff：KeePassXC FluentUIStyle Windows 真机执行

> 给新 harness（Windows + Qt6.2.4 + MSVC v143 会话）的一页纸。
> 地图：[KeePassXC FluentUIStyle 美化路线图](https://github.com/YRSB/keepassxc/issues/1)。
> 本机（当前环境）无 Qt/MSVC 工具链，以下两步只能在这里做。

## 已锁定的决策（不用重议）

- [Fluent 编译兼容实证](https://github.com/YRSB/keepassxc/issues/2，
  笔记 `research/fluent-compat` → `docs/research-fluent-compat.md`）：
  无 Private 头依赖；基样式 `windowsvista`（缺 DLL 降级 `fusion`）；
  `FLUENTUI3STYLE_COPY_TO_QT_DIR` 必须显式 OFF；C++17/20 同 MSVC 可链；
  **最大风险 R1：无 6.2.4 真机实证，先编过再接**。
- [旧主题/QSS 冲突清单](https://github.com/YRSB/keepassxc/issues/3，
  笔记 `research/theme-conflicts` → `docs/research-theme-conflicts.md`）：
  迁移决策总表（BaseStyle 停用、深浅 qss 禁用、`basestyle.qss` 拆分、
  classic 保留、调色板只换值、KMessageWidget 去渐变、选中补丁停用、
  Fluent 三路热切换）。
- [深浅映射决策](https://github.com/YRSB/keepassxc/issues/4）：
  三档改名标 Fluent、菜单四项不变、classic 重启与高对比保留、存量原地接管。

## 先做这张：submodule + CMake 接入骨架

1. 认领：`gh issue edit 6 --add-assignee @me`
2. 首动作真编（Qt6.2.4 + MSVC v143，`develop` 出 throwaway 分支）：
   `git submodule add https://github.com/XHY-ChuJian/FluentUIStyle.git 3rd/FluentUIStyle`
   后钉 tag/commit；configure 带
   `-DBUILD_EXAMPLES=OFF -DBUILD_FRAMELESS=OFF -DBUILD_QTCREATOR_PLUGIN=OFF -DFLUENTUI3STYLE_COPY_TO_QT_DIR=OFF`
3. 按笔记 §8 最小片段接 `add_subdirectory`＋插件部署（`styles/FluentUI3*.dll`
   手动随包，核对 `qwindowsvistastyle.dll` 在包内）。
4. 验收：configure＋build 通过、`QStyleFactory::keys()` 见 `FluentUI3`；
   然后 resolution 评论＋close＋地图 Decisions 追一行。一次一票。

## 再做这张：主窗口＋解锁＋欢迎页原型

认领 [原型票](https://github.com/YRSB/keepassxc/issues/5)（已解挡），throwaway 分支，
Win10/Win11 明暗截图＋切换录屏，重点验 Fluent 三路热切换与决策总表。
反馈挂票据评论；ExWidgets/无边框/打包体积/上游策略四条雾按反馈毕业或转新票。

## 别碰

`src/crypto`、`src/format`；`FLUENTUI3STYLE_COPY_TO_QT_DIR=ON`（禁写 Qt 目录）；
首批 `BUILD_FRAMELESS=ON`（QWindowKit 留后）。
