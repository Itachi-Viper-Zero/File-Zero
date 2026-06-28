主机发现
```bash
sudo nmap -sn -e eth0 192.156.169.0/24

使用nmap进行主机发现
-e 指定网卡（因为我安装了两个网卡 eth0:NAT模式 ; eth1:Host-Only模式)
```
![](assets/DarkHole-1/file-20260508151054367.png)
发现目标IP：192.156.169.134

进行端口发现
```bash
sudo nmap -e eth0 --min-rate 10000 -p- IP -oA EPT/scans/port
```
![](assets/DarkHole-1/file-20260508151054371.png)
如果发现端口较多可以进行多次扫描进行对比（降低速率为1000或更低）
最好是每次都进行至少三次的扫描，以免出现网络问题导致漏扫的情况

进行TCP与基本漏洞、服务、版本号扫描
```bash
sudo nmap -e eth0 -sT -sC -sV -O -p22,80 IP -oA EPT/scans/detail
```
![](assets/DarkHole-1/file-20260508151054388.png)

这里出现一个警告：
`Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port`
**OSScan结果可能不可靠，因为nmap没法找到至少一个开放、一个关闭的端口**
这其实是Nmap的判断机制，它通过对比开放与关闭端口返回的内容进行更加细致的判断
解决方案就是添加一个关闭的端口即可

```bash
sudo nmap -e eth0 -sT -sC -sV -O -p22,80,99(未开放端口) IP -oA EPT/scans/detail
```
![](assets/DarkHole-1/file-20260508151054390.png)


**详细内容分析**：
- **MAC Address: 00:0C:29:50:9C:07** (VMware) -> 虚拟机
- **Host ip up** -> 延迟低，说明与扫描机在同一网段
- **NetWork Distance**：网络距离 1hop(一跳) -> 同网段，中间无路由器
- **端口基本信息**：
	- 22 - 开放 - SSH - OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux)
		- ssh-hostkey -> SSH主机指纹
	- 80 - 开放 - HTTP - Apache httpd 2.4.41 (Ubuntu)
		- **http-title** : DarkHole (Web标题)
		- http-header：Apache/2.4.41 (Ubuntu)
		- http-cookie-flags：Cookie标志 -> `PHPSESSID` 的 `httponly` 标志未设置 → 这是一个低危但值得注意的配置缺陷，可能导致会话 Cookie 被客户端脚本读取（XSS 时可被窃取）
	- 99 -关闭
- **Running - OS CPE - OS details ->操作系统推测**：

分析完后的测试权重为：Http（80）-> SSH（22）

在进行UDP扫描（将信息收集做全，即使可能没有作用）
```bash
sudo nmap -e eth0 -sU --top-ports 100（常用的100个） IP EPT/scans/udp
```
![](assets/DarkHole-1/file-20260508151054370.png)
发现这里只有一个 68/udp 端口 服务是：DHCP（用于分配IP地址的）
几乎没有可利用处

进行Nmap脚本扫描
```bash
sudo nmap -e eth0 --script=vuln -p22,80 IP -oA EPT/scans/script
```
![](assets/DarkHole-1/file-20260508151054392.png)
**深度分析**：
- 22/tcp: 几乎没有其余可利用漏洞
- 80/tcp:
	- 没有找到XSS漏洞
	- Cookie标志 -> `PHPSESSID` 的 `httponly` 标志未设置 → 这是一个低危但值得注意的配置缺陷，可能导致会话 Cookie 被客户端脚本读取（XSS 时可被窃取）
	- 可能存在**CSRF漏洞**发现两个Path，未发现反 CSRF 的隐藏 token
	- 没有发现**CVE2017-1001000漏洞**
	- **目录/菜单发现**：/login.php、/config、/css、/js、/upload；**w/ listing**表示Apache 的目录索引（Indexes）是开启的，意味着你可以直接在浏览器里看到这些目录下的文件列表。
综合：优先进行网也访问，查看目录发现等内容是否有其他可收集信息

对发现的菜单进行查看
没有发现前端代码有什么隐藏（权重比放低，如实在找不到再在进行必要审计）
![](assets/DarkHole-1/file-20260508151054389.png)
在/css发现两张图 home.jpg -> 封面图  wp5805428.gif -> 动图（可能存在隐写等，这是CTF常用的图片格式）

![](assets/DarkHole-1/file-20260508151054394.png)
简单尝试登录（基本不会登录成功），这里尝试的结果也是一样。

可以尝试先注册在登录（其实这是很多渗透测试/src等的操作手段），只要不大面积泄露他人隐私信息，只对自己账户的一些测试完全是可以的（比如：信息回显啊等，因为这些都是你能直接看到的，你只是证明了他有漏洞并未通过非法渠道获取他人隐私信息---你不可能说明我自己的信息对于我自身而言是隐私吧，这也太荒唐了！）

![](assets/DarkHole-1/file-20260508151054397.png)
进行注册
![](assets/DarkHole-1/file-20260508151054395.png)
尝试登录

![](assets/DarkHole-1/file-20260508151054400.png)
发现登录进来了（如果发现一直在转圈，重新加载下就行）

![](assets/DarkHole-1/file-20260508151054403.png)
注意到网址的位置，内容可以看出是一个GET请求
那么可以尝试修改**id值**看是否存在越权漏洞
接下来就需要进行抓包了

![](assets/DarkHole-1/file-20260508151054401.png)
修改id值发包有发现返回的内容中提示*密码已经更新*

发现id=2没有用（可能就是干扰项而已），继续尝试id=1
![](assets/DarkHole-1/file-20260508151054406.png)
登录成功（Username几乎默认为: admin可以继续尝试，若不行，在进行详细的信息收集与分析)
这里出现了一个**Upload**功能，考虑会不会存在文件上传漏洞

接下来生产PHP一句话木马
可以使用Kali自带的反弹Shell
```bash
locate php-reverse-shell
```
![](assets/DarkHole-1/file-20260508151054399.png)
选择一个进行拷贝

![](assets/DarkHole-1/file-20260508151054411.png)
进行IP与Port修改，port选择443 -> 为了避免存在防火墙（即使存在放行443端口的概率也是较大的）

![](assets/DarkHole-1/file-20260508151054404.png)
这里出现上传文件类型，几乎只允许图片类型

不知道这是黑名单还是白名单
尝试修改一下php后缀进行测试

修改成mp4,发现可以上传 -> 证明该上传逻辑为黑名单
![](assets/DarkHole-1/file-20260508151054409.png)

熟悉PHP的人都知道，在旧版的的php后缀有很多，比如phtml、phar等 -> 基本都是历史遗留，新版本为例兼容旧版程序做出的妥协

使用`nc`进行443端口监听
点击运行php脚本
![](assets/DarkHole-1/file-20260508151054412.png)
反弹Shell成功，但是权限很不足

接下来就是进行bashshell却换(几乎大部分都会自带python)
```python
python -c 'import pty;pty spwan("/bin/bash")'
```
![](assets/DarkHole-1/file-20260508151054407.png)
如果`python`没有尝试`python3`，因为有些系统python默认是python2

**SUID**：
`SUID（Set User ID）是一种特殊权限位。当一个可执行文件被设置了 SUID 位时，**任何用户运行该文件时，都会以该文件**所有者**的身份执行**，而不是以当前用户的身份执行。`

接下来在shell中进行suid程序查找：
```bash
find / -perm -u=s -type f 2>/dev/null
-perm : 按权限进行查找
-u=s: u代表user，s代表suid
```

![](assets/DarkHole-1/file-20260508151054417.png)
发现一个有点特殊的执行文件（/home/john/toto）可以尝试进行运行
![](assets/DarkHole-1/file-20260508151054420.png)

接下来考虑进行劫持
![](assets/DarkHole-1/file-20260508151054415.png)
看到命令好提示符 -> 大致是成功了

![](assets/DarkHole-1/file-20260508151054423.png)
回到john目录进行文件查看获得第一个Flag：**DarkHole{You_Can_DO_It}**

![](assets/DarkHole-1/file-20260508151054419.png)
这里还有一个password的文件，里面是root123
尝试进行SSH登录

![](assets/DarkHole-1/file-20260508151054422.png)
对的这就是ssh的密码

![](assets/DarkHole-1/file-20260508151054425.png)
可以看出，john可以在不需要root权限时就可以执行一个叫`file.py`的文件

![](assets/DarkHole-1/file-20260508151054429.png)
文件是空的，那就自己写一个提权逻辑

![](assets/DarkHole-1/file-20260508151054428.png)
这是对简单的提权逻辑

执行`sudo /usr/bin/python3 /home/john/file.py`
![](assets/DarkHole-1/file-20260508151054426.png)

看见成功出现了root字符

![](assets/DarkHole-1/file-20260508151054435.png)
成功获得第二个Flag：**DarkHole{You_Are_Legend}

![](assets/DarkHole-1/file-20260508151054433.png)
成功拿下**DarkHole_1**靶机

**小结**：
基本流程：
信息收集/Nmap漏洞扫描 -> Web应用的交互进行测试交互/注册了用户 -> 根据URL的特征尝试了越权漏洞（具体是IDOR：水平越权漏洞；使用BP实现修改高权限用户的密码） -> 通过文件上传漏洞获得系统立足点（PHP的反弹Shell）-> 对目录进行详细的查询发现toto文件的特殊性（以john用户的身份运行类型id的命令）-> 进行劫持提权 -> 获取了关键文件的读取权（user.txt、password、file.py等）-> john用户可以在不需要root权限时就可以执行一个叫`file.py`的文件 -> 进行python脚本的提权书写 -> 运行，成功获取了Root权限
---> **强化点**：如何利用