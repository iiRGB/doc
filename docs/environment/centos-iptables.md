# CentOs7 安装使用 iptables

CentOS7默认的防火墙不是iptables,而是firewalle.

## 安装

### 安装iptable iptable-service

``` shell
#先检查是否安装了iptables
service iptables status
#安装iptables
yum install -y iptables
#升级iptables
yum update iptables 
#安装iptables-services
yum install iptables-services
```

### 禁用/停止自带的firewalld服务

``` shell
#停止firewalld服务
systemctl stop firewalld
#禁用firewalld服务
systemctl mask firewalld
```

### 设置现有规则

``` shell
#查看iptables现有规则
iptables -L -n
#先允许所有,不然有可能会杯具
iptables -P INPUT ACCEPT
#清空所有默认规则
iptables -F
#清空所有自定义规则
iptables -X
#所有计数器归0
iptables -Z
#允许来自于lo接口的数据包(本地访问)
iptables -A INPUT -i lo -j ACCEPT
#开放22端口
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
#开放21端口(FTP)
iptables -A INPUT -p tcp --dport 21 -j ACCEPT
#开放80端口(HTTP)
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
#开放443端口(HTTPS)
iptables -A INPUT -p tcp --dport 443 -j ACCEPT
#允许ping
iptables -A INPUT -p icmp --icmp-type 8 -j ACCEPT
#允许接受本机请求之后的返回数据 RELATED,是为FTP设置的
iptables -A INPUT -m state --state  RELATED,ESTABLISHED -j ACCEPT
#其他入站一律丢弃
iptables -P INPUT DROP
#所有出站一律绿灯
iptables -P OUTPUT ACCEPT
#所有转发一律丢弃
iptables -P FORWARD DROP
```

### 其他规则设定

``` shell
#如果要添加内网ip信任（接受其所有TCP请求）
iptables -A INPUT -p tcp -s 45.96.174.68 -j ACCEPT
#过滤所有非以上规则的请求
iptables -P INPUT DROP
#要封停一个IP，使用下面这条命令：
iptables -I INPUT -s ***.***.***.*** -j DROP
#要解封一个IP，使用下面这条命令:
iptables -D INPUT -s ***.***.***.*** -j DROP
```

### 保存规则设定

``` shell
service iptables save
```

### 开启iptables服务

``` shell
#注册iptables服务
#相当于以前的chkconfig iptables on
systemctl enable iptables.service
#开启服务
systemctl start iptables.service
#查看状态
systemctl status iptables.service
```

### 删除规则

``` shell
#将所有iptables以序号标记显示，执行：
iptables -L INPUT --line-number
#比如要删除INPUT里序号为1的规则，执行：
iptables -D INPUT 1
```



## 使用

### 屏蔽IP

``` shell
#屏蔽单个IP的命令是
iptables -I INPUT -s 123.45.6.7 -j DROP
#封整个段即从123.0.0.1到123.255.255.254的命令
iptables -I INPUT -s 123.0.0.0/8 -j DROP
#封IP段即从123.45.0.1到123.45.255.254的命令
iptables -I INPUT -s 124.45.0.0/16 -j DROP
#封IP段即从123.45.6.1到123.45.6.254的命令是
iptables -I INPUT -s 123.45.6.0/24 -j DROP
```

### NAT 转发

外网IP只有一个时，可以通过iptables nat转发实现内网机器连接外部网络

1. 公网机器开启IP转发

``` shell
#需启用ip转发
vi /etc/sysctl.conf
net.ipv4.ip_forward = 1
# 重加载生效
sysctl -p
```

2. 公网机器添加规则

``` shell
#分别表示：来自内网、出口为ens192的包接受转发；来自eth0、目标地址为内网，且连接状态为建立、相关的包接受转发.
iptables -A FORWARD -s 192.168.10.0/24 -o ens192 -j ACCEPT
iptables -A FORWARD -d 192.168.10.0/24 -m state --state ESTABLISHED,RELATED -i ens192 -j ACCEPT

# nat转发  -s 表示源网络，即内网地址；-o 为连接因特网的接口
iptables -t nat -A POSTROUTING -s 192.168.10.0/24 -o ens192 -j SNAT --to-source 222.xxx.xxx.xxx
```

3. 内网机器添加网关

``` shell
# 设置默认网关
route add default gw 192.168.10.8
# 检查网关设置
route -n
0.0.0.0         192.168.10.8    0.0.0.0         UG    100    0        0 ens192
192.168.10.0    0.0.0.0         255.255.255.0   U     100    0        0 ens192

# 测试网络连接
ping www.baidu.com
```

