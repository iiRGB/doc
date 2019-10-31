# CentOS安装Tomcat

记录一次安装tomcat的过程，以供将来查看。需要JAVA环境，参考[JAVA安装](environment/setup-jdk)。

## 下载

从[官网]( https://tomcat.apache.org/ )进入，这次选择8.X版本。如下图所示，下载文件。

![image-20191031102801613](..\image\image-20191031102801613.png)

## 上传解压

``` bash
# 上传
scp ./tomcat.tar.gz  root@xxx.xxx.xxx.xxx:/usr/share/tomcat/

# 解压
tar -zxvf apache-tomcat-8.x.tar.gz -C /usr/share
```

## 配置

设置systemctl控制启停及自启动。

创建文件 

``` bash
vim /lib/systemd/system/tomcat.service
```

输入如下配置：

``` shell
[Unit]
Description=tomcat
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/share/apache-tomcat-8.5.43/bin/startup.sh
ExecStop=/usr/share/apache-tomcat-8.5.43/bin/shutdown.sh
ExecReload=/bin/kill -s HUP $MAINPID
RemainAfterExit=yes

Environment='JAVA_HOME=/opt/java'
Environment='JRE_HOME=/opt/java/jre'

[Install]
WantedBy=multi-user.target
```

可执行如下命令：

```bash
# 启动
systemctl start tomcat
# 停止
systemctl stop tomcat
# 重启
systemctl restart tomcat
# 设置自启动
systemctl enable tomcat
# 查看状态
systemctl status tomcat
```

