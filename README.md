## 关闭ubuntu服务器53服务

默认安装好ubuntu server 20.04LTS后，系统会自动开放53的dns服务为本机提供服务。

Ubuntu的systemd-resolved将默认监听在53号端口，如果我们需要运行自己定义的dns服务器，端口已经在使用会导致端口冲突。

1、查看端口情况
```
root@ub20:/home/sk# netstat -lnpt|grep 53
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      2119/systemd-resolv
或者
root@ub20:/home/sk# sudo lsof -i :53
COMMAND    PID            USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
systemd-r 2119 systemd-resolve   12u  IPv4  67939      0t0  UDP localhost:domain
systemd-r 2119 systemd-resolve   13u  IPv4  67940      0t0  TCP localhost:domain (LISTEN)
```

2、如何停止ubuntu上的systemd-resolved服务使用53

我们可以修改/etc/systemd/resolved.conf中DNSStubListener的注释行，它将不再打开dns服务
```
root@srv201:~# cat /etc/systemd/resolved.conf
#  This file is part of systemd.
#
#  systemd is free software; you can redistribute it and/or modify it
#  under the terms of the GNU Lesser General Public License as published by
#  the Free Software Foundation; either version 2.1 of the License, or
#  (at your option) any later version.
#
# Entries in this file show the compile time defaults.
# You can change settings by editing this file.
# Defaults can be restored by simply deleting this file.
#
# See resolved.conf(5) for details
[Resolve]
#DNS=
#FallbackDNS=
#Domains=
#LLMNR=no
#MulticastDNS=no
#DNSSEC=no
#DNSOverTLS=no
#Cache=no-negative
#DNSStubListener=yes  将这行的注释拿掉,改为no保存，如下
DNSStubListener=no  
#ReadEtcHosts=yes
```
3、将下面的文件创建一个软链接到etc文件夹下
```
sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
#s表示软链接
#f表示存在即覆盖
```
4、生效配置
```
systemctl restart systemd-resolved.service
或者
reboot
```
检查53是否使用：
```
lsof -i :53
```

## 设置私有 Burp Collaborator 服务器

先决条件：

	- 一个 VPS（我使用的是 vultr VPS）
	- 域名（子域也可以）
	- 通配符 DNS 证书（免费 LetsEncrypt 证书）
	- Burp Suite Pro（您不需要许可证，这意味着包括 BurpCommunity 用户在内的任何人都可以部署私有协作者服务器）

1、设置 VPS
```
sudo apt-get update
sudo apt-get install openjdk-17-jdk
sudo mkdir -p /usr/local/collaborator/
```

[现在从这里](https://portswigger.net/burp/releases#professional)下载最新的 BurpSuite Pro并将其推送到/usr/local/collaborator目录。在终端中运行ifconfig命令并查找您的内部和外部 IP。

使用以下命令
```
nano /usr/local/collaborator/collaborator.config
```
在我们的工作目录/usr/local/collaborator下创建一个collaborator.config文件。

代码如下：

```
{
  "serverDomain" : "outofbandconnections.yourdomain.com",
  "workerThreads" : 10,
  "eventCapture": {
      "localAddress" : [ "139.180.X.X" ],
      "publicAddress" : "139.180.X.X",
      "http": {
         "ports" : 80
       },
      "https": {
          "ports" : 443
      },
      "smtp": {
          "ports" : [25, 587]
      },
      "smtps": {
          "ports" : 465
      },
      "ssl": {
          "certificateFiles" : [
              "/usr/local/collaborator/keys/privkey.pem",
              "/usr/local/collaborator/keys/cert.pem",
              "/usr/local/collaborator/keys/fullchain.pem" ]
      }
  },
  "polling" : {
      "localAddress" :  "139.180.X.X",
      "publicAddress" :  "139.180.X.X",
      "http": {
          "port" : 39090
      },
      "https": {
          "port" : 39443
      },
      "ssl": {
          "certificateFiles" : [
              "/usr/local/collaborator/keys/privkey.pem",
              "/usr/local/collaborator/keys/cert.pem",
              "/usr/local/collaborator/keys/fullchain.pem" ]

      }
  },
  "metrics": {
      "path" : "jnaicmez8",
      "addressWhitelist" : ["0.0.0.0/1"]
  },
  "dns": {
      "interfaces" : [{
          "name":"ns1.outofbandconnections.yourdomain.com",
          "localAddress":"139.180.X.X",
          "publicAddress":"139.180.X.X"
      }],
      "ports" : 53
   },
   "logLevel" : "INFO"
}

```
在**localAddress** 和 **publicAddress** 中，通过运行 ifconfig 命令输入您的 VPS IP，并将**serverDomain** 替换为您的域名。

2、设置通配符 SSL 证书

使用以下命令
```
nano /usr/local/collaborator/configure_certs.sh
```
在我们的工作目录下创建一个configure_certs.sh文件。

代码如下：
```
CERTBOT_DOMAIN=$1
if [ -z $1 ];
then
    echo "Missing mandatory argument. "
    echo " - Usage: $0  <domain> "
    exit 1
fi
CERT_PATH=/etc/letsencrypt/live/$CERTBOT_DOMAIN/
mkdir -p /usr/local/collaborator/keys/

if [[ -f $CERT_PATH/privkey.pem && -f $CERT_PATH/fullchain.pem && -f $CERT_PATH/cert.pem ]]; then
        cp $CERT_PATH/privkey.pem /usr/local/collaborator/keys/
        cp $CERT_PATH/fullchain.pem /usr/local/collaborator/keys/
        cp $CERT_PATH/cert.pem /usr/local/collaborator/keys/
        chown -R collaborator /usr/local/collaborator/keys
        echo "Certificates installed successfully"
else
        echo "Unable to find certificates in $CERT_PATH"
fi
```

要安装 Let's Encrypt 证书，请运行以下命令。
```
snap install --classic certbot
```
获取证书运行一下命令
```
certbot certonly -d outofbandconnections.yourdomain.com -d *.outofbandconnections.yourdomain.com  --server https://acme-v02.api.letsencrypt.org/directory --manual --agree-tos --no-eff-email --manual-public-ip-logging-ok --preferred-challenges dns-01
```
**注意**：使用子域 outofbandconnections.yourdomain.com 需指向（您的VPS外部IP）的一条A记录，否则申请通配符证书无法通过验证

按照指南进行操作（它会要求您插入电子邮件）。

之后，您将看到有关如何显示 DNS TXT 记录的第一条消息。

按 Enter 键，让它给您第二条消息。

现在您有两个不同的 TXT 记录需要设置，请转到 DNS 服务器并配置这两个记录。使用相同的名称："_acme-challenge.outofbandconnections"

运行以下命令来安装证书
```
chmod +x /usr/local/collaborator/configure_certs.sh && /usr/local/collaborator/configure_certs.sh outofbandconnections.yourdomain.com
```
现在让我们第一次通过 VPS 运行我们的协作服务器。运行以下命令并查看我们的端口是否正确映射。

**注意**：其他服务可能正在使用我们在 collaborator.config 文件中定义的这些端口。因此，请确保没有其他服务正在使用这些端口，如果是这样，请先关闭这些服务，然后运行以下命令。

```
bash -c  "java -Xms10m -Xmx200m -XX:GCTimeRatio=19 -jar /usr/local/collaborator/burpsuite_pro.jar --collaborator-server --collaborator-config=/usr/local/collaborator/collaborator.config"
```
如果一切正常，那么我们就可以进入下一阶段，即设置DNS。按住CTRL + C一段时间，停止服务。

3、域名系统设置

转到您的 DNS 服务器并创建两条新记录。

	- 创建一条NS记录				outofbandconnections.yourdomain.com 指向 ns1.outofbandconnections.yourdomain.com
	- 创建指向（您的VPS外部IP）的A记录 	ns1.outofbandconnections.yourdomain.com 指向 139.180.X.X
	
就是这样！我们到这里就完成了。

为了持续运行协作者服务，我们可以创建一个服务。请按照以下步骤创建协作者服务。
```
sudo nano /etc/systemd/system/collaborator.service
```
将以下代码复制到collaborator.service文件中
```
[Unit]
Description=Burp Collaborator Server Daemon
After=network.target

[Service]
Type=simple
UMask=007
ExecStart=/usr/bin/java -Xms10m -Xmx200m -XX:GCTimeRatio=19 -jar /usr/local/collaborator/burpsuite_pro.jar --collaborator-server --collaborator-config=/usr/local/collaborator/collaborator.config
Restart=on-failure

# Configures the time to wait before service is stopped forcefully.
TimeoutStopSec=300

[Install]
WantedBy=multi-user.target
```
启用该服务：
```
systemctl enable collaborator
```
最后，启动服务：
```
systemctl start collaborator
```

4、BurpSuite 设置

打开 Burp Suite，转到"Project Options > Misc"选项卡，然后配置以下设置：

	- 服务器位置(Server location)：outofbandconnections.yourdomain.com
	- 轮询位置(Polling location)：outofbandconnections.yourdomain.com:39443

然后单击"运行运行状况检查(Run health check)..."

5、要监控性能和使用情况统计信息

请访问服务器的指标页面 URL 为：https://outofbandconnections.yourdomain.com/jnaicmez8/metrics


6、参考连接

	- https://portswigger.net/burp/documentation/collaborator/server/private
 	- https://portswigger.net/burp/documentation/collaborator/server/private/example
  	- https://portswigger.net/burp/documentation/collaborator/server/private#setting-up-the-configuration-file
	- https://blog.roughwire.com/?p=24
	- https://www.linkedin.com/pulse/setting-up-private-burp-collaborator-server-google-cloud-mark-sowell-3fglc
	- https://www.nuharborsecurity.com/blog/creating-a-private-burp-collaborator-in-amazon-aws-with-a-letsencrypt-wildcard-certificate
	- https://takshilp.medium.com/private-burp-collaborator-af18ac8649e4
	- https://teamrot.fi/self-hosted-burp-collaborator-with-custom-domain/
