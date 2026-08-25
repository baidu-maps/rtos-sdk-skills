# 完整 Demo 示例

所有示例均源自 SDK 全流程示例，可直接对照集成。头文件路径以实际工程 `includes/` 为准。

---

## main：初始化全流程

```cpp
using namespace baidu::rtos_map;
using namespace baidu::rtos_map::auth;
using namespace baidu::rtos_map::base;
using namespace baidu::rtos_map::view;

int main() {
    BaseApi* baseApi = BaseApi::GetInstance();
    baseApi->SetPackageName("your_package_name");

    AuthLicenseApi* authApi = AuthLicenseApi::GetInstance();
    authApi->SetAk("your_ak");
    authApi->RequestLicense([](license::LicenseErrorCode code) { /* ... */ });
    authApi->Authenticate([](MapAuthErrorCode code) { /* ... */ });

    MapViewHandle h = MapViewApi::Create();

    auto baiduMapCanvas = std::make_shared<MapCanvasImpl>();
    // 传入平台真实 canvas/context，例如 OHOS 先创建 UICanvasExt 再 setCanvas
    baiduMapCanvas->setCanvas(platformCanvasContext);
    MapViewApi::SetCanvas(h, baiduMapCanvas);

    // SDK 内部渲染数据更新时触发此回调，在此调用 RequestRender 拉取新数据
    MapViewApi::SetUIThreadFunc(h, [h]() {
        MapViewApi::RequestRender(h);
    });
    MapViewApi::InitMap(h);
    MapViewApi::RequestRender(h);

    // ... 业务 Demo ...

    MapViewApi::Destroy(h);
    return 0;
}
```

---

## RunMapStateAndOverlayDemo：地图状态、Marker、折线

```cpp
static void RunMapStateAndOverlayDemo(MapViewHandle h) {
    MapViewApi::setCenterPoint(h, VDPOINT(116.404, 39.915));
    MapViewApi::setZoom(h, 15.0f);
    MapViewApi::zoomIn(h);
    MapViewApi::zoomOut(h);
    MapViewApi::setRotationAngle(h, 30.0);
    MapViewApi::SetMapBackgroundColor(h, VColor(18, 18, 24, 255));
    MapViewApi::setViewBound(h, utils::geo::VRect(0, 0, 800, 480));

    const VDPOINT pixel  = MapViewApi::LatLngToScreenPixel(h, VDPOINT(116.404, 39.915));
    const VDPOINT restore = MapViewApi::ScreenPixelToLatLng(h, pixel);

    const int layerId   = MapViewApi::createLayer(h, LayerType::OVERLAYER, "demo_overlay_layer");

    // Marker
    const int markerId = MapViewApi::CreateMarker(h);
    MapViewApi::setMarkerPosition(h, markerId, VDPOINT(116.404, 39.915));
    MapViewApi::setMarkerSize(h, markerId, utils::geo::VSize(36, 36));
    MapViewApi::setMarkerOffset(h, markerId, utils::geo::VSize(18, 36));
    MapViewApi::setMarkerAngle(h, markerId, 0.0);
    MapViewApi::addOverlay(h, layerId, markerId);
    MapViewApi::showOverlay(h, markerId);

    // 折线（点集）
    const int polylineId = MapViewApi::CreatePolyline(h);
    std::vector<VDPOINT> routePoints = {
        VDPOINT(116.400, 39.910),
        VDPOINT(116.404, 39.915),
        VDPOINT(116.410, 39.920)
    };
    MapViewApi::setPolylinePoints(h, polylineId, routePoints);
    MapViewApi::setPolylineFillColor(h, polylineId, VColor(0, 153, 255, 220));
    MapViewApi::setPolylineStrokeColor(h, polylineId, VColor(255, 255, 255, 255));
    MapViewApi::setPolylineLineWidth(h, polylineId, 8);
    MapViewApi::setPolylineStrokeWidth(h, polylineId, 2);
    MapViewApi::addOverlay(h, layerId, polylineId);
    MapViewApi::showOverlay(h, polylineId);
    MapViewApi::updateLayer(h, layerId);

    const utils::geo::VRect bounds = MapViewApi::getOverlayBounds(h, polylineId);
    MapViewApi::setViewBound(h, bounds);
    MapViewApi::RequestRender(h);
}
```

> Marker 图标（`setMarkerVImage`）依赖 `VImage::SetImageProvider` 注册的 Provider；honor/oppo/huawei/xiaoniu 已内置默认实现，simulator 等平台需应用侧自行注册（见 [overlay-map-control.md § 图片适配](overlay-map-control.md)）。

---

## RunTouchDemo：触摸事件

```cpp
static void RunTouchDemo(MapViewHandle h) {
    auto makeEvent = [](TouchEvent::Type type, float x, float y) {
        TouchEvent e;
        e.type = type;
        Touch p;
        p.clientX = x;
        p.clientY = y;
        e.touches.push_back(p);
        e.changedTouches.push_back(p);
        return e;
    };

    MapViewApi::OnTouchDown(h, makeEvent(TouchEvent::Type::TOUCH_START, 100, 100));
    MapViewApi::OnTouchMove(h, makeEvent(TouchEvent::Type::TOUCH_MOVE, 180, 160));
    MapViewApi::OnTouchUp(h, makeEvent(TouchEvent::Type::TOUCH_END,  220, 180));
}
```

---

## RunSearchAndRoutePlanDemo：POI 搜索与步行路线规划

```cpp
static void RunSearchAndRoutePlanDemo() {
    using namespace baidu_search;
    SearchApi& searchApi = SearchApi::GetInstance();

    PoiCitySearchOption poiOption;
    poiOption.city       = "北京";
    poiOption.keyword    = "地铁站";
    poiOption.isCityLimit = true;
    poiOption.scope      = POI_SEARCH_SCOPE_TYPE::DETAIL_INFORMATION;
    poiOption.pageIndex  = 0;
    poiOption.pageSize   = 10;
    searchApi.PoiCitySearch(poiOption,
        [](void*, PoiSearchResult* result, SEARCH_ERROR_CODE code, const std::string&) {
            // result 可能为 nullptr
        });

    WalkingSearchOption walkOption;
    walkOption.from.pt       = Coordinate(39.915, 116.404);   // 纬度, 经度
    walkOption.from.cityName = "北京";
    walkOption.to.pt         = Coordinate(39.99466, 116.502966);
    walkOption.to.cityName   = "北京";
    searchApi.RouteWalkingSearch(walkOption,
        [](void*, WalkingSearchResult* result, SEARCH_ERROR_CODE code, const std::string&) {
            // 回调在 HTTP 线程，操作 MapViewApi 须切回 UI 线程
        });
}
```

> `Coordinate` 构造顺序为 **(纬度, 经度)**，勿颠倒。

---

## RunOfflineDemo：离线地图

```cpp
static void RunOfflineDemo() {
    using namespace baidu::rtos_map::offline;
    MapOfflineApi* api = MapOfflineApi::GetInstance();

    api->RegisterRequestVersionCallback([](const MapOfflineGetVersionCode& code) { });
    api->RegisterDownloadProgressCallback(
        [](const std::string& cityName, MapOfflineDownloadStatus status,
           int progress, int64_t downloaded, int64_t total) { });

    api->RequestVersion();   // 异步，版本回调后列表才有效

    std::vector<OfflineCityInfo> downloadable, downloaded;
    api->GetDownloadableCityList(downloadable);
    api->GetDownloadedCityList(downloaded);

    std::vector<std::string> queryCities = {"北京", "上海"};
    std::vector<OfflineCityInfo> cityInfos;
    api->GetOfflineCityInfo(queryCities, cityInfos);  // 先清空再写入，匹配不到的跳过

    const std::string city = downloadable.empty() ? "北京" : downloadable.front().name;
    int code = api->StartDownloadByCityName(city);
    // code == OfflinePackageNeedDelete：先 DeleteDownloadedCityPackageFile 再重试
}
```

---

## RunRouteTileDemo：路线瓦片预加载

```cpp
static void RunRouteTileDemo(MapViewHandle h) {
    const std::string routeFile = "route.json";   // 文件名，非完整路径

    MapViewApi::LoadRouteMapData(h, routeFile, [](const TilePreloadResponse& resp) {
        // resp.progressPercent == -1 表示瓦片数量超限
    });

    MapViewApi::CancelMapDataDownload(h);
    MapViewApi::DeleteRouteMapData(h, routeFile);
}
```

---

## RunNaviDemo：导航

```cpp
static void RunNaviDemo() {
    using namespace baidu::rtos_map::navi;
    NaviApi* navi = NaviApi::GetInstance();

    navi->RegisterGuideInfoCallback([](NaviType type, const std::string& info) { });
    navi->RegisterYawingCallback([](NaviType type, NaviYawingStatus status) { });
    navi->RegisterRemainDistanceCallback([](NaviType type, int meters) { });
    navi->RegisterRemainTimeCallback([](NaviType type, int seconds) { });
    navi->RegisterTrackCallback(
        [](NaviType type, int stepIndex, const std::string& curLoc,
           const std::string& nextPoint, const std::string& routeInfo) { });

    navi->Init("route.json", [](bool ok) { });
    navi->UpdateRoute("new_route.json", [](bool ok) { });
    navi->SetNaviType(NaviType::WALKING);
    navi->StartNavi();

    // 持续喂入传感器数据
    navi->UpdateCompass(45.0f);
    location::Location loc(39.915, 116.404);
    loc.setTime(1234567890);
    loc.setDirection(45.0);
    loc.setSpeed(1.2);
    navi->UpdateLocation(loc);

    navi->ExitNavi();
}
```

---

## 多实例地图示例

```cpp
MapViewHandle h1 = MapViewApi::Create();
MapViewHandle h2 = MapViewApi::Create();
auto canvas1 = std::make_shared<MapCanvasImpl>(); canvas1->setCanvas(ctx1);
auto canvas2 = std::make_shared<MapCanvasImpl>(); canvas2->setCanvas(ctx2);
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

两个实例各自独立生命周期，互不共享地图状态（`renderDataCache_`、图层、覆盖物等），仅进程级全局状态（如 `VImage::SetImageProvider`）共享。
