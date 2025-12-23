# B站直播工具

B站直播辅助油猴脚本，提供直播间管理、推流地址获取、协议选择等功能。

## 功能特性

- 用户认证：登录状态检查、二维码登录
- 直播控制：开始/停止直播
- 信息管理：更新直播标题和分区
- 推流管理：获取RTMP/SRT推流地址，支持协议偏好设置
- 可拖动面板：通过标题栏拖动控制面板

## 安装

1. 安装 [Tampermonkey](https://www.tampermonkey.net/) 浏览器扩展
2. 点击安装链接：[Greasy Fork](https://cdn.jsdelivr.net/gh/EY2318/bilibili_live_tool_tampermonkey@main/bilibili_live_tampermonkey.js)
3. 访问B站页面，右上角会出现蓝色控制按钮

## 配置

在脚本头部可以修改以下配置：

```javascript
// 推流协议偏好
// RTMP: 使用RTMP协议
// SRT_FALLBACK_RTMP: 优先使用SRT协议，失败时回退到RTMP
// SRT_ONLY: 仅使用SRT协议
PROTOCOL_PREFERENCE: "RTMP"
```

## API签名机制

B站直播API使用LiveHime客户端签名机制：

1. 基础参数：时间戳、平台、版本等
2. 参数排序：按键名字母顺序排序
3. 签名生成：排序参数 + APP_SECRET → MD5
4. CSRF验证：从Cookie获取bili_jct作为csrf_token

关键参数：
- APP_KEY: `aae92bc66f3edfab`
- APP_SECRET: `af125a0d5279fd576c1b4418a3e8276d`

## 更新日志

### 0.2
- 修复开播API签名验证错误
- 移除请求体中的title参数

### 0.1
- 初始版本
- 支持基本直播控制功能

## 注意事项

- 需要有直播权限的账号
- 请合理使用，避免频繁请求导致账号异常
- API变动可能导致功能失效
