# MAIN 通道 WebSocket 通信模拟 (MAIN Channel WebSocket Communication Simulation)

本文档基于实际代码实现，模拟展示云桌面 MAIN 通道的完整 WebSocket 通信过程。

This document simulates the complete WebSocket communication process of the Cloud Desktop MAIN channel based on the actual code implementation.

---

## 通信流程概览 (Communication Flow Overview)

```
客户端 (Client)                                    服务器 (Server)
    |                                                    |
    |---- 1. WebSocket 连接请求 ------------------------>|
    |                                                    |
    |<--- 2. WebSocket 连接建立 (101 Switching) ---------|
    |                                                    |
    |---- 3. JSON 握手消息 (Text) ---------------------->|
    |     包含 SSL 证书信息                               |
    |                                                    |
    |---- 4. 初始 REDQ 保活包 (Binary) ----------------->|
    |                                                    |
    |     [保持连接，等待挑战]                            |
    |                                                    |
    |<--- 5. RSA 挑战包 (Binary, REDQ 0x5245445102) -----|
    |                                                    |
    |---- 6. RSA 加密响应 (Binary) ---------------------->|
    |                                                    |
    |     [循环：步骤 5-6 重复]                          |
    |                                                    |
    |---- 7. 60秒到，主动断开连接 ----------------------->|
    |                                                    |
    |---- 8. 重新连接，回到步骤 1 ----------------------->|
```

---

## 详细通信步骤 (Detailed Communication Steps)

### 步骤 1: WebSocket 连接请求 (Step 1: WebSocket Connection Request)

**客户端请求**:
```http
GET wss://example.ctyun.server:port/clinkProxy/22312730/MAIN HTTP/1.1
Host: example.ctyun.server:port
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Sec-WebSocket-Protocol: binary
Origin: https://pc.ctyun.cn
User-Agent: Mozilla/5.0 ...
```

**关键点**:
- URL 路径: `/clinkProxy/{desktopId}/MAIN`
- 子协议: `binary`
- Origin: `https://pc.ctyun.cn`

---

### 步骤 2: WebSocket 连接建立 (Step 2: WebSocket Connection Established)

**服务器响应**:
```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
Sec-WebSocket-Protocol: binary
```

此时 WebSocket 连接已建立，可以进行双向二进制通信。

---

### 步骤 3: JSON 握手消息 (Step 3: JSON Handshake Message)

**客户端发送** (WebSocketMessageType.Text):
```json
{
  "type": 1,
  "ssl": 1,
  "host": "example.host.com",
  "port": "8443",
  "ca": "-----BEGIN CERTIFICATE-----\nMIID...(CA证书)...=\n-----END CERTIFICATE-----",
  "cert": "-----BEGIN CERTIFICATE-----\nMIID...(客户端证书)...=\n-----END CERTIFICATE-----",
  "key": "-----BEGIN RSA PRIVATE KEY-----\nMIIE...(客户端私钥)...=\n-----END RSA PRIVATE KEY-----",
  "servername": "remote.server.com:9443"
}
```

**说明**:
- 消息类型: 文本 (Text)
- 包含完整的 SSL/TLS 证书信息
- `type: 1` 表示连接类型
- `ssl: 1` 表示启用 SSL

**代码实现**:
```csharp
var connectMessage = new ConnecMessage {
    type = 1,
    ssl = 1,
    host = desktop.DesktopInfo.ClinkLvsOutHost.Split(":")[0],
    port = desktop.DesktopInfo.ClinkLvsOutHost.Split(":")[1],
    ca = desktop.DesktopInfo.CaCert,
    cert = desktop.DesktopInfo.ClientCert,
    key = desktop.DesktopInfo.ClientKey,
    servername = desktop.DesktopInfo.Host + ":" + desktop.DesktopInfo.Port
};
await client.SendAsync(msgBytes, WebSocketMessageType.Text, true, token);
```

---

### 步骤 4: 初始 REDQ 保活包 (Step 4: Initial REDQ Keep-Alive Packet)

**等待 500ms 后，客户端发送** (WebSocketMessageType.Binary):

**Base64 编码**:
```
UkVEUQIAAAACAAAAGgAAAAAAAAABAAEAAAABAAAAEgAAAAkAAAAECAAA
```

**十六进制 (Hex)**:
```
52 45 44 51 02 00 00 00 02 00 00 00 1a 00 00 00
00 00 00 00 01 00 01 00 00 00 01 00 00 00 12 00
00 00 09 00 00 00 04 08 00 00
```

**协议解析**:
```
52 45 44 51    # "REDQ" 魔数 (Magic Number)
02 00 00 00    # 版本或类型标识
02 00 00 00    # 字段 1
1a 00 00 00    # 字段 2 (26)
00 00 00 00    # 字段 3
01 00 01 00    # 字段 4
00 00 01 00    # 字段 5
00 00 12 00    # 字段 6 (18)
00 00 09 00    # 字段 7 (9)
00 00 04 08    # 字段 8
00 00          # 字段 9
```

**代码实现**:
```csharp
const string InitialPayloadBase64 = "UkVEUQIAAAACAAAAGgAAAAAAAAABAAEAAAABAAAAEgAAAAkAAAAECAAA";
await Task.Delay(500, token);
await client.SendAsync(Convert.FromBase64String(InitialPayloadBase64), 
                       WebSocketMessageType.Binary, true, token);
```

---

### 步骤 5: 等待并接收 RSA 挑战包 (Step 5: Receive RSA Challenge Packet)

**服务器发送挑战包** (WebSocketMessageType.Binary):

**包头标识**:
```
52 45 44 51 02 ...  # REDQ + 版本 0x02
```

**完整示例** (变长，包含 RSA 公钥):
```
52 45 44 51 02 00 00 00 XX XX XX XX ...
│  │  │  │  │              │
│  │  │  │  │              └─ 后续数据（包含公钥等）
│  │  │  │  └─ 版本/类型 (0x02)
│  │  │  └─ 'Q'
│  │  └─ 'D'
│  └─ 'E'
└─ 'R'
```

**数据结构** (从索引 16 开始是有效载荷):
```
[0-15]:   头部信息
[16+]:    有效载荷
  [32-160]:  RSA 公钥的 n 值 (129 字节)
  [163-165]: RSA 公钥的 e 值 (3 字节, 24位整数)
```

**客户端接收代码**:
```csharp
var result = await ws.ReceiveAsync(new ArraySegment<byte>(buffer), token);
var data = buffer.AsSpan(0, result.Count).ToArray();
var hex = BitConverter.ToString(data).Replace("-", "");

if (hex.StartsWith("5245445102", StringComparison.OrdinalIgnoreCase))
{
    // 这是 RSA 挑战包
    Console.WriteLine("收到保活校验");
}
```

---

### 步骤 6: RSA 加密响应 (Step 6: RSA Encrypted Response)

**客户端处理挑战并返回响应**:

#### 6.1 解析公钥
```csharp
// 从接收到的挑战数据中提取公钥
Memory<byte> payload = new Memory<byte>(data, 16, data.Length - 16);
byte[] nBytes = payload.ToArray().AsMemory(32, 129).ToArray();  // 公钥 n
int e = Read24BitValue(payload.ToArray(), 163);                // 公钥 e
```

#### 6.2 生成随机数并构造填充
```csharp
// 生成 20 字节随机数
byte[] random = new byte[20];
RandomNumberGenerator.Fill(random);

// OAEP 填充处理
// 1. 计算填充长度 (128 - 1 - 20 = 107)
// 2. 使用 SHA-1 哈希生成种子
// 3. MGF1 掩码生成函数
// 4. XOR 操作混合数据
```

#### 6.3 RSA 公钥加密
```csharp
// 使用服务器提供的公钥进行 RSA 加密
BigInteger message = new BigInteger(paddedData);
BigInteger encrypted = BigInteger.ModPow(message, e, n);
byte[] encryptedBytes = encrypted.ToByteArray();
```

#### 6.4 构造响应包
```csharp
// 在加密数据前添加 auth_mechanism (4 字节)
using (var ms = new MemoryStream())
using (var writer = new BinaryWriter(ms))
{
    writer.Write((uint)1);           // auth_mechanism = 1
    writer.Write(encryptedBytes);    // RSA 加密后的数据
    return ms.ToArray();
}
```

**响应包格式**:
```
01 00 00 00    # auth_mechanism (4 字节, 值为 1)
XX XX XX ...   # RSA 加密数据 (128 字节)
```

**发送响应**:
```csharp
var response = encryptor.Execute(data);
await ws.SendAsync(response, WebSocketMessageType.Binary, true, token);
Console.WriteLine("发送保活响应成功");
```

---

### 步骤 7: 循环处理 (Step 7: Loop Processing)

**持续监听**:
```csharp
while (ws.State == WebSocketState.Open && !token.IsCancellationRequested)
{
    var result = await ws.ReceiveAsync(new ArraySegment<byte>(buffer), token);
    
    if (result.MessageType == WebSocketMessageType.Close)
        break;
    
    if (result.Count > 0)
    {
        var data = buffer.AsSpan(0, result.Count).ToArray();
        var hex = BitConverter.ToString(data).Replace("-", "");
        
        // 检测并处理 REDQ 挑战包
        if (hex.StartsWith("5245445102", StringComparison.OrdinalIgnoreCase))
        {
            var response = encryptor.Execute(data);
            await ws.SendAsync(response, WebSocketMessageType.Binary, true, token);
        }
    }
}
```

**特点**:
- 被动响应模式
- 只处理特定的 REDQ 挑战包 (0x5245445102)
- 不主动发送心跳
- 不干扰其他类型的消息

---

### 步骤 8: 60秒重连策略 (Step 8: 60-Second Reconnection Strategy)

**定时器触发**:
```csharp
using var sessionCts = CancellationTokenSource.CreateLinkedTokenSource(globalToken);
sessionCts.CancelAfter(TimeSpan.FromMinutes(1));  // 60秒后自动取消

try {
    await ReceiveLoop(client, desktop, sessionCts.Token);
}
catch (OperationCanceledException) {
    Console.WriteLine("60秒时间到，准备重连...");
}
```

**优雅关闭**:
```csharp
if (client.State == WebSocketState.Open)
{
    await client.CloseOutputAsync(WebSocketCloseStatus.NormalClosure, 
                                   "Timeout Reset", 
                                   CancellationToken.None);
}
```

**重新连接**:
- 回到步骤 1，建立新的 WebSocket 连接
- 重新发送握手和初始包
- 继续保活循环

---

## 时序图 (Sequence Diagram)

```
时间轴 →

T=0s     : 连接建立
T=0.1s   : 发送 JSON 握手
T=0.6s   : 发送初始 REDQ 包
T=5s     : 收到挑战包 → 发送响应
T=15s    : 收到挑战包 → 发送响应
T=25s    : 收到挑战包 → 发送响应
T=35s    : 收到挑战包 → 发送响应
T=45s    : 收到挑战包 → 发送响应
T=55s    : 收到挑战包 → 发送响应
T=60s    : 主动断开连接
T=60.5s  : 重新连接（回到 T=0s）
```

---

## 实际数据示例 (Real Data Example)

### 示例 1: 握手消息
```json
{
  "type": 1,
  "ssl": 1,
  "host": "192.168.1.100",
  "port": "8443",
  "ca": "-----BEGIN CERTIFICATE-----\nMIIDXTCCAkWgAwIBAgI...",
  "cert": "-----BEGIN CERTIFICATE-----\nMIIDYTCCAkmgAwIBAg...",
  "key": "-----BEGIN RSA PRIVATE KEY-----\nMIIEowIBAAKCAQE...",
  "servername": "desktop.ctyun.cn:9443"
}
```

### 示例 2: 初始 REDQ 包
```
方向: Client → Server
类型: Binary
时间: T+600ms
数据: 52 45 44 51 02 00 00 00 02 00 00 00 1a 00 00 00 00 00 00 00 
      01 00 01 00 00 00 01 00 00 00 12 00 00 00 09 00 00 00 04 08 00 00
长度: 42 字节
```

### 示例 3: RSA 挑战包
```
方向: Server → Client
类型: Binary
时间: T+5s
数据: 52 45 44 51 02 00 00 00 [变长，包含 RSA 公钥数据]
长度: 约 180-200 字节
标识: 开头 5 字节为 52 45 44 51 02
```

### 示例 4: RSA 响应包
```
方向: Client → Server
类型: Binary
时间: T+5.1s
数据: 01 00 00 00 [128 字节 RSA 加密数据]
长度: 132 字节 (4 字节头 + 128 字节数据)
```

---

## 关键代码片段 (Key Code Snippets)

### 完整的保活循环
```csharp
static async Task KeepAliveWorkerWithForcedReset(Desktop desktop, CancellationToken globalToken)
{
    const string InitialPayloadBase64 = "UkVEUQIAAAACAAAAGgAAAAAAAAABAAEAAAABAAAAEgAAAAkAAAAECAAA";
    var uri = new Uri($"wss://{desktop.DesktopInfo.ClinkLvsOutHost}/clinkProxy/{desktop.DesktopId}/MAIN");

    while (!globalToken.IsCancellationRequested)
    {
        using var sessionCts = CancellationTokenSource.CreateLinkedTokenSource(globalToken);
        sessionCts.CancelAfter(TimeSpan.FromMinutes(1));

        using var client = new ClientWebSocket();
        client.Options.SetRequestHeader("Origin", "https://pc.ctyun.cn");
        client.Options.AddSubProtocol("binary");

        try
        {
            // 1. 连接
            await client.ConnectAsync(uri, sessionCts.Token);

            // 2. 握手
            var connectMessage = new ConnecMessage { /* ... */ };
            var msgBytes = JsonSerializer.SerializeToUtf8Bytes(connectMessage);
            await client.SendAsync(msgBytes, WebSocketMessageType.Text, true, sessionCts.Token);

            // 3. 初始包
            await Task.Delay(500, sessionCts.Token);
            await client.SendAsync(Convert.FromBase64String(InitialPayloadBase64), 
                                   WebSocketMessageType.Binary, true, sessionCts.Token);

            // 4. 接收循环
            await ReceiveLoop(client, desktop, sessionCts.Token);
        }
        catch (OperationCanceledException)
        {
            // 60秒到，准备重连
        }
        finally
        {
            if (client.State == WebSocketState.Open)
                await client.CloseOutputAsync(WebSocketCloseStatus.NormalClosure, 
                                               "Timeout Reset", CancellationToken.None);
        }
    }
}
```

### RSA 加密处理
```csharp
public byte[] Execute(byte[] challengeData)
{
    // 1. 提取载荷（从索引 16 开始）
    Memory<byte> payload = new Memory<byte>(challengeData, 16, challengeData.Length - 16);
    
    // 2. 提取公钥
    byte[] nBytes = payload.ToArray().AsMemory(32, 129).ToArray();
    int e = Read24BitValue(payload.ToArray(), 163);
    
    // 3. 生成随机数和 OAEP 填充
    byte[] random = new byte[20];
    RandomNumberGenerator.Fill(random);
    byte[] paddedData = OAEPPadding(random, 128);
    
    // 4. RSA 加密
    BigInteger message = new BigInteger(paddedData);
    BigInteger n = new BigInteger(nBytes);
    BigInteger encrypted = BigInteger.ModPow(message, e, n);
    
    // 5. 构造响应
    using (var ms = new MemoryStream())
    using (var writer = new BinaryWriter(ms))
    {
        writer.Write((uint)1);  // auth_mechanism
        writer.Write(encrypted.ToByteArray());
        return ms.ToArray();
    }
}
```

---

## 为什么不影响正常客户端 (Why It Doesn't Affect Normal Clients)

1. **独立的 WebSocket 连接**: MAIN 通道是单独的保活连接，不占用实际的远程桌面数据通道
2. **被动响应模式**: 只响应服务器的挑战包，不主动发送任何控制指令
3. **特定包识别**: 只处理 `0x5245445102` 开头的包，其他消息全部忽略
4. **无输入干扰**: 不发送鼠标、键盘等输入事件
5. **低资源消耗**: 大部分时间处于等待状态，只在收到挑战时才执行加密计算

---

## 总结 (Summary)

云桌面的 MAIN 通道保活机制是一个精心设计的 **被动响应式安全认证系统**：

1. **连接建立**: WebSocket + 子协议 binary
2. **身份握手**: JSON 格式的 SSL 证书交换
3. **初始化**: 发送 REDQ 协议初始包
4. **持续认证**: 被动响应 RSA 挑战-响应循环
5. **定期重连**: 60秒强制断开重连，防止连接僵死

这种设计既保证了会话的持续活跃，又通过 RSA 加密确保了安全性，同时不会对用户的正常操作产生任何影响。
