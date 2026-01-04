# CtYun 保活接口说明 (Keep-Alive Interface Documentation)

## 概述 (Overview)

本项目通过 WebSocket 连接实现天翼云桌面的保活功能，确保云桌面会话保持活跃状态，避免因长时间无操作而被服务器断开连接。

This project implements keep-alive functionality for CtYun (China Telecom Cloud) desktops through WebSocket connections, ensuring cloud desktop sessions remain active and preventing disconnection due to inactivity.

## 保活接口 (Keep-Alive Interface)

### WebSocket 连接端点 (WebSocket Endpoint)

```
wss://{ClinkLvsOutHost}/clinkProxy/{DesktopId}/MAIN
```

**参数说明 (Parameters):**
- `ClinkLvsOutHost`: 从连接接口获取的云桌面代理主机地址
- `DesktopId`: 云桌面的唯一标识符

**请求头 (Request Headers):**
```
Origin: https://pc.ctyun.cn
Sec-WebSocket-Protocol: binary
```

### 连接流程 (Connection Flow)

#### 1. 建立 WebSocket 连接
通过 WebSocket 协议连接到云桌面代理服务器。

#### 2. 发送握手信息
连接建立后，发送 JSON 格式的握手消息：

```json
{
  "type": 1,
  "ssl": 1,
  "host": "<ClinkLvsOutHost_without_port>",
  "port": "<port>",
  "ca": "<CaCert>",
  "cert": "<ClientCert>",
  "key": "<ClientKey>",
  "servername": "<Host>:<Port>"
}
```

这个握手消息包含了 SSL/TLS 连接所需的证书信息，用于建立安全的通信通道。

#### 3. 发送初始保活包
握手完成后（延迟 500ms），发送初始二进制保活包：

```
Base64: UkVEUQIAAAACAAAAGgAAAAAAAAABAAEAAAABAAAAEgAAAAkAAAAECAAA
Hex: 52 45 44 51 02 00 00 00 02 00 00 00 1a 00 00 00 00 00 00 00 01 00 01 00 00 00 01 00 00 00 12 00 00 00 09 00 00 00 04 08 00 00
```

这是 REDQ 协议的初始化数据包。

## REDQ 协议 (REDQ Protocol)

### 协议标识 (Protocol Identifier)
保活协议使用 REDQ (Remote Desktop Query) 协议，特征码为：
```
REDQ (0x52 0x45 0x44 0x51)
```

### 保活校验包 (Keep-Alive Challenge)
服务器会定期发送保活校验包，包头特征：
```
Hex: 52 45 44 51 02 ...
(REDQ + version byte 0x02)
```

### 响应机制 (Response Mechanism)
客户端接收到校验包后，需要通过加密算法处理并返回响应包。响应逻辑在 `Encryption.cs` 中实现，使用 RSA 加密算法对接收到的数据进行处理。

## 为什么不影响正常客户端使用 (Why It Doesn't Affect Normal Client Usage)

### 1. 独立连接 (Independent Connection)
保活程序建立的是**独立的 WebSocket 连接**，与正常用户使用的远程桌面客户端连接是分离的。

- 保活连接：专门用于发送和接收保活心跳包
- 用户连接：正常的远程桌面协议（RDP/VNC）连接，用于实际桌面操作

### 2. 只处理心跳包 (Only Handles Heartbeat Packets)
保活程序**仅响应特定的保活校验包**（0x5245445102 开头的数据包），不会：
- 发送鼠标/键盘输入
- 修改桌面内容
- 干扰用户操作
- 消耗显著的系统资源

### 3. 被动响应模式 (Passive Response Mode)
保活机制采用**被动响应**模式：
- 只在服务器发送校验包时才响应
- 不主动发送控制指令
- 不占用远程桌面的输入通道

### 4. 会话保持机制 (Session Maintenance)
保活的目的是保持服务器端的**会话记录活跃**，防止因空闲超时而被服务器回收资源。这与用户的实际操作是两个独立的层面：
- 会话层：保活维护会话不被超时
- 应用层：用户的实际远程桌面操作

### 5. 多设备并发支持 (Multi-Device Concurrency)
程序为每个云桌面创建独立的保活任务（`KeepAliveWorkerWithForcedReset`），支持多台设备同时保活，互不干扰。

## 技术实现细节 (Technical Implementation Details)

### 定时重连机制 (Timed Reconnection)
- 每个 WebSocket 连接维持 **60 秒**
- 60 秒到期后自动断开并重新建立连接
- 这样可以防止连接僵死，同时避免频繁重连

```csharp
sessionCts.CancelAfter(TimeSpan.FromMinutes(1)); // 60秒后自动触发取消
```

### 异常处理 (Error Handling)
- 连接失败时自动重试
- 重试间隔 5 秒，防止请求风暴
- 支持优雅关闭（Graceful Shutdown）

### 接收循环 (Receive Loop)
```csharp
while (ws.State == WebSocketState.Open && !ct.IsCancellationRequested)
{
    var result = await ws.ReceiveAsync(new ArraySegment<byte>(buffer), ct);
    
    if (result.MessageType == WebSocketMessageType.Close) break;
    
    if (result.Count > 0)
    {
        var data = buffer.AsSpan(0, result.Count).ToArray();
        var hex = BitConverter.ToString(data).Replace("-", "");
        
        // 检测保活校验包
        if (hex.StartsWith("5245445102", StringComparison.OrdinalIgnoreCase))
        {
            // 加密并返回响应
            var response = encryptor.Execute(data);
            await ws.SendAsync(response, WebSocketMessageType.Binary, true, ct);
        }
    }
}
```

## 安全性考虑 (Security Considerations)

1. **SSL/TLS 加密**: 使用 wss:// 协议，所有通信都经过 TLS 加密
2. **证书验证**: 使用服务器提供的 CA 证书、客户端证书和密钥进行双向认证
3. **设备绑定**: 需要通过短信验证码绑定设备，防止未授权访问
4. **请求签名**: API 请求使用 MD5 签名验证（ctg-signaturestr），防止篡改

## API 认证机制 (API Authentication)

保活功能需要先通过 CtYun API 进行身份认证：

1. **获取挑战码** (`genChallengeData`)
2. **登录验证** (`/api/auth/client/login`)
3. **设备绑定** (`/api/cdserv/client/device/binding`)
4. **获取桌面列表** (`/api/desktop/client/pageDesktop`)
5. **建立连接** (`/api/desktop/client/connect`)

所有 API 请求都包含以下认证头：
```
ctg-devicetype: 60
ctg-version: 103020001
ctg-devicecode: <设备码>
ctg-userid: <用户ID>
ctg-tenantid: <租户ID>
ctg-timestamp: <时间戳>
ctg-requestid: <请求ID>
ctg-signaturestr: <MD5签名>
```

## 使用场景 (Use Cases)

1. **长时间挂机**: 需要云桌面长时间运行任务但不需要实时操作
2. **批量设备管理**: 同时保持多台云桌面的活跃状态
3. **自动化任务**: 在云桌面上运行自动化脚本，避免会话超时
4. **资源预留**: 保持云桌面分配状态，避免被系统回收

## 总结 (Summary)

CtYun 保活接口通过 WebSocket 连接到云桌面代理服务器，使用 REDQ 协议发送和响应保活心跳包。由于采用独立连接和被动响应模式，只处理特定的保活校验包，不会干扰用户的正常远程桌面操作。这种设计既实现了会话保持的目的，又不影响用户体验。

The CtYun keep-alive interface connects to the cloud desktop proxy server via WebSocket and uses the REDQ protocol to send and respond to keep-alive heartbeat packets. Since it uses an independent connection and passive response mode, only handling specific keep-alive challenge packets, it does not interfere with normal user remote desktop operations. This design achieves session maintenance without affecting user experience.
