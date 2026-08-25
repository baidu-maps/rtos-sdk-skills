# 平台 Adapter 实现与 Canvas 适配

## 平台 Adapter 接口

SDK 通过一组 C 函数桩（`simulator_map_adapter.h`）和 C++ 抽象类（`MapThread/Mutex/...Impl`）将内部能力委托给平台实现。集成新平台时须提供以下全部实现。

### C 接口桩（`simulator_map_adapter.h`）

| 函数签名 | 说明 |
|---------|------|
| `void* simulator_map_malloc(unsigned int size)` | 内存分配 |
| `void* simulator_map_realloc(void* ptr, unsigned int size)` | 内存重分配 |
| `void  simulator_map_free(void* ptr)` | 内存释放 |
| `void* simulator_map_cjson_malloc(unsigned int size)` | cJSON 专用分配 |
| `void  simulator_map_cjson_free(void* ptr)` | cJSON 专用释放 |
| `unsigned int simulator_map_get_current_task_id()` | 返回当前任务/线程 ID |
| `long long simulator_map_getticks()` | 开机以来毫秒数（单调时钟） |
| `long long simulator_map_timestamps()` | Unix 毫秒时间戳 |
| `void simulator_map_log(level, file, line, fmt, ...)` | 错误/告警日志，输出到 stderr |
| `void simulator_map_debug(level, file, line, fmt, ...)` | 调试日志，输出到 stdout |

`LogLevelDef` 枚举：`LV_NONE/FATAL/ERROR/WARN/INFO/DEBUG`。

宏 `SIMULATOR_MAP_LOG` / `SIMULATOR_MAP_DEBUG` 自动填入 `__FILE__` 和 `__LINE__`，SDK 内部日志均通过这两个宏输出。

### C++ 适配类（`src/<platform>_adapter_impl/`）

每个类对应 SDK 内部一种能力，均须完整实现：

| 文件 | 适配能力 |
|------|---------|
| `MapThreadImpl.cpp` | 线程创建/启动/停止（`start/stop/detach/exit/delay/yield/id`） |
| `MapMutexImpl.cpp` | 互斥锁（`lock/unlock/trylock`） |
| `MapSemaphoreImpl.cpp` | 信号量（`take/give/reset`） |
| `MapEventImpl.cpp` | 事件标志组 |
| `MapMsgQueueImpl.cpp` | 消息队列（`send/receive`） |
| `MapTimerImpl.cpp` | 软件定时器（`start/stop/restart`） |
| `MapClockImpl.cpp` | 系统时钟（`getTickMs/getTimestampMs`） |
| `MapFileImpl.cpp` | 文件读写（`open/read/write/seek/close`） |
| `MapFileUtilImpl.cpp` | 文件/目录工具（`exists/mkdir/remove/listDir`） |
| `MapPlatformUtilImpl.cpp` | 平台杂项（获取 cache 路径、资源路径等） |

参考实现见模拟器 `src/adapter_impl/`（共用实现）、`src/adapter_impl/posix/posix_map_adapter.cpp`（POSIX/macOS C 接口桩入口）、`src/adapter_impl/windows/win_map_adapter.cpp`（Windows C 接口桩入口）。Canvas 适配参考 `src/adapter_impl/map_canvas_impl.cpp`，网络层参考 `src/adapter_impl/mapsdk_network_stub.cpp`。

---

## Canvas 适配层（`MapCanvasImpl`）

### 接口基类

`MapCanvasImpl` 须继承并实现 `CanvasContextCommonInterface`（`includes/` 中提供），该接口是仿 HTML5 Canvas 2D API 的纯虚类。

**必须实现的关键方法：**

| 方法 | 要求 |
|------|------|
| `setCanvas(void* ctx)` | 接收平台原生 canvas/渲染上下文指针并持久化；后续所有绘制通过此 ctx 操作平台 API |
| `drawImage(image, dx, dy [, dw, dh])` | 传入 PNG 文件路径；若平台有特定图像格式要求，平台方须提供对应图像格式头文件，`drawImage` 实现中按平台接口完成图像解码与绘制；图标路径形如 `getResPath() + "mapRes/xxx.png"` |
| `measureText(text, width, height)` | 必须返回真实宽高，不能返回 0；依赖字体文件，字体在首次调用时从 `getAppCachePath() + "/simhei.ttf"` 懒加载，确保 `simhei.ttf` 存在于该路径 |
| `setTextAlign` / `setTextBaseline` | 至少支持 `"center"` / `"middle"` |
| `stroke()` | 建议支持合理的 `lineJoin`/`lineCap`，否则折点密集处视觉断续 |
| `save()` / `restore()` | 须正确维护状态栈（transform/fill/stroke/alpha） |

完整接口列表见 `includes/adapter/platform/simulator/canvas/MapCanvasImpl.h`（约 40+ 个纯虚函数）。

### 绑定流程

`SetCanvas` 调用之前必须先完成平台 canvas context 绑定：

```cpp
auto canvas = std::make_shared<MapCanvasImpl>();
canvas->setCanvas(platformCanvasContext);   // 传入平台原生渲染上下文
MapViewApi::SetCanvas(h, canvas);
```

`platformCanvasContext` 由平台决定：
- macOS 模拟器：`SDL_Renderer*`（强转为 `void*` 传入，内部再 `static_cast` 回来）
- OHOS：`UICanvasExt*` 或等效 canvas 对象指针
- 其他平台：对应平台的 2D 渲染上下文指针

### 线程安全

`RequestRender` 与 Canvas 所有绘制调用必须在同一固定线程执行。SDK 不做内部线程调度，由应用层保证所有 Canvas 绘制和 `RequestRender` 调用在同一线程发起。可在 `MapCanvasImpl` 构造或 `setCanvas` 时记录线程 ID，在关键绘制入口断言线程，方便排查跨线程调用问题（参考 `canvas_detail::WarnIfNotMainThread`）。

### 排障

| 现象 | 检查 |
|------|------|
| 地图白屏但鉴权成功 | `setCanvas` 是否在 `SetCanvas` 前调用；`platformCanvasContext` 是否为有效指针 |
| 图标显示为灰块 | `drawImage` 是否实现了 PNG 解码 |
| 文字不显示或位置偏 | `measureText` 返回值是否为真实宽高；`textAlign`/`textBaseline` 是否生效 |
| 折线断续 | `lineJoin`/`lineCap` 未实现或近似精度不足 |
| 非主线程绘制崩溃 | 确认所有 Canvas 绘制和 `RequestRender` 始终在同一线程调用；`SetUIThreadFunc` 注册的回调须在该线程触发 |
