<!---

MIT License

Copyright (c) 2025 Burning_TNT (pangyl08@163.com)

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

-->

# Scaffolding：Minecraft 联机客户端数据交换协议

## 目标

- 在 [EasyTier](https://easytier.cn/) 的基础上建立协议；
- 允许联机客户端相互交换和 Minecraft 联机的相关数据；
- 带有充足可拓展性：允许各联机客户端自定义拓展协议交换更多数据。

## 名词定义

- 联机客户端：运行在联机成员各自的电脑上，拼接参数并拉起 EasyTier 实例的应用程序；
- 联机中心：承担 Scaffolding 协议中心服务器职责的联机客户端，运行在与 Minecraft 服务器 相同的物理机上；
- 联机房客：处于联机网络中的其他联机客户端；
- 联机房间码：由联机中心向联机房客通过外部渠道传递的形如 `U/NNNN-NNNN-SSSS-SSSS` 代码，用于标识唯一的联机房间。

# 协议定义

## 联机房间码标准

联机中心应在 Minecraft 服务器启动后，通过密码学安全的随机数生成器，创建符合 `U/NNNN-NNNN-SSSS-SSSS` 形式的联机房间码，满足以下约束：

- N 和 S 为任意大写字母（除去 I 和 O）和数字；
- 按照 0-9、A-H、J-N、P-Z顺序映射至 \[0, 33\] 后，依“小端序”读得的整型应能被7整除。

## 联机网络标准

EasyTier 网络名称应为 `scaffolding-mc-NNNN-NNNN`，网络密钥应为 `SSSS-SSSS`。
例如，房间码 `U/YNZE-U61D-2206-HXRG` 对应的 EasyTier 网络名称和密钥应分别为 `scaffolding-mc-YNZE-U61D` 和 `2206-HXRG`。

## 联机中心发现协议

每个 EasyTier 网络有且应仅有一联机中心。
其 Hostname (https://easytier.cn/guide/network/configurations.html#其他设置) 应当为：
scaffolding-mc-server-{port: uint16}。
其中，port 必须是小于等于 65535 且大于 1024 的整数。例如：

- scaffolding-mc-server-33768：合法；
- scaffolding-mc-server-77844：不合法，77844 大于 65535；
- scaffolding-mc-server-n8：不合法，n8 不是整数。
  如果一个网络中找不到合法的联机中心，联机客户端应提示玩家联机中心发现失败，并不再运行后续协议。

> [!CAUTION]
> 若一个 EasyTier 网络中存在多个符合上述协议的联机中心，则将为【未定义行为】

## 联机信息交换协议

该协议建立于 TCP 协议之上，地址为联机中心的虚拟 IP 地址，端口为联机中心的 port 参数。
联机中心将在该接口开放 TCP服务器，接受符合如下标准的请求（大端序）：

| 代号     | 偏移量        | 类型     | 解释                                                       |
|--------|------------|--------|----------------------------------------------------------|
| 请求类型长度 | 0          | uint8  | 字段*请求类型*的字节长度                                            |
| 请求类型   | 1          | byte[] | 请求类型，应符合 {namespace: ascii string}:{value: ascii string} |
| 请求体长度  | 1 + 请求类型长度 | uint32 | 字段*请求体*的字节长度                                             |
| 请求体    | 5 + 请求类型长度 | /      | 请求体，视*请求类型*而定                                            |

其中，namespace 和 value 都应当是仅包含小写字母、下划线、数字的ASCII字符串。
联机中心应返回如下标准的请求：

| 代号    | 偏移量 | 类型     | 解释                  |
|-------|-----|--------|---------------------|
| 状态    | 0   | uint8  | 响应状态                |
| 响应体长度 | 1   | uint32 | 字段*响应体*的字节长度（可能为 0） |
| 响应体   | 5   | /      | 响应体（可能为空）           |

如果处理请求成功，状态应为 0，响应体将视*请求类型*而定；如果请求失败，则应返回以下状态：

| 状态       | 错误类型    | 响应体            | 响应体格式   |
|----------|---------|----------------|---------|
| [32, 64) | *请求类型*定义的错误 | 若未由*请求类型*明确指出，默认为空 | /       |
| 255      | 未知错误    | 错误详细内容         | UTF8字符串 |

### 请求类型

请求体和响应体由*请求类型*规定。请求类型分为核心、标准和拓展请求类型

- 核心请求类型：所有联机中心和联机房客都必须完整实现的最小请求类型集合，包括且：`c:ping`, `c:protocols`,` c:server_port`, `c:player_ping`, `c:player_profiles_list`；
- 标准请求类型：由 Scaffolding-MC 规范直接规定的请求类型，以 c (community) 作为命名空间的全部请求类型；
- 拓展请求类型：由社区自发实现的其他请求类型。

所有核心请求类型都是标准请求类型。
拓展请求类型的制定者可递交 SEP 来添加特定拓展请求类型到标准请求类型中。

#### 字段解释

##### machine_id: string

各联机客户端应根据硬件信息生成足够长的字符串，避免碰撞，并尽力保证对同一设备始终产出稳定的结果。

#### 标准请求类型

##### c:ping [核心请求类型]

测试联机中心是否正常运行

- 请求体：任意内容
- 请求体格式：二进制内容，长度应小于 32 字节
- 响应体：与请求体内容一致
- 响应体格式：二进制内容，长度应与请求体一致

##### c:protocols [核心请求类型]

与联机中心协商请求类型列表

- 请求体：联机房客所支持的请求类型列表；
- 请求体格式：由 `\0` 分割的多个 ASCII String，如 c:protocols\0c:server_addresses\0c:player_name；
- 响应体：联机中心所支持的请求类型列表；
- 响应体格式：由 `\0` 分割的多个 ASCII String，如 c:protocols\0c:server_addresses\0c:player_name。

##### c:server_port [核心请求类型]

获取 Minecraft 服务器的地址

- 请求体：空；
- 响应体：Minecraft服务器的在联机中心上的端口；
- 响应体格式：大端序编码的 2 字节 u16 数据，FF FF。
- 错误状态：
  * 32：服务器未启动

##### c:player_ping [心跳/5秒] [核心请求类型]

- 请求体：玩家 ID 和设备 machine_id
- 请求体格式（JSON）：{ name: string, machine_id: string, vendor: string }
- 响应体：空

##### c:player_profiles_list [核心请求类型]

- 请求体：空
- 响应体：玩家列表（包括联机中心）
- 响应体格式（JSON）：[{ name: string, machine_id: string, vendor: string, kind: 'HOST' | 'GUEST' }]

#### 拓展请求类型

各联机客户端可以自行实现请求类型，选用自己的命名空间。
请尽量选用联机客户端的名称作为命名空间，以避免命名空间冲突！

## 联机流程

1. 联机客户端加入对应 EasyTier 网络；
2. 联机客户端根据联机中心发现协议，确定联机信息交换协议的 TCP 服务器；
3. 立刻发送 `c:player_ping` 心跳包（在请求类型协商之前）；
4. （如果客户端仅支持核心请求类型，可跳过此步）在联机信息交换协议上发送 `c:protocols`，获取联机中心支持的请求类型，并与当前联机客户端支持的请求类型取交集，确定本次联机可用的全部请求类型；
5. 在联机信息交换协议上发送 `c:server_port`，获取Minecraft服务器地址；
6. 每隔 5s 发送一次 `c:player_ping` 心跳包。
