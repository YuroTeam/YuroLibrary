---
tags:
  - 网络协议/icmp
  - wireshark
  - 以太网
  - ttl
  - 网络抓包
  - 计算机网络
  - 数据链路层
  - mac地址
  - ipv4
  - ping命令
---

在 WireShark 中抓取 `ping baidu.com` 时的 ICMP 数据包

WireShark 是非常强大的网络抓包工具，并且被广泛用于网络安全等领域，是学校计算机网络的好工具。

WireShark 抓包的机制非常底层，可能工作在**数据链路层**（或更低），它捕获的是网卡接收到的完整链路层数据单元。也就是二进制比特流，WireShark 会将二进制比特流解析为数据链路层的以太网帧，能看帧里的 MAC 地址、类型字段（`0x0800` 表示里面装的是 IPv4）、VLAN 标签等。

打开 WireShark，因为我们使用的是无限局域网，所以选择 Wi-Fi:en0 即可，电脑如果接了网线，可能会有一个以太网的选项。

具体就是看哪个网卡是有流量的，例如我这里就是 Wi-Fi:en0 和 Loopback:lo0 是有流量的，而我们知道 Loopback 是环回地址，所以真正应该抓包的应该是 Wi-Fi:en0 的数据。

![[Pasted image 20260518211516.png]]

然后可以看到 WireShark 就在不停地抓包：

![[Pasted image 20260518211601.png]]

因为我们要对 `ping baidu.com` 并抓包，而 `ping` 走的是 ICMP 协议，所以我们可以直接过滤：

![[Pasted image 20260518211729.png]]

过滤以后，当前是什么都没有，因为此时我们没有 `ping` 任何网址：

![[Pasted image 20260518211814.png]]

来到终端，使用 `ping baidu.com` 让 WireShark 抓一会包：

![[Pasted image 20260518211942.png]]

经过了 4 次 ICMP，我们已经可以在 WireShark 中看到 Ping 的结果了：

![[Pasted image 20260518212121.png]]

如果已经拿到了想要的数据就可以暂停抓包了，我这里已经暂停了：

![[Pasted image 20260518214854.png]]

百度的 IP 是 124.237.177.164，所以找到对应的条目就可以了，一个请求对应一个响应。

以编号为 37709 号帧为例，我们可以读到，包的序号、时间、来源 IP 地址、目标 IP 地址、协议、长度和详细信息，详细信息里面写了：

| 片段                      | 含义                                                                                                                                             |
| ----------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| **Echo (ping) request** | 这是一个 **ICMP Echo 请求**，也就是你电脑发出去的 `ping` 探测包。`Echo` 是 ICMP 的术语，表示"回声请求"。                                                                        |
| **id=0x70f2**           | **标识符（Identifier）**。同一个 `ping` 进程发出的所有包，这个 ID 都相同。如果你同时开两个窗口 `ping`，它们的 ID 会不一样，操作系统靠这个区分是哪个进程的包。`0x70f2` 是十六进制表示。                             |
| **seq=0/0**             | **序列号（Sequence Number）**。`0` 是原始值，`/0` 是 WireShark 换算后的"相对序列号"。表示这是这个进程发出的 **第 1 个** ping 包（从 0 开始计数）。下一个包你会看到 `seq=1/256`，再下一个是 `seq=2/512`…… |
| **ttl=64**              | **生存时间（Time To Live）**。这个包从发出时 TTL 为 64（Linux/macOS 默认值），每经过一个路由器减 1。如果减到 0 还没到达，就会被丢弃，防止包在网络里无限循环。                                            |
| **(reply in 37710)**    | WireShark 的**关联提示**。意思是"这个请求对应的回复在 **第 37710 号帧**"。你往下看，37710 号就是 `Echo (ping) reply`，而且它的 Info 里会写 `(request in 37709)`，两者互相指认。               |

接下来我们可以阅读详细的信息：

![[Pasted image 20260518213428.png]]

红色箭头 1 指向的两个箭头指的就是请求与回复的意思，红色箭头 2 指向的就是以太网帧的具体数据了，我们将会重点研究这里的数据，左边是数据解析后的数据的文字描述，右边则是真正的二进制数据，并且用十六进制表示。

Frame 是 WireShark自己加的“元数据层”，里面有非常详细的信息：

- **Arrival Time**：精确到微秒的到达时间
- **Epoch Time**：Unix 时间戳
- **Time delta from previous captured frame**：和上一个帧的时间间隔
- **Frame Number**：37709
- **Frame Length**：98 bytes
- **Capture Length**：98 bytes
- **Interface id**：从哪块网卡抓的
- **Encapsulation type**：`Ethernet`（表示下面封装的是以太网）
- **Protocols in frame**：`eth:ethertype:ip:icmp`（这一帧里包含的协议栈）

![[Pasted image 20260518215619.png]]

我们直接来看 Ethernet II，也就是 Ethernet V2 协议的头部信息：

![[Pasted image 20260518220645.png]]

我们可以看到目标 MAC 地址，我们的网关应该是锐捷的：

![[Pasted image 20260518220716.png]]

也可以看到源 MAC 地址：

![[Pasted image 20260518220905.png]]

还能看到协议类型，这里是 IPv4：

![[Pasted image 20260518220944.png]]

没有 FCS 是因为 FCS 如果校验通过，网卡会抛弃 FCS，如果校验没通过，网卡直接将整个以太网帧抛弃掉。

接下来是网络层的头部，后面再详细讲：

![[Pasted image 20260518221158.png]]

最后就是数据部分：

![[Pasted image 20260518221446.png]]

数据就是这样一层一层被封装然后发送出去的。