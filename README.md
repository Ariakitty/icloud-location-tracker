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

## 这个项目是做什么的？

这个项目的起因很简单：我想在 VPS 上随时知道 iPhone 的位置。

听起来不难，但真正动手做的时候踩了不少坑。iCloud 的登录态会过期、中国的地图坐标跟 GPS 坐标不一样、高德 API 有各种小陷阱……这篇教程把我从零搭建过程中遇到的问题和解决思路都记录下来了，希望能帮后来的人少走弯路。

最终实现的效果：

- **每 10 分钟自动查一次 iPhone 位置**，daemon 模式常驻运行，不需要反复登录
- **坐标自动转换**，GPS 坐标转成中国地图能用的格式
- **经纬度翻译成地址**，看到的不是一串数字，而是"深圳市南山区XX路"
- **识别常用地点**，到家了、到学校了、离开了，自动推送通知
- **离家太远时警报**，超过设定距离自动提醒
- **顺手查天气**，反正坐标都有了，零成本加一个

```
你的 iPhone
    ↓  pyicloud 通过 Find My iPhone API 拿到 GPS 坐标
你的 VPS（Python daemon，常驻进程）
    ├── 坐标转换：GPS 坐标 → 中国地图坐标
    ├── 高德 API：坐标 → 可读地址 + 附近地标
    ├── Open-Meteo：坐标 → 实时天气
    ├── 匹配常用地点 → 检测到达 / 离开事件
    └── 打包推送到你的后端
```

<p align="center"><img src="assets/divider.png" width="500"/></p>

## 快速开始

> 假设你已经有一台 VPS、Python 3.10+ 和一个[高德开放平台](https://lbs.amap.com/) API Key（免费申请）。

```bash
# 1. 安装依赖
pip install pyicloud requests

# 2. 首次认证（需要交互式输入密码和 2FA 验证码）
ICLOUD_PW="你的密码" python3 -u geo-poll.py

# 3. 确认定位数据正常输出后 Ctrl+C，然后用 systemd 接管
sudo systemctl start geo-tracker
sudo journalctl -u geo-tracker -f   # 看日志
```

> [!IMPORTANT]
> 密码只在首次认证时使用，认证完成后只存在进程内存中。**不要把密码写进任何文件。**

下面是每个模块的详细讲解。如果你只想快速跑起来，到这里就够了。

<p align="center"><img src="assets/divider-gold.png" width="500"/></p>

## 深入讲解

<br/>

### <img src="assets/icon-daemon.png" width="28"/> iCloud 会话管理：为什么用 daemon 不用 cron

我最开始的想法很自然：写个 Python 脚本，用 cron 每 10 分钟跑一次，简单直接。但试了之后发现完全走不通。

**问题出在 iCloud 的登录态（session）上。** pyicloud 这个库在你登录后会把 session token 存到本地文件里，下次启动时会尝试复用。但这个 token 只有几个小时的寿命，一旦过期，cron 下一次触发就会认证失败。在 VPS 上你没法每隔几小时手动输一次密码和验证码，这条路走不通。

> [!TIP]
> 打个比方：cron 就像每隔 10 分钟敲一次门，每次都要重新证明"我是我"。daemon 就像进了门之后一直坐在客厅里，隔一会儿跟主人打个招呼，让他知道你还在。

**解决方案是 daemon 模式**：Python 进程不退出，`PyiCloudService` 实例一直活在内存里。每 30 分钟做一次轻量的 API 调用（比如 `list(api.devices)`），iCloud 就不会让 session 过期。

```python
from pyicloud import PyiCloudService
import time

# 首次启动时认证（密码通过环境变量传入）
api = PyiCloudService("your@apple.id", password, china_mainland=True)

if api.requires_2fa:
    code = input("验证码: ")
    api.validate_2fa_code(code)

# daemon 主循环
last_keepalive = time.time()

while True:
    poll_location(api)  # 每 10 分钟查一次位置

    # 每 30 分钟心跳保活
    if time.time() - last_keepalive > 1800:
        list(api.devices)  # 轻量调用，让 iCloud 知道我们还在
        last_keepalive = time.time()

    time.sleep(30)
```

为什么心跳间隔选 30 分钟？这是试出来的甜蜜点。太频繁浪费请求，太长有时候赶不上 token 刷新窗口。

还有一个细节：daemon 重启之后怎么办？pyicloud 会把 session 缓存在本地（默认 `/tmp/pyicloud/`），重启时可以先尝试无密码恢复：

```python
def try_resume():
    api = PyiCloudService("your@apple.id", china_mainland=True)  # 不传密码
    if api.requires_2fa:
        return None  # session 过期了，需要重新认证
    try:
        list(api.devices)  # 验证一下是否真的有效
        return api
    except:
        return None
```

> [!NOTE]
> `/tmp` 在有些系统上重启后会被清空。如果你发现每次重启都要重新认证，可以把 `cookie_directory` 指向一个持久路径，比如 `/var/lib/geo-tracker/session/`。

<p align="center"><img src="assets/divider.png" width="500"/></p>

### <img src="assets/icon-coord.png" width="28"/> 坐标转换：WGS-84 → GCJ-02

这一步是整个项目里**最容易出错、也最难调试**的部分。

先说背景：GPS 卫星给出的坐标用的是一套叫 WGS-84 的全球标准坐标系，你在世界上任何地方拿到的 GPS 坐标都是这个格式。但中国出于测绘安全考虑，所有国内地图服务必须使用一套偏移过的坐标系，叫做 **GCJ-02**，江湖人称**"火星坐标系"**。高德、腾讯地图用的都是 GCJ-02。

这意味着什么？如果你拿 iCloud 返回的 GPS 坐标直接去调高德 API，返回的地址会**偏 500 到 2000 米**。你明明在家，查出来的地址可能是 2 公里外的某个地方。而且这种错误**不会报错**，API 照常返回一个格式完全正确的 JSON，只是内容对不上。

转换算法是公开的，核心思路是对经纬度施加一组非线性的三角函数偏移。代码不长但有一个特别阴险的坑：

```python
# 核心：以 (35, 105) 为参考点计算偏移量
x = lon - 105.0   # ← 这步减法很多网上版本漏了
y = lat - 35.0

# 后续所有偏移计算都用 x 和 y，不是原始的 lon 和 lat
dlat = -100.0 + 2.0*x + 3.0*y + 0.2*y*y + ...
dlon = 300.0 + x + 2.0*y + 0.1*x*x + ...
```

网上流传的很多版本直接用原始经纬度代入计算，漏掉了这步减法。结果就是偏移方向和幅度都不对，但程序不会报错，你只能靠肉眼对比地图才能发现问题。

> [!TIP]
> **怎么验证？** 找一个你确切知道位置的地点（比如你家楼下的某个标志性建筑），拿它的 GPS 坐标转一次，然后去高德地图搜转换后的坐标，看落点对不对。如果偏了几百米以上，转换算法肯定有问题。

<p align="center"><img src="assets/divider-gold.png" width="500"/></p>

### <img src="assets/icon-map.png" width="28"/> 高德逆地理编码：经纬度变成人话

有了正确的 GCJ-02 坐标，就可以调高德的逆地理编码 API 了。它能把一串经纬度翻译成"XX市XX区XX路XX号"这样的可读地址，还能告诉你附近有什么（POI，即兴趣点，比如商场、学校、餐厅）。

```python
def amap_reverse(lat, lon):
    gcj_lat, gcj_lon = wgs84_to_gcj02(lat, lon)

    # ⚠️ 注意：高德的参数顺序是 经度,纬度（跟直觉相反）
    url = f"https://restapi.amap.com/v3/geocode/regeo?key={AMAP_KEY}&location={gcj_lon},{gcj_lat}&extensions=all&radius=200"

    data = requests.get(url).json()
    regeocode = data.get("regeocode", {})
    address = regeocode.get("formatted_address", "")
    pois = regeocode.get("pois", [])

    # 只取 500 米内的 POI
    nearby = [p["name"] for p in pois if float(p.get("distance", 9999)) <= 500][:3]
    return address, nearby
```

这里有两个坑，我都踩了：

**第一个：参数顺序。** 高德 API 的 `location` 参数是**经度在前、纬度在后**（`lon,lat`）。我们日常说的是"纬度经度"，pyicloud 返回的也是 `latitude, longitude`，但高德偏偏要反过来。搞反了不会报错，只会返回一个完全不对的地址。

**第二个：距离字段的类型。** POI 列表里的 `distance` 字段是字符串类型的浮点数，比如 `"123.343"`。用 `int()` 解析会直接炸，必须用 `float()`。

> [!NOTE]
> `extensions=all` 让高德返回完整的 POI 列表。如果你不需要附近地标，用默认的 `extensions=base` 可以加快响应。高德免费版每天 5000 次调用，10 分钟一次一天才 144 次，完全够用。

<p align="center"><img src="assets/divider.png" width="500"/></p>

### <img src="assets/icon-place.png" width="28"/> 常用地点检测：到家了、出门了

光知道坐标和地址还不够，我还想知道"到家了没""什么时候离开的学校"。做法是预先定义一组常用地点，每次查到位置后算距离，看是否在某个地点的范围内。

```python
PLACES = [
    {"name": "home",   "label": "家",   "lat": xx.xxxx, "lon": xxx.xxxx, "radius": 200},
    {"name": "school", "label": "学校", "lat": xx.xxxx, "lon": xxx.xxxx, "radius": 200},
]
```

距离用 Haversine 公式算（考虑地球曲率的两点距离），如果距离小于设定半径就认为"在这个地方"。

更有意思的是**事件检测**。不只是知道"现在在哪"，还要知道"刚到了"还是"刚离开了"。做法是在 daemon 内存里记住上一次的位置状态，跟这次对比：

| 上次 | 这次 | 产生的事件 |
|:---:|:---:|:---:|
| 不在任何地点 | 在家 | `arrived_home` 到家了 |
| 在学校 | 不在任何地点 | `left_school` 离开学校了 |
| 不在任何地点 | 不在任何地点 + 离家 >1km | `far_from_home` 离家太远了 |

> [!TIP]
> `radius` 设成 200 米比较合适。GPS 本身有 10~50 米误差，建筑物遮挡可能更大。设太小会在边缘反复触发到达/离开事件，设太大会失去精度。

<p align="center"><img src="assets/divider-gold.png" width="500"/></p>

### <img src="assets/icon-weather.png" width="28"/> 天气

既然坐标都有了，顺手查一下天气几乎零成本。用的是 [Open-Meteo](https://open-meteo.com/)，完全免费，不需要 API Key，直接传经纬度就能拿到温度、湿度、风速、天气状况。

唯一要注意的是：Open-Meteo 用的是 WGS-84 坐标（标准 GPS 坐标），**不需要做 GCJ-02 转换**。只有调中国地图服务的 API 才需要转。

<p align="center"><img src="assets/divider.png" width="500"/></p>

### <img src="assets/icon-gear.png" width="28"/> 用 systemd 跑起来

daemon 写好之后，用 systemd 管理它，开机自启、崩溃有日志。

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

几个细节：
- **`-u` 参数**：让 Python 不缓冲输出，日志能实时出现在 `journalctl` 里
- **`Restart=no`**：不自动重启，因为如果退出了通常是 session 过期，需要人工重新认证
- **第一次必须手动跑**：需要输密码和验证码，不能直接用 systemd 启动

```bash
# 第一次：手动认证
ICLOUD_PW="你的密码" python3 -u /path/to/geo-poll.py
# 确认正常后 Ctrl+C

# 之后：systemd 接管
sudo systemctl daemon-reload
sudo systemctl start geo-tracker
sudo systemctl enable geo-tracker  # 开机自启
```

<p align="center"><img src="assets/divider-gold.png" width="500"/></p>

## 踩过的坑

做这个项目基本就是一路踩坑过来的。每个坑都不难解决，但如果你不知道它在那里，调试起来特别痛苦。

<br/>

**坐标转换少了一步减法**

> 症状：定位能正常跑，高德也返回了地址，但地址偏了将近 2 公里，附近地标全是远处的建筑。
>
> 原因：WGS-84 → GCJ-02 的算法里要先减去参考点 `(105, 35)`，很多网上的版本漏了这步。偏移方向和幅度都不对，但不会报错，只能肉眼发现。
>
> 教训：坐标相关的 bug 特别阴，因为返回的数据格式完全正常，内容就是不对。一定要用已知地点验证。

**高德 API 参数顺序反了**

> 症状：返回的地址完全对不上，人在深圳结果查出来是某个不知名的城市。
>
> 原因：高德的 `location` 参数是 `经度,纬度`（lon,lat），不是 `纬度,经度`（lat,lon）。反了不报错，照常返回结果。
>
> 教训：调地图 API 第一件事就是确认参数顺序，不同服务商的约定不一样。

**POI 距离用 `int()` 解析**

> 症状：daemon 跑着跑着突然崩溃，报 `ValueError`。
>
> 原因：高德返回的距离字段是 `"123.343"` 这样的字符串浮点数，`int()` 解析不了。
>
> 教训：不要假设 API 返回的数值是整数，先打印一条原始响应看看。

**cron 模式 session 频繁过期**

> 症状：每天要重新输密码和验证码。
>
> 原因：cron 每次启动新进程，没有心跳保活，token 几小时就过期。
>
> 教训：需要长期保持登录态的服务，daemon 比 cron 靠谱得多。

**session 缓存目录在 `/tmp`**

> 症状：VPS 重启后必须重新认证。
>
> 原因：pyicloud 默认把 session 存在 `/tmp/pyicloud/`，有些系统重启时会清空 `/tmp`。
>
> 教训：指定 `cookie_directory` 到持久路径。

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

- **密码不要写进文件**，通过环境变量传入，认证后只存在进程内存中
- session 文件包含登录 token，保护好文件权限（`chmod 700`）
- 定位数据是敏感信息，传输链路要加密

<br/>

---

<p align="center">
  <sub>MIT License</sub>
</p>
