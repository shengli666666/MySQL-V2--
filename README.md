# 欢迎来到SQL小白的学习笔记（运维篇）
* 本次学习课程为B站黑马程序员系列课程之“MySQL数据库入门到精通，从mysql安装到mysql高级、mysql优化全囊括”，承接基础和进阶篇。运维篇动手环节较多，练习过程中也遇到过很多问题，
  有些环节记录的内容比较口语化和简略，笔记中如有错误，欢迎指出。
* [基础部分](https://github.com/shengli666666/MySQL-/tree/main#%E5%9F%BA%E7%A1%80%E9%83%A8%E5%88%86begining)
	* [1.SQL概述](https://github.com/shengli666666/MySQL-/tree/main#1sql%E6%A6%82%E8%BF%B0)
	* [2.SQL基础以及分类](https://github.com/shengli666666/MySQL-/tree/main#2sql%E5%9F%BA%E7%A1%80%E5%8F%8A%E5%88%86%E7%B1%BB)
	* [3.SQL基础---函数](https://github.com/shengli666666/MySQL-/tree/main#3sql%E5%9F%BA%E7%A1%80---%E5%87%BD%E6%95%B0)
	* [4.基础约束](https://github.com/shengli666666/MySQL-/tree/main#4%E5%9F%BA%E7%A1%80%E7%BA%A6%E6%9D%9F)
	* [5.多表查询](https://github.com/shengli666666/MySQL-/tree/main#5%E5%A4%9A%E8%A1%A8%E6%9F%A5%E8%AF%A2)
	* [6.事务](https://github.com/shengli666666/MySQL-/tree/main#6%E4%BA%8B%E5%8A%A1)

# MySQL运维篇beginning
# 1.第一部分 日志
## 1.1 错误日志
```shell
错误日志是My5QL中最重要的日志之一，它记录了当mysqld启动和停止时，以及服务器在运行过程中发生任何严重错误时的相关信息。当数据库出现任何故障导致无法正常使用时，建议首先查看此日志。
mysql -u root -p#进入MySQL
该日志是默认开启的，默认存放目录/var/log/，默认的日志文件名为 mysqld.log。查看日志位置:
show variables like '%log_error%';查询错误日志，找到存放错误日志的文件位置

---在终端中输入
tail -50 ./XX#显示错误日志最后50行
tail -f ./XX实时显示错误日志
vim /var/lib/mysql/auto.cnf#可找到MySQL服务的id（server_uuid）
```

## 1.2 二进制日志
```shell
二进制日志（BINLOG）记录了所有的DDL（数据定义语言)语句和DML（数据操纵语言）语句，但不包括数据查询（SELECT、SHOW)语句。
作用:0.灾难时的数据恢复;2.MySQL的主从复制。在MySQL8版本中，默认二进制日志是开启着的，涉及到的参数如下:
show variables like '%log bin%'
开启binlog：

show variables like '%log_bin%';#查看二进制文件，log_bin显示的状态为on，
-- 也会显示存放的地址;两个二进制文件，一个是索引文件，一个是日志文件。

可查看系统里二进制文件情况：cd ./XXX
然后输入： ll

show variables like '%binlog_format%';#查看二进制日志格式，默认日志格式row

从行格式修改为其他格式：
1.vim /etc/my.cnf
2.在文件中添加语句binlog_format= STATEMENT
3.systemctl restart mysqld
4.ll 即可看到新生成的binlog

二进制日志文件工具：mysqlbinglog [参数] logfilename

手动清除日志：
1.purge master logs to 'binlog.000002';删除XX之前的所有日志
2.reset master；清除所有日志
自动清除日志：
show variables like '%binlog_expire%';查看二进制文件的过期时间，默认30天
```

## 1.3 查询日志
```shell
查询日志中记录了客户端的所有操作语句，而二进制日志不包含查询数据的SQL语句。默认情况下，查询日志是未开启的。
如果需要开启查询日志，可以设置以下配置︰

查看：show variables like '%general%';
开启：vim /etc/my.cnf 添加两个语句：general_log=1
general_log_file=/var/lib/mysql/mysql_query.log
去文件夹下找到该文件，显示实时变化，即可观察到所有的sql操作

vim /etc/my.cnf文件需要添加内容
--datadir=/var/lib/mysql
--socket=/var/lib/mysql/mysql.sock

--log-error= /var/log/mysqld.log
--pid-file=/var/run/mysqld/mysqld.pid

--general_log=1
--general_log_file=/var/lib/mysql/mysql_query.log
```

## 1.4 慢查询日志
```shell
慢查询日志记录了所有执行时间超过参数long_query_time设置值并且扫描记录数不小于min_examined_row_limit的所有的SQL语句的日志，默认未开启。
long_query_time默认为10秒，最小为0，精度可以到微秒。默认情况下，不会记录管理语句，也不会记录不使用索引进行查找的查询。
1.vim /etc/my.cnf添加如下语句
  slow_query_log=1
  long_query_time=2(单位秒)
 ```
 
 # 2. 第二部分 主从复制
 ```shell
 主从复制是指将主数据库的DDL和DML操作通过二进制日志传到从库服务器中,然后在从库上对这些日志重新执行（也叫重做)，从而使得从库和主库的数据保持同步。
 MySQL支持一台主库同时向多台从库进行复制，从库同时也可以作为其他从服务器的主库，实现链状复制。
 MySQL复制的有点主要包含以下三个方面:
 1.主库出现问题，可以快速切换到从库提供服务。2.实现读写分离，降低主库的访问压力。3.可以在从库中执行备份，以避免备份期间影响主库服务。
 2.原理
 主要分为三步：
 1）Master 主库在事务提交时，会把数据变更记录在二进制日志文件Binlog中。
 2）从库读取主库的二进制日志文件Binlog，写入到从库的中继日志Relay Log。
 3）slave重做中继日志中的事件，将改变反映它自己的数据。
 3.主从复制的搭建（需要两个服务器）
1） 准备服务器2）配置主库3）配置从库 4）测试主从复制
```

 # 3. 第三部分 分库分表
 ## 3.1 概念
 ```shell
问题分析：随着互联网及移动互联网的发展，应用系统的数据量也是成指数式增长，
若采用单数据库进行数据存储，存在以下性能瓶颈:

1. IO瓶颈:热点数据太多，数据库缓存不足，产生大量磁盘IO，效率较低。请求数据太多，带宽不够，网络IO瓶颈。
2. CPU瓶颈:排序、分组、连接查询、聚合统计等SQL会耗费大量的CPU资源，请求数太多，CPU出现瓶颈。
```
## 3.2 拆分
```shell
垂直拆分：
垂直分库:以表为依据，根据业务将不同表拆分到不同库中。
特点:1.每个库的表结构都不一样。2.每个库的数据也不一样。3.3.所有库的并集是全量数据。
垂直分表:以字段为依据，根据字段属性将不同字段拆分到不同表中。
特点：1.每个表的结构都不一样。2.每个表的数据也不一样，一般通过一列（主键/外键）关联 3．所有表的并集是全量数据。

水平拆分：
水平分库:以字段为依据，按照一定策略，将一个库的数据拆分到多个库。
特点：1.每个库的表结构都一样。2．每个库的数据都不一样。3．所有库的并集是全量数据。
水平分表：水平分表:以字段为依据，按照一定策略，将一爷表的数据拆分到多个表中。
特点：1.每个表的表结构都一样。2.每个表的数据都不一样。3．所有表的并集是全量数据。

分库分表的实现技术：
shardinglDBC∶基于AOP原理，在应用程序中对本地执行的SQL进行拦截，解析、改写、路由处理。需要自行编码配置实现，只支持java语言，性能较高。
MyCat:数据库分库分表中间件，不用调整代码即可实现分库分表，支持多种语言，性能不及前者。

分库分表的中心思想都是将数据分散存储，使得单一数据库/表的数据量变小来缓解单一数据库的性能问题，从而达到提升数据库性能的目的。
```shell
给linux文件授权 chmod 777 XXX 

启动mycat
zsl@zsl-virtual-machine:~$ cd mycat
zsl@zsl-virtual-machine:~/mycat$ bin/mycat start
Starting Mycat-server...
zsl@zsl-virtual-machine:~/mycat$ 
检查是否启动
cd /usr/local/mycat/
ll
tail -f logs/wrapper.log#如果输出了successfully就代表成功
```
## 3.3 mycat配置文件
```shell
三类配置文件：
schema.xml 用来配置逻辑库和逻辑表等信息
server.xml 配置mycat运行服务等相关的信息
rule.xml 配置分片规则等相关信息
```
## 3.4 mycat分片
```shell
mycat垂直拆分：垂直分库
在不同的分片中的各种表是不能实现联查的，因为涉及到了mycat的底层路由
需要将不同分片的表设置为全局表
即在schema.xml 加上type=“global”字段

mycat水平拆分：水平分表
水平拆分的核心：rule配置 例如：rule=“mod-long” 取模节点与mycat配置有关
```
## 3.5 mycat分片规则：
```shell
1.范围：根据指定的字段及其配置的范围与数据节点的对应情况，来决定该数据属于哪一个分片。
rule=auto-sharding-long_query_time
2.取模：根据指定的字段值与节点数量进行求模运算，根据运算结果，来决定该数据属于哪一个分片。
rule=mod-long
3.一致性哈希：所谓一致性哈希，相同的哈希因子计算值总是被划分到相同的分区表中，不会因为分区节点的增加而改变原来数据的分区位置。
rule=sharding-by-murmur
4.枚举：通过在配置文件中配置可能的枚举值,指定数据分布到不同数据节点上,本规则适用于按照省份、性别、状态拆分数据等业务。
rule=sharding-by-intfile-enumstatus
5.应用指定：运行阶段由应用自主决定路由到那个分片，直接根据字符子串（必须是数字）计算分片号。
rule=sharding-by-substring
6.固定分片hash算法：该算法类似于十进制的求模运算，但是为二进制的操作，例如，取id的二进制低10位与1111111111进行位&运算。
rule=sharding-by-long-hash
7.字符串哈希解析：截取字符串中的指定位置的子字符串,进行hash算法，算出分片。
rule=sharding-by-stringhash
8.按天（日期）分片：
rule=sharding-by-date
9.自然月
rule=sharding-by-month
```
## 3.6 mycat管理及监控
```shell
mycat-web安装
第一步 安装zookeeper
启动zookeeper：进入到zookeeper安装目录，输入bin/zkServer.sh start
查看状态：bin/zkServer.sh status 显示standalone即为成功
第二步 安装mycat-web
进入mycat-web
启动mycat：sh start.sh 即可
```
# 4. 第四部分 读写分离
```shell
读写分离,简单地说是把对数据库的读和写操作分开,以对应不同的数据库服务器。
主数据库提供写操作，从数据库提供读操作，这样能有效的减轻单台数据库的压力。
通过MyCat即可轻易实现上述功能，不仅可以支持MySQL，也可以支持Oracle和SQL Server。
```
## 4.1 一主一从 
```shell
原理：MySQL的主从复制，是基于二进制日志（ binlog）实现的。
一主一从读写分离：MyCat控制后台数据库的读写分离和负载均衡由schema.xml文件datahost标签的balance属性控制。
balance四个取值：
0：不开启读写分离机制,所有读操作都发送到当前可用的writeHost上
1：全部的readHost与备用的writeHost都参与select语句的负载均衡（主要针对于双主双从模式）
2：所有的读写操作都随机在writeHost , readHost上分发
3：所有的读请求随机分发到writeHost对应的readHost上执行, writeHost不负担读压力
准备：需要两台服务器
```

## 4.2 双主双从 
```shell
原理:一个主机Master1用于处理所有写请求，它的从机Sslave1和另一台主机Master2还有它的从机Slave2负责所有读请求。
当Master1主机宕机后，Master2主机负责写请求，Master1 、Master2互为备机。
双主双从读写分离：MyCat控制后台数据库的读写分离和负载均衡由schema.xml文件datahost标签的balance属性控制，通过writeType及switchType来完成失败自动切换的。
三个参数配置如下：
balance="1"：代表全部的readHost与stand by writHost参与select语句的负载均衡，简单的说，当双主双从模式(M1->S1，M2->S2，并且M1与M2互为主备。正常情况下，M2,S1,S2都参与select语句的负载均衡。
writeType：0:写操作都转发到第1台witeHost, writeHost1挂了,会切换到writeHost2上;1:所有的写操作都随机地发送到配置的writeHost上。
switchType：-1:不自动切换1:自动切换
准备：需要五台服务器
```
* 运维部分对一个SQL小白而言操作起来还是有难度的，不过跟着老师一步一步操作，还是能学到很多东西，至此，MySQL基础学习完结撒花，感谢老师，非常推荐大家学习黑马程序员系列课程，很适合小白。
