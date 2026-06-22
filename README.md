<p align="center">
  <img src="assets/cover.png" alt="iCloud Location Tracker" width="100%"/>
</p>

<h1 align="center">iCloud Location Tracker</h1>

<p align="center">
  自动追踪 iPhone 位置，告诉你"人在哪、在不在家、附近有什么、天气怎么样"
</p>

<p align="center">
  <img src="https://img.shields.io/badge/python-3.10+-94B8C9?style=flat-square&logo=python&logoColor=white"/>
  <img src="https://img.shields.io/badge/license-MIT-FDF5EF?style=flat-square"/>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/🐱_Aria-&-C9928E?style=for-the-badge&labelColor=FDF5EF"/>
  <img src="https://img.shields.io/badge/🦊_Claude-collaboration-94B8C9?style=for-the-badge&labelColor=FDF5EF"/>
</p>

> 💌 本篇教程受到 **North&盏** 的 [Geo 全球定位系统攻略](https://github.com)的启发。原版是面向全球的 pyicloud 定位方案（OSM、Google Maps、MCP 集成等），我们在这个基础上做了中国场景的适配：WGS-84 → GCJ-02 坐标转换、高德逆地理编码、常用地点事件检测、高德地图可视化。不限于中国大陆的话推荐看看原版。

<p align="center"><img src="assets/sep.png" width="120"/></p>

<img src="assets/h-intro.png" width="100%"/>

你有没有想过，能不能让一台服务器**自动帮你查 iPhone 在哪**，然后告诉你地址、附近有什么、天气怎么样？

这个项目就是做这件事的。

苹果手机有一个"查找我的 iPhone"功能（就是你丢手机时用来定位的那个），它的数据其实可以通过代码来读取。我们用一个叫 pyicloud 的 Python 库来做这件事，它相当于用代码模拟了"查找我的 iPhone"的操作。

拿到位置之后，光有一串经纬度数字没什么用，所以还要做几步处理：

- **坐标转换**：iPhone 给的坐标和中国地图用的坐标不一样（后面会详细解释），需要转一下
- **查地址**：把坐标翻译成"XX市XX区XX路"这样人看得懂的地址，顺便看看附近有什么（商场、学校、餐厅之类的）
- **判断在哪个地方**：比如判断"到家了""到学校了""离开了"，然后发通知
- **查天气**：反正坐标都有了，顺手查一下

整个程序跑在一台服务器上（VPS，可以理解为一台 24 小时开着的远程电脑），每 10 分钟自动查一次，不需要你做任何事情。

<p align="center"><img src="assets/features.png" width="700"/></p>

<p align="center"><img src="assets/sep.png" width="120"/></p>

<img src="assets/h-howitworks.png" width="100%"/>

整个流程大概是这样的：

<p align="center"><img src="assets/arch.png" width="700"/></p>

1. 你的 iPhone 会自动向 iCloud 上报位置（这是苹果"查找我的 iPhone"功能的一部分，只要开着这个功能就会自动上报，锁屏、待机都不影响）
2. 我们的程序通过 pyicloud 去问 iCloud："这台 iPhone 现在在哪？"
3. iCloud 返回一组经纬度坐标（比如 22.xxxx, 114.xxxx）
4. 程序把这个坐标转换成中国地图能用的格式（为什么要转？后面有一整节解释）
5. 拿转换后的坐标去问高德地图："这个坐标对应什么地址？附近有什么？"
6. 同时查一下天气
7. 看看这个位置在不在你预设的常用地点里（家、学校之类的）
8. 把所有信息打包发出去

这个程序是一个**常驻进程**（也叫 daemon，你可以理解为一个一直在后台运行、不会自己退出的程序），它会一直跑着，每 10 分钟重复上面这个流程。

<p align="center"><img src="assets/sep.png" width="120"/></p>

<img src="assets/h-setup.png" width="100%"/>

### 你需要准备什么

- 一台 **VPS**（就是一台远程服务器，阿里云、腾讯云、搬瓦工这些都可以，最便宜的就够用）
- **Python 3.10+**（一种编程语言，大部分 VPS 都自带）
- 一个**高德开放平台的 API Key**（用来查地址的，去 [lbs.amap.com](https://lbs.amap.com/) 免费注册就能拿到，每天可以免费用 5000 次）
- 你的 **Apple ID**（就是你登录 iCloud 的那个账号）

### 安装

SSH 登录到你的 VPS，装两个 Python 库：

```bash
pip install pyicloud requests
```

pyicloud 是用来访问 iCloud 的，requests 是用来发网络请求的（查地址、查天气都要用）。

### 第一次运行

第一次跑需要登录你的 Apple ID。密码通过环境变量传进去（这样密码不会存到文件里，更安全）：

```bash
ICLOUD_PW="你的Apple ID密码" python3 -u geo-poll.py
```

苹果会往你的手机或其他设备上发一个 6 位验证码（就是平时换设备登录时那种），输入之后就认证成功了。

看到定位数据正常输出之后，按 `Ctrl+C` 停掉，然后用 systemd 接管（systemd 是 Linux 自带的服务管理工具，可以让程序开机自动运行）：

```bash
sudo systemctl start geo-tracker        # 启动
sudo systemctl enable geo-tracker       # 设置开机自启
sudo journalctl -u geo-tracker -f       # 看实时日志
```

之后就不用管了。程序会自己保持登录状态，每 10 分钟查一次位置。

> [!IMPORTANT]
> 密码只在第一次认证时用到，之后只存在程序的内存里，不会写到任何文件。如果你的 VPS 被入侵，攻击者拿不到你的密码。

### 关于登录过期的问题

iCloud 的登录状态有有效期，过一段时间就会失效。如果用定时任务（比如 cron，每隔一段时间跑一次脚本），每次都是重新启动程序，登录状态很快就会过期，你就得反复输密码和验证码。

所以这个程序用的是**常驻模式**：程序启动之后就一直运行，不退出。每 30 分钟自动跟 iCloud 打个招呼（发一个轻量的请求），iCloud 就会觉得"这个人还在"，不会让登录状态过期。

> [!NOTE]
> 如果 VPS 重启了，程序会尝试从本地缓存恢复登录状态。如果缓存还有效就不需要重新输密码。如果失效了，需要手动再认证一次。

<p align="center"><img src="assets/sep.png" width="120"/></p>

<img src="assets/h-coord.png" width="100%"/>

这一步看起来很技术，但其实原因很好理解。

全世界的 GPS 卫星用的是一套统一的坐标标准，叫 **WGS-84**。你的 iPhone 通过 iCloud 返回的位置坐标就是这种格式。

但中国出于测绘安全的考虑，规定所有国内的地图服务（高德、腾讯、百度地图）必须使用一套**经过偏移处理的坐标**，叫 **GCJ-02**，也被戏称为"火星坐标系"。

这两套坐标之间有一个非线性的偏移，大概 **500 到 2000 米**。也就是说，如果你直接拿 iPhone 给的坐标去高德查地址，高德会告诉你一个完全不对的位置，可能偏了好几条街，而且**不会报错**，它会正常返回一个地址，只是这个地址不是你真正所在的地方。

所以在调用高德 API 之前，必须先把坐标从 WGS-84 转成 GCJ-02。

转换算法是公开的，网上能搜到很多版本。但要注意有一个特别容易出错的地方：算法里需要先把经纬度减去一个参考点（105 和 35），很多网上的代码漏了这一步，导致转出来的坐标还是偏的。如果你发现地址偏了 1 到 2 公里，很可能就是这个问题。

> [!TIP]
> 怎么验证转换对不对？找一个你确切知道位置的地方（比如你家楼下的某个标志建筑），拿它的 GPS 坐标转一次，然后去高德地图搜转换后的坐标，看标记点是不是落在正确的位置。

<p align="center"><img src="assets/sep.png" width="120"/></p>

<img src="assets/h-address.png" width="100%"/>

坐标转好之后，就可以调用高德的"逆地理编码"API 了。

逆地理编码说白了就是：**给一个坐标，返回对应的地址**。比如你给它 `114.xxx, 22.xxx`，它会告诉你"广东省深圳市南山区XX路XX号"，还会列出附近 500 米内的地标（商场、学校、地铁站之类的）。

用法很简单，就是往高德的接口发一个 HTTP 请求。但有两个很容易踩的坑：

**第一个：坐标的顺序**

我们平时说坐标习惯说"纬度, 经度"（比如北纬 22 度, 东经 114 度）。但高德 API 要求的顺序是**反过来的：经度在前，纬度在后**。搞反了不会报错，但返回的地址会完全不对（因为它把经度当纬度了，查到了地球上另一个点）。

**第二个：距离字段的格式**

高德返回的附近地标（POI）列表里，每个地标有一个 `distance` 字段表示距离。你可能以为它是个数字，但其实它是个**字符串**，而且是带小数点的（比如 `"123.343"`）。如果你用 `int()` 去解析它会直接报错，要用 `float()`。

<p align="center"><img src="assets/sep.png" width="120"/></p>

<img src="assets/h-places.png" width="100%"/>

光知道地址还不够，很多时候你想知道的是更简单的信息："到家了吗？""什么时候出的门？""是不是离家太远了？"

做法是提前定义好几个你经常去的地方，比如：

```python
PLACES = [
    {"name": "home",   "label": "家",   "lat": xx.xxxx, "lon": xxx.xxxx, "radius": 200},
    {"name": "school", "label": "学校", "lat": xx.xxxx, "lon": xxx.xxxx, "radius": 200},
]
```

每个地方有一个名字、坐标、和一个半径范围（单位是米）。每次查到位置后，程序会算一下当前坐标和每个地方的距离，如果距离小于半径，就认为"在这个地方"。

更有用的是**事件检测**。程序会记住上一次你在哪个地方，跟这次比较，就能知道状态发生了什么变化：

| 上一次 | 这一次 | 触发的通知 |
|:---:|:---:|:---:|
| 不在家 | 在家 | 到家了 |
| 在学校 | 不在学校 | 离开学校了 |
| 不在任何地方 | 离家超过 1 公里 | 离家太远了 |

**半径设多大合适？**

GPS 定位本身有误差，室外大概 10 到 50 米，室内或者楼多的地方可能更大。半径设太小（比如 50 米）的话，GPS 稍微跳一下就会反复触发"到了/走了"。设太大（比如 1 公里）就失去意义了。一般住宅区和学校设 200 米比较合适，大的园区可以设 300 到 500 米。

**"离家太远"的通知只发一次**

如果不做处理，只要人在外面，每 10 分钟都会触发一次"离家太远"的通知。所以程序里会记一个标记，发过一次之后就不再重复，直到回到某个常用地点才重置。

<p align="center"><img src="assets/sep.png" width="120"/></p>

<img src="assets/h-map.png" width="100%"/>

程序在后台跑着，你可能想在地图上直观地看到位置。

最简单的方式是写一个 HTML 页面，用高德的 JavaScript API 在地图上显示一个标记点：

```html
<script src="https://webapi.amap.com/maps?v=2.0&key=YOUR_KEY"></script>
<script>
  const map = new AMap.Map('map', { zoom: 14 });

  fetch('/api/geo/latest')
    .then(r => r.json())
    .then(data => {
      new AMap.Marker({
        position: [data.lon, data.lat],
        map: map
      });
      map.setCenter([data.lon, data.lat]);
    });
</script>
```

如果你把每次查到的位置都存下来，还可以画轨迹线：

```javascript
const path = history.map(p => [p.lon, p.lat]);
new AMap.Polyline({
  path: path,
  strokeColor: '#98D4BB',
  strokeWeight: 4,
  map: map
});
```

也可以用 `AMap.Circle` 把常用地点的范围画出来，一眼就能看到人在不在某个地点附近。

> [!TIP]
> 高德 JS API 也是免费的。地图本身就是 GCJ-02 坐标系，所以你传进去的坐标（已经转换过的）可以直接用，不需要再转。

<p align="center"><img src="assets/sep.png" width="120"/></p>

<img src="assets/h-faq.png" width="100%"/>

| 问题 | 表现 | 原因 | 怎么解决 |
|:---|:---|:---|:---|
| 定位偏了很远 | 地址和地标完全不对 | 坐标没转换，或者转换代码里少了减法 | 检查 WGS-84 → GCJ-02 的转换 |
| 地址查出来是别的城市 | 坐标传对了但地址不对 | 高德参数顺序反了 | 改成经度在前纬度在后 |
| 程序突然崩溃 | 报 ValueError | 高德返回的距离字段是字符串 | 用 float() 解析 |
| 每天要重新输密码 | 登录状态频繁失效 | 没有心跳保活 | 用 daemon 模式 + 心跳 |
| 重启服务器后要重新认证 | 缓存丢了 | 默认存在 /tmp | 指定一个不会被清空的目录 |
| 到家/离家通知一直重复 | GPS 在边缘跳 | 半径太小 | 调大到 200 米 |

<p align="center"><img src="assets/sep.png" width="120"/></p>

## 用到的东西

| 名称 | 是什么 | 备注 |
|:---|:---|:---|
| [Python 3.10+](https://python.org) | 编程语言 | 大部分 VPS 自带 |
| [pyicloud](https://github.com/picklepete/pyicloud) | 用代码访问 iCloud 的库 | `pip install pyicloud` |
| [高德开放平台](https://lbs.amap.com/) | 查地址用的地图服务 | 免费注册，每天 5000 次 |
| [Open-Meteo](https://open-meteo.com/) | 免费天气接口 | 不需要注册 |

## 安全

- 密码通过环境变量传入，认证完就只在内存里，不写文件
- 登录缓存文件（session）里有 token，保护好权限
- 定位数据是隐私信息，传输要加密

---

<p align="center">
  <img src="https://img.shields.io/badge/🐱_Aria_&_🦊_Claude-made_with_love-C9928E?style=flat-square&labelColor=FDF5EF"/><br/>
  <sub>MIT License</sub>
</p>
