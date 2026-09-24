# 平台移植契约与 Canvas 适配

契约以交付的 `common/` 目录为准，作为 `-I` 根之一，**须连同其所有子目录一起加**。

集成新平台须提供以下全部实现：

1. C 平台契约（`common/platform_adapter.h`）—— 内存 / 日志 / task id。**缺任意一项链接失败**
2. C++ 适配类（`common/base/*Impl.h`）—— 线程 / 锁 / 信号量 / 事件 / 消息队列 / 定时器 / 时钟 / 文件。**缺必需项链接失败**
3. 平台 util 自由函数（`common/{base,device,compass,location}/*PlatformUtil.h`）。**缺任意一项链接失败**
4. 网络契约（`common/fetch/network_fetch.h`）—— `BNetwork_*`。**缺任意一项链接失败**
5. Canvas 契约（`common/canvas/common/CanvasContextCommonInterface.h`）。**不参与链接**：漏实现纯虚函数是编译期错误（`invalid new-expression of abstract class`），全部补齐后即使功能不对也能正常链接，缺陷表现为白屏/图形错乱
6. 图片契约（`common/image/ImageProviderCommonInterface.h`）。**不产生链接符号**，未注册表现为图片永久不加载，不是链接错误

**六项的可编译空骨架见 [adapter-skeleton.md](adapter-skeleton.md)**：先粘骨架、编译链接通过，再按本文各节的要求逐项换成真实实现。

另见 § 7：`.a` 与目标环境的匹配。

---

## 1. C 平台契约（`common/platform_adapter.h`）

骨架见 [adapter-skeleton.md § `platform_adapter_impl.c`](adapter-skeleton.md)。

**7 个函数必须全部提供实现**，函数体可以只有一行转调。

| 函数 | 实现要求 |
|------|---------|
| `void* bd_map_malloc(unsigned int size)` | 转调平台 `malloc` |
| `void* bd_map_realloc(void* ptr, unsigned int size)` | 转调平台 `realloc`。实测未见任何符号引用它，漏了不会报错，但仍建议实现以防后续 SDK 版本使用 |
| `void bd_map_free(void* ptr)` | 转调平台 `free` |
| `void* bd_map_cjson_malloc(unsigned int size)` | cJSON 专用分配，可用独立堆 |
| `void bd_map_cjson_free(void* ptr)` | **必须与 `bd_map_cjson_malloc` 配对**，不与 `bd_map_free` 混用 |
| `unsigned int bd_map_get_current_task_id()` | 返回任务/线程 ID；取不到就 `return 0` |
| `void bd_map_log(BDLogLevel, const char* file, int line, const char* fmt, ...)` | 转到平台日志。不实现将无法排障 |

`BDLogLevel`：`BD_LOG_NONE/FATAL/ERROR/WARN/INFO/DEBUG/VERBOSE`。

- **禁止**把 `bd_map_cjson_malloc/free` 与 `bd_map_malloc/free` 交叉使用：各自配对即可，两套可以用不同的堆。
- **禁止**指望这套接口接管 SDK 的全部内存：它只覆盖一部分分配。需要全局内存池或内存统计的平台，另外在链接期替换全局 `operator new/delete`。
- 落地：新建一个 `.c/.cpp` 实现这 7 个函数，加进构建即可，不改 `platform_adapter.h`。

> **只实现 6 个会同时报 duplicate symbol 与一批异平台前缀的 undefined**（如 `simulator_map_malloc`）。看到这个组合就是"还差一个 `bd_map_*` 没实现"，先补齐再判断别的。常漏的是 `bd_map_log`（被引用最多）、`bd_map_malloc`/`bd_map_free`，`bd_map_realloc` 实测无人引用，漏了不会有这个症状。7 个都实现后仍报异平台前缀符号未定义，才是 `.a` 与平台不匹配，见 § 7。

---

## 2. C++ 适配类（`common/base/*Impl.h`）

骨架见 [adapter-skeleton.md § `base_impl.cpp`](adapter-skeleton.md)。

每个 `*Impl` 继承 `common/base/common/` 下对应的纯虚接口（pImpl 模式，头文件已给出声明），应用提供 `.cpp` 实现。方法签名与语义以头文件为准。

| 契约类 | 抽象接口 | 是否必需 |
|------|------|------|
| `MapMutexImpl` | `MutexCommonInterface` | **必需** |
| `MapThreadImpl` | `ThreadCommomInterface`（SDK 拼写为 "Commom"） | **必需**，含 `start/stop/resume/detach/exit/delay/yield/id` |
| `MapClockImpl` | `ClockCommonInterface` | **必需** |
| `MapFileImpl` | `FileCommonInterface` | **必需** |
| `MapFileUtilImpl` | `FileUtilCommonInterface` | **必需** |
| `MapMsgQueueImpl` | `MsgQueueCommonInterface` | **必需** |
| `MapSemaphoreImpl` | `SemaphoreCommonInterface` | 当前版本用不到，**不需要写** |
| `MapEventImpl` | `EventCommonInterface` | 当前版本用不到，**不需要写** |
| `MapTimerImpl` | `TimerCommonInterface` | 当前版本用不到，**不需要写** |

- `MapMutexImpl` / `MapThreadImpl` / `MapFileImpl` / `MapMsgQueueImpl` 的头文件声明了嵌套 `class Impl;`，**必须给出定义**（`class MapMutexImpl::Impl {};` 这样一行）。`MapClockImpl` / `MapFileUtilImpl` 没有嵌套 `Impl`，**不要**为它们写这一行。
- `MapClockImpl::getMillisecond()` **必须定义在 `.cpp` 里**，写成 inline 或放头文件里会导致链接缺 `vtable for MapClockImpl`。

---

## 3. 平台 util 自由函数

骨架见 [adapter-skeleton.md § `platform_util_impl.cpp`](adapter-skeleton.md)。

| 头文件 | 必须提供的函数 |
|--------|------|
| `common/base/ThreadPlatformUtil.h` | `thread_platform_util::delay(uint32_t ms)`、`yield()` |
| `common/device/DevicePlatformUtil.h` | `device_platform_util::getDeviceInfo/getDeviceId/getUserId/getAdvertisingId/getSerial/getMac/getDeviceICCID/getTotalStorage/getAvailableStorage/getCpuInfo`，返回 `DEVICE_CALL_SUCC(0)` / `DEVICE_CALL_FAIL(-1)` |
| `common/compass/CompassPlatformUtil.h` | `compass_platform_util::requestCompass/subscribeCompass/unsubscribeCompass`，回调 `void(double direction, int accuracy)`，**`direction` 按弧度** |
| `common/location/LocationPlatformUtil.h` | `location_platform_util::requestLocation/subscribeLocation/unsubscribeLocation`，回调 `void(const LocationData&)` |

`LocationData`：`{double longitude, latitude; float accuracy; double altitude; float speed, heading; uint64_t timestamp;}`。

- `getDeviceId` **必须返回有效值**：License 请求会用它。
- **要在地图上显示定位点，就必须让 `subscribeLocation` / `subscribeCompass` 真正把数据推给 SDK**；只做导航不显示定位点时，可以先给空实现。
- 导航推进**不依赖** subscribe，应用调 `NaviApi::UpdateLocation` / `UpdateCompass` 喂点即可。

---

## 4. 网络契约（`common/fetch/network_fetch.h`）

骨架见 [adapter-skeleton.md § `network_fetch_impl.cpp`](adapter-skeleton.md)。三个函数按头文件原样声明，**禁止**加 `extern "C"`，也不能放进 `.c` 文件。

必须实现的三个函数：

```cpp
int BNetwork_fetchCreateSession(void** session);
int BNetwork_fetchDestroySession(void* session);
int BNetwork_fetch(void* session, BNetwork_FetchConfig*, BNetwork_FetchCallbacks, void* user_data);
```

请求与回调结构：

- `BNetwork_FetchConfig : BDMapHeapBase { const char* url, header, method, data; int timeout; size_t data_size; bool is_large_file; }`
- `BNetwork_FetchResponse { int http_code; uint8_t* header/data; size_t header_size/data_size; void* user_data; BNetwork_FetchConfig* config; bool is_file; const char* file_path; }`
- `BNetwork_FetchCallbacks { succeeded_cb; failed_cb; completed_cb; }`，签名分别为 `void(void* session, BNetwork_FetchResponse*)`、`void(void* session, int errCode, BNetwork_FetchResponse*)`、`void(void* session, int opCode, BNetwork_FetchResponse*)`

规则：

- **必须**异步执行，先回 `succeeded_cb` 或 `failed_cb`，最后回 `completed_cb`。
- **必须**把三个回调全部投递到**同一个固定线程**，且该线程与调用 `MapViewApi`、执行 Canvas 绘制的线程一致。
- **禁止**在 HTTP worker 线程上直接回调进 SDK。
- 不实现即鉴权失败、地图/检索/导航全不可用。

---

## 5. Canvas 适配层

71 个方法的空骨架见 [adapter-skeleton.md § `canvas_impl.h`](adapter-skeleton.md)（已按下面三张清单分好组，直接粘）。

提供一个实现 `CanvasContextCommonInterface` 的类。两条硬性前提：

```cpp
#include "CanvasContextCommonInterface.h"

class MyCanvasContext : public baidu::rtos_map::view::CanvasContextCommonInterface {
    // 71 个纯虚函数必须全部定义
};
```

- **命名空间是 `baidu::rtos_map::view`**，漏掉报 `expected class-name`。
- **71 个纯虚函数必须全部定义**，否则 `std::make_shared` 报 `invalid new-expression of abstract class`（编译期报错，不是链接期）。最易漏 `setCanvas(void*)`。

### 必须写真实逻辑（23 项）

| 方法 | 要求 |
|------|------|
| `clear()` | 清屏，**并把变换矩阵复位为单位矩阵、清空当前路径** |
| `save()` / `restore()` | 状态栈 |
| `rotate(angle)` / `rotate(angle, cx, cy)` | 两个重载都要实现，**单位是弧度** |
| `translate(x, y)` | |
| `setFillStyle` / `setStrokeStyle` / `setLineWidth` | |
| `setLineDash` | **收到空数组时必须复位为实线** |
| `beginPath` / `closePath` / `moveTo` / `lineTo` / `fill` / `stroke` | 须能跨多次 `beginPath`/`stroke` 保持样式 |
| `fillRect` | |
| `setFont` / `getFont` | **必须按 `"<字号>px <字体族>"` 格式 round-trip** |
| `fillText` | 锚点见下方"文字锚点" |
| `measureText` | **必须返回真实宽高**，返回 0 会导致所有文字不显示 |
| `drawImage(image, dx, dy, dw, dh)` | 5 形参的那个重载。`image` 就是你的 `ImageProvider` 返回的对象，两边类型约定必须一致 |
| `setCanvas(void*)` | **必须能安全接受 `nullptr`** |

### 按用到的特性决定（5 项）

| 方法 | 何时需要 |
|------|---------|
| `setTextAlign` / `setTextBaseline` | 默认构建不调用，可空实现 |
| `getImageData` / `scale` | 默认构建不可达，可空实现 |
| `isCanvasValid() const` | 建议按真实状态返回 |

### 可空实现（43 项）

`getFillStyle`、`getStrokeStyle`、`getLineWidth`、`setLineCap`/`getLineCap`、`setLineJoin`/`getLineJoin`、`setGlobalAlpha`/`getGlobalAlpha`、`getTextAlign`、`getTextBaseline`、`transform`、`setTransform`、`rect`、`strokeRect`、`clearRect`、`quadraticCurveTo`、`bezierCurveTo`、`arcTo`、`arc`、`clip`、`isPointInPath`、`strokeText`、`drawImage`（3 形参与 9 形参重载）、`createLinearGradient`、`createRadialGradient`、`createPattern`、`putImageData`（两个重载）、`setShadowOffsetX/Y`、`getShadowOffsetX/Y`、`setShadowBlur`/`getShadowBlur`、`setShadowColor`/`getShadowColor`、`setGlobalCompositeOperation`/`getGlobalCompositeOperation`、`drawCircle`、`drawPath`、`clipRect`。

23 + 5 + 43 = 71。另有一个非纯虚的 `SetDrawPath`，不实现也能编译。

> 分类对应当前 SDK 版本。升级 SDK 后重新核对。

### 文字锚点

SDK 不通过 `setTextAlign`/`setTextBaseline` 下发对齐方式，而是按固定锚点计算文字位置。**当前设定为左上**（部分平台构建为左下）。

- 目标平台 `fillText` 的原生锚点与此一致 → 直接实现。
- 不一致但平台锚点可调 → 在你的 `fillText` 实现里换算。
- 不一致且无法调整 → 联系 SDK 方按你的平台锚点提供构建。

### 绑定渲染目标

开始渲染前，实现类内部必须已持有可用的渲染目标。两种方式任选一种。

方式一：注入。

```cpp
auto canvas = std::make_shared<MyCanvasContext>();
canvas->setCanvas(platformCanvasContext);
MapViewApi::SetCanvas(h, canvas);
```

方式二：构造时绑定（`setCanvas` 仍须定义，函数体可空）。

```cpp
auto canvas = std::make_shared<MyCanvasContext>(/* 平台渲染适配器等 */);
MapViewApi::SetCanvas(h, canvas);
```

### 线程

`RequestRender` 与所有 Canvas 绘制**必须**在同一个固定线程。数据更新时 SDK 通过 `SetUIThreadFunc` 注册的回调通知需要重绘，应在该回调里把渲染请求 post 到渲染线程。

---

## 6. 图片契约（`common/image/ImageProviderCommonInterface.h`）

骨架见 [adapter-skeleton.md § `image_provider_impl.h`](adapter-skeleton.md)。

实现两个方法并用 `VImage::SetImageProvider(...)` 注册一次（进程级全局）：

```cpp
std::shared_ptr<void> CreateImageFromPath(const std::string& path, int width, int height) override;
std::shared_ptr<void> CreateImageFromData(const char* data, unsigned int dataSize, int width, int height) override;
```

- **必须在第一个 `VImage` 被创建之前注册**，实践上放在 `InitMap` 之前。未注册时图片永久处于未加载状态、不会重试。
- 返回的对象由 Canvas 侧 `drawImage` 消费，两边类型约定必须一致。

---

## 移植落地清单

| 契约 | 目标平台需要提供 |
|------|---------|
| `bd_map_*` | 内存（cJSON 一对单独配对）、日志、task id，7 个全实现 |
| `*Impl` | 锁 / 线程 / 时钟 / 文件 / 文件工具 / 消息队列六个必需，其余不需要写 |
| Canvas 实现类 | 目标 2D 后端，命名空间 `baidu::rtos_map::view`，71 项全定义 |
| `BNetwork_fetch*` | 目标 HTTP/TLS 栈，三个回调收敛到同一固定线程 |
| `compass`/`location_platform_util` | 显示定位点时必须真正推数据 |
| `ImageProviderCommonInterface` | 图片解码，早于第一个 `VImage` |
| 资源 | 字体文件按 Canvas 实现约定落位；缓存目录先 `SetPackageName` |
| 输入坐标系 | 非 GCJ02 时应用侧先转换 |

---

## 7. `.a` 与目标环境的匹配

### 7.1 先核实

| 确认项 | 命令 |
|--------|------|
| 格式与架构 | `file libmapsdk.a` |
| 符号 | ELF 用 `nm -C`；Mach-O 用 `llvm-nm` 或 `otool -hv`（GNU `nm` 读不了 Mach-O） |
| C++ ABI | 符号含 `std::__1::` 是 libc++，含 `std::__cxx11::` 是 libstdc++。**与应用不一致必须换库**，不能只看"链接过了" |

不匹配时向 SDK 方索取按你的 toolchain 构建的库，**禁止**打补丁绕过。

### 7.2 与构建绑定的参数

| 项 | 默认值 | 应用侧能否改 |
|----|--------|-----------|
| 缩放级别范围 | 10 ~ 19 | **不能**，需要其它范围时联系 SDK 方 |
| 数据根目录、缓存目录 | 与构建绑定 | 不能直接指定；完整缓存路径含 `SetPackageName` 设的包名那一段 |
| 视口与画布尺寸 | 600×600（部分平台 512×512） | **能**，运行期 `MapViewApi::SetSize` 覆盖 |

- 索取 `.a` 时须提供：目标数据/缓存目录、需要的缩放级别范围。
- `BaseApi` 对外只有 `SetPackageName` / `GetPackageName` / `GetAppCachePath` / `GetMapVersion` / `ConvertCoord`，**没有路径 setter**。未调 `SetPackageName` 时 `GetAppCachePath()` 返回空串，所有依赖它的读写都会失败。
- 字体文件不在交付物内，由 Canvas 实现决定从哪加载，**必须随包落位**。

---

## 排障

| 现象 | 动作 |
|------|------|
| 链接缺 `bd_map_*` | 实现 7 个 C 函数并加进构建 |
| 链接同时报 `bd_map_*` duplicate symbol 与异平台前缀 undefined | 还差一个 `bd_map_*`，最常漏 `bd_map_log`/`bd_map_malloc`/`bd_map_free` |
| 链接缺 `BNetwork_fetch*` | 实现网络契约 |
| 链接缺 `*_platform_util::*` | 实现对应 `*PlatformUtil.h` |
| 链接过了但 `std::string` 参数错乱 / 运行期诡异崩溃 | ABI 不一致，见 § 7.1 |
| 编译报 `expected class-name` | Canvas 类补命名空间 `baidu::rtos_map::view` |
| 编译报 `invalid new-expression of abstract class` | 按报错里 `pure within` 列的名字补齐，最易漏 `setCanvas` |
| 白屏，日志有 `currentCanvas is null` | `SetCanvas` 没成功 |
| 图标显示为灰块 | `drawImage` 与 `ImageProvider` 的对象类型约定不一致，或图片没加载成功 |
| 文字完全不显示 | `measureText` 返回真实高度 |
| 文字整体偏半个字高 | 对齐锚点，见 § 5 |
| 每帧画面越跑越偏 | `clear()` 复位变换矩阵 |
| 该虚的线画成实线 / 实线带虚线相位 | 实现 `setLineDash`，空数组复位实线 |
| 折线断续 | 检查跨多次 `beginPath`/`stroke` 是否保持样式 |
| 非渲染线程绘制崩溃 | 所有绘制与 `RequestRender` 收敛到同一线程 |
| 缓存 / 离线 / 路线文件读写全失败 | 补 `BaseApi::SetPackageName` |
| 屏幕尺寸不对 | 调 `SetSize` |
| 定位蓝点不动（导航正常） | `subscribeLocation` 真正推数据 |
