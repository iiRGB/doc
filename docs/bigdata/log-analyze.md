# Hadoop案例-交易日志分析

## 数据来源和收集

数据来源是服务产生的日志文件，格式为：

``` shell
[info][2019/01/01 00:15:26][main]org.fone.ctf-[2019010160958][201901012056][3][TX76447][660088][CN9004510]:测试
# 级别 日期 线程 类 全局流水号 交易流水号 渠道 交易代码 柜员号 机构号 日志信息
```

通过命令将日志文件上传到`HDFS`系统。

```shell
# 切换用户
[root@H32 ~]# su hdfs
# 创建文件夹用于日志收集和清理存储
[hdfs@H32 root]$ hadoop fs -mkdir -p /log-analyze/input/20190101
[hdfs@H32 root]$ hadoop fs -mkdir -p /log-analyze/output/20190101
# 上传日志文件
[hdfs@H32 root]$ hadoop fs -put ~/20190101.log /log-analyze/input/20190101
```

## 数据清理

通过`MapReduce`对日志的每一行就行处理，收集有用信息。

``` shell
# 执行mapReduce
[hdfs@H32 root]$ hadoop jar ~/log-analyze.jar cn.rgb.hadoop.log.TransAnalyze /log-analyze/input/20190101 /log-analyze/output/20190101
```

## 统计分析

通过Hive实现统计分析，先将清洗后数据存入Hive中，以日期作为分区指标

``` sql
# 1.建立源数据表
CREATE EXTERNAL TABLE trans_orign(
level string,
transtime string,
guid string,
transid string,
channel string,
txcode string,
teller string,
org string) 
PARTITIONED BY (logdate string) ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t';
# 加载数据
load data inpath '/log-analyze/output/20190101' into table trans_orign partition(logdate='2019_01_01');

#2.创建明细表
drop table if exists trans_detail;
create table trans_detail(
status string,
transtime string,
daystr string,
timestr string,
year string,
month string,
day string,
hour string,
guid string,
txid string,
channel string,
code string,
teller string,
org string,
province string,
type string)
partitioned by (datestr string)
row format delimited
fields terminated by ',';


# 建立网点维度
create table trans_area(province string);
# 建立交易维度
create table trans_tx(txtype string);
# 建立渠道维度
create table trans_channel(channel string);
# 建立时间维度
create table trans_time(
year string,
month string,
day string,
hour string)
row format delimited
fields terminated by ',';


# 创建自己的函数，详见工程源码
add jar ~/log-analyze-util.jar;

# 创建临时函数
create temporary function getProvince as 'cn.rgb.hadoop.util.OrgUtil' USING JAR '/home/hdfs/log-analyze-util.jar';
create temporary function getTxType as 'cn.rgb.hadoop.util.TransTypeUtil' USING JAR '/home/hdfs/log-analyze-util.jar';


# 导入交易明细
insert into trans_detail partition(datestr='2019_01_01')
  select 
  	level as status,
  	transtime,
  	substring(transtime,0,10) as daystr,
  	substring(transtime,9) as timestr,
  	substring(transtime,0,4) as year,
  	substring(transtime,5,2) as month,
  	substring(transtime,8,2) as day,
  	substring(transtime,11,2) as hour,
  	guid,
  	txid,
  	channel,
  	code,
  	teller,
  	org,
  	getProvince(org) as province,
  	getTxType(type) as type 
  	from trans_orign;

# 导入网点维度表
insert into trans_area select distinct province from trans_detail;

# 导入交易类型维度表
insert into trans_tx select distinct type from trans_detail;

# 导入渠道维度表
insert into trans_channel select distinct channel from trans_detail;


# 小时维度来统计交易
drop table trans_sum_hour;
create table trans_sum_hour(year string,month string,day string,hour string,sum bigint) 
row format delimited
fields terminated by '\t';
# 插入数据
insert into table trans_sum_hour
select a.year as year ,a.month as month,a.day as day,a.hour as hour,
count(1) as sum from trans_detail a
group by a.yearstr,a.month,a.day,a.hour;
 
# 以天为维度来进行统计交易
drop table trans_sum_day;
create table trans_sum_day(year string,month string,day string,sum bigint)
row format delimited
fields terminated by '\t';
# 插入数据
insert into table trans_sum_day
select a.year as year,a.month as month,a.day as day,count(1) as sum from trans_detail a 
group by a.year,a.month,a.day;
 
# 以交易类型来进行统计
drop table trans_sum_txtype;
create table trans_sum_txtype(type string,year string,month string,day string,sum bigint)
row format delimited
fields terminated by '\t';
# 导入数据
insert into trans_sum_txtype
select a.type as type,
b.year as year,
b.month as month,
b.day as day,
sum bigint from trans_tx a
join trans_detail b
on a.type=b.type 
group by a.type,b.year,b.month,b.dayorder by sum desc;
 
# 操作渠道维度来进行统计
drop table trans_sum_channel;
create table trans_sum_channel(
channel string,
year string,
month string,
day string,
sum bigint,
);
 
insert into trans_sum_channel
select  a.channel as channel,
b.yearstr as year,
b.month as month,
b.day as day 
count(1) as sum from trans_channel a
join trans_detail b
on a.channel=b.channel 
group by a.channel,b.year,b.month,b.day order by pvs desc;
 
# 地域的维度去统计
drop table trans_sum_area;
create table trans_sum_area(province string,
year string,
month string,
day string,
sum bigint)
row format delimited
fields terminated by '\t';
# 导入数据
insert into trans_sum_area
select a.province as province,
b.year as year,
b.month as month,b.day as day，count(1) as sum from trans_area a
join trans_detail b on a.province=b.province
group by a.province,a.city,b.yearstr,month,day order by pvs sum;
```

