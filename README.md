# CtYun 天翼云桌面保活工具

本项目用于保持天翼云桌面的活跃状态，防止因长时间无操作而被服务器断开连接。

## 保活接口说明

本项目使用 WebSocket 连接到天翼云桌面代理服务器 (`wss://{host}/clinkProxy/{desktopId}/MAIN`)，通过 REDQ 协议发送和响应保活心跳包。保活程序采用独立连接和被动响应模式，只处理特定的保活校验包，**不会干扰用户的正常远程桌面操作**。

**技术文档**:
- [保活接口详细文档](KEEPALIVE_INTERFACE.md) - 完整的协议说明、对比分析
- [MAIN 通道通信模拟](MAIN_CHANNEL_SIMULATION.md) - WebSocket 通信流程模拟演示

## 使用指南

### windows用户直接下载Releases执行即可。

首次登录需要绑定新设备接收验证码,windows生成的设备信息在DeviceCode.txt文件中

### docker使用指南
> :warning: **提示：** docker第一次运行不要后台执行，不要加-d运行，要添加-it， 第一次软件会生成一个新的设备信息，需要接收短信来进行风控校验，需手动输入，提示设备 **保活任务启动** 即可后台运行。

设备号DEVICECODE是web_加上随机32大小写字母数字_
例如 **web_L53itptDslz6manpE8Uq2Op1OEoKi85t** 
不要填写案例，请自己生成或更改上方案例

Linux 可使用以下代码生成
```
echo "web_$(cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 32 | head -n 1)"
```
```
//第一次初始化运行这个。
docker run -it \
  --name ctyun \
  -e APP_USER="你的账号" \
  -e APP_PASSWORD='你的密码' \
  -e DEVICECODE='设备Id' \
  su3817807/ctyun:latest

```

```
//第一次运行不要加-d
docker run -d \
  --name ctyun \
  -e APP_USER="你的账号" \
  -e APP_PASSWORD='你的密码' \
  -e DEVICECODE='设备Id' \
  su3817807/ctyun:latest

```
### 查看日志检查是否登录并连接成功。

```
docker logs -f ctyun

```


验证码识别api方案来自 https://github.com/sml2h3/ddddocr
