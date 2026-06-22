<p align="center">
  <img src="assets/cover.png" alt="iCloud Location Tracker" width="100%"/>
</p>

<h1 align="center">iCloud Location Tracker</h1>

<p align="center">
  <b>在 VPS 上持久运行的 iPhone 定位系统</b><br/>
  <sub>pyicloud · GCJ-02 坐标转换 · 高德逆地理编码 · 常用地点检测 · 天气</sub>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/python-3.10+-blue?style=flat-square&logo=python&logoColor=white" alt="Python"/>
  <img src="https://img.shields.io/badge/pyicloud-Find_My_iPhone-grey?style=flat-square&logo=apple&logoColor=white" alt="pyicloud"/>
  <img src="https://img.shields.io/badge/AMap-逆地理编码-green?style=flat-square" alt="AMap"/>
  <img src="https://img.shields.io/badge/license-MIT-pink?style=flat-square" alt="MIT"/>
</p>

<br/>

## 这是什么

每 10 分钟自动查一次 iPhone 的位置，把坐标翻译成可读地址，推送到你的后端。跑在 VPS 上，daemon 模式常驻，不需要反复登录 iCloud。

除了基本定位，还支持：

- **常用地点识别**：到家了、到学校了、离开了，自动推送通知
- **远离警报**：离家超过设定距离时提醒
- **实时天气**：顺手拿当前位置的天气

```
iPhone
    ↓  pyicloud（Find My iPhone API）
VPS Daemon（Python 常驻进程）
    ├── 坐标转换：WGS-84 → GCJ-02
    ├── 高德 API：坐标 → 地址 + 附近地标
    ├── Open-Meteo：坐标 → 天气
    ├── 常用地点匹配 + 到达/离开事件
    └── 推送到后端（HTTP POST）
```

<p align="center"><img src="assets/divider.png" width="500"/></p>

## 快速开始

> 需要：一台 VPS、Python 3.10+、[高德开放平台](https://lbs.amap.com/) API Key（免费申请）。

```bash
# 1. 安装依赖
pip install pyicloud requests

# 2. 首次认证（交互式，需要输密码和 2FA 验证码）
ICLOUD_PW="你的密码" python3 -u geo-poll.py

# 3. 确认正常后 Ctrl+C，用 systemd 接管
sudo systemctl start geo-tracker
sudo journalctl -u geo-tracker -f   # 看日志
```

> [!IMPORTANT]
> 密码只在首次认证时使用，之后只存在进程内存中。**不要把密码写进文件。**

<p align="center"><img src="assets/divider-gold.png" width="500"/></p>

## 详细教程

<br/>

### <img src="assets/icon-daemon.png" width="28"/> iCloud 会话管理：daemon 模式

pyicloud 登录后会拿到一个 session token，但这个 token 只有几个小时寿命。如果用 cron 定时跑脚本，token 过期就要重新输密码和验证码，在 VPS 上根本不现实。

解决办法是 daemon 模式：进程常驻，`PyiCloudService` 实例一直活在内存里，再加一个心跳定时调用 API，iCloud 就不会让 session 过期。

> [!TIP]
> 简单理解：cron 是每次敲门都要重新验证身份，daemon 是进门之后一直待着，隔一会儿打个招呼就行。

```python
from pyicloud import PyiCloudService
import time

api = PyiCloudService("your@apple.id", password, china_mainland=True)

if api.requires_2fa:
    code = input("验证码: ")
    api.validate_2fa_code(code)

last_keepalive = time.time()

while True:
    poll_location(api)  # 每 10 分钟查位置

    # 每 30 分钟心跳保活
    if time.time() - last_keepalive > 1800:
        list(api.devices)
        last_keepalive = time.time()

    time.sleep(30)
```

重启后可以尝试从本地缓存恢复 session，不需要密码：

```python
def try_resume():
    api = PyiCloudService("your@apple.id", china_mainland=True)  # 不传密码
    if api.requires_2fa:
        return None  # token 过期了
    try:
        list(api.devices)
        return api
    except:
        return None
```

> [!NOTE]
> pyicloud 默认把 session 存在 `/tmp/pyicloud/`，有些系统重启时会清空 `/tmp`。建议指定 `cookie_directory` 到持久路径。

<p align="center"><img src="assets/divider.png" width="500"/></p>

### <img src="assets/icon-coord.png" width="28"/> 坐标转换：WGS-84 → GCJ-02

iCloud 返回的是 GPS 标准坐标（WGS-84），但中国所有地图服务用的是偏移过的坐标系 GCJ-02（"火星坐标系"）。不转换的话，拿 GPS 坐标去调高德 API，返回的地址会**偏 500 到 2000 米**。

转换算法的核心是对经纬度施加一组非线性偏移。有一个特别容易错的地方：

```python
# 要先减去参考点坐标，再做后续计算
x = lon - 105.0   # ← 很多网上版本漏了这步
y = lat - 35.0

dlat = -100.0 + 2.0*x + 3.0*y + 0.2*y*y + 0.1*x*y + 0.2*math.sqrt(abs(x))
# ... 后续三角函数偏移都基于 x 和 y
```

漏掉这步减法不会报错，但偏移方向完全错误。

> [!TIP]
> **验证方法**：拿一个已知位置的 GPS 坐标转一次，然后在高德地图上搜转换后的坐标，看落点是否正确。

<p align="center"><img src="assets/divider-gold.png" width="500"/></p>

### <img src="assets/icon-map.png" width="28"/> 高德逆地理编码

拿到 GCJ-02 坐标后，调高德的逆地理编码 API 可以把经纬度翻译成可读地址，同时获取附近的 POI（兴趣点）。

```python
def amap_reverse(lat, lon):
    gcj_lat, gcj_lon = wgs84_to_gcj02(lat, lon)
    # ⚠️ 高德参数顺序：经度在前，纬度在后
    url = f"...&location={gcj_lon},{gcj_lat}&extensions=all&radius=200"
    data = requests.get(url).json()
    pois = data["regeocode"]["pois"]
    nearby = [p["name"] for p in pois if float(p.get("distance", 9999)) <= 500][:3]
    return data["regeocode"]["formatted_address"], nearby
```

两个容易踩的坑：

- **参数顺序**：高德是 `经度,纬度`（lon,lat），跟常规的 `lat,lon` 相反，搞反了不报错但地址完全不对
- **距离字段类型**：POI 的 `distance` 是字符串浮点数如 `"123.343"`，用 `int()` 解析会报错，要用 `float()`

<p align="center"><img src="assets/divider.png" width="500"/></p>

### <img src="assets/icon-place.png" width="28"/> 常用地点检测

预定义几个地点，每次查到位置后用 Haversine 公式算距离，判断是否在某个地点的半径范围内：

```python
PLACES = [
    {"name": "home",   "label": "家",   "lat": xx.xxxx, "lon": xxx.xxxx, "radius": 200},
    {"name": "school", "label": "学校", "lat": xx.xxxx, "lon": xxx.xxxx, "radius": 200},
]
```

在 daemon 内存里记住上一次的状态，跟当前对比就能检测事件：

| 上次 | 这次 | 事件 |
|:---:|:---:|:---:|
| 不在任何地点 | 在家 | `arrived_home` |
| 在学校 | 不在任何地点 | `left_school` |
| 离家 >1km | 不在任何地点 | `far_from_home` |

> [!TIP]
> `radius` 建议设 200 米。GPS 有 10~50 米误差，设太小会在边缘反复触发事件，设太大会失去精度。

<p align="center"><img src="assets/divider-gold.png" width="500"/></p>

### <img src="assets/icon-weather.png" width="28"/> 天气

用 [Open-Meteo](https://open-meteo.com/)，完全免费，不需要 Key，传经纬度就能拿到温度、湿度、风速。

注意 Open-Meteo 用 WGS-84 坐标，**不需要转 GCJ-02**。只有中国地图服务才需要转。

<p align="center"><img src="assets/divider.png" width="500"/></p>

### <img src="assets/icon-gear.png" width="28"/> systemd 服务

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

- `-u`：关闭 Python 输出缓冲，日志实时出现在 journalctl 里
- `Restart=no`：退出通常意味着 session 过期，需要人工重新认证，自动重启没用
- 第一次必须手动跑（要输密码），之后再交给 systemd

```bash
ICLOUD_PW="你的密码" python3 -u /path/to/geo-poll.py  # 首次认证
# Ctrl+C 后
sudo systemctl daemon-reload && sudo systemctl start geo-tracker
sudo systemctl enable geo-tracker  # 开机自启
```

<p align="center"><img src="assets/divider-gold.png" width="500"/></p>

## 常见问题

| 问题 | 表现 | 原因 | 解决 |
|:---|:---|:---|:---|
| 坐标偏移 2km | 地址和附近地标全不对 | GCJ-02 转换时少了 `x = lon - 105` 的减法 | 确认算法里有减参考点这步 |
| 地址完全不对 | 人在 A 城市，返回 B 城市 | 高德参数顺序反了，应该是 `lon,lat` | 经度在前纬度在后 |
| `ValueError` 崩溃 | 解析 POI 距离时报错 | 距离是 `"123.343"` 字符串 | 用 `float()` 不要用 `int()` |
| 每天要重新登录 | session 频繁过期 | cron 模式没有心跳保活 | 改 daemon + 30 分钟心跳 |
| 重启后要重新认证 | session 文件丢失 | 默认存在 `/tmp`，重启被清空 | 指定持久化 `cookie_directory` |

<p align="center"><img src="assets/divider.png" width="500"/></p>

## 依赖

| 名称 | 用途 | 备注 |
|:---:|:---:|:---:|
| Python 3.10+ | 运行环境 | |
| [pyicloud](https://github.com/picklepete/pyicloud) | iCloud API | `pip install pyicloud` |
| [高德开放平台](https://lbs.amap.com/) Key | 逆地理编码 | 免费，每天 5000 次 |
| [Open-Meteo](https://open-meteo.com/) | 天气 | 免费，无需 Key |

<br/>

## 安全提示

- 密码不要写进文件，通过环境变量传入
- session 文件包含登录 token，保护文件权限（`chmod 700`）
- 定位数据是敏感信息，传输链路要加密

---

<p align="center"><sub>MIT License</sub></p>
