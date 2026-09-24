# 检索、导航与离线地图

完整可运行代码见 [demo.md](demo.md)（`RunSearchAndRoutePlanDemo` / `RunNaviDemo` / `RunOfflineDemo`）。

## 检索（SearchApi）

入口 `search/search_api.h`；`SearchApi::GetInstance()` 返回**引用**，不是指针。回调线程规则见 [init-auth.md § 线程模型](init-auth.md)，满足该契约时回调里可直接调 `MapViewApi`。

| 必须 | 禁止 |
|------|------|
| 坐标写 `Coordinate(纬度, 经度)`，坐标系 GCJ02，见 [SKILL.md § 输入坐标系](../SKILL.md) | 按地图侧 `VDPOINT(经度, 纬度)` 的顺序填 |
| 判 `SEARCH_ERROR_CODE` | 判 `result` 是否为空（它永远非空） |
| 业务侧自加超时兜底 | 只等回调；回调不触发时看日志 `token is empty, cannot perform ...`（鉴权未完成/失败）、`response is null`（平台返回空 response）、`request returned <非0>`（`BNetwork_fetch` 提交失败） |
| 忽略回调首参 `void* search` | 解引用它（已失效，且无可用信息） |
| `Authenticate` 成功后再发检索 | token 未就绪就发起 |

| 场景 | API |
|------|-----|
| 城市 POI | `PoiCitySearch(PoiCitySearchOption, callback)` |
| 周边 POI | `PoiNearbySearch(PoiNearbySearchOption, callback)` |
| 步行路线 | `RouteWalkingSearch(WalkingSearchOption, callback)` |
| 简化步行 | `RouteSimpleWalkingSearch(...)` |
| 逆地理 | `ReverseGeoCodeSearch(ReverseGeoCodeSearchOption, callback)` |

`scope` 是**结果详细程度**、不是地理范围：`POI_SEARCH_SCOPE_TYPE::BASIC_INFORMATION`(1) / `DETAIL_INFORMATION`(2)，且只有取 `DETAIL_INFORMATION` 时 `filter` 才生效。`scope` / `filter` / `pageIndex` / `pageSize` 都在基类 `PoiSearchOptionCommon` 上，`PoiCitySearchOption` / `PoiNearbySearchOption` / `PoiBoundSearchOption` 都继承它；`PoiDetailSearchOption` 有自己独立的 `scope`。**这些字段在头文件里没有默认初值，必须逐个显式赋值**，不要依赖注释里写的默认值。

## 导航（NaviApi）

入口 `navi/navi_api.h`。路线瓦片预加载用 `MapViewApi::LoadRouteMapData`，见 [overlay-map-control.md § 路线瓦片预加载](overlay-map-control.md)。

```cpp
// 第一参：相对 GetAppCachePath() 的纯文件名，禁止传绝对路径
// 第三参：默认 kDefaultMapViewHandle(1)，用了别的 handle 必须显式传
navi->Init("route.json", [](bool ok) { }, h);
navi->SetNaviType(NaviType::WALKING);   // 必须在 Init 之后
navi->EnableTrackRecording(true);       // 需要轨迹记录时，必须在 StartNavi 之前
navi->StartNavi();
navi->ExitNavi();                       // 复位导航类型，下一段导航要重新 SetNaviType
```

| 必须 | 禁止 |
|------|------|
| `SetNaviType` 在 `Init` 之后 | 放在 `Init` 之前（等于没设，推进不起来） |
| `EnableTrackRecording(true)` 在 `StartNavi` 之前 | `StartNavi` 之后再开（对本段导航无效） |
| 换路线用 `UpdateRoute(const std::vector<std::string>&, cb)`，或 `ExitNavi` 后重新 `Init` | 用单文件版 `UpdateRoute(const std::string&, cb)` 判成败（恒返回 `false`） |
| 切换路线前先 `ExitNavi` | 不退出就二次 `Init`（失败，且上次的覆盖物遗留在地图上） |
| 退出导航时重新注册空回调，或在捕获里加会话标识做失效判断 | 依赖 `ExitNavi` 清回调（它不清），并在回调里捕获会被销毁的 `this` |
| 每种回调只注册一次 | 重复注册（是覆盖，不是追加） |
| 回调里只做轻活 | 在回调里做重活（都在调 `UpdateLocation` 的线程上同步触发） |
| 同一时刻只有一个地图实例显示导航 | 让多个实例同时显示导航覆盖物 |

单位：`UpdateCompass(float)` 是**弧度**（45° 写 `45.0f * M_PI / 180.0f`）；`Location::setDirection(double)` 是**度**（0.0–359.9）；`Location::setTime(uint64_t)` 是**毫秒**，`setSpeed(double)` 是**米/秒**。`Location(纬度, 经度)`，坐标系 GCJ02；`location.h` 里经纬度的注释是对调的，按形参名传值。

`Init` 的静默失败：路线文件缺失或非法时 `Init` 仍返回 `true`、回调永不触发；handle 不存在时导航逻辑照跑但看不到导航覆盖物，`Init` 同样返回 `true`。五种回调的完整签名与"在 `Init` 回调里注册"的写法见 [demo.md § RunNaviDemo](demo.md)；`RegisterGuideInfoCallback` 的第二参是 JSON 字符串，需自行解析。

**导航不推进**（日志只有一行 `NaviEngine not initialization finished.`）时逐项核对：① 路线文件读到并解析成功；② 导航类型 `!= UNKNOWN`（`SetNaviType` 在 `Init` 之后调过）；③ 已通过 `UpdateLocation` 喂进至少一个有效点。`StartNavi` 不校验是否 `Init` 过，返回 `true` 不代表能推进。

## 离线地图（MapOfflineApi）

入口 `offline/map_offline_api.h`。顺序：`BaseApi::SetPackageName` → `RequestVersion(cb)` 并判返回值 → 回调成功后才用列表类接口 → `StartDownloadByCityName`。代码见 [demo.md § RunOfflineDemo](demo.md)。

| 必须 | 禁止 |
|------|------|
| 先 `BaseApi::SetPackageName` | 不设包名就下载（`StartDownloadByCityName` 仍返回 `0`，但写盘失败，只能从进度回调的 `Failed` 看出来） |
| 判 `RequestVersion` 返回值：`0 Ok` / `1 TokenEmpty` / `3 HttpFailed` | 只看回调（`1` 与 `3` 不会经回调给出，回调压根不来） |
| 城市名照抄 `GetDownloadableCityList` 的 `.name`（如 `"北京市"`） | 传 `"北京"`（按字节精确相等匹配，不做后缀归一化） |
| 用 `RegisterDownloadProgressCallback` 的回调确认下载在跑 | 把 `StartDownloadByCityName` 返回 `0` 当成"正在下载" |
| 恢复中断的下载：重新 `StartDownloadByCityName`（同版本从已下字节续传） | 调 `PauseDownloadByCityName`（空 no-op）、`CancelDownloadByCityName`（不停下载、永久置空进度回调，后续 Start 全返回 `8`，只能重启进程） |
| 与返回码比较写全作用域并显式转换：`code == static_cast<int>(MapOfflineStartDownloadCode::OfflinePackageNeedDelete)` | 把 `enum class` 直接与裸数字比较 |
| 用 `MapOfflineRemoveFileStatus` 解读 `DeleteDownloadedCityPackageFile`：`Ok(0)` / `CityNotExist(1)` / `FileNotExist(2)` / `TypeNotExist(3)` | 拿 `MapOfflineStartDownloadCode` 解读它（同一数字含义完全不同） |

`StartDownloadByCityName` 可达返回码：`Ok(0)`、`InvalidCityId(1)`（城市名为空或不在可下载列表）、`OfflinePackageNeedDelete(7)`（本地包已存在/版本不符，先删再重试）、`CityPkgDownloading(8)`（已有任务在下）、`NoDownloadableCityList(9)`（先 `RequestVersion`）。遇 `9` 按 `RequestVersion` → `DeleteDownloadedCityPackageFile` → `StartDownloadByCityName` 的顺序处理。

`GetOfflineCityInfo(cityNameVec, cityRecords)`：先清空 `cityRecords`，输出与 `cityNameVec` 顺序一致，匹配不到的城市跳过，须先 `RequestVersion`。没有 `RegisterRequestVersionCallback` 这个接口，版本回调是 `RequestVersion` 的参数。离线包生效的两个前提：已下载包的索引是一次性快照，本次运行新下完的包要**重启进程**才会被用上；zoom < 10 只查全国基础包，离线看全国图必须下全国包。

**mecp**：场景识别（给定坐标判断是否处于机场/地铁站/水域等场景内），与地图渲染/检索/导航互相独立，见 [mecp.md](mecp.md)。
