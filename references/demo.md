# 可照抄的示例集

- 头文件按 `<模块>/*_api.h` 前缀包含，`includes/` 与 `common/` 各自连同全部子目录作为 `-I` 根，见 [SKILL.md § 编译](../SKILL.md)；示例中的经纬度都是 **GCJ02**，WGS84 输入必须先转换，见 [SKILL.md § 输入坐标系](../SKILL.md)。
- `MapCanvasImpl` 指**你自己实现的** Canvas 类：命名空间 `baidu::rtos_map::view`，继承 `CanvasContextCommonInterface`，71 个纯虚函数全部定义，见 [adapter-build.md § 5](adapter-build.md)。

## main：初始化全流程

顺序固定：`SetPackageName` → `SetAk` → `RequestLicense` → 回调成功后 `Create` → `SetCanvas` → `SetSize` → `SetUIThreadFunc` → `InitMap` → `RequestRender`。`RequestLicense` 的回调可能在调用栈内同步触发，规则见 [init-auth.md](init-auth.md)。

```cpp
using namespace baidu::rtos_map;         using namespace baidu::rtos_map::auth;
using namespace baidu::rtos_map::base;   using namespace baidu::rtos_map::view;
static MapViewHandle g_h = 0;
static std::shared_ptr<MapCanvasImpl> g_canvas;   // 你自己实现的 Canvas 类
static void StartMap() {                          // 只能在 License 回调成功后调
    g_h = MapViewApi::Create();                   // 保存返回值，全程用它
    if (g_h == kInvalidMapViewHandle) { return; } // 0 = License 未通过
    g_canvas = std::make_shared<MapCanvasImpl>();
    g_canvas->setCanvas(platformCanvasContext);   // 平台渲染上下文，也可在构造函数里绑定
    MapViewApi::SetCanvas(g_h, g_canvas);
    MapViewApi::SetSize(g_h, 466, 466);           // 必须，与真实画布尺寸一致
    MapViewApi::SetUIThreadFunc(g_h, []() {       // 必须早于 InitMap
        MapViewApi::RequestRender(g_h);           // 多线程模型：改为 post 到渲染线程
    });
    MapViewApi::InitMap(g_h);
    MapViewApi::RequestRender(g_h);
}

int main() {
    BaseApi::GetInstance()->SetPackageName("your_package_name");
    AuthLicenseApi* authApi = AuthLicenseApi::GetInstance();   authApi->SetAk("your_ak");
    authApi->RequestLicense([authApi](license::LicenseErrorCode code) {
        if (code != license::LicenseErrorCode::NO_ERROR) { return; }
        authApi->Authenticate([](MapAuthErrorCode code) {
            if (code != MapAuthErrorCode::OK) { /* token 失败：网络类请求全失败 */ }
        });
        StartMap();                               // 到这里 Create() 才会成功
    });
    // ... 事件循环 / 渲染循环 ...
    MapViewApi::Destroy(g_h);
    return 0;
}
```

## 地图状态、覆盖物与触摸（RunMapStateAndOverlayDemo / RunTouchDemo）

覆盖物顺序：`createLayer` → `Create*` → 设属性 → `addOverlay` → `updateLayer` → 渲染一次；折线必须显式设线宽与填充色；角度参数一律弧度。其余规则见 [overlay-map-control.md](overlay-map-control.md)。

```cpp
static void RunMapStateAndOverlayDemo(MapViewHandle h) {
    MapViewApi::setCenterPoint(h, VDPOINT(116.404, 39.915));   // VDPOINT(经度, 纬度)
    MapViewApi::setZoom(h, 15.0f);
    MapViewApi::setRotationAngle(h, 30.0 * M_PI / 180.0);      // 弧度
    const int layerId = MapViewApi::createLayer(h, LayerType::OVERLAYER, "demo_overlay_layer");
    const int markerId = MapViewApi::CreateMarker(h);
    MapViewApi::setMarkerPosition(h, markerId, VDPOINT(116.404, 39.915));
    MapViewApi::setMarkerSize(h, markerId, utils::geo::VSize(36, 36));
    MapViewApi::setMarkerOffset(h, markerId, utils::geo::VSize(18, 36));
    MapViewApi::setMarkerAngle(h, markerId, 0.0);              // 弧度
    MapViewApi::addOverlay(h, layerId, markerId);
    const int polylineId = MapViewApi::CreatePolyline(h);
    std::vector<VDPOINT> pts = { VDPOINT(116.400, 39.910), VDPOINT(116.404, 39.915), VDPOINT(116.410, 39.920) };
    MapViewApi::setPolylinePoints(h, polylineId, pts);
    MapViewApi::setPolylineLineWidth(h, polylineId, 8);   MapViewApi::setPolylineFillColor(h, polylineId, VColor(0, 153, 255, 220));   // 两者必须
    MapViewApi::addOverlay(h, layerId, polylineId);
    MapViewApi::updateLayer(h, layerId);                       // 必须
    MapViewApi::RequestRender(h);
}
```

`TOUCH_MOVE` 的 `touches` **至少放 2 个元素**（只放 1 个会越界读内存），把同一个 touch push 两次即可。

```cpp
static void RunTouchDemo(MapViewHandle h) {
    auto makeEvent = [](TouchEvent::Type type, float x, float y) {
        TouchEvent e;   e.type = type;
        Touch p;        p.clientX = x;   p.clientY = y;
        e.touches.push_back(p);
        if (type == TouchEvent::Type::TOUCH_MOVE) {
            e.touches.push_back(p);        // MOVE 必须凑够 2 个元素
        }
        e.changedTouches.push_back(p);
        return e;
    };
    MapViewApi::OnTouchDown(h, makeEvent(TouchEvent::Type::TOUCH_START, 100, 100));
    MapViewApi::OnTouchMove(h, makeEvent(TouchEvent::Type::TOUCH_MOVE, 180, 160));
    MapViewApi::OnTouchUp(h, makeEvent(TouchEvent::Type::TOUCH_END, 220, 180));
}
```

## POI 搜索与步行路线（RunSearchAndRoutePlanDemo）

`Coordinate(纬度, 经度)`；判 `SEARCH_ERROR_CODE`，不判 `result` 是否为空；回调首参 `void* search` 已失效，禁止解引用。回调可能根本不触发，业务侧必须自加超时兜底，见 [search-navi-offline.md § 检索](search-navi-offline.md)。

```cpp
static void RunSearchAndRoutePlanDemo() {
    using namespace baidu_search;
    SearchApi& searchApi = SearchApi::GetInstance();        // 返回引用，不是指针
    PoiCitySearchOption poiOption;
    poiOption.city = "北京";   poiOption.keyword = "地铁站";   poiOption.isCityLimit = true;
    poiOption.scope = POI_SEARCH_SCOPE_TYPE::DETAIL_INFORMATION;   // 结果详细程度，非地理范围
    poiOption.pageIndex = 0;   poiOption.pageSize = 10;            // 无默认初值，必须显式赋
    searchApi.PoiCitySearch(poiOption,
        [](void* search, PoiSearchResult* result, SEARCH_ERROR_CODE code, const std::string&) {
            (void)search;                                  // 禁止解引用
            if (code != SEARCH_ERROR_CODE::NO_ERROR) { return; }
        });
    WalkingSearchOption walkOption;
    walkOption.from.pt = Coordinate(39.915, 116.404);       // (纬度, 经度)
    walkOption.to.pt   = Coordinate(39.99466, 116.502966);
    walkOption.from.cityName = "北京";   walkOption.to.cityName = "北京";
    searchApi.RouteWalkingSearch(walkOption,
        [](void*, WalkingSearchResult* result, SEARCH_ERROR_CODE code, const std::string&) {
            if (code != SEARCH_ERROR_CODE::NO_ERROR) { return; }
        });
}
```

## 离线地图与路线瓦片预加载（RunOfflineDemo / RunRouteTileDemo）

必须判 `RequestVersion` 的返回值（`1 TokenEmpty` / `3 HttpFailed` 不经回调），版本回调是它的参数、没有单独的注册接口；城市名照抄列表里的 `.name`；`StartDownloadByCityName` 返回 `0` 只表示已受理，下载是否在跑看进度回调。返回码全表见 [search-navi-offline.md § 离线地图](search-navi-offline.md)。

```cpp
static void RunOfflineDemo() {
    using namespace baidu::rtos_map::offline;
    MapOfflineApi* api = MapOfflineApi::GetInstance();
    api->RegisterDownloadProgressCallback(                  // 必须注册
        [](const std::string& cityName, MapOfflineDownloadStatus status,
           int progress, int64_t downloaded, int64_t total) { });
    const int versionRc = api->RequestVersion([](const MapOfflineGetVersionCode& code) {
        if (code != MapOfflineGetVersionCode::Ok) { return; }
        MapOfflineApi* api = MapOfflineApi::GetInstance();
        std::vector<OfflineCityInfo> downloadable;
        api->GetDownloadableCityList(downloadable);         // 回调成功后才可取列表
        if (downloadable.empty()) { return; }
        const std::string city = downloadable.front().name; // 照抄，如 "北京市"
        int code2 = api->StartDownloadByCityName(city);     // 返回 int，比较须显式转换
        if (code2 == static_cast<int>(MapOfflineStartDownloadCode::OfflinePackageNeedDelete)) {
            api->DeleteDownloadedCityPackageFile(city);
            code2 = api->StartDownloadByCityName(city);
        }
    });
    if (versionRc != 0) { /* 回调不会来，在此处理失败 */ }
}
```

`routeFile` 传文件名。geojson 源文件按进程 CWD 解析、瓦片账本相对 `GetAppCachePath()`，调用前先把 CWD 切到 `GetAppCachePath() + "/route"`。回调的三种结果见 [overlay-map-control.md § 路线瓦片预加载](overlay-map-control.md)。

```cpp
static void RunRouteTileDemo(MapViewHandle h) {
    const std::string routeFile = "route.json";      // 文件名，不是完整路径
    MapViewApi::LoadRouteMapData(h, routeFile, [](const TilePreloadResponse& resp) {
        if (resp.progressPercent == -1) { return; }  // 瓦片超 5000 张，已放弃预加载
    });
    MapViewApi::CancelMapDataDownload(h);
    MapViewApi::DeleteRouteMapData(h, routeFile);
}
```

## 导航（RunNaviDemo）

`SetNaviType` 必须在 `Init` 之后，`EnableTrackRecording(true)` 必须在 `StartNavi` 之前，每种回调只有一个槽位（重复注册是覆盖）。`Init` 首参是相对 `GetAppCachePath()` 的纯文件名（禁止绝对路径），第三参是 handle（默认 `kDefaultMapViewHandle`(1)，用别的 handle 必须显式传）。单位：`UpdateCompass` 弧度、`setDirection` 度、`setTime` 毫秒、`setSpeed` 米/秒。见 [search-navi-offline.md § 导航](search-navi-offline.md)。

```cpp
static void RunNaviDemo() {
    using namespace baidu::rtos_map::navi;
    NaviApi* navi = NaviApi::GetInstance();
    navi->Init("route.json", [navi](bool ok) {
        if (!ok) { return; }
        navi->RegisterGuideInfoCallback([](NaviType type, const std::string& info) { });  // info 是 JSON
        navi->RegisterYawingCallback([](NaviType type, NaviYawingStatus status) { });
        navi->RegisterRemainDistanceCallback([](NaviType type, int meters) { });
        navi->RegisterRemainTimeCallback([](NaviType type, int seconds) { });
        navi->RegisterTrackCallback([](NaviType type, int stepIndex, const std::string& curLoc,
                                       const std::string& nextPoint, const std::string& routeInfo) { });
        navi->EnableTrackRecording(true);         // 须早于 StartNavi
        navi->SetNaviType(NaviType::WALKING);    navi->StartNavi();   // SetNaviType 须在 Init 之后
    }, kDefaultMapViewHandle);
    navi->UpdateCompass(45.0f * static_cast<float>(M_PI) / 180.0f);   // 弧度
    location::Location loc(39.915, 116.404);    // (纬度, 经度)
    loc.setTime(1234567890);   loc.setDirection(45.0);   loc.setSpeed(1.2);  // 毫秒 / 度 / 米每秒
    navi->UpdateLocation(loc);                  // 导航靠这里推进
    navi->ExitNavi();                           // 复位导航类型，下一段要重新 SetNaviType
}
```

## 多实例地图

每个实例各自 `Create` → `SetCanvas` → `SetSize` → `InitMap` → `Destroy`，地图状态互不影响；handle 一律用 `Create()` 返回值传递。进程级共享项见 [overlay-map-control.md § 多实例地图](overlay-map-control.md)。

```cpp
MapViewHandle h1 = MapViewApi::Create(), h2 = MapViewApi::Create();   // License 通过后
auto c1 = std::make_shared<MapCanvasImpl>();   auto c2 = std::make_shared<MapCanvasImpl>();
c1->setCanvas(ctx1);   c2->setCanvas(ctx2);
MapViewApi::SetCanvas(h1, c1);  MapViewApi::SetSize(h1, 466, 466);  MapViewApi::InitMap(h1);
MapViewApi::SetCanvas(h2, c2);  MapViewApi::SetSize(h2, 466, 466);  MapViewApi::InitMap(h2);
MapViewApi::setCenterPoint(h1, {116.4074, 39.9042});   MapViewApi::setCenterPoint(h2, {121.4737, 31.2304});
MapViewApi::RequestRender(h1);  MapViewApi::RequestRender(h2);
MapViewApi::Destroy(h1);  MapViewApi::Destroy(h2);
```
