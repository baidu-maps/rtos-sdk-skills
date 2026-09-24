# 初始化顺序与线程模型

## 调用顺序

```
BaseApi::SetPackageName → AuthLicenseApi::SetAk → RequestLicense(cb)
cb 内确认 LicenseErrorCode::NO_ERROR 后：Authenticate(cb) → h = MapViewApi::Create() → SetCanvas(h, canvas) → SetSize(h, w, ht) → SetUIThreadFunc(h, func) → InitMap(h) → RequestRender(h) → 业务 → Destroy(h)
```

完整代码见 [demo.md § main 初始化全流程](demo.md)；多实例边界见 [overlay-map-control.md § 多实例地图](overlay-map-control.md)。

## 硬规则

| 必须 | 禁止 |
|------|------|
| 等 `RequestLicense` 回调成功后才 `Create()` | 发起 `RequestLicense` 后立刻 `Create()`（返回 0） |
| 回调代码按"可能在 `RequestLicense` 返回前就执行"来写 | 依赖"`RequestLicense` 之后那几行已经跑完" |
| license 请求只在一处发起 | 多处并发发起（重复调用会顶掉上一个回调） |
| 保存 `Create()` 返回值全程使用（有效 = 非 `kInvalidMapViewHandle`(0)） | 假设 handle 等于 `kDefaultMapViewHandle`(1) |
| `SetCanvas` 早于任何 `RequestRender` | 未设 Canvas 就渲染（白屏） |
| `SetSize`、`SetUIThreadFunc` 早于 `InitMap` | 漏 `SetSize`（按默认尺寸渲染，画面裁切/中心偏移）；`SetUIThreadFunc` 晚于 `InitMap`（丢首帧刷新通知） |
| `Authenticate` 与 `RequestLicense` 无先后依赖，首次接入按串行写 | 两条请求并行发起（失败时分不清是许可问题还是 token 问题） |
| 分开处理两种失败：token 失败 = 有地图但网络请求全失败；License 失败 = 拿不到 handle | 混为一谈 |

## 线程模型

1. `BNetwork_fetch` 的三个回调**必须**全部收敛到同一个固定线程，且与调用 `MapViewApi`、执行 Canvas 绘制的线程一致；marshal 必须在回调进入 SDK 之前完成。满足后，检索/离线/鉴权回调里可直接调 `MapViewApi`。**禁止**在 HTTP worker 线程直接回调进 SDK。
2. `RequestRender` 与所有 Canvas 绘制**必须**在同一固定渲染线程。`SetUIThreadFunc` 注册的回调在事件发生的那个线程上就地触发，多线程模型下应在该回调里把渲染请求 post 到渲染线程，不要就地 `RequestRender`。**禁止**跨线程调 `MapViewApi`（API 层非线程安全）。

## 首轮联调验收清单

1. `RequestLicense` 回调返回 `LicenseErrorCode::NO_ERROR`，`Authenticate` 回调返回 `MapAuthErrorCode::OK`
2. `Create()` 返回非 0 handle，且已存入变量供后续所有调用使用
3. `SetCanvas` → `SetSize` → `SetUIThreadFunc` → `InitMap` 全部调过，渲染线程收到刷新通知并调用 `RequestRender(h)`
4. 可见底图背景、矢量线、POI 文本/图标；拖拽时中心点变化与瓦片请求方向一致
5. 定位点落在预期位置；偏几百米时先把输入坐标转成 GCJ02，见 [SKILL.md § 输入坐标系](../SKILL.md)

## 日志关键字

需平台实现 `bd_map_log` 并放开 DEBUG 级别，见 [adapter-build.md § 1](adapter-build.md)。

| 现象 | 关键字 |
|------|--------|
| 白屏第一排查项 | `currentCanvas is null, please call setCanvas first` → `SetCanvas` 没成功 |
| 手势坐标是否连续 | `OnTouchDown` / `OnTouchMove` / `OnTouchUp` |
| Marker 图片未创建成功 | `markerData or image is null` |
| 瓦片来源 | `_state_cache req =`（缓存）、`notify_data _state_offline req:`（离线包）、`_state_remote req =`（远程） |
| 导航喂点但不推进 | `NaviEngine not initialization finished.` → 见 [search-navi-offline.md § 导航](search-navi-offline.md) |
