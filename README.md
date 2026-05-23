# pyEzvizApiCN

这是一个面向 EZVIZ/萤石设备的 Python API 客户端和命令行工具。项目基于 `pyezvizapi`，当前分支重点增加了对中国区萤石云 iOS 接口形态的适配，适合用于本地调试、脚本自动化，以及 Home Assistant 相关集成探索。

## 功能概览

- 登录 EZVIZ/萤石账号并保存 session token。
- 查询摄像头、灯泡、插座等设备状态。
- 查询原始设备资源数据，便于抓包对照和调试。
- 控制摄像头常见开关：隐私模式、休眠、音频、红外灯、状态灯等。
- 控制云台 PTZ、报警、防区和部分设备配置。
- 获取统一消息、云录像、SD 卡录像描述。
- 支持实验性的云端 VTM 流、本地 LAN SDK 流和部分加密视频解密流程。
- 对 `api.ys7.com` 做了 CN iOS 请求路径和响应结构兼容。

## 安装

建议使用 Python 3.12+。

```bash
cd /Users/clover/workspace/pyEzvizApi
/opt/homebrew/bin/python3.12 -m venv .venv
source .venv/bin/activate
python -m pip install -U pip wheel
python -m pip install -e '.[dev]'
```

安装完成后可以使用：

```bash
pyezvizapi --help
```

## 中国区配置

中国区萤石云使用：

```bash
export EZVIZ_REGION='api.ys7.com'
export EZVIZ_USERNAME='你的用户名'
export EZVIZ_PASSWORD='你的密码'
```

首次登录并保存 token：

```bash
pyezvizapi -r "$EZVIZ_REGION" \
  -u "$EZVIZ_USERNAME" \
  -p "$EZVIZ_PASSWORD" \
  --save-token devices status
```

后续如果 `ezviz_token.json` 中已有有效 session，可以直接复用 token：

```bash
pyezvizapi -r "$EZVIZ_REGION" --json devices status
```

注意：`ezviz_token.json` 包含登录 session，不建议提交到 Git 仓库。

## 本次 CN/iOS 适配

本分支已根据 iOS 端抓包对 `api.ys7.com` 做了独立适配，不影响默认国际区接口。

### 登录适配

`api.ys7.com` 登录接口改为：

```text
POST /v3/users/login/v3
```

登录请求使用 iOS VideoGo 风格 header：

```text
Accept: */*
Connection: keep-alive
Content-Type: application/x-www-form-urlencoded
Host: api.ys7.com
User-Agent: VideoGo/7.7.3 (iPhone; iOS 26.5; Scale/3.00)
appId: ys7
clientNo:
clientType: 1
clientVersion:
featureCode: 动态生成的设备标识
sessionId:
ssid:
```

登录 form 只保留：

```text
account
password
featureCode
msgType
bizType
cuName
```

`featureCode` 使用项目原有逻辑动态生成；`cuName` 使用原项目默认值 `SGFzc2lv`。同时兼容 CN iOS 登录成功响应中的 `sessionInfo`：

```json
{
  "sessionInfo": {
    "sessionId": "...",
    "rfSessionId": "...",
    "userName": "..."
  },
  "meta": {
    "code": 200
  }
}
```

### 路径替换

`api.ys7.com` 下已替换为 CN iOS 路径：

| 功能 | 原项目路径 | CN iOS 路径 |
| --- | --- | --- |
| 登录 | `/v3/users/login/v5` | `/v3/users/login/v3` |
| 设备资源列表 | `/v3/userdevices/v1/resources/pagelist` | `/v3/devices/resources` |
| 设备状态 | `/v3/userdevices/v1/devices/status` | `/v3/devices/statusInfo` |
| 流媒体 ticket | `/v3/cameras/ticketInfo` | `/v3/streaming/ticket/{serial}/{channel}` |
| 流媒体 relay/VTM | `/v3/streaming/vtm/{serial}/{channel}` | `/v3/streaming/query/relay/{serial}/{channel}` |

### 响应结构兼容

CN iOS 的 `/v3/devices/resources` 返回字段和原项目 pagelist 不一致，代码已做映射：

| CN iOS 字段 | 项目内部字段 |
| --- | --- |
| `statusInfos` | `STATUS` |
| `switchStatusInfos` | `SWITCH` |
| `connectionInfos` | `CONNECTION` |
| `wifiInfos` | `WIFI` |
| `cloudInfos` | `CLOUD` |
| `p2pInfos` | `P2P` |
| `secretKeyInfos` | `KMS` |
| `featureInfos` | `FEATURE_INFO` |
| `resources` | `resourceInfos` |
| `cameraInfos[].vtmInfo` | `VTM` |
| `cameraInfos` | `CHANNEL` |
| `cameraInfos[].videoQualityInfos` | `VIDEO_QUALITY` |

这样 `devices status`、`device_infos` 等现有命令可以继续使用。

## 常用命令

查看所有设备状态：

```bash
pyezvizapi -r "$EZVIZ_REGION" devices status
```

JSON 输出：

```bash
pyezvizapi -r "$EZVIZ_REGION" --json devices status
```

查看原始设备信息：

```bash
pyezvizapi -r "$EZVIZ_REGION" --json device_infos
```

查看某个设备：

```bash
pyezvizapi -r "$EZVIZ_REGION" --json device_infos --serial 设备序列号
```

摄像头状态：

```bash
pyezvizapi -r "$EZVIZ_REGION" camera --serial 设备序列号 status
```

云台移动：

```bash
pyezvizapi -r "$EZVIZ_REGION" camera --serial 设备序列号 move --direction up --speed 5
```

切换隐私模式：

```bash
pyezvizapi -r "$EZVIZ_REGION" camera --serial 设备序列号 switch --switch privacy --enable 1
```

统一消息/报警消息：

```bash
pyezvizapi -r "$EZVIZ_REGION" --json unifiedmsg
```

灯泡：

```bash
pyezvizapi -r "$EZVIZ_REGION" devices_light status
pyezvizapi -r "$EZVIZ_REGION" light --serial 设备序列号 status
pyezvizapi -r "$EZVIZ_REGION" light --serial 设备序列号 toggle
```

## 流媒体相关

查看 stream 子命令：

```bash
pyezvizapi stream --help
```

抓取 VTM 包元数据：

```bash
pyezvizapi -r "$EZVIZ_REGION" stream trace \
  --serial 设备序列号 \
  --channel 1 \
  --max-packets 20 \
  --json-lines
```

导出 MPEG-TS：

```bash
pyezvizapi -r "$EZVIZ_REGION" stream dump \
  --serial 设备序列号 \
  --channel 1 \
  --duration 30s \
  --output stream.ts
```

启动本地代理：

```bash
pyezvizapi -r "$EZVIZ_REGION" stream proxy \
  --serial 设备序列号 \
  --channel 1 \
  --listen-port 8558
```

本地 SDK 流需要设备暴露 `9010/9020` 端口，并需要 CAS/local stream 相关凭据：

```bash
pyezvizapi -r "$EZVIZ_REGION" stream local-sdk-keys \
  --serial 设备序列号
```

## 开发与测试

运行单元测试：

```bash
python -m pytest -q
```

运行本次 CN 适配相关测试：

```bash
python -m pytest -q \
  tests/test_auth.py \
  tests/test_pagelist.py \
  tests/test_http_helpers.py::test_ys7_status_and_ticket_use_ios_paths
```

运行 lint：

```bash
ruff check pyezvizapi tests
```

## 安全提示

- 不要把 `ezviz_token.json`、账号密码、HAR 原始抓包提交到公共仓库。
- HAR 中可能包含 session、设备序列号、局域网 IP、云端 ticket、密钥相关字段。
- 建议首次登录后使用 `--save-token`，后续尽量复用 token，减少明文密码出现在 shell 历史中的机会。

## 许可证

沿用上游项目许可证，详见 [LICENSE](LICENSE) 和 [LICENSE.md](LICENSE.md)。
