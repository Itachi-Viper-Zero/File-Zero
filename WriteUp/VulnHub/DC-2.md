来吧！DC系列的第二台靶机！！！
```table-of-contents
```

# 主机发现
依旧最基础的主机发现
![](assets/DC-2/file-20260513091544262.png)
主机发现的方法很多，具体使用需进行权衡对比

![](assets/DC-2/file-20260513091740724.png)
当然如果有多张网卡也可进行指定网卡选择`-I eth0`

![](assets/DC-2/file-20260513092123756.png)

当然还有其他优质的工具比如：`masscan`、`netdiscover`等

# 端口扫描

![](assets/DC-2/file-20260513101806672.png)

![](assets/DC-2/file-20260513101627016.png)
其实很明显可以看出工具相对而言更加全面一点（但受限于环境与需求，只能做出必要的权衡才行）

# TCP/UDP协议扫描

![](assets/DC-2/file-20260513102130738.png)

![](assets/DC-2/file-20260513102356683.png)

UDP只有一个dhcp（这是自动分配IP地址的服务），可以忽略（基本不作为立足点）
TCP：
- 80端口：http-title显示 **未跟随重定向到 dc-2** --> 这里待分析验证
- 7744端口：ssh服务，一般不作为立足点


# 默认脚本扫描

![](assets/DC-2/file-20260513102340295.png)

结合TCP扫描分析：
枚举除了 wordpress 的users：admin、tom、jerry
目录枚举版本却本过多难以直接利用，优先级排后

# 渗透流程

扫描获得的信息不多，知道是 WordPress 搭建的（这是一个常见的 Blog 搭建的CMS）
所以应该能从网页内容获取到一些信息！！！

![](assets/DC-2/file-20260513103116878.png)

![](assets/DC-2/file-20260513103143719.png)

通过IP地址访问，发现被重定向了（刚好符合 TCP 扫描所获得的信息）
解决方案：**在Hosts中配置即可**

![](assets/DC-2/file-20260513103401850.png)
再次访问

![](assets/DC-2/file-20260513103423308.png)

这里就需要考虑重新进行一次扫描操作了
TCP：
![](assets/DC-2/file-20260513103615550.png)

UDP：因为之前探测的端口不多且不重要，所以可以先不写扫描，需要时再进行操作即可！！！

Vuln：
![](assets/DC-2/file-20260513103701766.png)

这么一看的确有很多信息在第一次扫描是遗漏了
分析：
- WordPress的确定版本为：4.7.10 （熟悉的可以进行操作，不熟悉的可以搜索或暂时作为备选点）
- 还有一个新的 CSRF 出现，不过这里不需要我们手动实现CSRF操作（不进行手动实现的原因：脚本已经帮我探测出了；CSRF漏洞本身算是属于低危险的，除非配合其他漏洞联合使用）--> 因此优先级靠后（可暂时作为信息收集的渠道）

接下来回到网站页面：
![](assets/DC-2/file-20260513104329364.png)
这里直接就有Flag 1了
```flag1
Your usual wordlists probably won’t work, so instead, maybe you just need to be cewl.

More passwords is always better, but sometimes you just can’t win them all.

Log in as one to see the next flag.

If you can’t find it, log in as another.
```
--->
你通常的单词列表可能不起作用，所以相反，也许你只需要变得精明。
更多的密码总是更好，但有时你不可能全部成功。
以一个身份登录以查看下一个标志。
如果你找不到它，用另一个身份登录。

这里的提示是需要进行登录，之前拿到了三个用户名：admin、tom、jerry
接下来应该就是字典了，但是通常的字典列表却不行，所以得考虑从该网站进行字典构建
当然人为的一个一个去构建自然最安全，但效率未必如意
在网络上搜索一些构建字典的工具：
![](assets/DC-2/file-20260513105105293.png)

![](assets/DC-2/file-20260513105114312.png)

通过AI搜索发现Flag1其实给出了提示：使用**cewl**

⚠️扩展思路一：其实也可以合理猜测的，提示中几乎每个单词都能在翻译软件上找到单独的意思，唯独**cewl**没有具体的意思，进行搜索就可以得出这是一个工具！！！
![](assets/DC-2/file-20260513105432286.png)

⚠️扩展思路二：像Kali这一个专业的系统必然自带了 通过网站内容生成自定义字典的工具（因为构建必要的针对性的字典其实是很重要的，为了提高效率必然会被写出自动化哪怕不是万能的，也会在某些时刻提高渗透效率）。稍微检索一下就可以得到。
![](assets/DC-2/file-20260513105906701.png)
这些都是通过浏览器检索到的较为全面的Kali自动的字典工具，其中也包含了**cewl**
它也给出了详细的使用方法

![](assets/DC-2/file-20260513110229110.png)
一次性就构建了1689个密码内容

手动构建一下用户字典（因为之前，探测出来了一些且数量不多，测试也花不了多长的时间，就先以这些进行尝试）

接下来就是使用爆破工具进行尝试吧！！！
这里使用`Hydra`，也可使用`wpsacn`等其他工具都可以
```bash
# hydra的使用方法
hydra -l [用户名] -P [密码字典] [目标IP] [协议] [URL路径] [参数]

hydra -L dc2_username.txt -P dc2_passwd.txt dc-2 http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:is incorrect" -o dc2_hydra.txt

# -L 自定用户字典
# -o 进行输出保存
```

解释一下具体指令构造过程（从协议开始）：
**http-post-from**：指定的是POST请求
![](assets/DC-2/file-20260513111053230.png)
**参数的log、pwd与is incorrect**的信息来源：
**log、pwd**是从前端表单**name**获取的
![](assets/DC-2/file-20260513111155328.png)
**is incorrect**是报错特征，写其他的也可以（最好是通用性高的字段）
![](assets/DC-2/file-20260513111224761.png)

![](assets/DC-2/file-20260513111703817.png)成功爆破出来密码！！！
![](assets/DC-2/file-20260513111815878.png)

进行登录后浏览内容，发现了flag2
```flag2
If you can't exploit WordPress and take a shortcut, there is another way.

Hope you found another entry point.
```
--->
如果您无法利用WordPress走捷径，还有另一种方法。
希望您找到了另一个入口点。

这里给的提示显示无法利用WordPress获取shell等就尝试另一种方法!
什么方法呢？之前信息收集时有一个ssh的端口开放（只是端口从 22->7744），是否可以利用
尝试登录：
![](assets/DC-2/file-20260513112046418.png)
尝试之后只有tom用户可以登录，jerry无法通过ssh登录

![](assets/DC-2/file-20260513112228458.png)
没法使用cat，显示 **-rbash**，最低权限！！！
看一下usr
![](assets/DC-2/file-20260513112318996.png)
好像只能使用这四个命令

这里选择使用vi查看flag3.txt（选择自己最熟悉最佳）
![](assets/DC-2/file-20260513112416612.png)
```flag3.txt
Poor old Tom is always running after Jerry. Perhaps he should sue for all the stress he causes.
```
--->
可怜的老汤姆总是追着杰瑞跑。也许他应该就他所造成的所有压力提起诉讼。

看来tom这个账号不行，是否是需要切换到jerry这个账号？？？
看一下是否存在jerry
![](assets/DC-2/file-20260513112554359.png)
有的，看来应该就是切换到jerry
但是能用的指令只有4个，其中没有cd
![](assets/DC-2/file-20260513112646623.png)
得想办法绕过 **-rbash**才行
只能从less、ls、scp、vi入手
搜索一下能否从这个四个或其他手段绕过 **-rbash**
参考文章：[RBash - 受限的Bash绕过-腾讯云开发者社区-腾讯云](https://cloud.tencent.com/developer/article/1680551)

看来方法很多啊！！！
里面刚好存在编辑器vi的实现，参考文章实现即可：
```vi
输入指定：
set shell=/bin/bash
shell
```

![](assets/DC-2/file-20260513113355963.png)
看来是可以的！！！

![](assets/DC-2/file-20260513113425845.png)
cat依旧不可用，但这次是 **bash**

这下思路就很简单了！
思路一：使用vi或less查看![](assets/DC-2/file-20260513113528831.png)

思路二：使用有bash权限的cat查看即可
![](assets/DC-2/file-20260513113551144.png)

```flag4.txt
Good to see that you've made it this far - but you're not home yet. 

You still need to get the final flag (the only flag that really counts!!!).  

No hints here - you're on your own now.  :-)

Go on - git outta here!!!!
```
--->
很高兴看到你已经走了这么远——但你还没有到家。
你仍然需要获得最终的标志（唯一真正重要的标志！！！）。
这里没有提示——你现在只能靠自己了。 :-)
继续——快离开这里！！！！

这里竟然不是最终目的地
看来得继续深入，获取root权限才行了！！！
但不知道从哪里入手
先进行信息收集吧！！！
![](assets/DC-2/file-20260513113946393.png)
看来权限还是有很大限制

⚠️发现重大问题：目前的操作的用户依旧还是tom
得想办法切换到jerry尝试
看一下能功过 `/bin` 执行那些内容吧！！！
![](assets/DC-2/file-20260513115239496.png)
可以使用`su`进行切换用户，尝试一下！！！

![](assets/DC-2/file-20260513115416479.png)
可以的，接下来应该就是提权！！！

![](assets/DC-2/file-20260513115509919.png)

先看下`.bash_history`，看作者运行了那些指令
![](assets/DC-2/file-20260513115617965.png)
结合`sudo -l`，看来重点得先放在`git`上，

搜索一下git如何提权利用
参考网站：[https://gtfobins.org/](https://gtfobins.org/gtfobins/git/#shell)
![](assets/DC-2/file-20260513115941396.png)
应该是运用`Inherit`
![](assets/DC-2/file-20260513120115023.png)
成功了！！！（是建立在多次尝试的基础上）
需要添加sudo以root权限运行才可提权到root！！！

![](assets/DC-2/file-20260513120228025.png)


**小结**：从开始的立足点探寻时就需要有一定的基础与‘敏感度’（**一**：英语基础在渗透测试上的重要性，渗透测试时会有很多这些专有名词或俚语、工具等需要进行一定的积累；**二**：在没法直接找到立足点时就需要想到该如何破解，这里只探测出了*http*服务、*ssh*服务，所以可能想到的就是爆破与专有字典的构建） ---> 但如果探测的端口与服务较多是否能想到进行结合，探测出了几个基本的用户名接下来应该就是弱口令尝试（若无其他明显暴露的问题，就只有尝试进行爆破）
其次在进行逃逸时的理解，这里就出现了`rbash`的逃逸知识，也要懂得利用浏览器搜索必要的利用方案与突破点（当然这里面的信息收集也是必不可少的，从直接想要读取flag的命令显示除了有所限制，得去探测一下权限边界开放了那些可用的指令等再考虑结合运用）
--->**重点需要提升的点**：英语基础，对一些常见的专有名词要敏感；信息收集能力，是贯穿始终的步骤必须强化；权限绕过与提权手段，结合收集的信息权衡最终逃逸、提权的手段