# 鉴权、初始化与 Canvas 适配

## 初始化顺序

```
BaseApi::SetPackageName
→ AuthLicenseApi::SetAk → RequestLicense → Authenticate（异步）
→ MapViewHandle h = MapViewApi::Create()
→ SetCanvas(h, ...)（先调 MapCanvasImpl::setCanvas(platformCanvasContext)）
→ SetUIThreadFunc(h, ...)
→ InitMap(h)
→ RequestRender(h)
→ ... 业务逻辑 ...
→ Destroy(h)
```

多实例：每个地图实例独立执行以上全部步骤（各自的 `h`），互不共享地图状态；进程级全局状态（如 `VImage::SetImageProvider`）只需设置一次。

完整可运行代码见 [demo.md § main 初始化全流程](demo.md)。

---

## Canvas 适配层要点

Canvas 接口实现要求与绑定流程见 [adapter-build.md](adapter-build.md)。

`SetUIThreadFunc`：注册 SDK 数据刷新回调，SDK 内部渲染数据更新时触发，应用在回调里调用 `RequestRender` 拉取新数据。

**线程要求：**
- `RequestRender` 必须始终在同一固定线程调用（通常为应用的渲染/主线程），SDK 不做内部线程调度，由应用层保证。
- 其他 SDK 接口（含网络/搜索回调）应在应用主线程调用；若回调在非主线程触发（如 HTTP 线程），须在调用 SDK 接口前切回主线程（mapAPP 中用 `EnqueueMainThreadMapWork`）。

**`SetUIThreadFunc` 使用方式：**
SDK 内部数据有更新时会调用此回调通知应用层，应用可在收到通知后的合适时机调用 `RequestRender` 拉取最新数据。应用也可以不依赖此回调，自行在合适时机主动调用 `RequestRender`。两种方式均可，`RequestRender` 须始终在同一固定线程调用。

---

## 首轮联调验收清单

1. `Authenticate` 成功
2. `MapViewApi::Create()` 拿到有效 `h`
3. `SetCanvas(h, ...)` + `SetSize(h, ...)`
4. `SetUIThreadFunc(h, ...)`
5. `InitMap(h)`
6. UI 线程收到渲染请求并调用 `RequestRender(h)`
7. 可见底图背景、矢量线、POI 文本/图标
8. 拖拽时中心点变化与瓦片请求方向一致

---

## 日志排障关键字

| 场景 | 关键字 |
|------|--------|
| 渲染 | `BMap::RequestRender start/end`、`context.renderDataList size` |
| 手势 | `OnTouchDown/OnTouchMove/OnTouchUp`（坐标是否连续） |
| 绘制 | `drawImage begin`、图层 `paint` 统计 |
| 数据 | 瓦片请求命中（cache/local/remote）日志 |
