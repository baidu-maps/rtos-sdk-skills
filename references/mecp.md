# mecp（场景识别）

给定一个坐标，判断它在哪个机场 / 火车站 / 商场 / 医院 / 水域等场景（AOI）的内部或附近：数据在线下载，识别在本地完成。与地图渲染、`SearchApi`、`NaviApi`、`MapOfflineApi` 互相独立，不需要该能力时可整块排除。入口头文件：`mecp/include/rtos_mecp.h` + `mecp/include/rtos_mecp_types.h`（一套 C 接口）。

## 接入四步

1. 实现平台依赖（见下表）。
2. `BaseApi::SetPackageName()` → `mecp_init()`。两步都是同步的，可在启动早期完成；**`mecp_init` 的唯一前置就是 `SetPackageName`**，与 License、网络无关，**不要放进 `RequestLicense` 回调里等**。
3. `AuthLicenseApi::SetAk` → `Authenticate` + `RequestLicense`。这是**业务接口**（下载 / 识别 / AOI 提取）的前置，与 `mecp_init` 无先后依赖。
4. `mecp_update_request_radius`（可选）→ `mecp_request_data` → `mecp_recognize`。

## 平台依赖

与地图共用 `common/` 契约，实现要求见 [adapter-build.md](adapter-build.md)。**下表 5 项均为链接期必需**，缺任意一项链接失败（mecp 不使用 Canvas / ImageProvider 契约，不受这两项的运行期失败模式影响）。

| 契约 | mecp 是否必需 |
|------|------|
| `common/platform_adapter.h`：`bd_map_malloc/realloc/free`、`bd_map_cjson_malloc/free`、`bd_map_get_current_task_id`、`bd_map_log` | 必需 |
| `common/base/MapClockImpl.h`、`MapFileImpl.h`、`MapFileUtilImpl.h` | 必需 |
| `common/base/Map{Thread,Mutex,Semaphore,Event,Timer,MsgQueue}Impl.h` | 仅地图 / 导航等模块需要，mecp-only 可不实现 |
| `common/device/DevicePlatformUtil.h`，`getDeviceId` 必须返回有效值 | 必需（鉴权用） |
| `common/fetch/network_fetch.h`：`BNetwork_fetch*`，三个回调必须收敛到同一线程 | 必需 |

## 数据目录

固定为 `<存储根目录>/<packageName>/cache/mecp/{meta,data}`，**不能传自定义路径**，应用层唯一可变量是 `PackageName`。未设 `PackageName` 时 `mecp_init()` 返回 `MECP_ERR_NOT_INIT`。运行期变更 `PackageName` 必须 `mecp_uninit()` → `mecp_init()` 才生效。

## API 速查

| 生命周期 | 作用 |
|------|------|
| `int mecp_init(void)` | 初始化并建数据目录。已初始化时幂等早退，不更新数据目录 |
| `void mecp_uninit(void)` | 释放资源，在途 HTTP 请求安全；之后业务接口一律返回 `MECP_ERR_NOT_INIT`(-2)，需重新 `mecp_init` |
| `int mecp_update_config(int request_encrypt_enable)` | 坐标加密开关：`1` 开 / `0` 关 |
| `void mecp_set_request_key(const uint8_t* key32)` | 注入 32 字节加密 Key，通常不必调用；**传 `nullptr` 是空操作**，不会清除已设置的 Key |

| 下载与本地数据 | 作用 |
|------|------|
| `int mecp_request_data(MecpLocation location, MecpPlaceType type, MecpRequestCallback cb, void* user_data)` | 下载该坐标 + 类型的场景数据。**只有 4 个参数**，没有半径 / 粒度参数 |
| `int mecp_update_request_radius(int radius)` | 设 WATER 下载半径（米）。**须在 `mecp_init` 之后、`mecp_request_data` 之前**；`radius <= 0`（含负数）一律按 1000m，不报参数错 |
| `int mecp_query_data(MecpQueryCallback cb, void* user_data)` | 列举已下载数据集。同步回调，不校验 License |
| `int mecp_delete_data(const MecpOfflineDataModel* model, MecpDeleteCallback cb, void* user_data)` | 删除指定数据集。同步回调，不校验 License |

| 识别与 AOI 提取 | 作用 |
|------|------|
| `int mecp_recognize(MecpLocation location, const MecpPlaceRequest* requests, int request_count, MecpRecognizeCallback cb, void* user_data)` | 按场景类型识别，一次可带多条请求。同步执行，回调在调用栈内，可每次 GPS 更新调用 |
| `int mecp_recognize_by_aoi_uid(MecpLocation location, const char* uid, MecpRecognizeCallback cb, void* user_data)` | 判断坐标与指定 AOI 的关系。同步回调 |
| `int mecp_pickup_aoi_list(MecpLocation location, int radius, MecpPlaceType aoi_type, MecpPickupAoiCallback cb, void* user_data)` | 取坐标周边 `radius` 米内指定类型的 AOI，一次最多 `MECP_MAX_AOI_LIST_SIZE`(50) 个。同步回调 |
| `int mecp_pickup_aoi_by_uid(const char* uid, MecpPickupAoiCallback cb, void* user_data)` | 按 UID 取单个 AOI。同步回调 |

识别与提取都只读**本地已下载**数据、只匹配 `place_type` 完全相同的数据集，没有匹配数据即 `MECP_ERR_NO_DATA`；每个识别响应最多 `MECP_MAX_RESULT_SIZE`(20) 条，`results[0]` 最相关。

## 错误码

| 错误码 | 值 | 传递方式 | 含义 |
|--------|----|------|------|
| `MECP_OK` | 0 | 返回值 + 回调 | 返回值 = 请求已受理；回调 = 执行成功 |
| `MECP_LOCAL_DATA_EXIST` | 1 | **仅回调**（`MecpRequestCallback`，同步触发） | 本地已有相同区域数据，跳过下载。不是错误 |
| `MECP_ERR_INVALID_PARAM` | -1 | **仅返回值** | 参数非法 |
| `MECP_ERR_NOT_INIT` | -2 | 返回值为主，也会经回调 | 未 `mecp_init` / 未 `SetPackageName` / 已 `mecp_uninit` |
| `MECP_ERR_DB` | -3 | 返回值 **+** 回调 | 本地存储失败；`mecp_delete_data` 两处都会给出 |
| `MECP_ERR_NET` | -4 | **仅回调** | 网络请求失败或服务端状态异常 |
| `MECP_ERR_NO_DATA` | -5 | **仅回调** | 服务端或本地无数据。服务端状态 240 时**本地全部 mecp 数据已被清空**，收到后不要假设旧数据还在 |
| `MECP_ERR_NO_PERMISSION` | -6 | **仅返回值** | License 未授权或 token 为空。重新 `Authenticate` / `RequestLicense` 即可，无需重新 `mecp_init` |

## 场景类型

`MecpPlaceType`（`rtos_mecp_types.h`）：`MECP_PLACE_TYPE_NONE`(0)、`AIRPORT`(1)、`TRAIN_STATION`(2)、`SHOPMALL`(3)、`SUBWAY_STATION`(4)、`BUS_STATION`(5)、`FILLING_STATION`(6)、`SCHOOL`(7)、`HOSPITAL`(8)、`RESIDENTIAL_AREA`(9)、`SCENIC_AREA`(10)、`PARK`(11)、`FREEWAY_SERVICE`(12)、`WATER`(13)。

- SDK **不做客户端类型拦截**：13 类都会正常发出请求；**当前只有 `MECP_PLACE_TYPE_WATER` 有服务端数据**，其它类型通常回 `MECP_ERR_NO_DATA`。
- `SUBWAY_STATION` / `BUS_STATION` 按 POI 处理：`is_in_aoi_area` 恒为 `0`，`distance_near == distance_to_poi`。

## 关键结构体

`MecpLocation`：`double lat` / `double lng`（纬度 / 经度）+ `MecpCoordType coord_type`，取 `MECP_COORD_WGS84`(0) / `MECP_COORD_GCJ02`(1) / `MECP_COORD_BD09LL`(2)，按实际来源填。

| `MecpPlaceRequest` | 说明 |
|------|------|
| `int index` | 原样回传，用于对应响应 |
| `MecpPlaceType place_type` | 参与识别的数据集类型，必须完全相同才匹配 |
| `int restrict_recognize_radius` | 米，`0` = 不限；**仅当坐标在 AOI 外部时参与过滤** |
| `int restrict_poi_radius` | 米，**`<= 0` 按 1000m 处理**，不是"不限" |
| `int city_code` | `0` = 不按城市过滤 |

| `MecpPlaceData` | 说明 |
|------|------|
| `uid` / `name` / `place_type` | AOI/POI 标识、名称、类型 |
| `is_in_aoi_area` | `1` = 在内部，`0` = 在外部 |
| `distance_near` / `distance_to_poi` | 到最近边界的距离（内部时为 `0`）/ 到多边形顶点算术平均质心的 haversine 距离，均为米 |
| `dataset_hashcode` | 所属数据集标识 |


`MecpPickupAoiResponse` 的 `aois[i].polygon` 是 `char[4096]` 字符串，格式 `"x,y|x,y|..."`（`|` 分点、`,` 分坐标、**x 即经度方向在前**）：**坐标值是 bd09mc 墨卡托米，不是经纬度**，画到地图上前需自行做 mc → ll 转换；超长多边形会被**静默截断到 4095 字节**，解析前自检末尾是否为完整的 `x,y` 对。

## 必须 / 禁止

| 必须 | 禁止 |
|------|------|
| `SetPackageName` → `mecp_init()`，启动早期同步做完 | 把 `mecp_init` 放进 `RequestLicense` 回调里等 |
| `mecp_update_request_radius` 在 `mecp_init` 之后、`mecp_request_data` 之前 | 在 `mecp_init` 之前设半径（会被覆盖）；用 `<= 0` 表示"不限"（按 1000m） |
| 权限 / 参数错误查**返回值** | 在回调里等 `MECP_ERR_NO_PERMISSION` / `MECP_ERR_INVALID_PARAM`（回调不触发） |
| `mecp_request_data` 自加超时兜底 | 把返回 `MECP_OK` 当成"回调一定会来" |
| 回调按"可能仍在 `mecp_request_data` 调用栈内"且"可能不在主线程"来写 | 在回调里假设函数已返回、或假设自己在主线程 |
| 收到 `MECP_ERR_NO_DATA` 后按需重新下载 | 假设服务端状态 240 之后本地旧数据还在 |
| 变更 `PackageName` 后 `mecp_uninit()` → `mecp_init()` | 重复调 `mecp_init` 期望换目录 |
| `polygon` 先 mc → ll 转换并自检完整性 | 把 `polygon` 直接当经纬度用 |
| 只对 `MECP_PLACE_TYPE_WATER` 期望有结果 | 期望其它 12 类返回数据 |
| `mecp_recognize` 直接在每次 GPS 更新时调 | 为它另起线程或队列 |

## 最小完整示例

```cpp
#include "base/base_api.h"
#include "auth/auth_license_api.h"
#include "mecp/include/rtos_mecp.h"
#include "mecp/include/rtos_mecp_types.h"
using namespace baidu::rtos_map::auth;
using namespace baidu::rtos_map::base;

static void on_recognize(MecpRecognizeResponse* responses, int response_count, void* /*ud*/) {
    for (int i = 0; i < response_count; i++) {
        MecpRecognizeResponse* r = &responses[i];
        if (r->error_code != MECP_OK || r->result_count == 0) { continue; }
        MecpPlaceData* best = &r->results[0];            // results[0] 最相关
        if (best->is_in_aoi_area) { /* 在 best->name 内部 */ }
        else { /* 在 best->name 附近，距边界 best->distance_near 米 */ }
    }
}

static void on_download(int error_code, MecpOfflineDataModel* /*models*/,
                        int /*count*/, void* /*ud*/) {
    // MECP_LOCAL_DATA_EXIST 在 mecp_request_data 调用栈内同步到达；
    // MECP_OK / MECP_ERR_NET / MECP_ERR_NO_DATA 在网络回调线程到达
    if (error_code != MECP_OK && error_code != MECP_LOCAL_DATA_EXIST) { return; }
    MecpLocation loc = {39.915, 116.404, MECP_COORD_BD09LL};
    MecpPlaceRequest req{};
    req.index = 0;
    req.place_type = MECP_PLACE_TYPE_WATER;
    req.restrict_poi_radius = 0;                         // 0 等于 1000m
    mecp_recognize(loc, &req, 1, on_recognize, nullptr);
}

void app_start() {
    BaseApi::GetInstance()->SetPackageName("your.app.name");
    if (mecp_init() != MECP_OK) { return; }              // 失败即 MECP_ERR_NOT_INIT
    mecp_update_request_radius(1000);                    // 必须在 mecp_init 之后

    AuthLicenseApi* auth = AuthLicenseApi::GetInstance();
    auth->SetAk("your_ak_here");
    auth->Authenticate([](MapAuthErrorCode err) {
        if (err != MapAuthErrorCode::OK) { return; }      // token 为空 → 下载返回 NO_PERMISSION
    });
    auth->RequestLicense([](LicenseErrorCode err) {
        if (err != LicenseErrorCode::NO_ERROR) { return; }
        MecpLocation loc = {39.915, 116.404, MECP_COORD_BD09LL};
        int ret = mecp_request_data(loc, MECP_PLACE_TYPE_WATER, on_download, nullptr);
        if (ret != MECP_OK) { return; }                  // 无权限/未初始化/参数非法：回调不会来
        // ret == MECP_OK：回调可能已同步跑完、可能稍后到达、也可能不来 → 需超时兜底
    });
}

// 每次 GPS 更新时调用：同步、无 I/O，回调在本函数调用栈内触发
void on_location_update(double lat, double lng) {
    MecpLocation loc = {lat, lng, MECP_COORD_BD09LL};
    MecpPlaceRequest req{};
    req.index = 0;
    req.place_type = MECP_PLACE_TYPE_WATER;
    mecp_recognize(loc, &req, 1, on_recognize, nullptr);
}
```
