# OpenClaw WeChat Plugin - API Instructions

## 1. 核心消息API (iLink Bot API)

### 基础配置
- **基础URL**: `https://ilinkai.weixin.qq.com`
- **HTTP方法**: POST
- **内容类型**: `application/json`

### 公共请求头
```
Content-Type: application/json
AuthorizationType: ilink_bot_token
Authorization: Bearer {token}
X-WECHAT-UIN: {random uint32 base64}
iLink-App-Id: {pkg.ilink_appid}
iLink-App-ClientVersion: {version encoded as uint32}
SKRouteTag: {optional route tag}
```

---

## 2. API端点详细说明

### 2.1 getUpdates - 长轮询获取新消息

**端点**: `POST /ilink/bot/getupdates`

**超时**: 35秒（服务器可通过响应字段调整）

**请求体**:
```json
{
  "get_updates_buf": "string or empty",
  "base_info": {
    "channel_version": "string",
    "bot_agent": "string (optional)"
  }
}
```

**响应体**:
```json
{
  "ret": 0,
  "errcode": -14,
  "errmsg": "string",
  "msgs": [
    {
      "seq": 1,
      "message_id": 12345,
      "from_user_id": "xxx@im.wechat",
      "to_user_id": "xxx",
      "client_id": "string",
      "create_time_ms": 1624000000000,
      "update_time_ms": 1624000000000,
      "delete_time_ms": 1624000000000,
      "session_id": "string",
      "group_id": "string",
      "message_type": 1,
      "message_state": 0,
      "item_list": [
        {
          "type": 1,
          "text_item": {
            "text": "string"
          }
        }
      ],
      "context_token": "string"
    }
  ],
  "get_updates_buf": "string",
  "longpolling_timeout_ms": 35000
}
```

**关键说明**:
- `ret === 0`: 请求成功
- `errcode === -14`: 会话过期，需暂停60分钟后重试
- `get_updates_buf`: 必须保存到本地磁盘，用于下次请求
- `context_token`: 必须回显在回复消息中
- 若无消息时会持续等待直到超时，无需频繁轮询

---

### 2.2 sendMessage - 发送文本消息

**端点**: `POST /ilink/bot/sendmessage`

**超时**: 15秒

**请求体**:
```json
{
  "msg": {
    "to_user_id": "string",
    "from_user_id": "",
    "client_id": "string",
    "message_type": 2,
    "message_state": 2,
    "context_token": "string",
    "item_list": [
      {
        "type": 1,
        "text_item": {
          "text": "string"
        }
      }
    ]
  },
  "base_info": {
    "channel_version": "string",
    "bot_agent": "string (optional)"
  }
}
```

**响应体**:
```json
{
  "ret": 0,
  "errmsg": "string"
}
```

**关键说明**:
- `message_type`: 始终为2 (BOT消息)
- `message_state`: 始终为2 (FINISH状态)
- `context_token`: 必须从入站消息获取并回显
- 文本内容最大4000字符
- 单次发送一条消息，若超长需分割发送

---

### 2.3 getUploadUrl - 获取媒体上传URL

**端点**: `POST /ilink/bot/getuploadurl`

**超时**: 15秒

**请求体**:
```json
{
  "filekey": "string",
  "media_type": 1,
  "to_user_id": "string",
  "rawsize": 1024,
  "rawfilemd5": "string",
  "filesize": 1040,
  "thumb_rawsize": 512,
  "thumb_rawfilemd5": "string",
  "thumb_filesize": 528,
  "no_need_thumb": false,
  "aeskey": "string",
  "base_info": {
    "channel_version": "string",
    "bot_agent": "string (optional)"
  }
}
```

**参数说明**:
- `media_type`: 1=IMAGE, 2=VIDEO, 3=FILE, 4=VOICE
- `rawsize`: 原始文件大小（字节）
- `rawfilemd5`: 原始文件MD5（16进制）
- `filesize`: AES-128-ECB加密+PKCS7填充后的大小
- `aeskey`: AES密钥（16进制）
- IMAGE/VIDEO必须提供thumb参数

**响应体**:
```json
{
  "ret": 0,
  "upload_param": "string",
  "thumb_upload_param": "string",
  "upload_full_url": "string",
  "thumb_upload_full_url": "string"
}
```

**关键说明**:
- 优先使用 `upload_full_url`，其次使用 `upload_param` 拼接CDN地址
- `upload_param` 用于构建CDN URL: `{cdnBaseUrl}?{upload_param}`
- 返回的参数仅在此次上传过程中有效

---

### 2.4 getConfig - 获取账号配置

**端点**: `POST /ilink/bot/getconfig`

**超时**: 10秒

**请求体**:
```json
{
  "ilink_user_id": "string",
  "context_token": "string (optional)",
  "base_info": {
    "channel_version": "string",
    "bot_agent": "string (optional)"
  }
}
```

**响应体**:
```json
{
  "ret": 0,
  "errmsg": "string",
  "typing_ticket": "string"
}
```

**关键说明**:
- `typing_ticket`: Base64编码，用于sendTyping请求
- 建议缓存24小时，避免频繁调用
- 可选的 `context_token` 用于特定会话的配置获取

---

### 2.5 sendTyping - 发送打字指示器

**端点**: `POST /ilink/bot/sendtyping`

**超时**: 10秒

**请求体**:
```json
{
  "ilink_user_id": "string",
  "typing_ticket": "string",
  "status": 1,
  "base_info": {
    "channel_version": "string",
    "bot_agent": "string (optional)"
  }
}
```

**参数说明**:
- `status`: 1=开始输入, 2=停止输入
- `typing_ticket`: 从getConfig获取

**响应体**:
```json
{
  "ret": 0,
  "errmsg": "string"
}
```

**关键说明**:
- 在AI开始处理前调用 `status=1`
- 在消息发送后调用 `status=2`
- 保活间隔：5秒

---

### 2.6 notifyStart - 通知网关启动

**端点**: `POST /ilink/bot/msg/notifystart`

**超时**: 10秒

**请求体**:
```json
{
  "base_info": {
    "channel_version": "string",
    "bot_agent": "string (optional)"
  }
}
```

**响应体**:
```json
{
  "ret": 0,
  "errmsg": "string"
}
```

---

### 2.7 notifyStop - 通知网关停止

**端点**: `POST /ilink/bot/msg/notifystop`

**超时**: 10秒

**请求体**:
```json
{
  "base_info": {
    "channel_version": "string",
    "bot_agent": "string (optional)"
  }
}
```

**响应体**:
```json
{
  "ret": 0,
  "errmsg": "string"
}
```

---

## 3. QR码登录API

### 3.1 get_bot_qrcode - 获取QR码

**端点**: `POST /ilink/bot/get_bot_qrcode?bot_type=3`

**基础URL**: `https://ilinkai.weixin.qq.com`（固定，不可自定义）

**请求体**:
```json
{
  "local_token_list": ["token1", "token2"]
}
```

**响应体**:
```json
{
  "qrcode": "string",
  "qrcode_img_content": "string"
}
```

**关键说明**:
- `local_token_list`: 已注册的bot token列表，最多10个（用于账号绑定检查）
- `qrcode`: 二维码内容标识
- `qrcode_img_content`: 二维码URL，可直接在浏览器打开或转换为二维码显示

---

### 3.2 get_qrcode_status - 轮询QR码状态

**端点**: `GET /ilink/bot/get_qrcode_status?qrcode={qrcode}&verify_code={code}`

**基础URL**: `https://ilinkai.weixin.qq.com`（可能通过redirect_host重定向）

**超时**: 35秒

**响应体**:
```json
{
  "status": "wait",
  "bot_token": "string",
  "ilink_bot_id": "string",
  "baseurl": "string",
  "ilink_user_id": "string",
  "redirect_host": "string"
}
```

**状态值**:
- `wait`: 等待扫描，继续轮询
- `scaned`: 已扫描，等待用户确认
- `confirmed`: 登录成功，返回bot_token和ilink_bot_id
- `expired`: QR码过期，需刷新（最多3次）
- `need_verifycode`: 需要输入配对码
- `verify_code_blocked`: 配对码输入错误过多
- `scaned_but_redirect`: 需要IDC重定向，使用redirect_host更新polling地址
- `binded_redirect`: 此bot已绑定到当前OpenClaw实例，无需重复登录

**关键说明**:
- 轮询间隔：1秒
- 总超时：480秒
- 若状态为 `scaned_but_redirect`，更新polling地址为 `https://{redirect_host}`
- 若收到 `verify_code`，需从stdin读取并在下次轮询时作为query参数发送

---

## 4. 消息数据结构

### 4.1 WeixinMessage (入站消息)
```json
{
  "seq": 1,
  "message_id": 12345,
  "from_user_id": "xxx@im.wechat",
  "to_user_id": "xxx",
  "client_id": "string",
  "create_time_ms": 1624000000000,
  "update_time_ms": 1624000000000,
  "delete_time_ms": 0,
  "session_id": "string",
  "group_id": "string",
  "message_type": 1,
  "message_state": 0,
  "item_list": [],
  "context_token": "string"
}
```

**字段说明**:
- `message_type`: 1=USER消息, 2=BOT消息
- `message_state`: 0=NEW, 1=GENERATING, 2=FINISH
- `context_token`: **必须保存并在回复时原样返回**
- `item_list`: 消息内容列表

---

### 4.2 MessageItem (消息项)

#### 4.2.1 文本项
```json
{
  "type": 1,
  "text_item": {
    "text": "string"
  }
}
```

#### 4.2.2 图片项
```json
{
  "type": 2,
  "image_item": {
    "media": {
      "encrypt_query_param": "string",
      "aes_key": "string (base64)",
      "full_url": "string"
    },
    "thumb_media": {
      "encrypt_query_param": "string",
      "aes_key": "string (base64)",
      "full_url": "string"
    },
    "aeskey": "string (hex)",
    "url": "string",
    "mid_size": 1024,
    "thumb_size": 512,
    "thumb_height": 100,
    "thumb_width": 100,
    "hd_size": 2048
  }
}
```

#### 4.2.3 语音项
```json
{
  "type": 3,
  "voice_item": {
    "media": {
      "encrypt_query_param": "string",
      "aes_key": "string (base64)"
    },
    "encode_type": 6,
    "bits_per_sample": 16,
    "sample_rate": 16000,
    "playtime": 3000,
    "text": "string (可选，语音转文字结果)"
  }
}
```

#### 4.2.4 文件项
```json
{
  "type": 4,
  "file_item": {
    "media": {
      "encrypt_query_param": "string",
      "aes_key": "string (base64)"
    },
    "file_name": "string",
    "md5": "string",
    "len": "1024"
  }
}
```

#### 4.2.5 视频项
```json
{
  "type": 5,
  "video_item": {
    "media": {
      "encrypt_query_param": "string",
      "aes_key": "string (base64)"
    },
    "video_size": 1024000,
    "play_length": 30000,
    "video_md5": "string",
    "thumb_media": {
      "encrypt_query_param": "string",
      "aes_key": "string (base64)"
    },
    "thumb_size": 512,
    "thumb_height": 100,
    "thumb_width": 100
  }
}
```

#### 4.2.6 引用消息项
```json
{
  "type": 1,
  "text_item": {
    "text": "string"
  },
  "ref_msg": {
    "title": "string (摘要)",
    "message_item": {}
  }
}
```

---

### 4.3 MessageItem类型常量
```
1 = TEXT
2 = IMAGE
3 = VOICE
4 = FILE
5 = VIDEO
```

---

### 4.4 MessageType常量
```
1 = USER (用户消息)
2 = BOT (机器人消息)
```

---

### 4.5 MessageState常量
```
0 = NEW (新消息)
1 = GENERATING (生成中)
2 = FINISH (完成)
```

---

### 4.6 UploadMediaType常量
```
1 = IMAGE
2 = VIDEO
3 = FILE
4 = VOICE
```

---

### 4.7 TypingStatus常量
```
1 = TYPING (开始输入)
2 = CANCEL (停止输入)
```

---

## 5. 媒体上传和下载

### 5.1 CDN上传流程

**CDN基础URL**: `https://novac2c.cdn.weixin.qq.com/c2c`

**步骤**:
1. 计算文件MD5: `md5(plaintext)`
2. 生成AES密钥: `crypto.randomBytes(16)`
3. 生成文件key: `crypto.randomBytes(16).toString("hex")`
4. 计算加密后大小: `aesEcbPaddedSize(rawsize)` = AES-128-ECB + PKCS7填充
5. 调用 `getUploadUrl()` 获取CDN URL
6. 使用AES-128-ECB加密文件: `encryptAesEcb(plaintext, aeskey)`
7. POST加密数据到CDN URL: 
   - 方法: POST
   - Content-Type: `application/octet-stream`
   - Body: 加密的二进制数据
8. 从响应头 `x-encrypted-param` 获取下载参数
9. 获取响应成功（status 200），返回上传结果

**重试策略**:
- 最多重试3次
- 4xx错误立即失败
- 5xx错误重试
- 超时重试

**构建CDN URL**（当返回upload_param时）:
```
{cdnBaseUrl}?{upload_param}
```

---

### 5.2 媒体下载流程

**下载参数来自**:
- 入站消息中的 `message_item.xxx_item.media.encrypt_query_param`
- 上传后CDN返回的 `x-encrypted-param` 响应头

**步骤**:
1. 从消息项获取 `encrypt_query_param` 和 `aes_key`
2. 从CDN下载加密数据:
   ```
   GET {cdnBaseUrl}?{encrypt_query_param}
   ```
3. 使用AES-128-ECB解密: `decryptAesEcb(ciphertext, aeskey)`
4. 保存本地文件

**注意**:
- `aes_key` 在JSON中为Base64编码
- 若消息项中有 `aeskey` 字段（16进制），优先使用该字段

---

### 5.3 AES-128-ECB加密参数

**算法**: AES-128-ECB
**密钥长度**: 16字节（128位）
**填充方式**: PKCS7
**模式**: ECB（不使用初始向量）

**填充计算**:
```
paddedSize = ((plaintext.length + 15) / 16) * 16
// 或
paddedSize = ((plaintext.length - 1) / 16 + 1) * 16
```

---

## 6. 请求头构建

### 6.1 X-WECHAT-UIN 生成
```
随机uint32 → 转为十进制字符串 → Base64编码
```

**示例代码**:
```javascript
function randomWechatUin() {
  const uint32 = crypto.randomBytes(4).readUInt32BE(0);
  return Buffer.from(String(uint32), "utf-8").toString("base64");
}
```

---

### 6.2 iLink-App-ClientVersion 编码

**格式**: `(major << 16) | (minor << 8) | patch`

**示例**:
- 版本 `2.0.11` → `((2 & 0xff) << 16) | ((0 & 0xff) << 8) | (11 & 0xff)` → `0x0002000B` → `131083`

---

### 6.3 Bot Agent 格式

**格式** (User-Agent风格):
```
Name/Version [Name/Version ...] [(comment)]
```

**规则**:
- 默认值: `OpenClaw`
- 最大长度: 256字节（UTF-8）
- 无效token自动丢弃
- 若清理后为空，使用默认值

**有效示例**:
- `OpenClaw`
- `MyBot/1.2.0`
- `MyBot/1.2.0 (region=cn)`
- `MyBot/1.2.0 LangChain/0.3.5`

---

## 7. 错误处理

### 7.1 HTTP错误
- HTTP状态码非2xx时，抛出错误
- 响应体作为错误详情

### 7.2 API响应错误
- `ret !== 0` 或 `errcode !== 0` 表示错误
- `errcode === -14`: 会话过期，暂停60分钟后重试
- 其他错误: 指数退避重试（2秒 → 30秒）

### 7.3 长轮询超时
- AbortError且abortSignal已中止: 正常退出
- AbortError但abortSignal未中止: 客户端超时，返回空响应继续

### 7.4 会话过期保护
```
如果accountId在过期列表中，暂停所有API调用
剩余暂停时间从getUpdates返回的-14错误时开始计算
```

---

## 8. 流程图

### 8.1 消息处理完整流程
```
getUpdates()
  ↓ 检查响应
  ├─ ret=0且无errcode: 继续处理
  ├─ errcode=-14: 暂停60分钟，返回sleep状态
  └─ 其他错误: 指数退避重试
  
  ↓ 保存get_updates_buf到磁盘
  
  ↓ 遍历msgs列表
  ├─ 下载+解密媒体（可选）
  ├─ 保存context_token
  ├─ 路由到Agent
  └─ 处理回复：
      ├─ 获取typing_ticket (getConfig)
      ├─ 发送typing开始指示 (sendTyping status=1)
      ├─ 等待AI处理
      ├─ 发送typing结束指示 (sendTyping status=2)
      └─ 发送消息:
          ├─ 若有媒体: getUploadUrl → CDN上传 → sendImageMessage/sendVideoMessage/sendFileMessage
          └─ 若无媒体: sendMessage
```

### 8.2 QR码登录流程
```
startWeixinLoginWithQr()
  ↓ GET /ilink/bot/get_bot_qrcode?bot_type=3
  ↓ 返回qrcodeUrl和sessionKey
  
waitForWeixinLogin(sessionKey)
  ↓ 循环轮询 GET /ilink/bot/get_qrcode_status?qrcode={qrcode}
  ├─ status=wait: 继续轮询
  ├─ status=scaned: 已扫描，继续轮询
  ├─ status=scaned_but_redirect: 更新polling地址，继续轮询
  ├─ status=need_verifycode: 从stdin读取配对码，包含在下次请求中
  ├─ status=verify_code_blocked: 刷新QR码，重新轮询
  ├─ status=expired: 刷新QR码（最多3次）
  ├─ status=binded_redirect: 已绑定，返回alreadyConnected=true
  └─ status=confirmed: 返回botToken、ilink_bot_id、baseUrl、userId
```

### 8.3 媒体上传流程
```
uploadMediaToCdn()
  ↓ 读取文件
  ├─ 计算MD5
  ├─ 生成AES密钥
  ├─ 生成filekey
  ├─ 计算加密后大小
  ↓ 调用 getUploadUrl()
  ↓ 获取upload_full_url或upload_param
  ↓ AES-128-ECB加密
  ↓ uploadBufferToCdn()
    ├─ POST加密数据到CDN
    ├─ 重试3次（4xx立即失败，5xx重试）
    └─ 从响应头获取x-encrypted-param
  ↓ 返回downloadEncryptedQueryParam、aeskey、fileSize、fileSizeCiphertext
```

### 8.4 媒体下载流程
```
downloadMediaFromItem()
  ├─ IMAGE: 获取encrypt_query_param和aes_key → 下载 → 解密 → 保存
  ├─ VOICE: 获取参数 → 下载 → 解密SILK → 转码WAV（可选） → 保存
  ├─ FILE: 获取参数 → 下载 → 解密 → 保存
  └─ VIDEO: 获取参数 → 下载 → 解密 → 保存
```

---

## 9. 配置项

### 9.1 Base URLs
- **API基础URL**: `https://ilinkai.weixin.qq.com`
- **CDN基础URL**: `https://novac2c.cdn.weixin.qq.com/c2c`
- **QR码固定URL**: `https://ilinkai.weixin.qq.com`（不可自定义）

### 9.2 超时配置
- **getUpdates**: 35秒
- **sendMessage**: 15秒
- **getUploadUrl**: 15秒
- **getConfig**: 10秒
- **sendTyping**: 10秒
- **notifyStart/notifyStop**: 10秒

### 9.3 重试配置
- **CDN上传**: 3次重试，4xx立即失败，5xx重试
- **API错误**: 指数退避，2秒 → 30秒
- **QR刷新**: 最多3次

### 9.4 缓存配置
- **getConfig缓存**: 24小时TTL，随机刷新
- **Context Token**: 内存+磁盘双层缓存
- **getUpdates同步缓冲**: 磁盘持久化

---

## 10. 常量定义

### 10.1 消息类型
```
MESSAGE_TYPE_USER = 1
MESSAGE_TYPE_BOT = 2

MESSAGE_ITEM_TYPE_TEXT = 1
MESSAGE_ITEM_TYPE_IMAGE = 2
MESSAGE_ITEM_TYPE_VOICE = 3
MESSAGE_ITEM_TYPE_FILE = 4
MESSAGE_ITEM_TYPE_VIDEO = 5

MESSAGE_STATE_NEW = 0
MESSAGE_STATE_GENERATING = 1
MESSAGE_STATE_FINISH = 2

UPLOAD_MEDIA_TYPE_IMAGE = 1
UPLOAD_MEDIA_TYPE_VIDEO = 2
UPLOAD_MEDIA_TYPE_FILE = 3
UPLOAD_MEDIA_TYPE_VOICE = 4

TYPING_STATUS_TYPING = 1
TYPING_STATUS_CANCEL = 2

QR_STATUS_WAIT = "wait"
QR_STATUS_SCANED = "scaned"
QR_STATUS_CONFIRMED = "confirmed"
QR_STATUS_EXPIRED = "expired"
QR_STATUS_NEED_VERIFYCODE = "need_verifycode"
QR_STATUS_VERIFY_CODE_BLOCKED = "verify_code_blocked"
QR_STATUS_SCANED_BUT_REDIRECT = "scaned_but_redirect"
QR_STATUS_BINDED_REDIRECT = "binded_redirect"

SESSION_EXPIRED_ERRCODE = -14
SESSION_PAUSE_DURATION_MS = 3600000 // 60分钟

DEFAULT_ILINK_BOT_TYPE = "3"
DEFAULT_BOT_AGENT = "OpenClaw"

QR_LONG_POLL_TIMEOUT_MS = 35000
```

---

## 11. 数据验证规则

### 11.1 文件大小计算
```
filesize (AES加密后) = ((rawsize + 15) >> 4) << 4
// 等同于:
filesize = Math.ceil(rawsize / 16) * 16
```

### 11.2 MD5计算
```
rawfilemd5 = md5(plaintext).toString("hex")
// 结果为小写16进制字符串
```

### 11.3 AES密钥格式
```
JSON中: base64编码的AES密钥
请求中: 16进制字符串格式的AES密钥
转换: hex → Buffer.from(hex, "hex").toString("base64")
```

### 11.4 Context Token验证
```
必须：
- 从入站消息获取并保存
- 在回复消息中原样返回
- 用于维持对话上下文
```

---

## 12. 特殊处理

### 12.1 多账户场景
```
单个OpenClaw可同时运行多个微信账户
各账户通过accountId隔离
context_token通过 accountId:userId 的二维映射维护
```

### 12.2 IDC重定向
```
若QR轮询返回 status=scaned_but_redirect
使用响应中的 redirect_host 更新polling地址
后续所有请求使用新地址
```

### 12.3 已绑定检测
```
若QR轮询返回 status=binded_redirect
表示此bot已绑定到当前OpenClaw
无需重新登录，本地凭证仍然有效
返回 alreadyConnected=true
```

### 12.4 消息分割
```
单条文本消息最大4000字符
超长文本需分割为多条消息逐一发送
```

### 12.5 长轮询机制
```
getUpdates采用服务端长轮询：
- 若有消息立即返回
- 若无消息等待直到超时
- 不需要客户端频繁轮询
- 服务器可通过longpolling_timeout_ms调整超时
```

