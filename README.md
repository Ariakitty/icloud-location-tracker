![iCloud Location Tracker](assets/banner.svg)

# iCloud Location Tracker

用 pyicloud + 高德逆地理编码 + 常用地点检测，在 VPS 上搭建一个持久运行的 iPhone 定位追踪系统。

> 这个项目是我和 Claude 一起做的监控台的定位模块。它每 10 分钟查询一次 iPhone 位置，经过坐标转换和地理编码后推送到后端，还能识别"到家了""到学校了""离开常用地点"这些状态变化。

## 功能

- **持久 iCloud 会话**：daemon 模式运行，自动心跳保活，不需要反复输入密码
- **WGS-84 → GCJ-02 坐标转换**：iCloud 返回的是 WGS-84 坐标，中国地图 API（高德、百度）用的是 GCJ-02，不转换会偏移 500-2000 米
- **高德逆地理编码**：把经纬度翻译成人类看得懂的地址和附近地标
- **常用地点检测**：自定义多个地点（家、学校、公司……），自动检测到达和离开，触发事件推送
- **天气查询**：顺便拿当前位置的实时天气
- **远离警报**：离家超过设定距离且不在任何常用地点时，推送一次位置通知

## 架构概览

```
iPhone (iCloud)
    ↓ pyicloud (Find My iPhone API)
VPS Daemon (Python)
    ├── 坐标转换 WGS-84 → GCJ-02
    ├── 高德逆地理编码 → 地址 + POI
    ├── Open-Meteo → 天气
    ├── 常用地点匹配 + 事件检测
    └── 推送到后端 (HTTP POST)
```

## 核心思路

### 1. iCloud 会话保活

pyicloud 的会话 token 几个小时就会过期。如果用 cron 每 10 分钟跑一次脚本，token 过期后就需要重新输入密码和 2FA 验证码，非常烦。

解决方案是把脚本改成 **daemon 模式**——进程常驻，在内存里维持 `PyiCloudService` 实例。每 30 分钟做一次轻量心跳（调一下 `api.devices`），保持 session 不过期。

```python
from pyicloud import PyiCloudService

# 首次启动时认证（密码只在内存中，不写盘）
api = PyiCloudService("your@apple.id", password, china_mainland=True)

# 如果需要 2FA
if api.requires_2fa:
    code = input("验证码: ")
    api.validate_2fa_code(code)

# 之后 daemon 循环里定期调用，保持 session 活着
while True:
    # 每 10 分钟查位置
    poll_location(api)
    # 每 30 分钟心跳保活
    if time.time() - last_keepalive > 1800:
        list(api.devices)
        last_keepalive = time.time()
    time.sleep(30)
```

重启后 pyicloud 会先尝试从本地 session 文件恢复。如果 session 还有效就不需要密码：

```python
def try_resume():
    api = PyiCloudService("your@apple.id", china_mainland=True)  # 不传密码
    if api.requires_2fa:
        return None  # session 过期了
    try:
        list(api.devices)  # 测试 session 是否有效
        return api
    except:
        return None
```

### 2. 获取 iPhone 位置

```python
def find_iphone(api):
    for device in api.devices:
        status = device.status()
        if "iPhone" in status.get("deviceDisplayName", ""):
            return device
    return None

def poll_location(device):
    for _ in range(6):  # 最多重试 6 次
        device.status()  # 触发位置更新
        loc = device.location
        if loc and loc.get("latitude"):
            return loc
        time.sleep(5)
    return None
```

`device.location` 返回的坐标是 **WGS-84**（即 GPS 原始坐标）。在中国使用地图 API 之前必须转换。

### 3. WGS-84 → GCJ-02 坐标转换

中国有一套国家标准的坐标偏移算法（GCJ-02，俗称"火星坐标系"），所有中国地图服务商（高德、腾讯、百度）都使用这套坐标。如果你直接拿 iCloud 返回的 WGS-84 坐标去调高德 API，返回的地址会偏 500 到 2000 米。

转换算法的核心：

```python
def wgs84_to_gcj02(lat, lon):
    # 中国范围外不转换
    if lon < 72.004 or lon > 137.8347 or lat < 0.8293 or lat > 55.8271:
        return lat, lon

    a = 6378245.0  # 长半轴
    ee = 0.00669342162296594  # 偏心率平方

    # ⚠️ 关键：用偏移后的坐标计算，不是原始坐标
    x = lon - 105.0
    y = lat - 35.0

    dlat = -100.0 + 2.0*x + 3.0*y + 0.2*y*y + 0.1*x*y + 0.2*math.sqrt(abs(x))
    dlat += (20.0*math.sin(6.0*x*math.pi) + 20.0*math.sin(2.0*x*math.pi)) * 2.0/3.0
    dlat += (20.0*math.sin(y*math.pi) + 40.0*math.sin(y/3.0*math.pi)) * 2.0/3.0
    dlat += (160.0*math.sin(y/12.0*math.pi) + 320*math.sin(y*math.pi/30.0)) * 2.0/3.0

    dlon = 300.0 + x + 2.0*y + 0.1*x*x + 0.1*x*y + 0.1*math.sqrt(abs(x))
    dlon += (20.0*math.sin(6.0*x*math.pi) + 20.0*math.sin(2.0*x*math.pi)) * 2.0/3.0
    dlon += (20.0*math.sin(x*math.pi) + 40.0*math.sin(x/3.0*math.pi)) * 2.0/3.0
    dlon += (150.0*math.sin(x/12.0*math.pi) + 300.0*math.sin(x/30.0*math.pi)) * 2.0/3.0

    radlat = lat / 180.0 * math.pi
    magic = math.sin(radlat)
    magic = 1 - ee * magic * magic
    sqrtmagic = math.sqrt(magic)
    dlat = (dlat * 180.0) / ((a * (1-ee)) / (magic * sqrtmagic) * math.pi)
    dlon = (dlon * 180.0) / (a / sqrtmagic * math.cos(radlat) * math.pi)

    return lat + dlat, lon + dlon
```

> **踩坑记录**：网上很多版本的转换算法直接用 `lon` 和 `lat` 做计算，但正确的做法是先减去偏移量 `x = lon - 105.0`, `y = lat - 35.0`。我们最初的版本就是少了这步减法，导致定位偏了将近 2 公里，附近地标全是 2km 以外的。

### 4. 高德逆地理编码

拿到 GCJ-02 坐标后，调高德 API 得到可读的地址和附近地标（POI）：

```python
def amap_reverse(lat, lon):
    gcj_lat, gcj_lon = wgs84_to_gcj02(lat, lon)
    # ⚠️ 高德的参数顺序是 经度,纬度（不是纬度,经度）
    url = f"https://restapi.amap.com/v3/geocode/regeo?key={AMAP_KEY}&location={gcj_lon},{gcj_lat}&extensions=all&radius=200"
    data = requests.get(url).json()

    regeocode = data.get("regeocode", {})
    address = regeocode.get("formatted_address", "")
    pois = regeocode.get("pois", [])
    # 只取 500 米内的 POI
    nearby = [p["name"] for p in pois if float(p.get("distance", 9999)) <= 500][:3]
    return address, nearby
```

> **注意**：高德 API 的 `location` 参数是 **经度在前，纬度在后**（`lon,lat`），跟大多数人的直觉相反。而且 POI 的 `distance` 字段是字符串类型的浮点数（比如 `"123.343"`），用 `int()` 解析会报错，要用 `float()`。

### 5. 常用地点检测

定义一组常用地点，每次查到位置后计算距离，判断当前在哪个地点范围内：

```python
PLACES = [
    {"name": "home",   "label": "家",   "lat": xx.xxxx, "lon": xxx.xxxx, "radius": 200},
    {"name": "school", "label": "学校", "lat": xx.xxxx, "lon": xxx.xxxx, "radius": 200},
]

def find_named_place(lat, lon):
    best = None
    best_dist = float('inf')
    for p in PLACES:
        d = haversine(lat, lon, p["lat"], p["lon"])
        if d <= p["radius"] and d < best_dist:
            best = p
            best_dist = d
    return best
```

在 daemon 的内存里跟踪上一次的地点，检测状态变化：

```python
prev_place = None

def detect_event(current_place, dist_home):
    global prev_place
    event = None

    if current_place and current_place != prev_place:
        event = "arrived_" + current_place  # arrived_home, arrived_school
    elif not current_place and prev_place:
        event = "left_" + prev_place  # left_home, left_school

    # 离家超过 1km 且不在任何常用地点
    if not current_place and dist_home > 1000:
        event = event or "far_from_home"

    prev_place = current_place
    return event
```

### 6. 用 systemd 管理 daemon

写一个 systemd service 让它开机自启、崩溃自重启：

```ini
[Unit]
Description=iCloud Location Tracker Daemon
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 -u /path/to/geo-poll.py
Restart=no
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

首次启动需要手动认证（输入密码），之后 daemon 会保持 session 活着。如果 session 意外过期（比如 Apple 强制登出），daemon 会退出并记录日志，你需要重新手动认证一次。

```bash
# 首次启动（交互式输入密码）
ICLOUD_PW="你的密码" python3 -u /path/to/geo-poll.py
# 认证成功后 Ctrl+C，再用 systemd 启动
systemctl start geo-tracker
```

## 依赖

- Python 3.10+
- [pyicloud](https://github.com/picklepete/pyicloud)（`pip install pyicloud`）
- [高德开放平台](https://lbs.amap.com/) API Key（免费申请，每天 5000 次逆地理编码）
- [Open-Meteo](https://open-meteo.com/)（免费天气 API，无需 key）

## 安全提示

- **不要把 Apple ID 密码存在文件里**。用环境变量传入，认证完成后密码只存在进程内存中
- session 文件（`/tmp/pyicloud/`）包含登录 token，保护好文件权限
- 高德 API Key 有调用频率限制，10 分钟查一次完全够用

## 踩过的坑

| 坑 | 症状 | 原因 |
|---|---|---|
| 坐标转换少减偏移量 | 定位偏了 2km | WGS-84→GCJ-02 算法里 `x` 要用 `lon-105` 不是 `lon` |
| 高德参数顺序反了 | 返回的地址完全不对 | `location=经度,纬度`，不是纬度,经度 |
| POI 距离用 int() 解析 | `ValueError` 崩溃 | distance 是 `"123.343"` 这样的浮点字符串 |
| cron 模式 token 频繁过期 | 每天要重新登录 | 改成 daemon + 心跳保活解决 |
| pyicloud session 在 /tmp | 重启后 session 丢失 | 可以指定 `cookie_directory` 到持久路径 |

## License

MIT
