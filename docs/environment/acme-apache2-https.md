# ACME.sh 在 Linux服务器Apache2部署HTTPS证书

## 环境及工具

- OS : Debian GNU/Linux 9
- web服务器: Debian GNU/Linux 9
- acme.sh V2.8.4

## 部署流程

### 一. 安装acme.sh

 **acme.sh** 实现了 `acme` 协议, 可以从 letsencrypt 生成免费的证书。更多高级用法参考项目地址： https://github.com/Neilpang/acme.sh 

安装命令：

``` shell
curl  https://get.acme.sh | sh
```

普通用户和 root 用户都可以安装使用. 安装过程进行了以下几步:

1. 把 acme.sh 安装到你的 **home** 目录下:

``` bash
~/.acme.sh/
```

并创建 一个 bash 的 alias, 方便你的使用: `alias acme.sh=~/.acme.sh/acme.sh`

2. 自动为你创建 cronjob, 每天 0:00 点自动检测所有的证书, 如果快过期了, 需要更新, 则会自动更新证书.

更高级的安装选项请参考: https://github.com/Neilpang/acme.sh/wiki/How-to-install

**安装过程不会污染已有的系统任何功能和文件**, 所有的修改都限制在安装目录中: `~/.acme.sh/`

### 二. 生成证书

 **acme.sh** 实现了 **acme** 协议支持的所有验证协议. 一般有两种方式验证: http 和 dns 验证. 此处使用了 http 方式。 由于使用的 **apache**服务器, **acme.sh** 还可以智能的从 **apache**的配置中自动完成验证 

``` bash
acme.sh  --issue  -d mydomain.com -d www.mydomain.com  --apache
```

### 三. copy/安装 证书

1. 创建目录存储证书

   ``` bash
   mkdir -p /etc/apache2/2.2/ssl
   ```

2. 运行amce.sh 安装证书

   ``` bash
   acme.sh --install-cert -d online.domain.com \
   --cert-file /etc/apache2/2.2/ssl/online.domain.com-cert.pem \
   --key-file /etc/apache2/2.2/ssl/online.domain.com-key.pem \
   --fullchain-file /etc/apache2/2.2/ssl/letsencrypt.pem \
   --reloadcmd "service apache2 force-reload"
   ```

3. 配置SSL

   修改`/etc/apache2/sites-available/default-ssl.conf`

   `ServerAdmin `下另起一行加上:

   `ServerName 你的域名:443 `

   找到`SSLEngine `修改

   ``` apacheconf
   SSLCertificateFile /etc/apache2/2.2/ssl/online.domain.com-cert.pem
   SSLCertificateKeyFile /etc/apache2/2.2/ssl/online.domain.com-key.pem
   SSLCertificateChainFile "/etc/apache2/2.2/ssl/letsencrypt.pem"
   
   SSLCACertificatePath "/etc/apache2/2.2/ssl/"
   SSLCACertificateFile "/etc/apache2/2.2/ssl/letsencrypt.pem"
   ```

4. 重启验证

   ``` bash
   service apache2 restart
   ```

   