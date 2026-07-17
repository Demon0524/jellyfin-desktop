# 第三方 Jellyfin 桌面播放器产品方案

> 状态：方案记录，尚未完成产品命名与品牌决策  
> 更新日期：2026-07-17  
> 当前基础：`jellyfin/jellyfin-desktop` 3.0.0-dev（CEF + mpv）

## 1. 产品定位

目标是维护一个 Windows 优先、基于官方 Jellyfin Desktop 内核的第三方桌面客户端，重点解决以下问题：

- 播放可靠性：普通媒体、STRM、转码、内嵌/外置字幕都应有可预测的行为。
- 高可用：播放失败可以诊断、降级、恢复和回滚，避免无限重试外部服务。
- 自定义：字幕、音频、快捷键、主题和 OSD 可配置。
- 个性化：使用独立名称、Logo 和视觉语言，不伪装成 Jellyfin 官方客户端。
- 可维护：持续合并上游修复，个性化功能尽量与上游核心代码隔离。

这是一套“稳定增强发行版”，不计划从零重新实现 Jellyfin 协议、CEF 或 mpv 播放内核。

## 2. 品牌与许可证边界

产品名称尚未决定，当前候选：

- `Aching Player`（推荐工作名称，与现有 `aching.cn` 体系一致）
- `Felix Theater`
- `Asteria Player`

推荐描述格式：

```text
Aching Player
A reliable desktop client for Jellyfin
```

公开发行前必须完成：

- 使用不同于 Jellyfin 的产品名称与 Logo。
- 不使用或变形使用 Jellyfin 三角形 Logo。
- 可以使用紫蓝渐变，但 Logo 轮廓必须独立设计。
- 程序名、窗口标题、About 页面、设备上报名称和配置目录独立。
- 建议配置目录为 `%APPDATA%\AchingPlayer`，避免覆盖官方客户端。
- 保留 GPL-2.0 许可证和上游版权信息，并为发行的二进制提供对应源码。

官方依据：

- [Jellyfin 品牌规范](https://jellyfin.org/docs/general/contributing/branding/)
- [Jellyfin Desktop 仓库与 GPL-2.0 许可证](https://github.com/jellyfin/jellyfin-desktop)

## 3. 版本规范

产品使用独立的语义化版本，不直接继承官方 `3.0.0-dev` 作为产品版本。

```text
MAJOR.MINOR.PATCH-预发布阶段.序号
```

示例：

```text
0.1.0-alpha.1
0.1.0-alpha.2
0.1.0-beta.1
0.1.0-rc.1
0.1.0
0.1.1
0.2.0
1.0.0
```

- `MAJOR`：存在不兼容的配置、插件或架构变化。
- `MINOR`：增加向后兼容的用户功能。
- `PATCH`：兼容性 Bug 修复或小范围稳定性改进。
- `alpha`：小范围开发验证。
- `beta`：主要功能完成，扩大日常使用测试。
- `rc`：候选正式版，只接受阻断性修复。
- 无后缀：稳定版本。

上游版本独立记录，不塞进产品版本号：

```text
Product: Aching Player 0.1.0-alpha.1
Upstream engine: jellyfin-desktop 3.0.0-dev
Upstream commit: <commit>
Product build: <commit>
Build date: <UTC timestamp>
```

标签和资产示例：

```text
Tag: v0.1.0-alpha.1
AchingPlayer-0.1.0-alpha.1-windows-x64.zip
AchingPlayer-0.1.0-alpha.1-windows-arm64.zip
AchingPlayer-0.1.0-alpha.1-SHA256SUMS.txt
```

### 发布通道

| 通道 | 用途 | 分发位置 |
|---|---|---|
| Nightly | 每次主分支构建，开发验证 | Actions Artifact |
| Alpha | 小范围功能验证 | GitHub Prerelease |
| Beta | 日常使用测试 | GitHub Prerelease |
| RC | 正式版候选 | GitHub Prerelease |
| Stable | 推荐普通用户使用 | GitHub Release |

当前声音、字幕修复完成实机验证后，第一个建议版本为 `v0.1.0-alpha.1`。

## 4. GitHub Release 流程

目标流程：

```text
提交/合并代码
  -> 自动检查
  -> 创建 v* 标签
  -> Windows x64/ARM64 构建
  -> 生成 SHA256
  -> 创建 Draft + Prerelease
  -> 上传 Release Assets
  -> Windows 实机验收
  -> 手动转为正式发布
```

工作流要求：

- 使用工作流自带的 `GITHUB_TOKEN`，并设置 `permissions: contents: write`。
- Release 只允许从已通过测试的标签构建。
- Alpha/Beta/RC 默认同时标记为 Draft 和 Prerelease。
- Release Notes 至少包含功能、修复、已知问题、上游基线和校验值。
- Nightly 继续使用 Artifact；可长期下载的版本进入 Release。
- Windows 作为第一阶段唯一阻断平台，Linux/macOS 后续逐步加入。

## 5. 高可用定义

这里的“高可用”不是服务端集群，而是客户端面对媒体、网络和播放内核异常时仍能恢复或给出明确原因。

- Jellyfin 媒体元数据为空时，允许 mpv 对真实媒体进行兜底探测。
- 明确区分“未知”“用户关闭”和“明确选择”的音轨/字幕状态。
- Direct Play 失败时提供受控降级，不能制造无限转码或无限请求。
- URL 401/403、超时和临时网络故障使用有限重试、退避与熔断。
- 对 OpenList、115 等外部服务限制请求频率，避免触发风控。
- 播放器崩溃或退出后保留播放进度，并允许一键恢复。
- 新版本出现严重回归时支持回滚上一稳定版本。
- 可选提供外部 mpv/VLC 兜底，但不作为默认播放路径。

## 6. 功能路线

### v0.1：可靠播放基础

- STRM 缺少媒体信息时自动探测音频和默认字幕。
- 手动添加本地 ASS/SSA/SRT/VTT 字幕。
- 音频输出设备选择。
- 音频直通配置模板和安全回退。
- 播放失败自动导出诊断报告。
- 日志隐藏 Token、Cookie、签名 URL 和账号信息。
- GitHub Prerelease 自动发布。
- 版本回滚说明与上一版本保留策略。

### v0.2：字幕与个性化

- 字幕文件拖放。
- 字幕字体、颜色、大小、描边和位置设置。
- 快捷键自定义。
- 深色、影院和简洁主题。
- 自定义强调色、背景和 OSD 密度。
- 配置导入、导出和重置。

### v0.3：网络适应与恢复

- Direct Play 失败后的受控降级。
- 播放地址失效后的有限刷新。
- 超时、重试、指数退避和熔断。
- 多 Jellyfin 服务器保存和快速切换。
- 异常退出后的播放恢复。
- 可选外部播放器兜底。

### v0.4：高级视频能力

- HDR/Dolby Vision 检测和安全回退。
- 刷新率与分辨率切换。
- 硬件解码和渲染状态显示。
- PiP 和多显示器配置。
- 4K/高码率性能诊断。

### v1.0：稳定发行

发布 `1.0.0` 前至少满足：

- Windows x64 安装、升级和卸载流程稳定。
- 自动更新或明确的受控更新机制可用。
- 关键播放选择逻辑有自动测试。
- 普通文件、STRM、Direct Play、转码和外置字幕均通过验收。
- 日志脱敏完成。
- 回滚流程经过实际验证。
- 与上游同步流程经过至少一次完整演练。

## 7. 自动测试与验收

新增 `check.yml`，至少执行：

- `cargo fmt --check`
- `cargo test -p jfn-mpv`
- JavaScript 语法检查
- 音轨与字幕选择纯函数测试
- Windows x64 完整构建

关键用例：

- STRM 元数据为空：`audio=-1 sub=-1`。
- 用户关闭音频或字幕：参数为 `0`。
- 已知内嵌轨道：使用明确轨道编号。
- Jellyfin 外置字幕：调用 `sub-add`。
- 转码输出：不错误套用源文件轨道编号。
- 没有字幕的媒体：播放正常且不报错。
- 失效的默认轨道索引：不能覆盖成错误状态。

播放诊断报告至少包含：

- 客户端、上游和构建版本。
- Direct Play/Direct Stream/Transcode。
- Jellyfin 是否提供 MediaStreams。
- 最终音轨、字幕参数和 mpv 实际选择结果。
- 音频输出设备、硬件解码和关键错误。
- 所有 Token、Cookie、签名参数必须脱敏。

## 8. 上游 Issue 观察清单

### P0：播放可靠性

- [Windows 无声音 #412](https://github.com/jellyfin/jellyfin-desktop/issues/412)
- [音频直通后无声 #474](https://github.com/jellyfin/jellyfin-desktop/issues/474)
- [字幕选项崩溃与字幕重复 #390](https://github.com/jellyfin/jellyfin-desktop/issues/390)
- [Dolby Vision 导致崩溃 #387](https://github.com/jellyfin/jellyfin-desktop/issues/387)
- [fMP4 转码拖动异常 #319](https://github.com/jellyfin/jellyfin-desktop/issues/319)

### P1：Windows 体验

- [选择音频输出设备 #386](https://github.com/jellyfin/jellyfin-desktop/issues/386)
- [Windows 高 DPI #485](https://github.com/jellyfin/jellyfin-desktop/issues/485)
- [安装与更新流程 #487](https://github.com/jellyfin/jellyfin-desktop/issues/487)
- [Verbose 日志不可用 #502](https://github.com/jellyfin/jellyfin-desktop/issues/502)

### P2：自定义

- [字幕样式设置 #583](https://github.com/jellyfin/jellyfin-desktop/issues/583)
- [客户端插件 API #371](https://github.com/jellyfin/jellyfin-desktop/issues/371)
- [设置系统 #559](https://github.com/jellyfin/jellyfin-desktop/issues/559)
- [PiP #561](https://github.com/jellyfin/jellyfin-desktop/issues/561)

不能假定标题相似的问题具有同一根因。纳入本项目路线前仍需复现、收集日志并确认影响范围。

## 9. 参考项目

| 项目 | 借鉴内容 | 定位 |
|---|---|---|
| [Jellyfin Desktop](https://github.com/jellyfin/jellyfin-desktop) | CEF、mpv、Jellyfin Web 和协议兼容 | 上游基础 |
| [旧版 Qt Desktop](https://github.com/jellyfin-archive/jellyfin-desktop-qt) | 设置、快捷键和成熟桌面体验 | 已归档，只参考 |
| [Jellyfin MPV Shim](https://github.com/jellyfin/jellyfin-mpv-shim) | mpv 配置、可靠播放和外部播放 | 播放层参考 |
| [NipaPlay Reload](https://github.com/AimesSoft/NipaPlay-Reload) | 本地字幕、弹幕、跨平台和个性化 UI | 功能设计参考 |
| [Jellyfin Kodi](https://github.com/jellyfin/jellyfin-kodi) | TV 模式和长期运行稳定性 | 交互与可靠性参考 |
| [MPC-JF](https://github.com/Damocles-fr/MPC-JF) | Windows 外部播放器兜底 | 降级路径参考 |

## 10. 上游同步策略

建议分支模型：

```text
upstream/main
  -> sync/upstream-YYYYMMDD
  -> 自动测试和冲突处理
  -> main
  -> feature/* 或 fix/*
  -> 版本标签与 Release
```

- 保留 `upstream` 远端，不改写已发布的公共历史。
- 每次同步使用独立分支和 Pull Request。
- 品牌、诊断、可靠性和自定义逻辑尽量放在独立模块。
- 上游已修复的问题优先回归上游实现，减少永久补丁数量。
- 每次 Release 记录上游 commit，保证可以追踪和复现。

建议逐步形成：

```text
src/flavor/          品牌与主题
src/diagnostics/     播放诊断与脱敏
src/reliability/     重试、回退、恢复与熔断
src/customization/   字幕、快捷键和播放器设置
```

## 11. 已完成和下一步

当前已完成：

- STRM 音频元数据为空时由 mpv 自动选择音轨。
- STRM 字幕元数据为空时由 mpv 自动选择默认字幕。
- Windows x64 构建、打包和 Artifact 上传验证。

下一步按顺序执行：

1. 确认产品名称和 Logo 方向。
2. 建立 `0.1.0-alpha.1` 版本信息与 About 展示。
3. 增加自动检查和 Windows Prerelease 工作流。
4. 将当前声音、字幕修复合入自己的 `main`。
5. 实现手动添加本地字幕。
6. 实现可脱敏的一键播放诊断报告。

