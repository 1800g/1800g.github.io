### MySQL基准测试

什么是基准测试？

> 基准测试是一种测量和评估软件性能指标的活动用于建立某个时刻的性能基准，以便当系统发生软硬件变化时重新进行基准测试以评估变化对性能的影响。

#### 基准测试的目的
- 建立MySQL服务器的性能基准线
- 模拟比当前系统更高的负载，以找出系统的扩展瓶颈
- 测试不同的硬件、软件和操作系统的配置
- 证明新的 硬件设备是否配置正确

#### 如何进行基准测试
- 从系统入口进行测试(如网站Web前端，手机APP前端)
  ​	优点：
  - 能够测试整个系统的性能，包括web服务器、缓存、数据库等
  - 能反映出系统中各个组件接口间的性能问题体现真实性能情况
    缺点:
  - 测试设计复杂，消耗时间长
- 单独对MySQL进行基准测试
    优点
	- 测试设计简单，所需耗费时间短 
	缺点：
	- 无法全面了解整个系统的性能基线

#### MySQL基准测试的常见指标
- 单位时间内所处理的事务数(TPS)
- 单位时间内所处理的查询数(QPS)
- 响应时间
	平均响应时间、最小响应时间、最大响应时间、各时间所占百分比
- 并发量:同时处理的查询请求的数量
	正在工作中的并发的操作数或同时工作的数量

#### 基准测试的步骤
计划和设计基准测试
- 对整个系统还是某一组件
- 使用什么样的数据
- 准备基准测试及数据收集脚本
	CPU使用率、IO、网络流量、状态与计数器信息等
- 运行基准测试
- 保存及分析基准测试结果


信息收集脚本示例
```shell
#!/bin/bash
INTERVAL=5
PREFIX=$HOME/benchmarks/$INTERVAL-sec-status
RUNFILE=$HMOE/benchmarks/running
echo "1" > $RUNFILE
MYSQL=/usr/local/mysql/bin/mysql
$MYSQL -e "show global variables" >> mysql-variables
while test -e $RUNFILE; do
	file=$(date +%F_%I)
	sleep=$(date +%s.%N | awk '{print 5 - ($1 % 5)}')
	sleep $sleep
	ts="$(date +"TS %s.%N %F %T")"
	loadavg="$(uptime)"
	echo "$ts $loadavg" >> $PREFIX-${file}-status
	$MYSQL -e "show global status" >> $PREFIX-${file}-status &
	echo "$ts $loadavg" >> $PREFIX-${file}-innodbstatus
	$MYSQL -e "show engine innodb status" >> $PREFIX-${file}-innodbstatus &
	echo "$ts $loadavg" >> $PREFIX-${file}-processlist
	$MYSQL -e "show full processlist\G" >> $PREFIX-${file}-processlist &
	echo $ts
done
echo Exiting because $RUNFILE does not exists

```

分析脚本示例
```shell
#!/bin/bash
awk '
   BEGIN {
     printf "#ts date time load QPS";
     fmt=" %.2f";
   }
   /^TS/ {
   ts = substr($2,1,index($2,".")-1);
   load = NF -2;
   diff = ts - prev_ts;
   printf "\n%s %s %s %s",ts,$3,$4,substr($load,1,length($load)-1);
   prev_ts=ts;
   }
   /Queries/{
   printf fmt,($2-Queries)/diff;
   Queries=$2
   }
   ' "$@"

```

#### 基准测试中容易忽略的问题
- 使用生产环境数据时只使用了部分数据
- 在多用户场景中，只做单用户的测试
  推荐:使用多线程并发测试
- 在单服务器上测试分布式应用
  推荐:使用相同架构进行测试
- 反复执行同一查询
  容易缓存命中，无法反映真实查询性能

#### 常用的基准测试工具介绍
**mysqlslap**
下载及安装
MySQL服务器自带的基准测试工具,随MySQL一起安装
特点:
- 可以模拟服务器负载，并输出相关统计信息
- 可以指定也可以自动生成查询语句
常用参数说明
--auto-generate-sql 由系统自动生成SQL脚本进行测试
--auto-generate-sql-add-autoincrement 在生成的表中增加自增ID
--auto-generate-sql-load-type 指定测试中使用的查询类型
--auto-generate-sql-write-number 指定初始化数据时生成的数据量
--concurrency 指定并发线程的数量
--engine 指定要测试表的存储引擎，可与用逗号分隔多个存储引擎
--no-drop 指定不清理测试数据
--iterations 指定测试运行的次数
--number-of-queries 指定每一个线程执行的查询数量
--debug-info 指定输出额外的内存及CPU统计信息
--number-int-clos 指定测试表中包含的INT类型列的数量
--number-char-cols 指定测试表中包含的varchar类型列的数量
--create-schema 指定了用于执行测试的数据库的名字
--query 用于指定自定义SQL的脚本
--only-print 并不运行测试脚本，而是把生成的脚本打印出来



**sysbench**
安装说明
https://github.com/akopytov/sysbench/archive
./autogen.sh
./configure --with-mysql-includes=/usr/local/mysql/include/ \
--with-mysql-libs=/usr/local/mysql/lib/

make && make install
常用参数:
--test 用于指定所要执行的测试类型，支持一下参数
	Fileio 文件系统I/O性能测试
	cpu cpu性能测试
	memory 内存性能测试
	Oltp 测试要指定具体的Lua脚本
--mysql-db 用于指定执行基准测试的数据库名
--mysql-table-engine 用于指定所使用的存储引擎
--oltp-table-count 指定测试的表的数量
--oltp-tables-size 指定每个表中的数据行数
--num-threads 指定测试的并发线程数量
--max-time 指定最大的测试时间
--report-interval 指定间隔多长时间输出一次统计信息
--mysql-user 指定执行测试的MySQL用户
--mysql-password 指定执行测试的MySQL用户的密码
prepare 用于准备测试数据
run 用于实际进行测试
cleanup 用于清理测试数据
