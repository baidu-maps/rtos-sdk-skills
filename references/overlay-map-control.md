# 地图状态、覆盖物与图片适配

完整可运行代码见 [demo.md](demo.md)（`RunMapStateAndOverlayDemo` / `RunTouchDemo` / `RunRouteTileDemo` / 多实例示例）。

## 地图状态与坐标

| 项 | 规则 |
|----|------|
| 坐标系 | 传入 SDK 的经纬度一律按 **GCJ02** 处理；WGS84 必须先转换，见 [SKILL.md § 输入坐标系](../SKILL.md)。不转不报错，整体偏几百米 |
| 坐标序 | 地图侧 `VDPOINT(经度, 纬度)`，检索侧 `Coordinate(纬度, 经度)`，勿混 |
| 角度单位 | `setRotationAngle(h, angle)`、`setMarkerAngle(h, id, angle)` 都是**弧度**。30° 写 `30.0 * M_PI / 180.0` |
| 按覆盖物全览 | `setViewBound(h, getOverlayBounds(h, overlayId))`；地图有旋转时先 `setRotationAngle(h, 0)` 复位 |
| 像素↔经纬度 | `LatLngToScreenPixel` / `ScreenPixelToLatLng`，经纬度侧同为 GCJ02 |
| `SetMapStyle` | 必须在 `InitMap` 之后；切换的是**全局主题**（见"多实例地图"）。返回 `false` = `themeId` 越界，或当前 `.a` 不支持主题切换 |

## 触摸事件

`TouchEvent` 需填 `type`（`TouchEvent::Type::TOUCH_START` / `TOUCH_MOVE` / `TOUCH_END`）、`touches`、`changedTouches`，按 `OnTouchDown` → `OnTouchMove` → `OnTouchUp` 投递。构造写法见 [demo.md § RunTouchDemo](demo.md)。

| 入口 | 取值要求 |
|------|------|
| `OnTouchDown` | 取 `touches[0]` |
| `OnTouchMove` | 取 `touches[1]`：**`touches` 至少放 2 个元素**，只放 1 个会越界读内存（可把同一个 touch push 两次） |
| `OnTouchUp`、图层事件 | 优先 `changedTouches[0]`，为空回落 `touches[0]`；两个数组都空则事件被跳过 |

## 覆盖物与图片适配

顺序：`createLayer` → `CreateMarker` / `CreatePolyline` → 设属性 → `addOverlay(h, layerId, overlayId)` → `updateLayer(h, layerId)` → 渲染一次（已 `SetUIThreadFunc` 时 `updateLayer` 末尾会自动发出刷新通知，未注册回调才需自己调 `RequestRender`）。

| 必须 | 禁止 |
|------|------|
| 设完属性调 `updateLayer`，并确保渲染发生一次 | 靠 `showOverlay` 让覆盖物可见（它只用于撤销 `hideOverlay`，不是必需项） |
| 折线显式设 `setPolylineLineWidth` | 不设线宽（行为未定义：可能不画，可能任意粗细） |
| 折线一起设 `setPolylineFillColor`（`setPolylineStrokeColor` / `setPolylineStrokeWidth` 为可选描边） | 不设填充色（默认不透明黑 (0,0,0,255)，深色底图上看不见） |
| `updateLayer` / `addOverlay` 只传 `createLayer` 返回的 layerId | 传未创建的 layerId（直接崩） |
| `addOverlay` 的目标图层类型必须是 `LayerType::OVERLAYER` | 往其它类型图层加覆盖物（静默失败，无日志无返回值） |

**Marker 图标两种设法**，都依赖已注册的图片 Provider：`setMarkerIcon(h, markerId, resPath)` 传 SDK 可读的绝对路径；`setMarkerVImage(h, markerId, image)` 传内存位图，免文件 I/O。日志 `drawMarker: pixel=(0.000000, 0.000000)` 是 marker 与地图中心重合时的正常输出，不是错误。

**折线点集也可来自 GeoJSON 文件**：`setPolylinePointsURI(h, polylineId, "/abs/path/routes.geojson")`，之后照常设线宽与颜色 → `addOverlay` → `updateLayer` → 渲染一次。该路径直接交平台文件实现打开，**不经过 `GetAppCachePath()`**：给绝对路径，相对路径按进程 CWD 解析；文件读不到或不是合法 GeoJSON `FeatureCollection` 时点集保持为空，无报错无返回值，表现为"折线加了但什么都没画"。

**图片 Provider** 必须在 `InitMap` 之前注册一次，进程级全局；实现要求见 [adapter-build.md § 6](adapter-build.md)。`CreateImageFromPath` / `CreateImageFromData` 返回的对象由 Canvas 侧 `drawImage` 消费，两边必须约定同一具体类型。

```cpp
#include "base/v_image.h"
using namespace baidu::rtos_map::image;
VImage::SetImageProvider(std::make_shared<MyImageProvider>());   // 只需一次
auto markerImage = std::make_shared<VImage>("marker_icon.png", 36, 36);
MapViewApi::setMarkerVImage(h, markerId, markerImage);
```

## 路线瓦片预加载

`LoadRouteMapData` / `DeleteRouteMapData` 的 `routeFile` 传**文件名**，不是完整路径；取消用 `CancelMapDataDownload`。代码见 [demo.md § RunRouteTileDemo](demo.md)。

- geojson 源文件（`routeFile` 本身）交平台文件实现打开，**不经过 `GetAppCachePath()`**，相对路径按进程 CWD 解析；瓦片账本文件 `route/<routeFile>_tile.data` **相对 `GetAppCachePath()`** 解析。让两者落在同一目录：调用前把 CWD 切到 `GetAppCachePath() + "/route"`，结束后切回。

| 回调结果 | 含义 |
|------|------|
| `progressPercent == -1`（`isCompleted` 同为 `true`） | 范围内瓦片超过 5000 张，放弃预加载；`totalCount` 为实际张数 |
| 一次都不触发 | 瓦片范围无效或算出 0 张（geojson 没读到 / 解析失败 / 点集为空）。不是"还在下载中" |
| 一次即完成，`totalCount=0`、`progressPercent=100`、瓦片列表为空 | 账本文件已存在，跳过下载；要重下先 `DeleteRouteMapData` |

## 多实例地图

各实例独立 `Create` / `SetCanvas` / `InitMap` / `Destroy`，**地图状态**（中心点、缩放、旋转、图层、覆盖物）互不影响。以下为**进程级共享**：

| 共享项 | 规则 |
|------|------|
| 底图瓦片缓存、图片 Provider | 缓存在首个 `Create()` 时建立、最后一个 `Destroy()` 后释放；`VImage::SetImageProvider` 全局唯一 |
| 地图主题 | `SetMapStyle` 对任一实例调用都影响其它实例；按全局设置统一管理，不要期望每实例不同主题 |
| handle 计数器 | `Destroy` 不复位，销毁后重建拿不回 1、2；handle 一律用 `Create()` 返回值传递，禁止写常量 |
