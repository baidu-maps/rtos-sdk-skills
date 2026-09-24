# 工作区

[English](README_EN.md)

本仓库提供面向 **百度地图 RTOS SDK（mapsdk-rtos）** 的 AI Coding Agent Skill `mapsdk-rtos-app-sdk`，帮助开发者在智能研发工具（如 Cursor / Claude Code 等）中高效完成 RTOS 地图应用层集成**与平台移植**：交付物结构、公开 API 头树、平台移植契约、初始化鉴权、Canvas 适配、地图组件控制、覆盖物绘制、检索路线、离线地图、导航与场景识别（mecp）等开发任务。

## Skill 能力概览

- **初始化与鉴权** — License 鉴权、基础配置、包名/缓存路径/版本信息、初始化顺序、线程模型与联调验收
- **平台移植契约与 Canvas 适配** — C 平台契约 `bd_map_*`、C++ 适配类 `*Impl`、平台 util 自由函数、网络契约 `BNetwork_*`、Canvas 契约（71 项）、图片 `ImageProvider`
- **最小适配骨架** — 可直接编译链接通过的空实现骨架（6 个文件、全部待实现函数、自检清单、从骨架到可用的替换顺序）
- **地图组件控制** — `MapViewApi` 初始化、生命周期、地图状态控制、渲染请求与触摸事件（静态方法 + `MapViewHandle`，支持多实例）
- **覆盖物绘制** — Marker、Polyline、点集 / GeoJSON、样式设置、显示隐藏与图层更新
- **检索与路线规划** — POI 检索、逆地理、步行/驾车路线规划、路线瓦片预加载
- **离线地图** — 版本请求、可下载城市列表、下载状态、离线包管理
- **导航能力** — `NaviApi` 初始化、路线结果接入、导航启动与状态监听
- **场景识别（mecp）** — AOI 场景识别（机场/火车站/商场/水域等），数据在线下载、识别本地完成，独立模块

适用于对接 `includes/` 下 `*_api.h` 公开头文件、并实现 `common/` 平台移植契约的 RTOS 地图应用开发场景。**输入坐标系必须为 GCJ02**，其它坐标系（如设备 GNSS 的 WGS84）须应用侧先转换。

## 适用环境

- **平台**：RTOS / 桌面模拟器（rtos-map-simulator）
- **开发语言**：C / C++
- **SDK**：mapsdk-rtos（交付物：`includes/` 54 个公开头、`common/` 平台移植契约、`libmapsdk.a`）
- **核心 API**：`MapViewApi`、`AuthLicenseApi`、`SearchApi`、`NaviApi`、`MapOfflineApi`、`BaseApi`、mecp C API（`rtos_mecp.h`）
- **集成方式**：对接 `includes/` 公开头文件与 `common/` 平台移植契约（`bd_map_*` / `*Impl` / 平台 util / `BNetwork_*` / Canvas / ImageProvider）

## 目录结构

```
rtos-sdk-skills/
├── README.md                         # 本文件（中文）
├── README_EN.md                      # English README
├── SKILL.md                          # Skill 定义文件（mapsdk-rtos-app-sdk）
└── references/
    ├── init-auth.md                  # 初始化顺序、线程模型、联调验收清单、日志关键字
    ├── adapter-skeleton.md           # 最小可编译适配骨架
    ├── adapter-build.md              # 平台移植契约与 Canvas 适配实现
    ├── overlay-map-control.md        # 地图状态控制与覆盖物、触摸、路线瓦片、多实例
    ├── search-navi-offline.md        # 检索、导航与离线地图
    ├── mecp.md                       # 场景识别（独立模块）
    └── demo.md                       # 全流程可运行示例
```

## 使用方式

### 1. 克隆本仓库

```bash
git clone https://github.com/baidu-maps/rtos-sdk-skills.git
cd rtos-sdk-skills
```

### 2. 从 Release 下载（可选）

你也可以直接从 [Releases](https://github.com/baidu-maps/rtos-sdk-skills/releases) 下载附件 `rtos-sdk-skills.zip`，然后解压使用：

```bash
unzip rtos-sdk-skills.zip
cd rtos-sdk-skills
```

### 3. 将 Skill 注册到你的 AI 助手

把本仓库目录链接或复制到当前环境对应的 skills 目录，这样 AI 在对话时会自动读取这些文档。

**Claude Code（本地）**

- Skills 目录一般为：`~/.claude/skills/`
- 注册（软链，推荐）：
  ```bash
  ln -sfn "$(pwd)" ~/.claude/skills/mapsdk-rtos-app-sdk
  ```
- 或直接把本仓库文件夹复制到 `~/.claude/skills/mapsdk-rtos-app-sdk` 下。

**Cursor**

- Skills 目录一般为：`~/.cursor/skills-cursor/`
- 注册（软链，推荐）：
  ```bash
  ln -sfn "$(pwd)" ~/.cursor/skills-cursor/mapsdk-rtos-app-sdk
  ```
- 或直接把本仓库文件夹复制到 `~/.cursor/skills-cursor/mapsdk-rtos-app-sdk` 下。

### 4. 在对话中使用

在支持 Skills 的客户端里，当你的问题涉及「RTOS 地图 SDK」「mapsdk-rtos」「移植到 RTOS」「MapViewApi」「AuthLicenseApi」「SearchApi」「NaviApi」「MapOfflineApi」「BaseApi / 坐标转换」「Canvas 适配」「平台移植契约 / common」「bd_map_* / BNetwork_*」「WGS84 / GCJ02」「mecp / 场景识别」「RTOS 覆盖物」「RTOS 导航」「RTOS 离线地图」「多实例地图」等关键词时，助手会优先参考本仓库文档来回答，从而给出更贴合百度地图 RTOS SDK、可编译可链接可移植的代码与用法。

## 参考文档

- [SKILL.md](SKILL.md) — Skill 触发说明、交付物、编译与 `-I` 根、输入坐标系、全局硬规则与现象速查
- [references/init-auth.md](references/init-auth.md) — 初始化顺序、线程模型、联调验收清单、日志关键字
- [references/adapter-skeleton.md](references/adapter-skeleton.md) — 可编译的最小适配骨架：6 个文件、全部待实现函数、自检清单、从骨架到可用的替换顺序
- [references/adapter-build.md](references/adapter-build.md) — 平台移植契约：`bd_map_*`、`*Impl`、平台 util、`BNetwork_*`、Canvas（71 项分类）、图片；`.a` 与目标环境匹配
- [references/overlay-map-control.md](references/overlay-map-control.md) — 地图状态、Marker、Polyline、触摸、图片、路线瓦片、多实例
- [references/search-navi-offline.md](references/search-navi-offline.md) — 检索、导航、离线地图
- [references/mecp.md](references/mecp.md) — 场景识别（独立模块）
- [references/demo.md](references/demo.md) — 可运行的全流程示例

## 许可证

本项目为百度内部项目，仅供授权使用。
