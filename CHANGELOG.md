# Changelog

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
