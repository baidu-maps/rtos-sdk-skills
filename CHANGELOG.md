# Changelog

## 2026-09-24
### 变更
- Skill 定位由"应用层集成开发助手"升级为"应用层集成**与移植规范**"，Skill 名由 `baidu-map-rtos-skills` 调整为 `mapsdk-rtos-app-sdk`
- 交付物新增 `common/` 平台移植契约（应用必须实现、SDK 会调用的接口）：C 平台契约 `bd_map_*`、C++ 适配类 `*Impl`、平台 util 自由函数、网络契约 `BNetwork_*`、Canvas 契约（71 项）、图片 `ImageProvider` 契约
- `includes/` 明确为 54 个公开 API 头；`includes/` 与 `common/` 均须作为 `-I` 根并连同其全部子目录一起加（多根编译）
- 明确输入坐标系为 **GCJ02**：WGS84 等其它坐标系必须应用侧先转换，并给出 `ConvertCoord` 的有效转换方向与坐标序约束
- 新增场景识别模块 mecp（`mecp/include/rtos_mecp.h`，AOI 识别，独立于地图/检索/导航/离线）
- 新增参考文档 `adapter-skeleton.md`（最小可编译适配骨架：6 个文件、全部待实现函数、自检清单、从骨架到可用的替换顺序）与 `mecp.md`（场景识别）
- 实现参考改为 `github.com/baidu-maps/rtos-map-simulator`（桌面可运行接入）；`demo.md` 由"在 mapAPP 工程扩展 Demo"改为可照抄的全流程示例集
- 补充全局硬规则表与"现象速查"排障表

## 2026-08-25
### 变更
- 地图组件接口由单实例 `MapComponentApi` 迁移为静态多实例接口 `MapViewApi`（按 `MapViewHandle` 区分实例）
- 公开头文件目录由 `outputIncludes/` 调整为 `includes/`
- Canvas 适配：`drawImage` 改为传入 PNG 文件路径由平台自行解码；`measureText` 依赖的 `simhei.ttf` 需存在于 `getAppCachePath()` 路径下
- Canvas 绘制与 `RequestRender` 的线程约束由"必须在 UI 主线程"调整为"必须在同一固定线程"（不再要求是 UI 主线程）
- 离线地图：移除 `GetOfflineCityInfoByCurrentLocation`，离线包路径改为由 SDK 内部管理
- Marker 图标（`setMarkerVImage`）依赖 `VImage::SetImageProvider` 注册的 Provider，simulator 等平台需应用侧自行注册
- 更新模拟器 Adapter 参考实现路径（`src/adapter_impl/` 及子目录）

## 2026-06-29
### 变更
- 初始版本发布
- 增加百度地图 RTOS SDK 应用层集成开发助手 (baidu-map-rtos-skills)
