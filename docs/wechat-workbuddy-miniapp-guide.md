# 微信小程序方案：用 xiaozhi-esp32 接入 Workbuddy（腾讯元宝 Copilot Work）

> 目标：做一个微信小程序，既能控制/查看 `xiaozhi-esp32` 设备状态，也能把语音/文本请求转发到 Workbuddy，再把回答回流到设备与小程序。

## 0. 先说明边界（务必先读）

1. **Workbuddy 的官方开放接口**：如果你当前账号有可用 OpenAPI/企业 API，优先走官方 API（推荐）。
2. 如果没有开放 API，你只能通过“中间层服务”做网页自动化或人工网关，这在稳定性与合规性上都较弱。
3. 微信小程序端不应直接保存 Workbuddy 凭证，统一走后端签发 token。

下面给出的是一套可落地的“**小程序 + 云函数 + 桥接服务 + xiaozhi-esp32**”实现模板。

---

## 1. 总体架构

```text
[微信小程序]
   | HTTPS / WSS
   v
[微信云函数(鉴权/会话/设备绑定)]
   | 内网调用
   v
[Bridge 服务 (Node.js)] <----> [Workbuddy API/Web Session]
   ^
   | WebSocket / MQTT (按 xiaozhi 协议)
[ xiaozhi-esp32 设备 ]
```

### 数据流

1. 小程序发起“提问” -> 云函数校验登录态 -> Bridge。
2. Bridge 把问题转发给 Workbuddy。
3. Workbuddy 回答 -> Bridge ->
   - 推送给小程序（用于实时显示）
   - 发送给 xiaozhi-esp32（用于播报/执行动作）
4. 设备状态（在线、音量、电量）由 Bridge 同步给小程序。

---

## 2. 最小目录设计

```text
project/
├─ miniprogram/
│  ├─ app.js
│  ├─ app.json
│  ├─ app.wxss
│  ├─ utils/request.js
│  └─ pages/index/
│     ├─ index.wxml
│     ├─ index.js
│     ├─ index.wxss
│     └─ index.json
├─ cloudfunctions/
│  ├─ login/index.js
│  ├─ bindDevice/index.js
│  └─ askWorkbuddy/index.js
└─ bridge/
   ├─ package.json
   ├─ .env.example
   ├─ src/server.js
   ├─ src/workbuddyClient.js
   └─ src/deviceHub.js
```

---

## 3. 小程序端代码

## 3.1 `miniprogram/app.js`

```js
App({
  globalData: {
    apiBase: 'https://<你的云托管域名>',
    deviceId: '',
    token: ''
  },
  async onLaunch() {
    try {
      const res = await wx.cloud.callFunction({ name: 'login' })
      this.globalData.token = res.result.token
    } catch (e) {
      console.error('login failed', e)
    }
  }
})
```

## 3.2 `miniprogram/utils/request.js`

```js
const app = getApp()

function request({ url, method = 'GET', data = {} }) {
  return new Promise((resolve, reject) => {
    wx.request({
      url: `${app.globalData.apiBase}${url}`,
      method,
      data,
      header: {
        Authorization: `Bearer ${app.globalData.token}`
      },
      success: (res) => {
        if (res.statusCode >= 200 && res.statusCode < 300) {
          resolve(res.data)
        } else {
          reject(new Error(res.data?.message || 'request failed'))
        }
      },
      fail: reject
    })
  })
}

module.exports = { request }
```

## 3.3 `miniprogram/pages/index/index.wxml`

```xml
<view class="container">
  <view class="card">
    <input class="input" placeholder="输入设备ID" value="{{deviceId}}" bindinput="onDeviceInput" />
    <button type="primary" bindtap="bindDevice">绑定设备</button>
  </view>

  <view class="card">
    <textarea class="textarea" placeholder="向 Workbuddy 提问" value="{{question}}" bindinput="onQuestionInput" />
    <button type="primary" bindtap="ask">发送</button>
  </view>

  <view class="card">
    <text class="title">设备状态：{{deviceStatus}}</text>
    <text class="answer">{{answer}}</text>
  </view>
</view>
```

## 3.4 `miniprogram/pages/index/index.js`

```js
const { request } = require('../../utils/request')
const app = getApp()

Page({
  data: {
    deviceId: '',
    question: '',
    answer: '',
    deviceStatus: 'unknown'
  },

  onLoad() {
    this.connectRealtime()
  },

  onUnload() {
    if (this.ws) this.ws.close()
  },

  onDeviceInput(e) {
    this.setData({ deviceId: e.detail.value })
  },

  onQuestionInput(e) {
    this.setData({ question: e.detail.value })
  },

  async bindDevice() {
    try {
      await wx.cloud.callFunction({
        name: 'bindDevice',
        data: { deviceId: this.data.deviceId }
      })
      app.globalData.deviceId = this.data.deviceId
      wx.showToast({ title: '绑定成功' })
    } catch (e) {
      wx.showToast({ title: '绑定失败', icon: 'none' })
    }
  },

  async ask() {
    if (!app.globalData.deviceId) {
      wx.showToast({ title: '请先绑定设备', icon: 'none' })
      return
    }
    try {
      const res = await wx.cloud.callFunction({
        name: 'askWorkbuddy',
        data: {
          deviceId: app.globalData.deviceId,
          question: this.data.question
        }
      })
      this.setData({ answer: res.result.answer || '处理中...' })
    } catch (e) {
      wx.showToast({ title: '发送失败', icon: 'none' })
    }
  },

  connectRealtime() {
    const token = app.globalData.token
    this.ws = wx.connectSocket({
      url: `wss://<你的bridge域名>/miniapp/realtime?token=${token}`
    })

    this.ws.onMessage((msg) => {
      const data = JSON.parse(msg.data)
      if (data.type === 'workbuddy.reply') {
        this.setData({ answer: data.payload.text })
      }
      if (data.type === 'device.status') {
        this.setData({ deviceStatus: data.payload.status })
      }
    })
  }
})
```

---

## 4. 云函数（微信侧）

## 4.1 `cloudfunctions/login/index.js`

```js
const cloud = require('wx-server-sdk')
const jwt = require('jsonwebtoken')
cloud.init({ env: cloud.DYNAMIC_CURRENT_ENV })

exports.main = async () => {
  const wxContext = cloud.getWXContext()
  const token = jwt.sign(
    { openid: wxContext.OPENID },
    process.env.JWT_SECRET,
    { expiresIn: '7d' }
  )
  return { token, openid: wxContext.OPENID }
}
```

## 4.2 `cloudfunctions/bindDevice/index.js`

```js
const cloud = require('wx-server-sdk')
cloud.init({ env: cloud.DYNAMIC_CURRENT_ENV })
const db = cloud.database()

exports.main = async (event) => {
  const { deviceId } = event
  const { OPENID } = cloud.getWXContext()

  await db.collection('user_devices').doc(`${OPENID}_${deviceId}`).set({
    data: {
      openid: OPENID,
      deviceId,
      createdAt: Date.now()
    }
  })
  return { ok: true }
}
```

## 4.3 `cloudfunctions/askWorkbuddy/index.js`

```js
const cloud = require('wx-server-sdk')
const axios = require('axios')
cloud.init({ env: cloud.DYNAMIC_CURRENT_ENV })
const db = cloud.database()

exports.main = async (event) => {
  const { deviceId, question } = event
  const { OPENID } = cloud.getWXContext()

  const bind = await db.collection('user_devices').doc(`${OPENID}_${deviceId}`).get()
  if (!bind.data) {
    throw new Error('device not bound')
  }

  const resp = await axios.post(
    process.env.BRIDGE_URL + '/api/ask',
    { openid: OPENID, deviceId, question },
    { headers: { 'x-bridge-key': process.env.BRIDGE_KEY } }
  )

  return { answer: resp.data.answer }
}
```

---

## 5. Bridge 服务（核心）

## 5.1 `bridge/src/server.js`

```js
import Fastify from 'fastify'
import websocket from '@fastify/websocket'
import jwt from 'jsonwebtoken'
import { askWorkbuddy } from './workbuddyClient.js'
import { DeviceHub } from './deviceHub.js'

const app = Fastify({ logger: true })
await app.register(websocket)

const deviceHub = new DeviceHub()
const miniappSockets = new Map() // openid -> ws

app.post('/api/ask', async (req, reply) => {
  if (req.headers['x-bridge-key'] !== process.env.BRIDGE_KEY) {
    return reply.code(401).send({ message: 'unauthorized' })
  }

  const { openid, deviceId, question } = req.body

  const answer = await askWorkbuddy({
    question,
    conversationId: `${openid}:${deviceId}`
  })

  // 推给设备
  deviceHub.sendText(deviceId, answer)

  // 推给小程序实时通道
  const ws = miniappSockets.get(openid)
  if (ws) {
    ws.send(JSON.stringify({
      type: 'workbuddy.reply',
      payload: { deviceId, text: answer }
    }))
  }

  return { answer }
})

app.get('/miniapp/realtime', { websocket: true }, (socket, req) => {
  const { token } = req.query
  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET)
    miniappSockets.set(payload.openid, socket)

    socket.on('close', () => {
      miniappSockets.delete(payload.openid)
    })
  } catch {
    socket.close()
  }
})

// 示例：设备上报状态时回推小程序
app.post('/device/status', async (req, reply) => {
  const { openid, status } = req.body
  const ws = miniappSockets.get(openid)
  if (ws) {
    ws.send(JSON.stringify({ type: 'device.status', payload: { status } }))
  }
  return reply.send({ ok: true })
})

app.listen({ port: 8080, host: '0.0.0.0' })
```

## 5.2 `bridge/src/workbuddyClient.js`

```js
import axios from 'axios'

// 如果你有 Workbuddy 官方 API，直接改成官方 endpoint + 鉴权。
export async function askWorkbuddy({ question, conversationId }) {
  const resp = await axios.post(
    process.env.WORKBUDDY_API_URL,
    {
      conversationId,
      message: question
    },
    {
      headers: {
        Authorization: `Bearer ${process.env.WORKBUDDY_API_KEY}`
      },
      timeout: 30000
    }
  )

  return resp.data?.answer || '未获取到有效回复'
}
```

## 5.3 `bridge/src/deviceHub.js`

```js
export class DeviceHub {
  constructor() {
    this.deviceSockets = new Map()
  }

  register(deviceId, socket) {
    this.deviceSockets.set(deviceId, socket)
    socket.on('close', () => this.deviceSockets.delete(deviceId))
  }

  sendText(deviceId, text) {
    const socket = this.deviceSockets.get(deviceId)
    if (!socket) return

    // 根据 xiaozhi 的消息协议组织 payload（示例）
    socket.send(JSON.stringify({
      type: 'tts.text',
      payload: { text }
    }))
  }
}
```

---

## 6. xiaozhi-esp32 对接步骤（重点）

> 这里用“配置接入”方式，不改固件核心逻辑。你也可以在 `main/protocols` 层扩展自定义指令。

1. 在你的 Bridge 暴露设备接入端（WebSocket 或 MQTT）。
2. 在 xiaozhi 的服务端配置中，把设备消息上行转发到 Bridge。
3. 确认 Bridge 能识别 `deviceId` 并建立 `deviceId -> socket` 映射。
4. 当小程序提问后，Bridge 返回答案并调用 `sendText(deviceId, answer)`。
5. 设备端按已有 TTS 播放链路播报文本。

### 建议的协议事件

- `device.hello`：设备上线，包含 `deviceId`、固件版本。
- `device.audio.chunk`：语音分片（可选，如果要设备直接上传语音）。
- `device.status`：在线状态、音量、电量。
- `tts.text`：下发播报文本。
- `device.action`：执行动作（灯光、按键、表情）。

---

## 7. 上线部署步骤（按顺序）

1. **部署 Bridge**（云服务器/容器均可）
   - 配置 `.env`：
     - `JWT_SECRET`
     - `BRIDGE_KEY`
     - `WORKBUDDY_API_URL`
     - `WORKBUDDY_API_KEY`
2. **配置 HTTPS/WSS 域名与证书**。
3. **微信小程序后台配置合法域名**（request、socket）。
4. **部署云函数**：`login` / `bindDevice` / `askWorkbuddy`。
5. **创建云数据库集合**：`user_devices`。
6. **烧录并联网 xiaozhi-esp32**，让设备连到你的 Bridge。
7. 小程序绑定 `deviceId` 后，发送测试问题：
   - “帮我总结今天的待办事项”
   - 观察小程序回包 + 设备播报是否同时成功。

---

## 8. 安全与稳定性建议

1. 小程序端只拿短期 token，不保存 Workbuddy 密钥。
2. Bridge 对 `x-bridge-key`、JWT、设备签名三层校验。
3. 所有回答入库（可选）便于审计和故障排查。
4. 做超时与重试：Workbuddy 超时后返回兜底文案。
5. 做限流：同一 openid 每分钟最多 N 次请求。

---

## 9. 你可以直接照抄的联调清单

1. 启动 Bridge：
   - `npm i`
   - `node src/server.js`
2. 部署云函数并填环境变量。
3. 微信开发者工具打开小程序，扫码登录。
4. 输入设备 ID 并绑定。
5. 输入问题并发送。
6. 看小程序实时回包 + 设备端 TTS 输出。

如果你愿意，我下一步可以继续给你：

- 一份**可直接运行的最小 Demo 仓库结构**（含 `package.json`、云函数 `package.json`、数据库索引）；
- 按你当前的 xiaozhi 实际部署方式（WebSocket 或 MQTT）给你对应的“设备侧消息适配代码”。
