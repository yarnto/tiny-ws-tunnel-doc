## 1. 自写原因

之前自己架设在家里的树莓派和香橙派上的个人网盘，日记等服务分别使用过`frp`和`nps`作为内网穿透代理工具。
奈何frp没有web控制台，且配置略繁琐，而nps在使用过程中发现不定期在没有任何实际业务负载的情况下cpu占用率持续100%的情况。
这两个工具使用的痛点让我最终选择业余时间自写了一个微型的内网穿透代理工具`tiny-ws-tunnel`或者简称`twt`，供自己使用。

**实现原理图：**

![#707px #300px](https://www.wavesxa.com/opendoc/attachment/278-schematic-diagram.jpg?id=59)

## 2. tiny-ws-tunnel的部分特性

- 支持 `tls`
- 服务端可前置nginx或caddy等反向代理。让反向代理支持tls
- agent 支持 tls 证书验证
- agent 支持连接认证
- 数据传输支持aes(aead)加密 (在不使用tls时推荐应用)
- 数据流量统计
- 支持 web api
- buffer调优
- 自带一个基于curl的管理工具脚本
- 支持在docker中运行
- 经过时间考验的tcp代理以及试用阶段的udp（udp over tcp）代理

目前自己已使用了半年多时间，运行`稳定`，`占用资源少`，圆满满足自己的需求，并提供给公司使用。下面简述使用方法。且后面视情况可能开源。

## 3. 简明使用方式（quick start）

### 3.1 首先下载

github https://github.com/yarnto/tiny-ws-tunnel-doc 的release页面下载。包括 linux amd64 arm64 armv7 的程序包，还有windows的 amd64 程序包。
备用下载地址：https://pan.hk.zhangyt.fun/s/bRtk

绿色软件，下载解压即安装。

**程序包目录结构**

```bash
conf  # 配置文件目录，初次启动为空目录，会在执行命令的时候自动生成相关配置文件
tiny-ws-tunnel  # 程序本身
twt # 脚本工具
```

> 以下假设所有命令都在安装目录下执行

### 3.2 服务端启动

> 通常服务端在有公网 IP 或出口的设备上启动

启动脚本

```shell
# 通过 ./tiny-ws-tunnel server -h 查看更多参数选项
./tiny-ws-tunnel server -serverAddr=:3699 \
  -needAgentConnectingAuthorized=true
```

> 启动后如果发现：
> `ERRO[0000] read conf/port-mappings.json err: open conf/port-mappings.json: no such file or directory`
> 的错误日志可忽略，下面会通过twt脚本工具手动创建一个认证的agent key(密钥)，自动生成该文件

该命令启动twt server，监听在`:3699`(即0.0.0.0:3699)地址，且设置了agent连接需要验证。
简单示例起见，此时没有启用单独加密和tls。可通过`./tiny-ws-tunnel version -h`参数配置tls和（或）aes单独加密。

### 3.2 添加一个认证的agent密钥

添加认证的agent。用于agent连接时的认证参数使用

```shell
# 添加一个 agentId 为 test-agent-1 ，密钥为 ta1-key 的agent认证记录
zhangyt@ubt20-server:~/tmp/twt$ ./twt addAuthorizedAgent test-agent-1 ta1-key
{
    "code": 10000,
    "msg": "ok"
}
```

列表已经添加的认证agent

```shell
zhangyt@ubt20-server:~/tmp/twt$ ./twt listAuthorizedAgents
{
    "code": 10000,
    "msg": "ok",
    "data": {
        "test-agent-1": {
            "AgentId": "test-agent-1",
            "Key": "ta1-key",
            "CreateTime": "2023-02-22T15:08:42.854401696+08:00"
        }
    }
}
```

### 3.3 启动agent

> 通常 agent 在一个没有公网 IP 的内网设备启动，作为例子，本agent在相同地机器上启动

```shell
# 通过 ./tiny-ws-tunnel agent -h 查看更多参数选项
zhangyt@ubt20-server:~/tmp/twt$ ./tiny-ws-tunnel agent -agentId=test-agent-1 -key=ta1-key -serverAddr="ws://127.0.0.1:3699/twt/ws"
2023/02/22 15:21:32 start dnsCache timeout check goruntinue
INFO[0000] heart beat will start in 15 second           
INFO[0000] agent test-agent-1 connected url ws://127.0.0.1:3699/twt/ws 
```

说明：

- `-agentId=test-agent-1`， `-key=ta1-key` 指定agentId和key
- `-serverAddr="ws://127.0.0.1:3699/twt/ws"` 指定要连接的server(通常是公网地址，https(tls)的以wss开头)


### 3.4 查看已经连接的agent

这时服务端可以通过如下命令查看已经连接的agent
明细会有非常有用的agent状态信息

```
zhangyt@ubt20-server:~/tmp/twt$ ./twt listConnectedAgents
{
    "code": 10000,
    "msg": "ok",
    "data": [
        {
            "AgentId": "test-agent-1",
            "ReadBytes": 0,
            "WriteBytes": 0,
            "FirstConnectTime": "2023-02-22T15:19:56.837436745+08:00",
            "LastConnectTime": "2023-02-22T15:21:32.94364389+08:00",
            "RemoteAddr": "127.0.0.1:51208",
            "LastAgentPingTime": "2023-02-22T15:24:48.00824876+08:00",
            "Connected": "on",
            "HReadBytes": "0 Bytes",
            "HWriteBytes": "0 Bytes"
        }
    ]
}
```

### 3.5 添加一个地址端口映射（添加代理）

把公网server端的地址端口映射到agent所在局域网设备地址，本质上是添加了一个代理。`这是我们地最终目标`

我们假设本地有一个httpbin的web的服务监听在8077端口，以下命令是在server机器上启动一个本地:8077的代理监听在0.0.0.0:8078的端口上

```shell
# addPortMapping listeningAddr targetAddr agentId note
zhangyt@ubt20-server:~/tmp/twt$ ./twt addPortMapping :8078 127.0.0.1:8077 test-agent-1 "a local agent"
{
    "code": 10000,
    "msg": "ok",
    "data": {
        "listenerId": 1
    }
}
```

说明：

- listeningAddr 公网监听地址
- targetAddr 目标地址，对于udp的代理映射，需要在地址后面加"/udp"
- agentId agent id
- note 简要描述（可选）

这时我们可以检测一下访问本机的8078端口是否等同于访问8077端口

```
root@ubt20-server:/opt/httpbin# curl http://localhost:8077/user-agent
{
  "user-agent": "curl/7.68.0"
}
root@ubt20-server:/opt/httpbin# curl http://localhost:8078/user-agent
{
  "user-agent": "curl/7.68.0"
}
```

上面的输出示例，说明已经成功了。

假设本机服务端启动在外网，agent启动在一个能访问外网server的内网，访问外网的8078端口就相当于访问内网的8077端口，实现了内网穿透访问。

## 4. 关于前置nginx或caddy代理

前面特性描述已经提到过使用 nginx 或 caddy 等作为前置代理。如果前置代理监听在标准的80和443端口，且已经有正规的tls证书。那twt server则可以直接利用前置代理的tls，基本可以省去使用aes加密，如果不用tls则推荐使用自带的aes加密以保护数据安全。

如下为caddy的配置示例 `Caddyfile`

```bash
your.public.domain {
  handle /* {
    file_server browse
  }
  
  # /twt/是 twt web server 默认地contextPath，可通过 -serverContextPath 指定别地
  handle /twt/* {
    reverse_proxy 127.0.0.1:3699 {
      header_up Tiny-Real-IP {remote}
    }
  }
}
```

在配置中我设置了一个头信息 `Tiny-Real-IP` 是为了让twt server知道从caddy连接过来的连接地源地址是什么。nginx的配置方式类似，就是设置一个反向代理，记得设置传递一个头信息。
