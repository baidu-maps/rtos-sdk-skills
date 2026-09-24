# 最小适配骨架

一套能直接编译通过的空实现骨架。按下表建文件、粘代码、编译通过后，再把标注"必须真实实现"的函数体逐个换成平台调用。

- 契约与签名以 `common/` 为准；本文件只是把 `common/` 里需要应用提供的东西**全部列齐**，避免漏项。
- 编译 flag 与 `-I` 根见 [SKILL.md § 编译](../SKILL.md)。骨架在 `-std=c++11` 下编译通过。
- 各契约的语义要求、单位、必须/禁止见 [adapter-build.md](adapter-build.md)。

## 文件清单

| 文件 | 内容 | 空实现能否跑起来 | 参考实现 |
|------|------|------|------|
| `platform_adapter_impl.c` | 7 个 `bd_map_*` | 内存和 task id 可空实现；**`bd_map_log` 建议立刻接真实日志**，否则无法排障 | `src/adapter_impl/posix/posix_map_adapter.cpp` |
| `base_impl.cpp` | 6 个 `*Impl` | **不能**。锁 / 线程 / 时钟 / 文件 / 消息队列必须真实实现 | `src/adapter_impl/Map{Mutex,Thread,Clock,File,FileUtil,MsgQueue}Impl.cpp` |
| `platform_util_impl.cpp` | 15 个自由函数 | 部分可以。`getDeviceId` 必须返回有效值；显示定位点还需 `subscribeLocation`/`subscribeCompass` 真实推数据 | `src/adapter_impl/MapPlatformUtilImpl.cpp`（其中 device / location / compass 是桩，**不要照抄**） |
| `network_fetch_impl.cpp` | 3 个 `BNetwork_*` | **不能**。空实现即鉴权失败、地图/检索/导航全不可用 | `src/adapter_impl/mapsdk_network_stub.cpp` |
| `canvas_impl.h` | Canvas 71 个纯虚 | **不能**。23 项必须真实实现 | `includes/adapter/platform/simulator/canvas/MapCanvasImpl.h` + `src/adapter_impl/map_canvas_impl.cpp` |
| `image_provider_impl.h` | ImageProvider 2 个方法 | 可以。返回 `nullptr` 则图片不显示，地图其余部分正常 | `src/image_provider_impl.h` / `.cpp` |

参考实现路径相对 `https://github.com/baidu-maps/rtos-map-simulator` 仓库根。它是桌面环境的一份可运行接入，**只看适配层的结构与调用顺序**；它的 `includes/` 布局与交付的 `includes/` 不同，不要参照。

`MapSemaphoreImpl` / `MapEventImpl` / `MapTimerImpl` 当前版本用不到，**不需要写**，本骨架不含它们。

---

## 1. `platform_adapter_impl.c`

可以用 `.c` 编译（`common/platform_adapter.h` 已带 `extern "C"`）。

```c
#include "platform_adapter.h"
#include <stdlib.h>

void* bd_map_malloc(unsigned int size)              { return malloc(size); }
void* bd_map_realloc(void* ptr, unsigned int size)  { return realloc(ptr, size); }
void  bd_map_free(void* ptr)                        { free(ptr); }
void* bd_map_cjson_malloc(unsigned int size)        { return malloc(size); }
void  bd_map_cjson_free(void* ptr)                  { free(ptr); }
unsigned int bd_map_get_current_task_id(void)       { return 0; }

/* 必须真实实现：转到平台日志，并放开 DEBUG 级别 */
void bd_map_log(BDLogLevel level, const char* file, int line, const char* fmt, ...) {
    (void)level; (void)file; (void)line; (void)fmt;
}
```

---

## 2. `base_impl.cpp`

`MapMutexImpl` / `MapThreadImpl` / `MapFileImpl` / `MapMsgQueueImpl` 的头文件里声明了嵌套的 `class Impl;`，**必须给出定义**（下面的 `class XxxImpl::Impl {};` 那几行），否则报 `invalid application of 'sizeof' to incomplete type`。`MapClockImpl` 与 `MapFileUtilImpl` 没有嵌套 `Impl`，**不要**为它们写这一行。

`MapClockImpl::getMillisecond()` **必须定义在 `.cpp` 里**，不能写成 inline 或放在头文件里，否则链接报缺 `vtable for MapClockImpl`。

```cpp
#include "MapMutexImpl.h"
#include "MapThreadImpl.h"
#include "MapClockImpl.h"
#include "MapFileImpl.h"
#include "MapFileUtilImpl.h"
#include "MapMsgQueueImpl.h"

// ---------------- 锁：必须真实实现 ----------------
class MapMutexImpl::Impl {};
MapMutexImpl::MapMutexImpl(const std::string& name, uint8_t flag) : pImpl(new Impl()) {
    (void)name; (void)flag;
}
MapMutexImpl::~MapMutexImpl() {}
bool MapMutexImpl::take(int timeout) { (void)timeout; return true; }   // timeout 单位见头文件
bool MapMutexImpl::try_take() { return true; }
bool MapMutexImpl::release() { return true; }

// ---------------- 线程：必须真实实现 ----------------
class MapThreadImpl::Impl {};
MapThreadImpl::MapThreadImpl(const std::string& name, std::function<void(void*)> func,
                             void* arg, uint32_t stack_size, int8_t priority)
    : pImpl(new Impl()) {
    (void)name; (void)func; (void)arg; (void)stack_size; (void)priority;
}
MapThreadImpl::~MapThreadImpl() {}
bool MapThreadImpl::start()  { return true; }
bool MapThreadImpl::stop()   { return true; }
bool MapThreadImpl::resume() { return true; }
bool MapThreadImpl::detach() { return true; }
bool MapThreadImpl::exit()   { return true; }
void MapThreadImpl::delay(uint32_t ms) { (void)ms; }
void MapThreadImpl::yield()  {}
uint64_t MapThreadImpl::id() { return 0; }

// ---------------- 时钟：必须真实实现，且必须在 .cpp 里 ----------------
uint64_t MapClockImpl::getMillisecond() { return 0; }
```

同一个 `base_impl.cpp` 续：

```cpp
// ---------------- 文件：必须真实实现 ----------------
class MapFileImpl::Impl {};
MapFileImpl::MapFileImpl(const std::string& file_name) : pImpl(new Impl()) { (void)file_name; }
MapFileImpl::~MapFileImpl() {}
bool MapFileImpl::open(FileMode mode) { (void)mode; return false; }
bool MapFileImpl::is_opened() const { return false; }
bool MapFileImpl::close() { return true; }
int  MapFileImpl::read(void* buf, int count)        { (void)buf; (void)count; return -1; }
int  MapFileImpl::write(const void* buf, int count) { (void)buf; (void)count; return -1; }
int  MapFileImpl::get_size() const     { return -1; }
int  MapFileImpl::get_position() const { return -1; }
int  MapFileImpl::seek(int offset, int whence) { (void)offset; (void)whence; return -1; }
int  MapFileImpl::seek_to_begin() { return -1; }
int  MapFileImpl::seek_to_end()   { return -1; }
bool MapFileImpl::exists() const  { return false; }
bool MapFileImpl::remove()        { return false; }
bool MapFileImpl::rename_to(const std::string& newName) { (void)newName; return false; }

// ---------------- 文件工具：必须真实实现，无嵌套 Impl ----------------
MapFileUtilImpl::MapFileUtilImpl() {}
MapFileUtilImpl::~MapFileUtilImpl() {}
bool MapFileUtilImpl::rename(const std::string& old_name, const std::string& new_name) {
    (void)old_name; (void)new_name; return false;
}
bool MapFileUtilImpl::make_dir(const std::string& dirName) { (void)dirName; return false; }
bool MapFileUtilImpl::is_directory_exist(const std::string& dir_name) { (void)dir_name; return false; }
bool MapFileUtilImpl::is_file_exist(const std::string& file_name) { (void)file_name; return false; }
int  MapFileUtilImpl::get_free_storage_space() { return 0; }

// ---------------- 消息队列：必须真实实现 ----------------
class MapMsgQueueImpl::Impl {};
MapMsgQueueImpl::MapMsgQueueImpl(const std::string& name, uint32_t msg_size,
                                 uint32_t max_msgs, uint8_t flag) : pImpl(new Impl()) {
    (void)name; (void)msg_size; (void)max_msgs; (void)flag;
}
MapMsgQueueImpl::~MapMsgQueueImpl() {}
bool MapMsgQueueImpl::send(const void* buffer, uint32_t size) { (void)buffer; (void)size; return false; }
bool MapMsgQueueImpl::send_wait(const void* buffer, uint32_t size, int timeout) {
    (void)buffer; (void)size; (void)timeout; return false;
}
bool MapMsgQueueImpl::urgent(const void* buffer, uint32_t size) { (void)buffer; (void)size; return false; }
bool MapMsgQueueImpl::recv(void* buffer, uint32_t size, int timeout) {
    (void)buffer; (void)size; (void)timeout; return false;
}
int  MapMsgQueueImpl::count() const { return 0; }
```

---

## 3. `platform_util_impl.cpp`

```cpp
#include "ThreadPlatformUtil.h"
#include "DevicePlatformUtil.h"
#include "CompassPlatformUtil.h"
#include "LocationPlatformUtil.h"

namespace thread_platform_util {
void delay(uint32_t ms) { (void)ms; }        // 必须真实实现
void yield() {}                              // 必须真实实现
}

namespace device_platform_util {
// getDeviceId 必须返回稳定且唯一的值；其余可先留空
int getDeviceId(std::string& deviceId)     { deviceId = "PUT-A-STABLE-UNIQUE-ID-HERE"; return DEVICE_CALL_SUCC; }
int getDeviceInfo(CVDeviceInfo& info)      { (void)info; return DEVICE_CALL_SUCC; }
int getUserId(std::string& v)              { v.clear(); return DEVICE_CALL_SUCC; }
int getAdvertisingId(std::string& v)       { v.clear(); return DEVICE_CALL_SUCC; }
int getSerial(std::string& v)              { v.clear(); return DEVICE_CALL_SUCC; }
int getMac(std::string& v)                 { v.clear(); return DEVICE_CALL_SUCC; }
int getDeviceICCID(std::string& v)         { v.clear(); return DEVICE_CALL_SUCC; }
int getTotalStorage(int* v)                { if (v) { *v = 0; } return DEVICE_CALL_SUCC; }
int getAvailableStorage(int* v)            { if (v) { *v = 0; } return DEVICE_CALL_SUCC; }
int getCpuInfo(std::string& v)             { v.clear(); return DEVICE_CALL_SUCC; }
}

namespace compass_platform_util {
// 要显示定位点方向就必须真实实现：回调 direction 按弧度
int subscribeCompass(CompassDataCallback callback, void* userData) {
    (void)callback; (void)userData; return COMPASS_CALL_SUCC;
}
int unsubscribeCompass(CompassDataCallback callback) { (void)callback; return COMPASS_CALL_SUCC; }
int requestCompass(CompassDataCallback callback)     { (void)callback; return COMPASS_CALL_SUCC; }
}

namespace location_platform_util {
// 要显示定位点就必须真实实现：把 LocationData 真正推给回调，坐标须为 GCJ02
int subscribeLocation(LocationDataCallback callback, void* userData) {
    (void)callback; (void)userData; return LOCATION_CALL_SUCC;
}
int unsubscribeLocation(LocationDataCallback callback) { (void)callback; return LOCATION_CALL_SUCC; }
int requestLocation(LocationDataCallback callback)     { (void)callback; return LOCATION_CALL_SUCC; }
}
```

---

## 4. `network_fetch_impl.cpp`

三个函数按 `common/fetch/network_fetch.h` 原样声明，**不要加 `extern "C"`**，也不能放进 `.c` 文件。真实实现的三条硬性要求（异步、三个回调收敛到同一固定线程、该线程与调 `MapViewApi` 及绘制的线程一致）见 [adapter-build.md § 4](adapter-build.md)。

```cpp
#include "network_fetch.h"

int BNetwork_fetchCreateSession(void** session) {
    if (!session) { return -1; }
    *session = nullptr;
    return 0;
}

int BNetwork_fetchDestroySession(void* session) {
    (void)session;
    return 0;
}

// 必须真实实现：发起异步请求，先回 succeeded_cb 或 failed_cb，最后回 completed_cb
int BNetwork_fetch(void* session, BNetwork_FetchConfig* config,
                   BNetwork_FetchCallbacks callbacks, void* user_data) {
    (void)session; (void)config; (void)callbacks; (void)user_data;
    return 0;
}
```

---

## 5. `image_provider_impl.h`

```cpp
#pragma once
#include "ImageProviderCommonInterface.h"

class MyImageProvider : public baidu::rtos_map::image::ImageProviderCommonInterface {
public:
    // 返回的对象由 Canvas 侧 drawImage 消费，两边必须约定同一具体类型
    std::shared_ptr<void> CreateImageFromPath(const std::string& path,
                                              int width, int height) override {
        (void)path; (void)width; (void)height;
        return nullptr;
    }
    std::shared_ptr<void> CreateImageFromData(const char* data, unsigned int dataSize,
                                              int width, int height) override {
        (void)data; (void)dataSize; (void)width; (void)height;
        return nullptr;
    }
};
```

注册一次，必须早于第一个 `VImage` 被创建，实践上放在 `InitMap` 之前：

```cpp
#include "base/v_image.h"
baidu::rtos_map::image::VImage::SetImageProvider(std::make_shared<MyImageProvider>());
```

---

## 6. `canvas_impl.h`

71 个纯虚函数全部在此，一个都不能少。分组与各项的具体要求（`clear` 必须复位变换矩阵、`setLineDash` 空数组必须复位实线、`setFont`/`getFont` 必须 round-trip、`measureText` 必须返回真实宽高、`setCanvas` 必须能接受 `nullptr`、文字锚点）见 [adapter-build.md § 5](adapter-build.md)。

```cpp
#pragma once
#include "CanvasContextCommonInterface.h"
#include <string>
#include <vector>
#include <memory>

class MyCanvasContext : public baidu::rtos_map::view::CanvasContextCommonInterface {
public:
    // ---- 必须真实实现（23）：把函数体换成平台绘图调用 ----
    void setFillStyle(const std::string& style) override {}
    void setStrokeStyle(const std::string& style) override {}
    void setLineWidth(double width) override {}
    void setLineDash(const std::vector<double>& segments) override {}
    void setFont(const std::string& font) override {}
    std::string getFont() const override { return {}; }
    void save() override {}
    void restore() override {}
    void rotate(double angle) override {}
    void rotate(double angle, double centerX, double centerY) override {}
    void translate(double x, double y) override {}
    void fillRect(double x, double y, double width, double height) override {}
    void beginPath() override {}
    void closePath() override {}
    void fill() override {}
    void stroke() override {}
    void moveTo(double x, double y) override {}
    void lineTo(double x, double y) override {}
    void fillText(const std::string& text, double x, double y, double maxWidth = -1) override {}
    bool measureText(const std::string& text, double& width, double& height) override { return true; }
    void drawImage(const std::shared_ptr<void>& image,
                   double dx, double dy, double dw, double dh) override {}
    void setCanvas(void* canvas) override {}
    void clear() override {}

    // ---- 按用到的特性决定（5）：默认可空 ----
    void setTextAlign(const std::string& align) override {}
    void setTextBaseline(const std::string& baseline) override {}
    void scale(double x, double y) override {}
    std::shared_ptr<void> getImageData(double sx, double sy, double sw, double sh) override { return {}; }
    bool isCanvasValid() const override { return true; }
```

同一个类续（可空实现 43 项，保持空即可）：

```cpp
    std::string getFillStyle() const override { return {}; }
    std::string getStrokeStyle() const override { return {}; }
    double getLineWidth() const override { return 0; }
    void setLineCap(const std::string& cap) override {}
    std::string getLineCap() const override { return {}; }
    void setLineJoin(const std::string& join) override {}
    std::string getLineJoin() const override { return {}; }
    void setGlobalAlpha(double alpha) override {}
    double getGlobalAlpha() const override { return 0; }
    std::string getTextAlign() const override { return {}; }
    std::string getTextBaseline() const override { return {}; }
    void transform(double a, double b, double c, double d, double e, double f) override {}
    void setTransform(double a, double b, double c, double d, double e, double f) override {}
    void rect(double x, double y, double width, double height) override {}
    void strokeRect(double x, double y, double width, double height) override {}
    void clearRect(double x, double y, double width, double height) override {}
    void quadraticCurveTo(double cpx, double cpy, double x, double y) override {}
    void bezierCurveTo(double cp1x, double cp1y, double cp2x, double cp2y,
                       double x, double y) override {}
    void arcTo(double x1, double y1, double x2, double y2, double radius) override {}
    void arc(double x, double y, double radius, double startAngle, double endAngle,
             bool anticlockwise = false) override {}
    void clip() override {}
    bool isPointInPath(double x, double y) override { return true; }
    void strokeText(const std::string& text, double x, double y, double maxWidth = -1) override {}
    void drawImage(const std::shared_ptr<void>& image, double dx, double dy) override {}
    void drawImage(const std::shared_ptr<void>& image, double sx, double sy, double sw, double sh,
                   double dx, double dy, double dw, double dh) override {}
    std::shared_ptr<void> createLinearGradient(double x0, double y0, double x1, double y1) override { return {}; }
    std::shared_ptr<void> createRadialGradient(double x0, double y0, double r0,
                                               double x1, double y1, double r1) override { return {}; }
    std::shared_ptr<void> createPattern(const std::shared_ptr<void>& image,
                                        const std::string& repetition) override { return {}; }
    void putImageData(const std::shared_ptr<void>& imageData, double dx, double dy) override {}
    void putImageData(const std::shared_ptr<void>& imageData, double dx, double dy,
                      double dirtyX, double dirtyY, double dirtyWidth, double dirtyHeight) override {}
    void setShadowOffsetX(double offsetX) override {}
    double getShadowOffsetX() const override { return 0; }
    void setShadowOffsetY(double offsetY) override {}
    double getShadowOffsetY() const override { return 0; }
    void setShadowBlur(double blur) override {}
    double getShadowBlur() const override { return 0; }
    void setShadowColor(const std::string& color) override {}
    std::string getShadowColor() const override { return {}; }
    void setGlobalCompositeOperation(const std::string& operation) override {}
    std::string getGlobalCompositeOperation() const override { return {}; }
    void drawCircle(double x, double y, double radius, void* style) override {}
    void drawPath(void* path, void* style) override {}
    void clipRect(double x, double y, double width, double height) override {}
};
```

23 + 5 + 43 = 71。`SetDrawPath` 不是纯虚，不实现也能编译。

接进地图：

```cpp
auto canvas = std::make_shared<MyCanvasContext>();
canvas->setCanvas(platformCanvasContext);          // 也可在构造函数里绑定
baidu::rtos_map::view::MapViewApi::SetCanvas(h, canvas);
```

---

## 骨架自检

按顺序确认，每一条都过了再往下接业务代码：

| # | 检查项 | 不过时的报错 |
|---|--------|------------|
| 1 | 六个骨架文件都编译通过 | `../base/v_point.h: No such file` → `-I` 没加子目录 |
| 2 | `expected class-name` 没有出现 | Canvas 类漏了命名空间 `baidu::rtos_map::view` |
| 3 | `std::make_shared<MyCanvasContext>()` 编译通过 | `invalid new-expression of abstract class` → 按报错里 `pure within` 列的名字补齐 |
| 4 | `class MapXxxImpl::Impl {};` 只给了 Mutex / Thread / File / MsgQueue 四个 | `qualified name does not name a class`（多写了）或 `sizeof to incomplete type`（少写了） |
| 5 | `MapClockImpl::getMillisecond()` 在 `.cpp` 里 | 链接缺 `vtable for MapClockImpl` |
| 6 | 7 个 `bd_map_*` 一个不少 | 链接同时报 `bd_map_*` duplicate symbol 与一批异平台前缀 undefined |
| 7 | `BNetwork_*` 三个都在，且没加 `extern "C"` | 链接缺 `BNetwork_fetch*` |
| 8 | 15 个 `*_platform_util` 自由函数一个不少 | 链接缺 `*_platform_util::*` |
| 9 | 链接通过后能跑到 `RequestLicense` 回调 | 见 [init-auth.md § 首轮联调验收清单](init-auth.md) |

## 从骨架到可用

空实现能链接通过、能跑起来，但屏幕上什么都没有。按下表逐项换成真实实现，每换一项都能看到明确变化：

| 顺序 | 换哪个 | 换完能看到 |
|------|--------|-----------|
| 1 | `bd_map_log` | 日志开始输出，后面所有排障都靠它 |
| 2 | `MapClockImpl` / `MapMutexImpl` / `MapThreadImpl` / `MapMsgQueueImpl` | SDK 内部调度能跑 |
| 3 | `MapFileImpl` / `MapFileUtilImpl` | 缓存能落盘 |
| 4 | `BNetwork_fetch*` | `RequestLicense` / `Authenticate` 回调开始成功 |
| 5 | `device_platform_util::getDeviceId` | License 请求不再被拒 |
| 6 | Canvas 的 23 项 | 底图、矢量线、POI 文字与图标出现 |
| 7 | `MyImageProvider` 两个方法 | Marker 图标不再是灰块 |
| 8 | `location_platform_util::subscribeLocation` / `compass_platform_util::subscribeCompass` | 地图上的定位点出现并跟随移动 |







