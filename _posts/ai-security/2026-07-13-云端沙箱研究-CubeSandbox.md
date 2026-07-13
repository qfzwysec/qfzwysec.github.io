---
layout: post
title: 云端沙箱研究：CubeSandbox
date: 2026-07-13 15:00:00 +0800
categories: AI安全
tag: 智能体安全
description: 梳理腾讯云开源 CubeSandbox 的控制面、数据面、MicroVM 生命周期、网络出入站控制、CubeProxy 路由以及 Agent 接入方式。
cover: /styles/images/cubesandbox-research/cover.png
---

CubeSandbox 是腾讯云开源的一套 Agent 沙箱基础设施。它给每个任务启动一台轻量、短生命周期的 Linux 微虚拟机，并在外部统一管理它的生命周期、文件、网络、访问凭据和审计记录。

# 一、整体架构

CubeSandbox 可以分为控制面和数据面。

- **控制面**负责接收请求、调度节点、保存状态和提供管理界面。
- **数据面**负责真正启动微虚拟机、准备磁盘、转发网络和执行安全策略。

![CubeSandbox 整体架构](/styles/images/cubesandbox-research/e9189695ad5588823b04f041096fd483.png)

## 1. Cube API：对外入口

CubeAPI 是兼容 E2B 接口的 REST API 网关。Agent 或程序调用 `Sandbox.create()` 时，请求先到 CubeAPI。CubeAPI 负责参数转换、可选鉴权以及把请求转交给 CubeMaster。

![CubeAPI 接入示例](/styles/images/cubesandbox-research/file-20260713130026434.png)

兼容 E2B 的意义在于，原本使用 E2B SDK 的程序通常只需要修改 API 地址和模板 ID，就可以改为连接自己的 CubeSandbox。

E2B配置

```bash
export E2B_API_URL=http://10.122.220.95:3000
export E2B_API_KEY=e2b_000000
```

沙箱模版

```bash
# 使用 CubeSandbox 中已经创建好的模板
export CUBE_TEMPLATE_ID=tpl-5c1f414b65bb414ba1fd2c7b
```

Agent执行

```python
import os
from e2b_code_interpreter import Sandbox

with Sandbox.create(template=os.environ["CUBE_TEMPLATE_ID"]) as sandbox:
    result = sandbox.commands.run("echo hello cube && uname -m")
    print(result.stdout)
```

此时`Sandbox.create()` 的创建请求会发送到`http://10.122.220.95:3000`，然后CubeAPI 将它转换并转交给 CubeMaster；CubeMaster 调度 Cubelet 从指定模板启动 MicroVM。随后 `commands.run()` 的执行请求经 CubeProxy 进入该 MicroVM。离开 `with` 代码块时，SDK 会请求销毁沙箱。


## 2. CubeMaster：集群调度中心

CubeMaster 决定沙箱应该在哪台计算节点上运行，并协调创建、销毁、暂停、恢复、快照和克隆等操作。Redis 用于共享生命周期事件和路由状态，MySQL 保存需要持久化的控制面数据。

![CubeMaster 调度与生命周期管理](/styles/images/cubesandbox-research/file-20260713130543448.png)


## 3. Cubelet：计算节点管理员

每台计算节点运行一个 Cubelet。它接到 CubeMaster 的指令后，准备沙箱需要的磁盘、内存、网络和运行时资源，然后调用 CubeShim 启动微虚拟机。可以把 Cubelet 理解为每台宿主机上的“现场管理员”。

## 4. CubeShim 与 CubeHypervisor：真正启动微虚拟机

CubeShim 实现 containerd Shim v2 接口，把上层的容器式生命周期操作转换成微虚拟机操作。CubeHypervisor 基于 RustVMM 和 Linux KVM，负责创建 vCPU、内存、虚拟磁盘、虚拟网卡和 vsock 通信设备。

## 5. CubeCoW：模板、快照和克隆

CubeCoW 利用 XFS 的 reflink 和 Linux `FICLONE` 能力管理沙箱磁盘。模板中的大部分磁盘数据可以由多个沙箱共享；只有发生修改的块才需要单独写入。

预制模板
   ├── CoW 克隆 → 沙箱 A 的可写层
   ├── CoW 克隆 → 沙箱 B 的可写层
   └── CoW 克隆 → 沙箱 C 的可写层
   
因此，启动一个临时小电脑时，不需要重新安装 Linux、Python、浏览器和其他依赖。平台直接复制一份模板的“使用视图”，这个复制主要是元数据操作。

如果沙箱运行期间执行 `apt install` 或 `pip install`，新文件只写入该沙箱的可写层。沙箱销毁后，这些修改默认消失。若希望以后继续使用，应把环境制作成新模板或快照。

## 6. CubeVS：宿主机内核中的网络数据面

每个沙箱通过独立 TAP 设备连接宿主机。CubeVS 使用 eBPF 在宿主机内核中处理网络转发、NAT、连接跟踪和按沙箱划分的 IP/CIDR 策略。

主要负责网络层问题：

- 这个沙箱能不能连接某个 IP？
- 私网、宿主机地址和云元数据地址是否应该拒绝？
- 这条连接应该直接出网，还是交给 CubeEgress 做更细的 HTTP 检查？

它需要处理
1. `from_cube`：处理沙箱发出的数据包
2. `from_world`：处理返回沙箱的数据包

## 7. CubeEgress：受控出网代理

CubeEgress 是部署在宿主机上的透明 HTTP/HTTPS 代理。CubeVS 将需要七层检查的流量通过 TPROXY 引导到 CubeEgress。CubeEgress 可以检查域名、TLS SNI、HTTP Host、方法和路径，并按照规则允许、拒绝、审计或添加请求头。
注意，但不是所有的流量都要经过CubeEgress。

在创建沙箱时，可以配置入站出站规则

![沙箱网络规则配置](/styles/images/cubesandbox-research/file-20260713141946856.png)

ui中其实缺少了一些功能，如果只在ui中配置了ip规则，这个CubeEgress是用不到的

所有的ip规则在创建后，会被统一解析到一个`allow_out_v2`，后续CubeVS看的就是这个

沙箱模版其实还接受这种写法

```yaml
{
  "templateID": "模板ID",
  "network": {
    "rules": [
      {
        "name": "api_rule",
        "match": {
          "sni": "api.example.com"
        },
        "action": {
          "allow": true
        }
      }
    ]
  }
}
```

当匹配到这种规则时，rule中的sni或者host会被提取为`L7AllowOut`写入`allow_out_v2`，并携带一个标志

```
NET_POLICY_FLAG_L7_REQUIRED = 1
```

处理结果就是

``` 
api.example.com
      ↓ DNS解析
203.0.113.10
      ↓
allow_out_v2：
203.0.113.10/32
flags = L7_REQUIRED
```

当CubeVS看到目标`should_redirect_to_l7_proxy()`也就是带有`L7_REQUIRED`标记就会交给CubeEgress。

```
CubeVS把TCP 80/443流量交给CubeEgress
                ↓
CubeEgress识别这是哪个沙箱的请求
                ↓
读取SNI、Host、Method、Path、Scheme
                ↓
按顺序匹配该沙箱的rules
                ↓
执行action
                ↓
允许：转发给原始服务器
拒绝：返回HTTP 403
```

也就是说，比如用户访问一个ip。这个ip在allow_out_v2没有L7_REQUIRED 就直接放行，如果有就需要再交给CubeEgress看对应的域名是否正确。

## 8.  CubeProxy：从客户端访问具体沙箱

CubeEgress 管理“沙箱向外访问”，CubeProxy 则管理“客户端向沙箱访问”，两者方向不同。

E2B SDK 会访问类似下面的域名：

```text
<端口>-<sandbox_id>.cube.app
```

CubeProxy 从域名中解析目标端口和沙箱 ID，再查询沙箱所在位置，将请求转发到对应 MicroVM。配套的 lifecycle manager 还能在请求到来时恢复已经自动暂停的沙箱。

假设创建了一个沙箱：
```
sandbox_id = sb-abc123
sandbox IP = 192.168.0.20
所在节点 = 10.0.0.15
```
然后在沙箱中启动一个网页服务：
```
python3 -m http.server 8000
```
这个服务实际监听在：
```
192.168.0.20:8000
```
但客户端不能直接访问这个沙箱IP。客户端使用CubeProxy提供的地址：
```
https://8000-sb-abc123.cube.app
```
完整处理过程如下。

#### 第一步：请求到达CubeProxy
浏览器发送：
```
GET / HTTP/1.1
Host: 8000-sb-abc123.cube.app
```
`*.cube.app` 会被解析到CubeProxy地址，所以请求首先进入CubeProxy，而不是直接进入沙箱。
#### 第二步：从Host中提取目标

CubeProxy解析：
```
8000-sb-abc123.cube.app
│    │
│    └── sandbox_id = sb-abc123
└─────── container_port = 8000
```
对应源码会执行：
```
local container_port, ins_id =
    hostname:match("^(%d+)%-([^%.]+)")
```
见 `rewrite_phase.lua (line 5)`（`CubeProxy/lua/rewrite_phase.lua:5`）。
现在CubeProxy知道：
```
要访问的沙箱：sb-abc123
要访问的沙箱端口：8000
```
#### 第三步：检查沙箱是否暂停
CubeProxy查询本地保存的沙箱状态。
如果状态是：
```
running
```
就直接继续路由。
如果状态是：
```
paused
```
CubeProxy先向配套sidecar发送内部请求：
```
POST /internal/resume?sandbox_id=sb-abc123
```
恢复完成后，原来的浏览器请求才继续处理：
```
浏览器请求
    ↓
发现沙箱已暂停
    ↓
请求恢复sb-abc123
    ↓
MicroVM恢复运行
    ↓
继续处理原浏览器请求
```
因此客户端不需要先手动调用 `Sandbox.connect()`。恢复逻辑见 `sandbox_state.lua (line 78)`（`CubeProxy/lua/sandbox_state.lua:78`）。

#### 第四步：查询沙箱所在位置
CubeProxy根据 `sandbox_id` 查询Redis中的路由元数据：
```
sb-abc123
├── HostIP     = 10.0.0.15
├── SandboxIP  = 192.168.0.20
└── 8000       = 节点映射端口
```
这些数据由CubeMaster在创建或恢复沙箱时写入。
查询代码见 `sandbox_backend.lua (line 60)`（`CubeProxy/lua/sandbox_backend.lua:60`）。
接下来分两种情况。
如果沙箱就在CubeProxy本机：
```
CubeProxy → 192.168.0.20:8000
```
CubeProxy直接访问沙箱IP和原始端口。
如果沙箱在另一台计算节点：
```
CubeProxy → 10.0.0.15:映射端口 → 192.168.0.20:8000
```
CubeProxy先访问目标节点的映射端口，再由该节点上的CubeVS转发到沙箱。
对应判断见 `sandbox_backend.lua (line 173)`（`CubeProxy/lua/sandbox_backend.lua:173`）。

#### 第五步：转发请求并返回响应
CubeProxy确定最终后端地址后，将它设置为nginx的上游：
```
balancer.set_current_peer(backend_ip, backend_port)
```
见 `balancer_phase.lua (line 8)`（`CubeProxy/lua/balancer_phase.lua:8`）。
最终链路是：
```
浏览器
  ↓
https://8000-sb-abc123.cube.app
  ↓
CubeProxy解析：
  sandbox_id = sb-abc123
  port = 8000
  ↓
如果暂停，先恢复MicroVM
  ↓
查询Redis路由
  ↓
找到192.168.0.20:8000
  ↓
沙箱中的Python HTTP服务
  ↓
响应沿原路返回浏览器
```

其意义是：**给所有沙箱提供一个统一、稳定的访问入口，客户端不需要知道沙箱IP、所在节点和当前运行状态。**

# 二、接入方式

## 1. 非E2B Agent通过工具或Skill调用沙箱
有些Agent既没有使用E2B，也没有内置CubeSandbox支持，例如OpenClaw一类可以执行本地工具的Agent。
这种情况下，可以给Agent提供一个沙箱工具或使用说明，让它在需要执行代码时调用CubeSandbox：
官方有提供一个skill，CubeSandbox/examples/openclaw-integration/skills/cube-sandbox/SKILL.md
```
用户要求Agent处理工作区
        ↓
Agent调用沙箱工具
        ↓
创建MicroVM
        ↓
在/workspace中执行代码
        ↓
读取结果
        ↓
销毁沙箱
```
## 2. 兼容E2B的Agent通过SDK接入沙箱
前面有说到过，只需要修改E2B API地址和模板ID：
```
export E2B_API_URL=http://10.122.220.95:3000
export E2B_API_KEY=e2b_000000
export CUBE_TEMPLATE_ID=tpl-xxx
```
使用：
```
with Sandbox.create(template=os.environ["CUBE_TEMPLATE_ID"]) as sandbox:
    result = sandbox.commands.run("python3 task.py")
```
调用链从：
```
Agent → E2B云服务
```
变成：
```
Agent → CubeAPI → CubeMaster → Cubelet → MicroVM
```
这种方式适合已经支持E2B的Agent、代码解释器和开发工具，迁移成本最低。
这里要说明的是如果不希望agent所有行为都在沙箱里，那就不能再提供别的操作本地文件，代码执行的工具。提供的所有工具都要封装成sandbox。例如
```
sandbox.commands.run("python3 task.py")
```
## 3. 直接使用内置AgentHub

也可以直接使用内置 AgentHub：

![内置 AgentHub](/styles/images/cubesandbox-research/file-20260713150835672.png)
