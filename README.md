# B站直播控制面板助手

## 项目介绍

这是一个基于Tampermonkey的B站直播辅助脚本，为主播提供了一套完整的直播间管理工具。通过简洁的UI界面，用户可以快速开关直播、更新直播信息、获取推流地址等，无需频繁切换B站官方直播后台，大幅提高直播管理效率。

## 主要功能

- **用户认证**：检查登录状态并提供可视化反馈
- **二维码登录**：快速扫码登录B站账号
- **直播控制**：开始/停止直播
- **信息管理**：更新直播标题和分区
- **推流管理**：获取RTMP推流地址
- **直播监控**：查看直播状态和历史记录
- **可拖动面板**：通过标题栏拖动控制面板到任意位置

## 安装使用

1. 安装 [Tampermonkey](https://www.tampermonkey.net/) 浏览器扩展
2. 创建新脚本，复制并粘贴 `bilibili_live_tampermonkey.js` 的内容
3. 保存并启用脚本
4. 访问B站页面，右上角会出现一个蓝色控制按钮
5. 点击按钮打开直播控制面板

## B站API接口详解

### 用户认证相关API

| API接口 | 请求方式 | 功能说明 | 返回数据 |
| --- | --- | --- | --- |
| `https://api.bilibili.com/x/web-interface/nav/stat` | GET | 检查用户登录状态 | 包含登录状态和基本用户信息 |
| `https://passport.bilibili.com/qrcode/getLoginUrl` | GET | 获取登录二维码URL | 包含二维码URL和oauthKey |
| `https://passport.bilibili.com/qrcode/getLoginInfo` | POST | 检查二维码扫描状态 | 包含登录结果和用户凭证 |
| `https://api.bilibili.com/x/web-interface/nav` | GET | 获取当前登录用户信息 | 包含用户ID、昵称、头像等详细信息 |
| `https://passport.bilibili.com/login/exit/v2` | POST | 退出登录 | 退出结果状态 |

### 直播控制相关API

| API接口 | 请求方式 | 功能说明 | 关键参数 | 返回数据 |
| --- | --- | --- | --- | --- |
| `https://api.live.bilibili.com/xlive/web-ucenter/v1/start/get_preview` | GET | 获取开播前预览信息 | 无 | 包含推荐分区、历史开播配置等 |
| `https://api.live.bilibili.com/room/v1/Room/getRoomInfoOld` | GET | 获取直播间基本信息 | `mid`: 用户ID | 包含房间ID、直播状态、标题等 |
| `https://api.live.bilibili.com/xlive/web-room/v1/index/getRoomBaseInfo` | GET | 获取房间详细信息 | `room_ids`: 房间ID列表 | 包含直播状态、观看人数等详情 |
| `https://api.live.bilibili.com/xlive/web-ucenter/v1/record/GetList` | GET | 获取历史直播记录 | `page`, `pageSize` | 包含历史开播时间、时长等记录 |
| `https://api.live.bilibili.com/room/v1/Area/getList` | GET | 获取所有直播分区 | 无 | 包含主分区和子分区信息 |
| `https://api.live.bilibili.com/room/v1/Room/startLive` | POST | 开始直播 | `area_v2`, `csrf_token` | 包含开播结果和推流地址 |
| `https://api.live.bilibili.com/room/v1/Room/stopLive` | POST | 结束直播 | `room_id`, `csrf_token` | 包含停播结果状态 |
| `https://api.live.bilibili.com/room/v1/Room/update` | POST | 更新直播信息 | `title`, `area_id`, `csrf_token` | 包含更新结果状态 |
| `https://api.live.bilibili.com/live_stream/v1/StreamList/get_stream_by_roomId` | GET | 获取推流信息 | `room_id` | 包含RTMP推流地址和密钥 |

### 签名验证机制

B站API的安全机制主要包括以下几种：

#### 1. APP_KEY与APP_SECRET签名

B站直播API使用了类似OAuth的签名机制，特别是针对官方客户端(如LiveHime)的API。签名流程如下：

1. **基础参数准备**：构建包含时间戳、平台、版本等信息的基础参数
```javascript
function basePayload() {
    return {
        access_key: "",
        build: "9240",         // LiveHime客户端版本build号
        platform: "pc_link",   // 平台标识
        ts: Math.floor(Date.now() / 1000).toString(),  // 当前时间戳
        version: "7.16.0.9240" // LiveHime客户端版本
    };
}
```

2. **参数合并与排序**：将业务参数与基础参数合并，按键名字母顺序排序
```javascript
function orderPayload(obj) {
    return Object.keys(obj)
        .sort()
        .reduce((result, key) => {
            result[key] = obj[key];
            return result;
        }, {});
}
```

3. **生成签名字符串**：将排序后的参数转换为URL查询字符串，并附加密钥
```javascript
// 参数URL编码
function encodeParams(params) {
    return Object.keys(params)
        .map((key) => `${key}=${encodeURIComponent(params[key])}`)
        .join("&");
}

// 签名生成
const queryString = encodeParams(orderedParams);
const signStr = queryString + APP_SECRET; // APP_SECRET是应用密钥
```

4. **计算MD5哈希**：对签名字符串进行MD5哈希计算，得到最终签名
```javascript
const sign = md5(signStr);
orderedParams.sign = sign; // 将签名添加到参数中
```

5. **完整实现**：
```javascript
function livehimeSign(payload) {
    const signed = { ...basePayload() };
    signed.appkey = "aae92bc66f3edfab";  // LiveHime的APP_KEY
    Object.assign(signed, payload);

    const orderedParams = orderPayload(signed);
    const queryString = encodeParams(orderedParams);
    const signStr = queryString + "af125a0d5279fd576c1b4418a3e8276d"; // APP_SECRET

    const sign = md5(signStr);
    orderedParams.sign = sign;

    return orderedParams;
}
```

#### 2. CSRF令牌保护

大多数修改操作（POST请求）除了需要API签名外，还需要提供`csrf_token`参数进行双重验证：

```javascript
// 从Cookie获取CSRF令牌
function getCsrfToken() {
    const match = document.cookie.match(/bili_jct=([^;]+)/);
    return match ? match[1] : "";
}

// 使用示例（开始直播）
const payload = {
    room_id: roomId,
    area_v2: areaId,
    type: 2
};

// 添加csrf令牌
if (csrf) {
    payload.csrf = csrf;
    payload.csrf_token = csrf; // 某些API需要两种形式的csrf参数
}

// 生成带签名的参数
const params = livehimeSign(payload);

// 发送请求
const response = await utils.post(
    "https://api.live.bilibili.com/room/v1/Room/startLive",
    params
);
```

#### 3. Cookie认证

大部分API请求依赖Cookie中的登录凭证：
- `SESSDATA`：会话标识
- `bili_jct`：CSRF令牌
- `DedeUserID`：用户ID

这些凭证通过浏览器自动附加到请求中，脚本通过`GM_xmlhttpRequest`保持这些Cookie。

#### 4. 请求头验证

B站API会验证请求的`User-Agent`、`Referer`和`Origin`头，确保请求来自合法来源：

```javascript
GM_xmlhttpRequest({
    method: "POST",
    url: url,
    headers: {
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/137.0.0.0 Safari/537.36",
        "Content-Type": "application/x-www-form-urlencoded; charset=UTF-8"
    },
    data: formData,
    onload: function(response) { /* 处理响应 */ }
});
```

#### 5. 访问频率限制

B站API有请求频率限制，过于频繁的请求可能导致临时IP封禁。脚本中实现了延迟和错误重试机制：

```javascript
// 轮询检查二维码扫描状态，1秒一次
const checkInterval = setInterval(async () => {
    const result = await auth.checkQRCodeStatus(qrData.qrcodeKey);
    // 处理结果...
}, 1000);

// 设置超时，防止无限轮询
setTimeout(() => {
    clearInterval(checkInterval);
}, 60000);
```

### API参数详解

#### 1. 检查用户登录状态
- **接口**: `https://api.bilibili.com/x/web-interface/nav/stat`
- **请求方式**: GET
- **参数**: 无需额外参数，使用Cookie验证
- **返回示例**:
```json
{
  "code": 0,
  "message": "0",
  "ttl": 1,
  "data": {
    "following": 100,
    "follower": 20,
    "dynamic_count": 15
  }
}
```

#### 2. 获取登录二维码URL
- **接口**: `https://passport.bilibili.com/qrcode/getLoginUrl`
- **请求方式**: GET
- **参数**: 无
- **返回示例**:
```json
{
  "code": 0,
  "status": true,
  "ts": 1628956800,
  "data": {
    "url": "https://passport.bilibili.com/qrcode/h5/login?oauthKey=xxx",
    "oauthKey": "xxxxxxxxxxxxxxx"
  }
}
```

#### 3. 开始直播
- **接口**: `https://api.live.bilibili.com/room/v1/Room/startLive`
- **请求方式**: POST
- **参数**:
  - `area_v2`: 直播子分区ID
  - `platform`: 平台，默认为"pc"
  - `csrf_token`: CSRF令牌
- **返回示例**:
```json
{
  "code": 0,
  "msg": "success",
  "message": "success",
  "data": {
    "change": 1,
    "status": "success",
    "rtmp": {
      "addr": "rtmp://live-push.bilivideo.com/live-bvc/",
      "code": "?streamname=xxx&key=xxx"
    }
  }
}
```

#### 4. 更新直播信息
- **接口**: `https://api.live.bilibili.com/room/v1/Room/update`
- **请求方式**: POST
- **参数**:
  - `title`: 新的直播标题
  - `room_id`: 直播间ID
  - `area_id`: 直播分区ID
  - `csrf_token`: CSRF令牌
- **返回示例**:
```json
{
  "code": 0,
  "msg": "success",
  "message": "success",
  "data": []
}
```

## 工具函数说明

脚本提供了多个工具函数模块，用于处理不同的功能需求：

- **auth**: 处理用户认证相关功能
- **liveControls**: 处理直播控制相关功能
- **ui**: 处理用户界面相关功能
- **utils**: 提供通用工具函数，如请求、日志等

## 面板使用

### 主控面板功能区
- **直播状态区**：显示当前直播状态（直播中/未开播）
- **直播标题**：修改直播间标题
- **直播分区**：选择主分区和子分区
- **分区搜索**：支持拼音、首字母和ID搜索分区
- **操作按钮区**：
  - 开始直播：选择分区并开始直播
  - 结束直播：结束当前直播
  - 复制推流：复制RTMP地址到剪贴板
  - 更新信息：更新直播标题和分区

### 操作流程
1. 点击右上角蓝色按钮打开控制面板
2. 如未登录，会提示扫描二维码登录
3. 填写直播标题并选择分区
4. 点击"开始直播"按钮
5. 获取推流地址后，可点击复制按钮复制到推流软件

## 注意事项

1. 使用前需要登录B站账号
2. 需要有直播权限的账号才能操作
3. 因B站API变动，部分功能可能会失效
4. 请合理使用脚本，避免频繁请求导致账号异常

## 更新日志

- 添加了面板拖动功能，支持通过标题栏拖动
- 优化了登录状态检查，添加可视化提示
- 改进了面板初始位置，使其靠近悬浮按钮
- 修复了事件冒泡问题，优化交互体验
- 添加了边界检测，防止面板被拖出屏幕

---

*本文档详细说明了B站直播控制面板助手脚本使用的API接口，如有API变动，请以最新代码为准。*
