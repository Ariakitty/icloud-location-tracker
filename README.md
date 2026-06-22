<p align="center">
  <img src="assets/cover.png" alt="iCloud Location Tracker" width="100%"/>
</p>

<h1 align="center">iCloud Location Tracker</h1>

<p align="center">
  在 VPS 上跑一个 Python daemon，每 10 分钟自动查一次 iPhone 位置<br/>
  坐标转换 · 地址解析 · 常用地点识别 · 天气 · 事件推送
</p>

<p align="center">
  <img src="https://img.shields.io/badge/python-3.10+-94B8C9?style=flat-square&logo=python&logoColor=white"/>
  <img src="https://img.shields.io/badge/pyicloud-iCloud_API-C9928E?style=flat-square&logo=apple&logoColor=white"/>
  <img src="https://img.shields.io/badge/AMap-逆地理编码-98D4BB?style=flat-square"/>
  <img src="https://img.shields.io/badge/license-MIT-FDF5EF?style=flat-square"/>
</p>

<p align="center"><img src="assets/sep.png" width="120"/></p>

<img src="assets/h-about.png" width="100%"/>

用 [pyicloud](https://github.com/picklepete/pyicloud) 调 iCloud 的 Find My iPhone 接口拿到 iPhone 的 GPS 坐标，然后做几件事：转成中国地图能用的坐标、查出对应的地址和附近地标、判断在不在常用地点（家、学校之类的）、顺便查个天气，最后打包推送到你的后端。

整个东西跑在 VPS 上，是一个常驻进程，不需要反复登录，也不需要开 App。

<p align="center"><img src="assets/features.png" width="700"/></p>

<p align="center"><img src="assets/arch.png" width="700"/></p>

<p align="center"><img src="assets/sep.png" width="120"/></p>

<img src="assets/h-quickstart.png" width="100%"/>

需要准备：一台 VPS、Python 3.10+、一个[高德开放平台](https://lbs.amap.com/) API Key（免费申请就行）。

```bash
pip install pyicloud requests
```

第一次跑需要输密码和 2FA 验证码，通过环境变量传密码进去：

```bash
ICLOUD_PW="你的密码" python3 -u geo-poll.py
```

看到定位数据正常输出之后 `Ctrl+C`，然后交给 systemd：

```bash
sudo systemctl start geo-tracker
sudo journalctl -u geo-tracker -f   # 看日志
```

之后就不用管了，daemon 会自己保持登录态。

> [!IMPORTANT]
> 密码只在第一次认证时用，之后只存在进程内存里，不会写到文件。

<p align="center"><img src="assets/sep.png" width="120"/></p>

<img src="assets/h-session.png" width="100%"/>

iCloud 的登录态（session token）有效期只有几个小时。如果用 cron 定时跑脚本，token 过期就得重新输密码和验证码，VPS 上没法这么搞。

所以要用 daemon 模式：进程不退出，`PyiCloudService` 实例一直活在内存里。再加一个每 30 分钟的心跳请求（随便调个轻量 API 就行），iCloud 就不会让 session 过期。

```python
api = PyiCloudService("your@apple.id", password, china_mainland=True)

if api.requires_2fa:
    code = input("验证码: ")
    api.validate_2fa_code(code)

last_keepalive = time.time()

while True:
    poll_location(api)

    if time.time() - last_keepalive > 1800:
        list(api.devices)          # 心跳，告诉 iCloud 我还在
        last_keepalive = time.time()

    time.sleep(30)
```

daemon 重启的时候，pyicloud 会尝试从本地缓存恢复 session（默认在 `/tmp/pyicloud/`），如果 token 还没过期就不用重新输密码：

```python
def try_resume():
    api = PyiCloudService("your@apple.id", china_mainland=True)  # 不传密码
    if api.requires_2fa:
        return None       # 过期了
    list(api.devices)     # 试一下能不能用
    return api
```

> [!NOTE]
> `/tmp` 在有些系统重启后会清空，session 就没了。可以在创建 `PyiCloudService` 时指定 `cookie_directory` 到一个不会被清的路径。

<p align="center"><img src="assets/sep.png" width="120"/></p>

<img src="assets/h-coord.png" width="100%"/>

iCloud 返回的是标准 GPS 坐标（WGS-84），全世界通用。但中国所有地图服务（高德、腾讯、百度）用的是一套偏移过的坐标系 GCJ-02，也叫火星坐标系。

不转的话，拿 GPS 坐标直接去查地址会**偏 500 到 2000 米**，而且不会报错，返回的 JSON 格式完全正常，就是内容不对。

转换算法是公开的，对经纬度做一组非线性偏移。最关键的一步：

```python
x = lon - 105.0    # 先减参考点
y = lat - 35.0

# 后面的偏移计算全部基于 x 和 y，不是原始的 lon / lat
dlat = -100.0 + 2.0*x + 3.0*y + 0.2*y*y + ...
dlon = 300.0 + x + 2.0*y + 0.1*x*x + ...
```

网上很多版本漏了这步减法，直接拿原始坐标算，结果偏移方向完全不对。改起来就一行的事，但排查起来很头疼，因为程序不会报错。

> [!TIP]
> 验证方法：拿一个你知道确切位置的地点的 GPS 坐标转一下，然后去高德地图搜转换后的坐标，看落点对不对。

<p align="center"><img src="assets/sep.png" width="120"/></p>

<img src="assets/h-geocode.png" width="100%"/>

拿到 GCJ-02 坐标之后就可以调高德的逆地理编码 API 了，把经纬度变成"XX 市 XX 区 XX 路"这种可读地址，还能拿到附近的地标（商场、学校、餐厅之类的）。

```python
gcj_lat, gcj_lon = wgs84_to_gcj02(lat, lon)

# 高德参数顺序：经度在前，纬度在后（跟常规相反）
url = f"...&location={gcj_lon},{gcj_lat}&extensions=all&radius=200"

data = requests.get(url).json()
address = data["regeocode"]["formatted_address"]
pois = data["regeocode"]["pois"]
nearby = [p["name"] for p in pois if float(p.get("distance", 9999)) <= 500][:3]
```

两个容易踩的地方：

- `location` 参数是**经度在前纬度在后**，反了不会报错，但地址完全不对
- POI 的 `distance` 字段是字符串浮点数（比如 `"123.343"`），不能用 `int()` 解析，要用 `float()`

> [!NOTE]
> 高德免费版每天 5000 次调用，10 分钟查一次一天才 144 次，完全够。`extensions=all` 才会返回 POI 列表，默认只返回地址。

<p align="center"><img src="assets/sep.png" width="120"/></p>

<img src="assets/h-places.png" width="100%"/>

光有坐标和地址还不够，很多时候你想知道的是"到家了吗""什么时候离开的学校"这种状态。

做法是先定义一组常用地点：

```python
PLACES = [
    {"name": "home",   "label": "家",   "lat": xx.xxxx, "lon": xxx.xxxx, "radius": 200},
    {"name": "school", "label": "学校", "lat": xx.xxxx, "lon": xxx.xxxx, "radius": 200},
    {"name": "office", "label": "公司", "lat": xx.xxxx, "lon": xxx.xxxx, "radius": 300},
]
```

每次拿到坐标后用 Haversine 公式算和每个地点的距离，小于 radius 就算"在这个地方"。

然后是事件检测，在内存里记住上一次在哪，跟这次比较：

```python
prev_place = None

def detect_event(current_place, dist_home):
    global prev_place
    event = None

    if current_place and current_place != prev_place:
        event = "arrived_" + current_place      # 到了某个地方

    elif not current_place and prev_place:
        event = "left_" + prev_place             # 离开了某个地方

    if not current_place and dist_home > 1000:
        event = event or "far_from_home"         # 离家太远

    prev_place = current_place
    return event
```

| 上次状态 | 当前状态 | 触发事件 |
|:---:|:---:|:---:|
| 不在任何地点 | 在家 | `arrived_home` |
| 在学校 | 不在任何地点 | `left_school` |
| 不在任何地点 | 离家 >1km | `far_from_home` |

**关于 radius 的选择**

GPS 本身有 10~50 米的误差，室内或者建筑密集的地方可能更大。radius 设太小的话（比如 50 米），定位在边缘跳动会反复触发"到了/走了"。设太大（比如 1km）就失去意义了。200 米对住宅和学校来说比较合适，大型园区可以设到 300~500 米。

**防抖**

`far_from_home` 这个事件只触发一次。在 daemon 里加一个 flag，触发过之后就不再重复推送，直到回到某个常用地点或者回家才重置。不然每 10 分钟都会收到"你离家很远"的通知，很烦。

<p align="center"><img src="assets/sep.png" width="120"/></p>

<img src="assets/h-weather.png" width="100%"/>

坐标都有了，查天气几乎零成本。[Open-Meteo](https://open-meteo.com/) 完全免费，不要 Key，直接传经纬度就行。

注意它用的是 WGS-84 坐标，不用转 GCJ-02。只有中国地图 API 才需要转。

<p align="center"><img src="assets/sep.png" width="120"/></p>

<img src="assets/h-visual.png" width="100%"/>

daemon 在后台默默跑着，但你可能想看到位置在地图上长什么样。一个最简单的做法是写一个 HTML 页面，用高德的 JS API 在地图上画点。

```html
<script src="https://webapi.amap.com/maps?v=2.0&key=YOUR_KEY"></script>
<script>
  const map = new AMap.Map('map', { zoom: 14 });

  // 从你的后端拿最近的位置数据
  fetch('/api/geo/latest')
    .then(r => r.json())
    .then(data => {
      new AMap.Marker({
        position: [data.lon, data.lat],  // GCJ-02 坐标
        map: map
      });
      map.setCenter([data.lon, data.lat]);
    });
</script>
```

如果你想看历史轨迹，可以把每次查到的坐标存下来，然后用 `AMap.Polyline` 画一条线：

```javascript
const path = history.map(p => [p.lon, p.lat]);

new AMap.Polyline({
  path: path,
  strokeColor: '#98D4BB',
  strokeWeight: 4,
  map: map
});
```

也可以用不同颜色标注常用地点的范围（`AMap.Circle`），一眼就能看出人在不在某个地点附近。

> [!TIP]
> 高德 JS API 也是免费的，每天有额度限制但个人用完全够。显示的地图自动就是 GCJ-02 坐标系，不需要额外转换。

<p align="center"><img src="assets/sep.png" width="120"/></p>

<img src="assets/h-systemd.png" width="100%"/>

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

- `-u` 让 Python 不缓冲输出，`journalctl` 能实时看到日志
- `Restart=no` 是因为退出通常意味着 session 过期，需要人工重新认证，自动重启也没用
- 第一次必须手动跑（要输密码），跑通之后再交给 systemd

```bash
ICLOUD_PW="你的密码" python3 -u /path/to/geo-poll.py    # 手动认证
# 看到定位输出后 Ctrl+C

sudo systemctl daemon-reload
sudo systemctl start geo-tracker
sudo systemctl enable geo-tracker      # 开机自启
```

<p align="center"><img src="assets/sep.png" width="120"/></p>

<img src="assets/h-pitfalls.png" width="100%"/>

| 问题 | 表现 | 原因 | 解决 |
|:---|:---|:---|:---|
| 坐标偏了 2km | 地址和地标全不对 | GCJ-02 转换少了 `x = lon - 105` | 补上减法 |
| 地址完全对不上 | 查出来是别的城市 | 高德参数顺序反了 | `lon,lat` 经度在前 |
| `ValueError` 崩溃 | 解析 POI 距离报错 | distance 是 `"123.343"` 字符串 | 用 `float()` |
| 每天要重新登录 | session 过期 | cron 没有心跳保活 | 改 daemon + 心跳 |
| 重启后要重新认证 | session 丢了 | 默认存在 `/tmp` | 指定持久目录 |
| 到家/离家反复触发 | GPS 在边缘跳动 | radius 太小 | 设到 200m 以上 |

<p align="center"><img src="assets/sep.png" width="120"/></p>

## 依赖

| 名称 | 用途 | 备注 |
|:---:|:---:|:---:|
| Python 3.10+ | 运行环境 | |
| [pyicloud](https://github.com/picklepete/pyicloud) | iCloud API | `pip install pyicloud` |
| [高德开放平台](https://lbs.amap.com/) Key | 逆地理编码 + 地图 | 免费申请 |
| [Open-Meteo](https://open-meteo.com/) | 天气 | 免费，不要 Key |

## 安全

- 密码通过环境变量传入，不写文件
- session 文件有登录 token，权限设好（`chmod 700`）
- 定位数据是敏感信息，传输要加密

---

<p align="center"><sub>MIT License</sub></p>
