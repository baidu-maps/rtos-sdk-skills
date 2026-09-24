---
name: mapsdk-rtos-app-sdk
description: 百度地图 RTOS SDK（mapsdk-rtos）应用层集成与移植规范。帮助应用开发者把 mapsdk-rtos 接入自己的平台/RTOS：交付物结构、公开 API 头树（includes/，多根 -I 编译）、平台移植契约（bd_map_* C 接口、线程/锁/文件等 *Impl、网络 BNetwork_*、Canvas、平台 util）、初始化鉴权顺序、线程模型、输入坐标系（GCJ02）、地图控制与多实例、覆盖物、检索路线、离线地图、导航、场景识别 mecp。当用户提到：对接 RTOS 地图 SDK、集成 mapsdk-rtos、移植到自己的 RTOS、RTOS 地图初始化、MapViewApi、AuthLicenseApi、SearchApi、NaviApi、MapOfflineApi、Canvas 适配、平台 adapter、bd_map_malloc/bd_map_log、BNetwork_fetch、坐标系/WGS84/GCJ02 转换、RTOS 覆盖物/导航/离线地图、多实例地图时，必须使用本 skill 确保生成正确、可编译、可链接、可移植到目标平台的代码。
---

# mapsdk-rtos 应用层集成与移植规范

**适用**：用交付的 `includes/` 公开 API 写地图/检索/导航/离线/mecp 业务代码；实现 `common/` 下的平台移植契约；把地图能力移植到目标 RTOS；排查不渲染/覆盖物不可见/鉴权失败/链接缺符号等问题。
**不适用**：修改 SDK 内部实现。

## 接入前必须先向使用者确认

触发"集成 / 对接 / 移植"类请求时，先问清下列各项再写代码。

1. `libmapsdk.a` 是否用你的目标 toolchain 构建？架构、ABI（libc++ 还是 libstdc++）是否与应用一致？核实方法见 [adapter-build.md § 7](references/adapter-build.md)。
2. 定位源输出哪种坐标系？**SDK 只接受 GCJ02**，其它坐标系必须应用侧先转换（见下方"输入坐标系"）。
3. 平台的 HTTP/TLS 栈是什么？三个回调如何收敛到同一个固定线程？
4. Canvas 后端是什么？`drawImage` 接受的图像格式、`measureText` 用的字体文件路径、`fillText` 的原生锚点（左上还是左下）分别是什么？
5. 需要哪些能力（地图 / 覆盖物 / 检索 / 导航 / 离线 / mecp）？
6. 渲染线程是哪个？
7. 目标设备的包名、数据与缓存目录、需要的缩放级别范围？后两项与 `.a` 的构建绑定，须在索取 `.a` 时一并提供给 SDK 方。

## 交付物

| 交付物 | 说明 |
|--------|------|
| `includes/` | 公开 API 头树（54 个头）。作为 `-I` 根，**须连同其所有子目录一起加** |
| `common/` | 平台移植契约：应用必须实现、SDK 会调用的接口。作为 `-I` 根，**须连同其所有子目录一起加** |
| `libmapsdk.a` | 目标库。拿到后先核实架构/ABI 与应用一致 |

`includes/` 与本 skill 共版本，只对当前 SDK 版本负责；SDK 升级时整体替换。

**先用 [adapter-skeleton.md](references/adapter-skeleton.md) 的空骨架把工程编译链接通过，再逐项换成平台实现。** 骨架含全部需要应用提供的函数，一个不漏。

实现参考（可选）：`https://github.com/baidu-maps/rtos-map-simulator` —— 一份桌面环境的可运行接入。逐文件对照关系见 [adapter-skeleton.md § 文件清单](references/adapter-skeleton.md)；其中 device / location / compass 是桩，不要照抄。契约以交付的 `common/` 为准。

## 编译

```sh
CXXFLAGS += -Iincludes $(shell find includes -type d | sed 's/^/-I/')
CXXFLAGS += -Icommon   $(shell find common   -type d | sed 's/^/-I/')
```

- **禁止**只加 `-Iincludes -Icommon` 而不加子目录。
- 公开 API 按模块前缀包含：

```cpp
#include "view/map_view_api.h"        // 地图
#include "base/base_api.h"            // 基础能力、坐标转换
#include "auth/auth_license_api.h"    // 鉴权与 License
#include "search/search_api.h"        // 检索
#include "navi/navi_api.h"            // 导航
#include "offline/map_offline_api.h"  // 离线
#include "mecp/include/rtos_mecp.h"   // 场景识别（C API，按需）
```

- 其余头（`VDPOINT`、`VColor`、`VImage`、`VSize`/`VRect`、`Location`、`Coordinate`、检索各 option 与 result 等）都由 `*_api.h` 传递包含，**不需要手动包含**。
- 类名与静态方法一律带命名空间：`MapViewApi` 在 `baidu::rtos_map::view`，`BaseApi` 在 `::base`，`AuthLicenseApi` 在 `::auth`，`NaviApi` 在 `::navi`，`MapOfflineApi` 在 `::offline`，`SearchApi` 在 `baidu_search`，`VImage` 在 `::image`。

## 输入坐标系（错了不报错）

- **必须**：喂给 SDK 的所有坐标是 **GCJ02**。设备 GNSS 通常是 WGS84，**必须先转换**再传入。
- **禁止**：把原始 WGS84 传给 `setCenterPoint` / `setMarkerPosition` / `setPolylinePoints` / `NaviApi::UpdateLocation` / 检索选项。不会报错，位置偏几百米。
- 坐标系不可配置。需要 SDK 以其它坐标系为基准时，联系 SDK 方。

```cpp
using namespace baidu::rtos_map;
using namespace baidu::rtos_map::view;   // MapViewApi 在 view 命名空间下
VDPOINT gcj = base::BaseApi::GetInstance()->ConvertCoord(
                  wgs84Point, coordtype::CoordType::WGS84, coordtype::CoordType::GCJ02);
MapViewApi::setCenterPoint(h, gcj);
// 等价写法：coordtype::convertCoord(pt, from, to)
```

- **必须**用三参形式。单参 `convertCoord(point)` 是恒等变换，什么都不做。
- **只有 4 个方向会真的转换**：`WGS84→GCJ02`、`WGS84→BD09LL`、`GCJ02→BD09LL`、`BD09LL→GCJ02`。其它组合（含任何回转到 WGS84、任何 BD09MC 方向）**原样返回入参，不报错**。
- 没有把 SDK 侧坐标转回 WGS84 的接口；应用自己需要 WGS84 时须自行实现。
- 坐标**序**（与坐标系无关）：地图侧 `VDPOINT(经度, 纬度)`，检索侧 `Coordinate(纬度, 经度)`。

## 全局硬规则

| 必须 | 禁止 |
|------|------|
| 在 `RequestLicense` 回调成功后才调 `MapViewApi::Create()` | 发起 `RequestLicense` 后立刻 `Create()` |
| 保存 `Create()` 的返回值并全程使用 | 假设 handle 等于 1 |
| `Create` 后依次 `SetCanvas` → `SetSize` → `SetUIThreadFunc` → `InitMap` | 漏 `SetSize`（画面会裁切/偏移）；`SetUIThreadFunc` 晚于 `InitMap` |
| 调任何依赖路径的能力（离线/路线文件/mecp）前先 `BaseApi::SetPackageName` | 不设包名就读写缓存 |
| `BNetwork_fetch` 的三个回调全部收敛到同一个固定线程，且与调 `MapViewApi`、绘制的线程一致 | 在 HTTP worker 线程直接回调进 SDK |
| `RequestRender` 与所有 Canvas 绘制在同一个固定线程 | 跨线程调 `MapViewApi`（内部无锁） |
| 角度类参数按**弧度**：`setRotationAngle` / `setMarkerAngle` / `NaviApi::UpdateCompass` | 传角度值 |
| `Location::setDirection` 按**度**，`setTime` 按**毫秒** | 与上一条混用单位 |
| 覆盖物设置属性后调 `updateLayer`，并确保渲染发生一次 | 依赖 `showOverlay` 让覆盖物可见（它不是必需项） |
| 折线显式设 `setPolylineLineWidth` 与 `setPolylineFillColor` | 不设线宽（未定义行为） |
| `updateLayer` / `addOverlay` 只传 `createLayer` 返回的 layerId | 传未创建的 layerId（会崩） |
| `TouchEvent::Type::TOUCH_MOVE` 的 `touches` 至少放 2 个元素 | MOVE 事件只放 1 个 touch（越界读内存） |

## 参考文档

- [init-auth.md](references/init-auth.md) — 初始化顺序、线程模型、联调验收清单、日志关键字
- [adapter-skeleton.md](references/adapter-skeleton.md) — **可编译的最小适配骨架**：6 个文件、全部待实现函数、自检清单、从骨架到可用的替换顺序
- [adapter-build.md](references/adapter-build.md) — 平台移植契约：`bd_map_*`、`*Impl`、平台 util、`BNetwork_*`、Canvas（71 项分类）、图片；`.a` 与目标环境匹配
- [overlay-map-control.md](references/overlay-map-control.md) — 地图状态、Marker、Polyline、触摸、图片、路线瓦片、多实例
- [search-navi-offline.md](references/search-navi-offline.md) — 检索、导航、离线地图
- [mecp.md](references/mecp.md) — 场景识别（独立模块）
- [demo.md](references/demo.md) — 可运行的全流程示例

## 现象速查

| 现象 | 动作 |
|------|------|
| `g++` 报 `../base/v_point.h: No such file` | `-I` 没加子目录 |
| Canvas 类报 `expected class-name` | 补命名空间 `baidu::rtos_map::view` |
| Canvas 类报 `invalid new-expression of abstract class` | 71 个纯虚函数补齐，最易漏 `setCanvas` |
| 链接缺 `bd_map_*` | 实现 `common/platform_adapter.h` 的 7 个 C 函数 |
| 链接同时报 `bd_map_*` duplicate symbol 与异平台前缀 undefined | 7 个 `bd_map_*` 还差一个，最常漏 `bd_map_log`/`bd_map_malloc`/`bd_map_free` |
| 链接缺 `BNetwork_fetch*` | 实现 `common/fetch/network_fetch.h` |
| 链接缺 `*_platform_util::*` | 实现对应 `*PlatformUtil.h` |
| 位置整体偏几百米（不报错） | 输入坐标先转成 GCJ02 |
| 地图白屏 | 日志有 `currentCanvas is null` → `SetCanvas` 没成功；否则查 License 回调、`InitMap`、`RequestRender` |
| 画面裁切 / 中心偏移 | 补 `SetSize` |
| `Create()` 返回 0 | 等 `RequestLicense` 回调成功后再 `Create` |
| 折线 / Marker 不可见 | 设线宽与颜色；调 `updateLayer`；确认渲染发生过一次 |
| 文字完全不显示 | `measureText` 必须返回真实高度 |
| 文字整体偏半个字高 | 对齐 `fillText` 锚点，见 [adapter-build.md § 5](references/adapter-build.md) |
| 每帧画面越跑越偏 | `clear()` 必须复位变换矩阵 |
| 该虚的线画成实线 / 实线带虚线相位 | 实现 `setLineDash`，空数组时复位为实线 |
| POI / 步行检索无结果 | 查 token、网络实现、坐标序（检索为 纬度, 经度）、坐标系 |
| 导航喂点但不推进 | 日志 `NaviEngine not initialization finished.` → 路线、`SetNaviType`（须在 `Init` 之后）、当前位置三项补齐 |
| 定位蓝点不动（导航正常） | 实现 `location_platform_util::subscribeLocation` 并真正推数据 |
| 导航方向乱转 | `UpdateCompass` 传弧度 |
| 离线列表为空 | 先 `RequestVersion(cb)` 并判返回值，回调成功后再取列表 |
| 缓存 / 离线 / 路线文件读写全失败 | 补 `BaseApi::SetPackageName` |
| 网络回调后偶发崩溃 / 数据错乱 | 三个回调收敛到同一固定线程 |
| 链接 `.a` 失败 / 出现异平台符号 | `.a` 与目标 toolchain 不匹配，见 [adapter-build.md § 7](references/adapter-build.md) |
