# 地图状态、覆盖物与触摸

完整可运行示例见 [demo.md § RunMapStateAndOverlayDemo](demo.md) 和 [demo.md § RunTouchDemo](demo.md)。

## 地图状态与坐标

```cpp
MapViewApi::setCenterPoint(h, VDPOINT(lng, lat));  // VDPOINT 为经度/纬度顺序
MapViewApi::setZoom(h, level);
MapViewApi::zoomIn(h); MapViewApi::zoomOut(h);
MapViewApi::setRotationAngle(h, degrees);
MapViewApi::setViewBound(h, rect);
MapViewApi::LatLngToScreenPixel(h, center);
MapViewApi::ScreenPixelToLatLng(h, pixel);
```

## 触摸事件

构造 `TouchEvent`（类型：`TOUCH_START` / `TOUCH_MOVE` / `TOUCH_END`），填充 `touches` / `changedTouches` 后：

```cpp
MapViewApi::OnTouchDown(h, event);
MapViewApi::OnTouchMove(h, event);
MapViewApi::OnTouchUp(h, event);
```

## Overlay 层与 Marker

```cpp
int layerId  = MapViewApi::createLayer(h, LayerType::OVERLAYER, "layer_name");
int markerId = MapViewApi::CreateMarker(h);
MapViewApi::setMarkerPosition(h, markerId, VDPOINT(lng, lat));
MapViewApi::setMarkerSize(h, markerId, VSize(w, h));
MapViewApi::setMarkerOffset(h, markerId, VSize(ox, oy));
MapViewApi::setMarkerIcon(h, markerId, resPath);   // 须为可访问绝对路径
MapViewApi::addOverlay(h, layerId, markerId);
MapViewApi::showOverlay(h, markerId);
MapViewApi::updateLayer(h, layerId);
MapViewApi::RequestRender(h);
```

> `drawMarker: pixel=(0,0)` 日志是 marker 与地图中心重合的正常现象。

## 图片适配（VImage / ImageProviderCommonInterface）

`VImage`（`src/base/v_image.h`）用于 marker 图标（`setMarkerVImage`）和瓦片图层的图片加载。图片的实际解码/创建逻辑由 `ImageProviderCommonInterface`（`src/adapter/common/image/ImageProviderCommonInterface.h`）实现类提供：

内置四个平台（honor/oppo/huawei/xiaoniu）已由 SDK 在 `LayerMgr` 构造时按平台宏自动注册默认实现，应用侧无需额外调用；simulator 及未内置的新平台需应用侧实现 `ImageProviderCommonInterface`，在地图初始化（`MapViewApi::InitMap` 之前）调用一次：

```cpp
#include "v_image.h"

class MyImageProvider : public baidu::rtos_map::image::ImageProviderCommonInterface {
public:
    std::shared_ptr<void> CreateImageFromPath(const std::string& path, int width, int height) override {
        // 实现平台图片解码逻辑，返回类型抹平后的 shared_ptr<void>
    }
    std::shared_ptr<void> CreateImageFromData(const char* data, unsigned int dataSize, int width, int height) override {
        // 实现平台图片解码逻辑
    }
};

baidu::rtos_map::image::VImage::SetImageProvider(std::make_shared<MyImageProvider>());
```

注意：`VImage::SetImageProvider` 是进程级全局设置，只需调用一次；返回的 `shared_ptr<void>` 最终会被 Canvas 侧 `drawImage` 实现 `static_cast` 回具体类型消费，需与对应平台 `CanvasContextCommonInterface` 实现约定好类型。

```cpp
auto markerImage = std::make_shared<baidu::rtos_map::image::VImage>("marker_icon.png", 36, 36);
MapViewApi::setMarkerVImage(h, markerId, markerImage);
```

## 折线（点集）

```cpp
int polylineId = MapViewApi::CreatePolyline(h);
MapViewApi::setPolylinePoints(h, polylineId, points);
MapViewApi::setPolylineLineWidth(h, polylineId, 8);
MapViewApi::setPolylineFillColor(h, polylineId, VColor(0, 153, 255, 220));
MapViewApi::setPolylineStrokeColor(h, polylineId, color);   // 可选
MapViewApi::setPolylineStrokeWidth(h, polylineId, 2);        // 可选
MapViewApi::addOverlay(h, layerId, polylineId);
MapViewApi::showOverlay(h, polylineId);
MapViewApi::updateLayer(h, layerId);
MapViewApi::setViewBound(h, MapViewApi::getOverlayBounds(h, polylineId));
MapViewApi::RequestRender(h);
```

## 折线（GeoJSON 文件）

```cpp
MapViewApi::setPolylinePointsURI(h, polylineId, "/abs/path/to/routes.geojson");
MapViewApi::setPolylineLineWidth(h, polylineId, 12);
MapViewApi::setPolylineFillColor(h, polylineId, VColor::fromHex("#09cfed"));
// addOverlay → showOverlay → updateLayer → setRotationAngle(h, 0) → setViewBound → RequestRender
```

**必须显式设置线宽与颜色；勿依赖默认样式。**

## 路线瓦片预加载

```cpp
MapViewApi::LoadRouteMapData(h, routeFileName, [](const TilePreloadResponse& resp) {
    // resp.progressPercent == -1 表示瓦片数量超限
});
MapViewApi::CancelMapDataDownload(h);
MapViewApi::DeleteRouteMapData(h, routeFileName);  // routeFileName 为文件名，非完整路径
```

## 多实例地图

多个独立地图实例互不共享地图状态，各自 `Create`/`SetCanvas`/`InitMap`/`Destroy`：

```cpp
MapViewHandle h1 = MapViewApi::Create();
MapViewHandle h2 = MapViewApi::Create();
MapViewApi::SetCanvas(h1, canvas1);
MapViewApi::SetCanvas(h2, canvas2);
MapViewApi::InitMap(h1);
MapViewApi::InitMap(h2);
MapViewApi::setCenterPoint(h1, {116.4074, 39.9042}); // 北京
MapViewApi::setCenterPoint(h2, {121.4737, 31.2304}); // 上海
MapViewApi::RequestRender(h1);
MapViewApi::RequestRender(h2);
MapViewApi::Destroy(h1);
MapViewApi::Destroy(h2);
```

仅进程级全局状态（如 `VImage::SetImageProvider`）在多实例间共享，只需设置一次。
