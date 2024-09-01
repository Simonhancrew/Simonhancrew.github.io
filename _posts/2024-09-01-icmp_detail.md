---
title: icmp详解
date: 2024-09-01 11:11:00 +0800
categories: [Blogging, icmp, network]
tags: [writing]
---

Ping 是一个常用的网络诊断工具，其工作原理基于 ICMP（Internet Control Message Protocol，互联网控制消息协议）

1. 发送 ICMP Echo Request：
   - ping 命令会向目标主机发送 ICMP Echo Request（类型 8）数据包。
   - 这个数据包包含一个标识符、序列号和可选的数据负载。

2. 等待 ICMP Echo Reply：
   - 如果目标主机正常运行且网络连接正常，它会回复一个 ICMP Echo Reply（类型 0）数据包。
   - 回复包含与请求包相同的标识符和序列号。

3. 计算往返时间（RTT）：
   - ping 程序记录发送请求和接收回复的时间差，这就是往返时间（Round-Trip Time, RTT）。

4. 重复过程：
   - 通常，ping 会发送多个请求（默认情况下通常是 4 个），以获得更可靠的结果。

5. 统计和报告：
   - ping 最后会显示统计信息，如平均 RTT、最小和最大 RTT、丢包率等。

主要用途：

1. 检测网络连通性：确认目标主机是否可达。
2. 测量网络延迟：通过 RTT 了解网络响应速度。
3. 诊断网络问题：如丢包、路由问题等。

### ICMP 报文格式

ICMP头部通常是8字节长，紧跟在IP头部之后。

```ascii
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Type      |     Code      |          Checksum             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Identifier          |        Sequence Number        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

具体到每个字段：

1. Type (8 bits-uint8_t):
   - 表示ICMP消息的类型。
   - 类型 0：回显应答（Echo Reply）
   - 类型 3：目标不可达（Destination Unreachable）
   - 类型 5：重定向（Redirect）
   - 类型 8：回显请求（Echo Request）
   - 类型 11：超时（Time Exceeded）

2. Code (8 bits):
   - 进一步细分ICMP消息的类型。
   - 对于Echo请求和回复，这个值通常为0。

3. Checksum (16 bits):
   - 用于检测ICMP报文在传输过程中是否发生错误。
   - 计算方法包括ICMP头和数据。

4. Identifier (16 bits):
   - 用于匹配请求和回复。
   - 通常是发送进程的ID。

5. Sequence Number (16 bits):
   - 用于匹配请求和回复。
   - 每发送一个新的Echo请求，这个值就会增加。

因此实际上可以简单的定义一个结构体来表示ICMP头部：

```c
struct icmp_header {
    uint8_t type;
    uint8_t code;
    uint16_t checksum;
    uint16_t identifier;
    uint16_t sequence_number;
};
```

在Python中，我们可以使用struct模块来打包和解包这些字段：

```python
import struct

# 打包ICMP头
icmp_header = struct.pack('!BBHHH', 
    8,    # Type (8 for Echo Request)
    0,    # Code
    0,    # Checksum (初始值为0)
    1234, # Identifier
    1     # Sequence Number
)
```

也能注意到，ping的时候其实没有指定端口号，这是因为ICMP是一个基于IP层的协议，不需要端口号。

数据包按照普通的 IP 路由规则传输到目标主机，路由器根据 IP 地址进行转发，不关心 ICMP 的内容。

当目标主机收到 IP 数据包时，它会检查 IP 头部的协议字段，看到协议字段为 1，操作系统知道这是 ICMP 数据包。然后，数据包会被传递给操作系统的 ICMP 处理模块。

因此，配合socket_raw, 其实可以实现一个简单的ping client

```python
import os
import struct
import time
import select
import socket

ICMP_ECHO_REQUEST = 8

def checksum(packet):
    if len(packet) % 2 != 0:
        packet += b'\0'
    res = sum(struct.unpack("!H", packet[i:i+2])[0] for i in range(0, len(packet), 2))
    res = (res >> 16) + (res & 0xffff)
    res += res >> 16
    return (~res) & 0xffff

def create_packet(id, seq = 1):
    """Create a new echo request packet based on the given id."""
    header = struct.pack('!BBHHH', ICMP_ECHO_REQUEST, 0, 0, id, seq)
    data = b'Q' * 192
    my_checksum = checksum(header + data)
    header = struct.pack('!BBHHH', ICMP_ECHO_REQUEST, 0, my_checksum, id, seq)
    return header + data

def ping(host):
    """
    Returns either the delay (in seconds) or None on timeout.
    """
    dest_addr = socket.gethostbyname(host)
    my_socket = socket.socket(socket.AF_INET, socket.SOCK_RAW, socket.IPPROTO_ICMP)
    # my_socket.setsockopt(socket.SOL_SOCKET, socket.SO_RCVBUF, 65536)
    my_id = os.getpid() & 0xFFFF

    packet = create_packet(my_id)
    my_socket.sendto(packet, (dest_addr, 1))  # Port number is irrelevant for ICMP
    
    start_time = time.time()

    while True:
        ready = select.select([my_socket], [], [], 4)  # Timeout of 4 seconds
        if not ready[0]:  # Timeout
            return None

        received_packet, addr = my_socket.recvfrom(1024)
        icmp_header = received_packet[20:28]
        type, code, checksum, packet_id, sequence = struct.unpack('BBHHH', icmp_header)

        if packet_id == my_id:
            return time.time() - start_time

    my_socket.close()

def main():
    host = '127.0.0.1' # input("Enter the host to ping: ")
    print(f"Pinging {host}")
    
    for i in range(4):  # Send 4 ping requests
        delay = ping(host)
        if delay is None:
            print("Request timed out.")
        else:
            delay *= 1000  # Convert to milliseconds
            print(f"Received reply in {delay:.2f}ms")
        time.sleep(1)

if __name__ == "__main__":
    main()
```

### python的struct

Python的`struct`模块用于在字节串和Python数据类型之间进行转换。它非常有用，特别是在处理二进制数据、网络协议或文件格式时。让我详细解释一下如何使用`struct`模块：

1. 基本用法：

```python
import struct

# 打包（将Python数据类型转换为字节串）
packed = struct.pack(format, v1, v2, ...)

# 解包（将字节串转换为Python数据类型）
unpacked = struct.unpack(format, buffer)
```

2. 格式字符串：

格式字符串指定了如何解释字节。一些常用的格式字符包括：

- `B`: 无符号字符（1字节）
- `H`: 无符号短整型（2字节）
- `I`: 无符号整型（4字节）
- `Q`: 无符号长长整型（8字节）
- `h`: 有符号短整型（2字节）
- `i`: 有符号整型（4字节）
- `q`: 有符号长长整型（8字节）
- `f`: 单精度浮点数（4字节）
- `d`: 双精度浮点数（8字节）
- `s`: 字符串（字节数组）

3. 字节顺序：

- `<`: 小端序
- `>`: 大端序
- `!`: 网络字节序（大端序）
- `=`: 本机字节序

4. 示例：

```python
import struct

# 打包
# 格式：!BBHIH 表示网络字节序，依次为：无符号字符、无符号字符、无符号短整型、无符号整型、无符号短整型
packed = struct.pack('!BBHIH', 8, 0, 1000, 12345, 1)
print(packed)  # b'\x08\x00\x03\xe8\x00\x0009\x00\x01'

# 解包
unpacked = struct.unpack('!BBHIH', packed)
print(unpacked)  # (8, 0, 1000, 12345, 1)

# 字符串处理
name = "Alice"
#  表示一个无符号整数(4字节)， 5s: 表示一个5字节的字符串(字节串)
packed_name = struct.pack('!I5s', len(name), name.encode('utf-8'))
print(packed_name)  # b'\x00\x00\x00\x05Alice'

length, name = struct.unpack('!I5s', packed_name)
print(length, name.decode('utf-8'))  # 5 Alice

# 浮点数
packed_float = struct.pack('!f', 3.14)
print(packed_float)  # b'@I\x0f\xd0'

unpacked_float = struct.unpack('!f', packed_float)
print(unpacked_float)  # (3.140000104904175,)
```

5. 在网络编程中的应用（以ICMP为例）：

```python
import struct

def create_icmp_header(id, seq):
    # 创建ICMP头（类型8表示Echo请求，代码0）
    header = struct.pack('!BBHHH', 8, 0, 0, id, seq)
    
    # 计算校验和
    checksum = calculate_checksum(header)
    
    # 重新打包，这次包含计算好的校验和
    header = struct.pack('!BBHHH', 8, 0, checksum, id, seq)
    return header

def calculate_checksum(data):
    # 校验和计算（简化版）
    s = 0
    for i in range(0, len(data), 2):
        w = (data[i] << 8) + (data[i+1] if i+1 < len(data) else 0)
        s = s + w
    s = (s >> 16) + (s & 0xffff)
    s = ~s & 0xffff
    return s

# 使用示例
icmp_header = create_icmp_header(1234, 1)
print(icmp_header)

# 解析ICMP头
type, code, checksum, id, seq = struct.unpack('!BBHHH', icmp_header)
print(f"Type: {type}, Code: {code}, Checksum: {checksum}, ID: {id}, Sequence: {seq}")
```

### ip协议

```ascii
/*
 IPv4 Header Format

  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |Version|  IHL  |Type of Service|          Total Length         |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |         Identification        |Flags|      Fragment Offset    |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |  Time to Live |    Protocol   |         Header Checksum       |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |                       Source IP Address                       |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |                    Destination IP Address                     |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |                    Options                    |    Padding    |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

 Version:        4 bits
 IHL:            4 bits (Internet Header Length)
 Type of Service:8 bits
 Total Length:   16 bits
 Identification: 16 bits
 Flags:          3 bits
 Fragment Offset:13 bits
 Time to Live:   8 bits
 Protocol:       8 bits
 Header Checksum:16 bits
 Source IP:      32 bits
 Destination IP: 32 bits
 Options:        variable length, optional
 Padding:        variable length, used to ensure 32-bit alignment
*/
```

IPv4头部的详细字段描述：

1. 版本（Version）: 4位
   - 表示IP协议的版本，IPv4为4

2. 首部长度（Internet Header Length, IHL）: 4位
   - 表示IP头部的长度，以32位字（4字节）为单位
   - 最小值为5（20字节），最大值为15（60字节）

3. 服务类型（Type of Service）: 8位
   - 用于指定服务质量

4. 总长度（Total Length）: 16位
   - 整个IP数据报的长度，包括头部和数据，以字节为单位
   - 最大值为65535字节

5. 标识（Identification）: 16位
   - 用于标识数据报片段，同一数据报的所有片段该值相同

6. 标志（Flags）: 3位
   - 包括保留位、不分片（DF）和更多片段（MF）

7. 片偏移（Fragment Offset）: 13位
   - 表示该片段在原始数据报中的位置，以8字节为单位

8. 生存时间（Time to Live, TTL）: 8位
   - 限制数据报在网络中的生存时间，每经过一个路由器就减1

9. 协议（Protocol）: 8位
   - 指示上层协议，如TCP（6）或UDP（17）

10. 首部校验和（Header Checksum）: 16位
    - 用于检验IP头部的完整性

11. 源IP地址（Source IP Address）: 32位

12. 目的IP地址（Destination IP Address）: 32位

需要注意的是，虽然IP头部的标准长度是20字节（如图所示），但它可以包含额外的选项字段，最长可达60字节。这就是为什么我们需要"首部长度"（IHL）字段来指明实际的头部长度。

这种结构化的格式使得IP协议能够高效地在网络中传输数据，同时提供了必要的信息用于路由、分片和重组等网络操作。

```ascii
 OSI Model                    TCP/IP Model
 +-----------------------+    +------------------+
 |    7. Application     |    |                  |
 |-----------------------|    |                  |
 |    6. Presentation    |    |    Application   |
 |-----------------------|    |                  |
 |     5. Session        |    |                  |
 |-----------------------|    +------------------+
 |    4. Transport       |    |    Transport     |
 |-----------------------|    +------------------+
 |     3. Network        |    |    Internet      |
 |-----------------------|    +------------------+
 |    2. Data Link       |    |                  |
 |-----------------------|    |  Network Access  |
 |     1. Physical       |    |                  |
 +-----------------------+    +------------------+

 OSI Model Layers:
 7. Application:   High-level APIs, resource sharing, remote file access
 6. Presentation:  Data translation, encryption, compression
 5. Session:       Interhost communication, managing sessions
 4. Transport:     End-to-end connections, reliability, flow control
 3. Network:       Path determination and logical addressing
 2. Data Link:     Physical addressing, error detection and correction
 1. Physical:      Media, signal and binary transmission

 TCP/IP Model Layers:
 4. Application:   Combines OSI layers 5-7, handles high-level protocols
 3. Transport:     Corresponds to OSI layer 4, uses TCP and UDP
 2. Internet:      Corresponds to OSI layer 3, uses IP
 1. Network Access:Combines OSI layers 1-2, handles physical and data link
````
