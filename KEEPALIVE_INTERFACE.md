# CtYun 云桌面保活接口说明 (Cloud Desktop Keep-Alive Interface Documentation)

> **重要说明**: 本文档描述的是**天翼云桌面 (Cloud Desktop)** 的保活协议，与**天翼云手机 (Cloud Phone)** 使用的协议不同。云手机使用不同的心跳机制和多通道架构，详见文档末尾的对比说明。
>
> **Important Note**: This document describes the keep-alive protocol for **CtYun Cloud Desktop**, which differs from the **Cloud Phone** protocol. Cloud Phone uses a different heartbeat mechanism and multi-channel architecture. See comparison at the end.

## 概述 (Overview)

本项目通过 WebSocket 连接实现天翼云桌面的保活功能，确保云桌面会话保持活跃状态，避免因长时间无操作而被服务器断开连接。

This project implements keep-alive functionality for CtYun (China Telecom Cloud) **desktops** through WebSocket connections, ensuring cloud desktop sessions remain active and preventing disconnection due to inactivity.

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

### 6. 为什么不会顶号？(Why No Account Conflicts?)

**"顶号"是指多个客户端使用同一账号登录时，新的登录会踢掉旧的会话。本保活程序不会造成顶号，原因如下：**

#### 6.1 使用已绑定的设备码 (Uses Bound Device Code)
```csharp
var cyApi = new CtYunApi(deviceCode);  // 使用本地存储的设备码
```
- 每个设备有唯一的设备码 (Device Code)，格式为 `web_` + 32位随机字符串
- 设备码在首次使用时需要通过短信验证绑定到账号
- 后续使用该设备码不会触发顶号，因为服务器识别为**同一设备的多个连接**

#### 6.2 只建立监控连接，不创建新会话 (Monitoring Connection, Not New Session)
```csharp
var connectResult = await cyApi.ConnectAsync(d.DesktopId);
// API: POST /api/desktop/client/connect
```
- 调用的是 `connect` 接口，获取已存在桌面的连接信息
- **不是**创建新桌面或新登录会话的接口
- 仅建立 MAIN 通道的 WebSocket 连接进行保活监控

#### 6.3 连接类型不冲突 (Connection Type Compatibility)
保活程序建立的连接特点：
- **MAIN 通道**: 仅用于心跳和认证验证
- **无显示通道**: 不建立 DISPLAY、CURSOR、INPUTS 等控制通道
- **被动监听**: 只响应服务器的挑战包，不发送任何控制指令

正常用户连接的特点：
- **完整通道**: 建立 MAIN + DISPLAY + CURSOR + INPUTS 等完整通道
- **交互操作**: 发送鼠标、键盘输入，接收屏幕画面
- **主动控制**: 实际控制云桌面

两种连接类型可以**共存**，因为：
1. 服务器允许同一设备的多个连接（设备码相同）
2. MAIN 通道的保活连接不占用交互资源
3. 保活连接只维持会话状态，不执行实际操作

#### 6.4 设备绑定机制 (Device Binding Mechanism)
```csharp
// 首次使用需要绑定设备
await api.GetSmsCodeAsync(userphone);
await api.BindingDeviceAsync(verificationCode);
```
- 新设备首次使用时必须通过短信验证绑定
- 绑定后该设备码与账号关联，成为**授权设备**
- 授权设备的连接不会相互踢出，因为服务器认为是合法的多点接入

#### 6.5 会话层与连接层的区别 (Session vs Connection Layer)
```
用户账号 (Account)
  └─ 会话 (Session) ← 保活维护这一层，防止超时
      ├─ 保活连接 (Keep-alive Connection): MAIN 通道 WebSocket
      └─ 用户连接 (User Connection): 完整的远程桌面连接
```

- **会话层**: 账号登录状态，由 Token/Cookie 维护
- **连接层**: 实际的网络连接（WebSocket、RDP等）
- 保活维护的是**会话层**的活跃状态
- 不影响**连接层**的用户实际操作

#### 6.6 实际应用场景 (Practical Use Case)
```
情景：用户需要长时间运行任务但偶尔才需要查看

1. 用户在云桌面上启动长时间任务（如数据处理、渲染等）
2. 启动保活程序，维持会话不超时
3. 用户断开远程桌面连接（任务继续运行）
4. 保活程序继续工作，防止会话被回收
5. 用户需要时可以随时重新连接查看进度
6. ✓ 用户重连时不会被顶号，因为会话一直保持活跃
```

#### 6.7 技术验证 (Technical Verification)
从代码实现可以验证不会顶号：
```csharp
// 1. 使用绑定的设备码
client.DefaultRequestHeaders.Add("ctg-devicecode", deviceCode);

// 2. 只连接 MAIN 通道
var uri = new Uri($"wss://{host}/clinkProxy/{desktopId}/MAIN");

// 3. 只响应保活挑战，不发送控制命令
if (hex.StartsWith("5245445102")) {
    var response = encryptor.Execute(data);
    await ws.SendAsync(response, ...);
}
```

**总结**: 保活程序通过使用已绑定的设备码、仅建立监控性质的 MAIN 通道连接、被动响应模式等机制，确保不会与用户的正常连接冲突，实现了"保持会话活跃"而"不干扰用户操作"的目标。

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

## 云桌面 vs 云手机协议对比 (Cloud Desktop vs Cloud Phone Protocol Comparison)

### MAIN 通道的相似之处 (MAIN Channel Similarities)

云桌面和云手机的 MAIN 通道有很多共同点：

1. **相同的 WebSocket 路径格式**: 都使用 `wss://{server}/clinkProxy/{desktopId}/MAIN`
2. **相同的握手机制**: 建立连接后，首先发送 JSON 格式的握手消息，包含 SSL 证书信息
   ```json
   {
     "type": 1,
     "ssl": 1,
     "host": "<host>",
     "port": "<port>",
     "ca": "<CaCert>",
     "cert": "<ClientCert>",
     "key": "<ClientKey>",
     "servername": "<Host>:<Port>"
   }
   ```
3. **相同的 RSA 挑战-响应机制**: 两者都使用 RSA 加密的挑战响应来验证客户端身份
   - 服务器发送挑战数据（包含公钥）
   - 客户端使用相同的 RSA 加密算法处理并返回响应
   - 加密过程包括 SHA-1 哈希和 PKCS#1 填充
4. **二进制消息格式**: 都使用 ArrayBuffer (二进制) 格式传输数据
5. **独立的保活连接**: MAIN 通道都是独立于实际数据传输的控制/保活通道
6. **不干扰用户操作**: 保活机制都不会影响正常的远程桌面/手机操作

### MAIN 通道的区别 (MAIN Channel Differences)

虽然底层的 RSA 挑战-响应机制相同，但两者在保活策略和额外协议上有所不同：

| 特性 | 云桌面 (Cloud Desktop) | 云手机 (Cloud Phone) |
|------|----------------------|---------------------|
| **挑战包标识** | `52 45 44 51 02` (REDQ) | 可能使用相同或类似标识 |
| **主动心跳** | 无，仅被动响应挑战 | 有，每 5 秒发送 `07 00 00 00 00 00` |
| **心跳响应** | 无简单心跳 | 收到 `09 00 00 00 00 00` |
| **时间同步** | 无 | 有 (`03 00` / `04 00`) |
| **设备信息交换** | 无 | 有 (`6b 00` / `6d 00`) |
| **传感器通知** | 无 | 有 (`82 00`) |
| **重连策略** | 60秒强制重连 | 依靠持续心跳 |

### 其他差异 (Other Differences)

| 特性 | 云桌面 (Cloud Desktop) | 云手机 (Cloud Phone) |
|------|----------------------|---------------------|
| **WebSocket 通道数** | 仅 MAIN | 8 个通道 (MAIN + DISPLAY + CURSOR + RECORD + PLAYBACK + PORT×2 + DATA + INPUTS) |
| **显示传输** | 可能使用 RDP/VNC 等其他协议 | 通过独立 DISPLAY 通道 |
| **输入传输** | 可能使用 RDP/VNC 等其他协议 | 通过独立 INPUTS 通道 |
| **端口** | 动态分配 | 固定 9011 |

### 云桌面协议详情 (Cloud Desktop Protocol)

**本项目实现的协议**：
- **初始包**: `52 45 44 51 02 00 00 00 02 00 00 00 1a 00 ...` (REDQ + payload)
- **挑战包**: 服务器发送以 `52 45 44 51 02` 开头的校验数据
- **响应包**: 客户端使用 RSA 加密后返回
- **工作模式**: 被动响应，只在收到挑战时才回应
- **连接策略**: 每 60 秒强制断开重连，防止连接僵死

### 云手机协议详情 (Cloud Phone Protocol)

**心跳包示例**：
```
发送: 07 00 00 00 00 00  # 心跳请求
接收: 09 00 00 00 00 00  # 心跳响应
```

**时间同步包**：
```
发送: 03 00 0c 00 00 00 [12 bytes timestamp/sequence]
接收: 04 00 0c 00 00 00 [12 bytes timestamp/sequence]
```

**设备信息包**：
```
发送: 6b 00 [length] [payload]  # 分辨率、JSON配置
接收: 6d 00 [length] [payload]  # 设备状态、JSON配置
```

**传感器通知**：
```
接收: 82 00 0b 00 00 00 07 00 00 00 "battery"
接收: 82 00 10 00 00 00 0c 00 00 00 "acceleration"
接收: 82 00 0f 00 00 00 0b 00 00 00 "hinge_angle"
```

### 为什么有这些差异？(Why These Differences?)

1. **设备类型差异**: 云桌面是完整的虚拟机环境，云手机是模拟移动设备
2. **功能需求**: 云手机需要传感器数据 (电池、加速度、铰链角度等) 和更频繁的状态同步
3. **显示架构**: 云手机使用多个独立通道分离不同类型的数据流，云桌面可能使用集成的远程协议
4. **保活策略**: 云桌面使用周期性重连，云手机使用持续心跳维持连接

### 相同点 (Similarities)

上述 MAIN 通道的相似之处已经详细说明，核心共同点包括：
1. 相同的 WebSocket 连接路径和握手方式
2. **相同的 RSA 挑战-响应加密机制** (这是最重要的相似点)
3. 二进制消息格式和独立连接设计
4. 都不干扰正常的用户操作

## 总结 (Summary)

CtYun 云桌面保活接口通过 WebSocket 连接到云桌面代理服务器，使用 **REDQ 协议**发送和响应保活心跳包。由于采用独立连接和被动响应模式，只处理特定的保活校验包，不会干扰用户的正常远程桌面操作。这种设计既实现了会话保持的目的，又不影响用户体验。

**重要发现**: 云桌面和云手机的 MAIN 通道在核心机制上**高度相似**，特别是：
- 都使用相同的 WebSocket 路径格式和 JSON 握手
- **都使用相同的 RSA 挑战-响应加密机制**来验证客户端
- 主要差异在于云手机增加了额外的心跳包、时间同步、设备信息等辅助协议，以及使用了多通道架构

The CtYun cloud desktop keep-alive interface connects to the cloud desktop proxy server via WebSocket and uses the **REDQ protocol** to send and respond to keep-alive heartbeat packets. Since it uses an independent connection and passive response mode, only handling specific keep-alive challenge packets, it does not interfere with normal user remote desktop operations. This design achieves session maintenance without affecting user experience.

**Important Finding**: The MAIN channels of cloud desktop and cloud phone are **highly similar** in core mechanisms:
- Both use the same WebSocket path format and JSON handshake
- **Both use the same RSA challenge-response encryption mechanism** for client verification
- The main differences are that cloud phone adds additional heartbeat packets, time sync, device info, and other auxiliary protocols, plus uses a multi-channel architecture
