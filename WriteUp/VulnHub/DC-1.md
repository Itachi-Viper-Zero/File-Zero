基本的主机发现，端口扫描，服务与漏洞探测

![](assets/DC-1/file-20260508151102239.png)
发现该网站存在CVE （CVE-2014-3704）,先不急着进行利用
先进行网站的功能测试与信息收集

![](assets/DC-1/file-20260508151102243.png)

![](assets/DC-1/file-20260508151102242.png)
这里就注意到了一个问题，注册用户后并没有让用户设置密码而在登录是却需要密码进行登录
很明显是有问题的，继续测试看是否存在切入点

多次尝试后发现难以找到有用的信息（可能深度挖掘后会有，但是可能会耗费大量的时间）
所以做渗透测试（*如果是时间充裕或有其他需求另当别论*）还是要积累必要的经验，做到快速的信息收集（可以适当配合工具进行探测，可大大降低时间成本）
>`死磕`固然能加固知识的稳定性同时扩充知识广度，但是人的精力是有限的不应是任何问题都是`死磕`，而是要有判断力的分辨该知识是否是必要的或收益很大的才去选择性的`死磕`

利用`searchsploit`对漏洞进行查找
![](assets/DC-1/file-20260508151102244.png)
大致可以分析出该网站存在的漏洞应该就是 **SQL注入**（但具体是哪一个不太确定，如果有时间或精力可以尝试一下注入）

接下来尝试借用工具`MSF`进行利用：
```bash
msfconsole -q
```
![697](assets/DC-1/file-20260508151102246.png)
发现这里的也有很多
但是结合`searchsploit`的搜索分析可以判断的是：可以利用的 Payload 应该就是 序号1 （exploit/unix/webapp/drupal_drupalgeddon2）

这就是结合各种工具的便利性（当然也可以使用msf查找CVE，但是各工具相互佐证才会大大减少失误概率）

![](assets/DC-1/file-20260508151102247.png)
虽然也能检索出，但是明显发现可利用点减少了（描述多相对而言详细点）
---> 这里两个都可以利用（可以多尝试一下，这里以第一个为例）

![](assets/DC-1/file-20260508151102218.png)
选择需要运行的 payload 即可

# **注意**⚠️：运行时可能会出现一个错误``

![](assets/DC-1/file-20260508151102217.png)
使用`show options`查看配置需求，按照要求进行填写修改即可

![](assets/DC-1/file-20260508151102241.png)

`run`直接运行即可！！！
![](assets/DC-1/file-20260508151102215.png)
Payload利用成功！！！

![](assets/DC-1/file-20260508151102221.png)
观察所有文件名发现需要读取的文件，尝试`cat`一下

![](assets/DC-1/file-20260508151102219.png)
报错：`[-] stdapi_fs_stat: Operation failed: 1`

**解决方案**：进入原生 **Shell**

![](assets/DC-1/file-20260508151102212.png)
这样就可以尝试直接进行查看文件内容了

`Every good CMS needs a config file - and so do you.`
---> 每一个好的CMS需要一个配饰文件 - 你也应该这样做

看来这是一个引路灯啊！！！
接下来就应该去找到配置文件进行操作了！
但是 **Drupal** CMS 并不熟悉 ---> 无所谓我们熟不熟悉，有人熟悉、有人了解即可
浏览器检索：
![](assets/DC-1/file-20260508151102213.png)
有了！！！

![](assets/DC-1/file-20260508151102210.png)
**小记**：这里进行了反弹Shell（更加美观），查看文件基本信息（防止文件过大盲目打开直接卡死）
⚠️这里观察一下目录结构，除了配置文件所在的目录中还有一个`scripts`目录，可以先查看一下。当然其实也可以猜到，里面可能有一些网站实现的脚本，比如：密码加密，邮箱校验等

![](assets/DC-1/file-20260508151102207.png)
看来作者还是放水了，直接写到开头了！！！
```
 * Brute force and dictionary attacks aren't the
 * only ways to gain access (and you WILL need access).
 * What can you do with these credentials?
```
---> 暴力破解和字典攻击不是获取权限的唯一方法（你需要获得访问权限）。凭借这些证书你能获得吗？

看来是作者给出的提示，同时也像是一个战书！！！
Mysql配置信息：
```
'username' => 'dbuser',
'password' => 'R0ck3t',
```

`drupal_hash_salt = 'X8gdX7OdYRiBnlHoj0ukhtZ7eO4EDrvMkhN21SWZocs'`
还有 Hash 加盐（看来不能进行密码的暴力破解了！！！） ---> 这里其实就是信息收集的一部分（方便后续利用时整理思路）

![](assets/DC-1/file-20260508151102205.png)
接下来就是Mysql的操作即可

![](assets/DC-1/file-20260508151102209.png)
找到重点（在里面发现了我注册的用户信息）

虽然找到了数据库的内容，但是不知道明文也不能使用啊！！！
由于前面发现加盐了 ---> 所以思路就不再是分析密文特征再进行破解了

**我们直接能够操作数据库为何不找到加密逻辑的文件内容对指定明文密码加密后进行更新呢！！！**

这里就体现了之前的信息收集重要性了，省去了研究密码的时间

之前浏览目录结构时发现的`scripts`目录就起作用了
![](assets/DC-1/file-20260508151102203.png)
这里的bash文件是否就是加密逻辑呢？

```bash
#!/usr/bin/php
<?php

/**
 * Drupal hash script - to generate a hash from a plaintext password
 *
 * Check for your PHP interpreter - on Windows you'll probably have to
 * replace line 1 with
 *   #!c:/program files/php/php.exe
 *
 * @param password1 [password2 [password3 ...]]
 *  Plain-text passwords in quotes (or with spaces backslash escaped).
 */

if (version_compare(PHP_VERSION, "5.2.0", "<")) {
  $version  = PHP_VERSION;
  echo <<<EOF

ERROR: This script requires at least PHP version 5.2.0. You invoked it with
       PHP version {$version}.
\n
EOF;
  exit;
}

$script = basename(array_shift($_SERVER['argv']));

if (in_array('--help', $_SERVER['argv']) || empty($_SERVER['argv'])) {
  echo <<<EOF

Generate Drupal password hashes from the shell.

Usage:        {$script} [OPTIONS] "<plan-text password>"
Example:      {$script} "mynewpassword"

All arguments are long options.

  --help      Print this page.

  --root <path>

              Set the working directory for the script to the specified path.
              To execute this script this has to be the root directory of your
              Drupal installation, e.g. /home/www/foo/drupal (assuming Drupal
              running on Unix). Use surrounding quotation marks on Windows.

  "<password1>" ["<password2>" ["<password3>" ...]]

              One or more plan-text passwords enclosed by double quotes. The
              output hash may be manually entered into the {users}.pass field to
              change a password via SQL to a known value.

To run this script without the --root argument invoke it from the root directory
of your Drupal installation as

  ./scripts/{$script}
\n
EOF;
  exit;
}

$passwords = array();

// Parse invocation arguments.
while ($param = array_shift($_SERVER['argv'])) {
  switch ($param) {
    case '--root':
      // Change the working directory.
      $path = array_shift($_SERVER['argv']);
      if (is_dir($path)) {
        chdir($path);
      }
      break;
    default:
      // Add a password to the list to be processed.
      $passwords[] = $param;
      break;
  }
}

define('DRUPAL_ROOT', getcwd());

include_once DRUPAL_ROOT . '/includes/password.inc';
include_once DRUPAL_ROOT . '/includes/bootstrap.inc';

foreach ($passwords as $password) {
  print("\npassword: $password \t\thash: ". user_hash_password($password) ."\n");
}
print("\n");

```
还是一个php语言写的
分析一下：
- 改脚本运行前提：加载核心文件`/includes/password.inc`、`/includes/bootstrap.inc`
- 使用方法很简单：./scripts/password-hash.sh "password" ---> `print("\npassword: $password \t\thash: ". user_hash_password($password) ."\n");`还很良心的输出来了！！！（如果没有输出则需要考验我们的综合能力了：**PHP调试**、**动态分析**、**HOOK**、**内存观察**、**逆向执行流**等，涉及这些高级技术）
不过还好，这里直接给出来了（否则难度将上升一个档次）

![](assets/DC-1/file-20260508151102202.png)
直接运行即可（有兴趣的话可以研究下`password.inc`、`bootstrap.inc`，学习一下，强化自身的代码审计能力。当然也可借助AI进行辅助分析）

**小记**：这里补充一点，如果目录结构文件很多，使用`find`进行查找
![](assets/DC-1/file-20260508151102197.png)

![](assets/DC-1/file-20260508151102199.png)
成功将admin的密码进行了更新替换

![](assets/DC-1/file-20260508151102195.png)
登录成功了！！！

直接页面没有发现有用的信息，继续信息收集
![](assets/DC-1/file-20260508151102201.png)
很容易就发现了可能的利用点

![](assets/DC-1/file-20260508151102194.png)

`Special PERMS will help FIND the passwd - but you'll need to -exec that command to work out how to get what's in the shadow.`
特殊的 PERMS 将帮助 FIND 密码，但你需要 -exec 那个命令来弄清楚如何获取 shadow 中的内容。

有点不清楚这是什么意思？是让我们利用其他的点进行权限获取？

![](assets/DC-1/file-20260508151102192.png)
它的意思是使用 `find` 命令，进行**特殊权限**匹配查询

`find / -perm -u=s -type -f 2>/dev/null`:
**/ 全目录 ；-perm 按权限查询 ；-u=s 分组为 SUID ；-type 类型为文件 ；2>/dev/null 丢掉权限不匹配的错误提示**
![](assets/DC-1/file-20260508151102191.png)
这里面竟然有一个`find`可以利用！！！

浏览器检索一下
![](assets/DC-1/file-20260508151102189.png)

按照检索内容进行尝试：
![](assets/DC-1/file-20260508151102179.png)
可以的成功读取到了 `shadow` 的内容
发现都是加密的玩个鸡毛啊！！！

>[!important]
>**Linux/Unix 系统中`/etc/shadow`文件使用的 SHA-512 crypt 格式密码哈希**，是用于存储用户密码的单向加密结果

⚠️注意：虽然是单向且加盐，但如果过于简单说不定可以破解成功 --> 针对弱口令的离线爆破难度极低、耗时极短。

尝试一下：
`john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt`
![](assets/DC-1/file-20260508151102175.png)
除了`john`，`Hashcat`也可以

这里获取了用户 `flag4` 的密码（主要是简单密码） --> 可以暂停破解了（字典很大，耗时很久，还有一个root密码猜测应该难以破解）

![](assets/DC-1/file-20260508151102188.png)
哟！看来作者还是依旧放水了！！！

![](assets/DC-1/file-20260508151102186.png)

Can you use this same method to find or access the flag in root?
Probably. But perhaps it's not that easy. Or maybe it is?

你能使用同样的方法在root中找到或访问标志吗？
可能吧。但也许没那么容易。 或者也许很容易？

![](assets/DC-1/file-20260508151102174.png)
看来直接访问**root**是不行的！！！

查看了一遍貌似没有其他可收集的有用内容了！！！
接下来有两个思路
- 继续破解root密码（很大概率失败，且十分耗时）
- 返回到先前利用`find`（优先级高点）

重新回到提示：
`Special PERMS will help FIND the passwd - but you'll need to -exec that command to work out how to get what's in the shadow.`

发现提示的构造貌似刚好符合利用 `find` 的语句
`find /etc/shadow -exec cat {} \;`
![](assets/DC-1/file-20260508151102173.png)
既然它可以读取并输出 `flag4` 用户没法访问的目录文件
是否说明可以利用它进行 **提权** 呢！！！（刚好 find 拥有 **root 的 SUID 权限**！）

那么根据之前的搜索是否可以构建出一个语句
`find <path> -exec /bin/sh \;`
---> path：给定一个路径（这是语法规定）；`/bin/sh`因为`find`是以root权限运行的，所以启动的sh则是**root shell**

![](assets/DC-1/file-20260508151102171.png)
哦！！！成功了！！！

![](assets/DC-1/file-20260508151102169.png)

可以的拿到最终的flag！！！


>[!note]
>该靶机的难点就是通过**可执行文件**，绕过配置错误的系统中的本地安全限制
>而这样的**可执行文件**很多，因此有师傅将主流的、常见的列表都收集整合起来了
>参考网站：[GTFOBins](https://gtfobins.org/#) 



**小结**：该靶机的难度不大，如果作者将其中的提示删除获取难度会上一个档次。但这也是作者提供的宝贵的渗透思路。不仅要考虑SQL注入这一类的程序编写过程参数的漏洞，还需考虑**安全配置失误**的漏洞（即**OWASP TOP 10**的经典利用）
其实回头想一想，在最后的拿到 **shadow** 进行 root 哈希 爆破时就陷入了一个思维误区
早在`flag2`时就已经提醒过：
 * Brute force and dictionary attacks aren't the
 * only ways to gain access (and you WILL need access).
 * What can you do with these credentials?

>**启发：**
>当思路可能有瑕疵或错误时不妨停下来细细思考一下，合理的停止并不是半途而废的终言，而是通向成功的必经之路！！！有的时候回头看一看或与会有更加广阔的天地！