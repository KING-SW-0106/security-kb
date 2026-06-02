# 02 · Python——安全工具速通

> **⏪ 前置知识：** 无（C语言有帮助但非必须）
> **⏩ 学完后解锁：** 网络协议（04）、Web安全（06）、渗透实战（08）
> **⏱ 预计用时：** 2-3小时
> **🎯 核心目标：** 能用 Python 写简单的网络通信脚本、会调 requests 库、会用 pwntools 连远程程序

---

## 📊 本章进度

| 知识点 | 状态 |
|--------|:--:|
| 1. Python速通——和C语言对比 | ⬜ |
| 2. 字符串与字节——安全人的大坑 | ⬜ |
| 3. 📌 socket编程——手写网络工具 | ⬜ |
| 4. 📌 pwntools入门——二进制攻击利器 | ⬜ |
| 5. requests库——写爬虫和扫描器 | ⬜ |

**掌握自检：** `[ ]` 能用 socket 发一次 HTTP 请求获取百度首页

---

## 1. Python速通——对照C语言

### 🏢 生活类比

```
C语言 = 手动挡汽车
  → 你要管离合器、换挡、转速
  → 控制精细，但容易熄火（段错误）

Python = 自动挡汽车
  → 你只需要踩油门和刹车
  → 开起来快，但不知道引擎盖下面在干嘛
```

### 💻 C vs Python 核心差异

```python
# ===== 变量：不用声明类型 =====
# C语言:  int a = 10;
a = 10           # Python 自动判断这是整数
b = "hello"      # 也能自动判断是字符串

# ===== 数组/列表 =====
# C语言:  int arr[5] = {1, 2, 3, 4, 5};
arr = [1, 2, 3, 4, 5]       # 列表，想放多少放多少
arr.append(6)                # 动态追加

# ===== 条件 =====
# C语言:  if (x > 0) { ... }
if x > 0:                    # 没有括号，靠缩进判断层级！
    print("正数")
elif x == 0:
    print("零")
else:
    print("负数")

# ===== 循环 =====
for i in range(5):           # 0,1,2,3,4
    print(i)

for item in arr:             # 直接遍历元素
    print(item)
```

### 🐍 Python 为什么会自动判断类型？

```python
# Python 里一切都是对象
a = 10
print(type(a))    # <class 'int'>

a = "hello"
print(type(a))    # <class 'str'> —— 同一个变量名，类型变了

# 这意味着你不用像 C 那样 int a; char b; float c; 一个个声明
```

---

## 2. 字符串与字节——安全人的大坑

**这是安全领域最容易踩的坑。**

### 字符串 vs 字节

```python
s = "hello"            # 这是字符串（str类型）
b = b"hello"           # 这是字节（bytes类型）

# 它们不一样！
print(type(s))    # <class 'str'>
print(type(b))    # <class 'bytes'>

# 互相转换
s = b.decode()          # bytes → str
b = s.encode()          # str → bytes
```

### 🔥 为什么安全方向要特别小心？

```python
# 从网络收到的数据是 bytes
data = sock.recv(1024)   # 收到的是 bytes 类型

# 构造攻击 payload 时经常要混合不同类型
shellcode = b"\x31\xc0\x50\x68\x2f\x2f\x73\x68"  # shellcode 是 bytes
addr = 0x7fffffffde00                                # 地址是整数

# 要把地址转成 bytes 才能拼进 payload
payload = b"A" * 16 + addr.to_bytes(8, 'little')   # 正确做法
```

### 速查表

| 场景 | 代码 |
|------|------|
| 字符串→字节 | `"abc".encode()` |
| 字节→字符串 | `b"abc".decode()` |
| 整数→字节(小端) | `(0x7fff).to_bytes(8, 'little')` |
| 整数→字节(大端) | `(0x7fff).to_bytes(8, 'big')` |
| 字节→整数 | `int.from_bytes(b"\xff\x00", 'little')` |
| 十六进制字符串→字节 | `bytes.fromhex("deadbeef")` |
| 字节→十六进制字符串 | `b"\xde\xad\xbe\xef".hex()` |

---

## 3. 📌 socket编程——手写网络工具

### 🏢 生活类比

```
Socket = 电话
  → 你要给某人打电话：
    1. 先拨号码（IP地址 + 端口号）
    2. 对方接听（connect）
    3. 开始说话（send）
    4. 听对方说话（recv）
    5. 挂断（close）
```

### 💻 最简单的 TCP 客户端

```python
import socket

# 1. 买个电话机
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# 2. 拨号（连接百度的 80 端口）
sock.connect(("www.baidu.com", 80))

# 3. 说话（发送 HTTP 请求）
request = b"GET / HTTP/1.1\r\nHost: www.baidu.com\r\n\r\n"
sock.send(request)

# 4. 听回答（接收响应）
response = sock.recv(4096)
print(response.decode())

# 5. 挂断
sock.close()
```

### 跑一遍试试

把上面代码保存为 `test.py`，运行：

```bash
python test.py
# 你会看到百度的首页 HTML 被打印出来
```

**这意味着你已经用底层协议和互联网上的一台真实服务器完成了一次对话。** 渗透测试的很多工具本质就是这样的 socket 通信。

### 写一个最简单的端口扫描器

```python
import socket

def scan_port(host, port):
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.settimeout(0.5)  # 超时0.5秒
    result = sock.connect_ex((host, port))
    sock.close()
    if result == 0:
        print(f"[+] {host}:{port} 开放")

# 扫描本机常用端口
for port in [22, 80, 443, 3306, 8080]:
    scan_port("127.0.0.1", port)
```

---

## 4. 📌 pwntools——二进制攻击利器

**pwntools 是 CTF Pwn 题的标准工具库。** 它封装了 socket 连接、payload 构造、shellcode 生成等功能。

### 安装

```bash
pip install pwntools
```

### 核心用法

```python
from pwn import *

# ===== 连接目标 =====
# 本地程序
p = process("./vuln")

# 远程服务
p = remote("192.168.1.100", 9999)

# ===== 收发数据 =====
p.send(b"hello")               # 发送
p.sendline(b"hello")           # 发送 + 换行符
data = p.recv(1024)            # 接收最多1024字节
data = p.recvline()            # 接收一行
data = p.recvuntil(b"prompt>") # 接收直到出现 "prompt>"

# ===== 构造 payload =====
payload = b"A" * 16            # 16个A填充缓冲区
payload += p64(0x400123)       # 覆盖返回地址为 0x400123（p64=转小端序8字节）

# ===== 发送 payload =====
p.sendline(payload)

# ===== 获得交互式 shell =====
p.interactive()                # 手动交互模式

# ===== 生成 shellcode =====
shellcode = asm(shellcraft.sh())  # 生成执行 /bin/sh 的 shellcode

# ===== 查地址 =====
elf = ELF("./vuln")
print(hex(elf.symbols['win']))    # 打印 win 函数的地址
print(hex(elf.got['puts']))       # 打印 puts 的 GOT 表地址
```

### 💻 一个完整的 pwntools 攻击脚本

```python
from pwn import *

# 情景: 一个程序有栈溢出漏洞，我们要跳到 win 函数（0x400567）

p = process("./vuln")

# 程序先打印一句话，我们接收它
p.recvuntil(b"Enter your name: ")

# 构造 payload
payload = b"A" * 40          # 填充 40 字节到返回地址
payload += p64(0x400567)     # 覆盖返回地址为 win 函数

# 发送
p.sendline(payload)

# 拿到 shell！
p.interactive()
```

---

## 5. requests库——写爬虫和扫描器

**requests 是 Python 最流行的 HTTP 库，Web 安全方向必须熟练。**

### 安装

```bash
pip install requests
```

### 核心用法

```python
import requests

# GET 请求
r = requests.get("http://example.com")
print(r.status_code)    # 200
print(r.text)           # 响应体（字符串）
print(r.headers)        # 响应头

# POST 请求
r = requests.post("http://example.com/login",
                   data={"username": "admin", "password": "123456"})

# 带自定义请求头
r = requests.get("http://example.com",
                  headers={"User-Agent": "Mozilla/5.0"})

# 带 Cookie
r = requests.get("http://example.com",
                  cookies={"session": "abc123"})

# 设置代理（配合 BurpSuite）
r = requests.get("http://example.com",
                  proxies={"http": "http://127.0.0.1:8080"})
```

### 🔥 安全场景：写一个 SQL 注入检测脚本

```python
import requests

url = "http://target.com/product.php?id="

# 注入测试 payload
payloads = [
    "'",           # 单引号，看是否报错
    "' OR '1'='1", # 经典永真条件
    "' OR 1=1 --", # 注释掉后面
    "' AND 1=2 --", # 永假条件（对比永真看差异）
]

for payload in payloads:
    r = requests.get(url + payload)
    if "error" in r.text.lower() or "mysql" in r.text.lower():
        print(f"[!] 可能存在注入: {payload}")
    print(f"Payload: {payload} → 响应长度: {len(r.text)}")
```

---

## 🎬 推荐视频

| 视频 | 平台 | 说明 |
|------|------|------|
| [Python零基础入门2024](https://www.bilibili.com/video/BV1wF411F7xX/) | B站 | 搜索"Python 零基础 2024"，选播放量最高的 |
| [pwntools官方教程](https://docs.pwntools.com/) | 官网 | 英文文档，CTF必备 |
| [socket 编程入门](https://www.bilibili.com/) | B站 | 搜索"python socket 编程 入门" |
| [requests 库快速上手](https://www.bilibili.com/) | B站 | 搜索"python requests 库 教程" |

> 💡 **Pwn 方向必装：** `pip install pwntools` —— 然后跑一遍上面第4节的示例代码。

---

## 📝 复习自检

- [ ] 能用 Python 写一个 TCP 客户端，连上百度发 HTTP 请求
- [ ] 能区分 `str` 和 `bytes`，知道怎么互转
- [ ] 能用 pwntools 启动本地程序、发 payload、收数据
- [ ] 能用 requests 发 GET/POST 请求并查看响应

---

## 🔗 知识串联

```
socket 编程  → 《04-网络协议》你就能抓包看懂自己发的数据长什么样
pwntools    → 《07-二进制安全》写攻击脚本的核心工具
requests    → 《06-Web安全》自己写自动化扫描的基础
```

---

> 💬 **读完用一句话记下来：** "Python 在网安里就是你的瑞士军刀——快速验证想法、写自动化工具、调试漏洞。但你得先搞清楚 bytes 和 str 的区别。"

### 下一步：打开 `03-计算机体系-程序如何运行.md`
