# 1.注册亚马逊云平台配置

### 1.1实例EC2 Amazon Linux AMI (HVM),ssd Volume Type

### 1.2添加安全组 

入站:Custom udp rule  Port:1194 

### 1.3创建一个新的密钥 create a new key pair

key pair name --> download to location

# 2.ssh连接配置

putty 或者 xshell 导入*.pem 生成连接  使用ec2-user+xshell生成密钥连接

设置root和ec2-user密码

```
sudo passwd root
sudo passwd ec2-user
```

完成可以使用用户名密码连接ams 

# 3.设置openvpn的服务器server和客户端client

## 3.1 配置服务器server

- install

```bash
sudo yum install -y openvpn
sudo modprobe iptable_nat
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
sudo iptables -t nat -A POSTROUTING -s 10.4.0.1/2 -o eth0 -j MASQUERADE
sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
```

-  Setting up PKI authentication with easy-rsa (recommended)使用easy-rsa进行PKI身份验证

```bash
#1.启用epe1仓库
sudo yum install easy-rsa -y --enablerepo=epe1
#2.在openvpn安装目录中出啊关键一个easy-rsa目录
sudo mkdir /etc/openvpn/easy-rsa
#3.进入该目录
cd /etc/openvpn/easy-rsa
#4.将easy-rsa文件遍历复制到当前目录
sudo cp -RV /usr/share/easy-rsa/3.0.6 ./
#5.使用easyrsa初始化pki 并构建一个证书密钥对(输入pem密码,且提示输入一个通用名称,或者回车保持默认)
sudo ./easyrsa init-pki
sudo ./easyrsa build-ca
#6.设置一个diffie-hellman密钥(提供前向保密)
sudo ./easyrsa gen-dh
#7.生成dh.pem(不带密码保护)
sudo ./easyrsa gen-req server nopass
#8.回车,将通用名保存为服务器名,注册密钥,签署证书
sudo ./easyrsa sign-req server server
#9.输入yes确认证书,并输入CA密码(步骤5设置的密码)
#10.设置客户端,不要密码
sudo ./easyrsa gen-req client nopass
#11.设置客户端名称,按下回车保持通用就行
sudo ./easyrsa sign-req client client
#12.确认并输入ca密码
#13.设置TSL密钥(前向保密)
cd /etc/openvpn #打开openvpn目录
openvpn --genkey --secret pfs.key
#14.服务器配置文件
cd /etc/openvpn
sudo vi server.conf
------------------------- server.conf -----------------------------------------
port 1194
proto udp
dev tun
ca /etc/openvpn/easy-rsa/../pki/ca.crt   #在easy-rsa安装目录下的pki文件夹中
cert /etc/openvpn/easy-rsa/../pki/issued/server.crt #在easy-rsa安装目录下的pki文件夹中
key /etc/openvpn/easy-rsa/../pki/private/server.key #在easy-rsa安装目录下的pki文件夹中
dh /etc/openvpn/easy-rsa/../pki/dh.pem #在easy-rsa安装目录下的pki文件夹中
cipher AES-256-CBC
auth SHA512
server 10.8.0.0 255.255.255.0
push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 8.8.8.8"
push "dhcp-option DNS 8.8.4.4"
ifconfig-pool-persist ipp.txt
keepalive 10 120
comp-lzo
persist-key
persist-tun
status openvpn-status.log
log-append openvpn.log
verb 3
tls-server
tls-auth /etc/openvpn/pfs.key
-------------------------------------------------------------------------------

#15.启动openvpn
sudo service openvpn start


其他命令
sudo chkconfig openvpn on
server.sh脚本
vi server.sh
------------------------------server.sh------------------------------------
#!/bin/sh
echo 1 | sudo  tee /proc/sys/net/ipv4/ip_forward
sudo iptables -t nat -A POSTROUTING -s 10.4.0.1/2 -o eth0 -j MASQUERADE
sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
----------------------------------------------------------------------------
```

## 3.2 配置client客户端

以上服务器配置完成

配置客户端,需要将服务器中的证书和私钥文件拷贝到客户端

```bash
# 1.提升文件夹权限

sudo su  # 启用root用户

# 2.降低文件权限

cd /etc/openvpn  #打开openvpn目录
mkdir keys       #创建keys文件夹
3.#拷贝6个证书和私钥文件到keys文件夹中
cp pfs.key keys
cp /etc/openvpn/easy-rsa/../pki/dh.pem keys
cp /etc/openvpn/easy-rsa/../pki/ca.crt keys
cp /etc/openvpn/easy-rsa/../pki/private/ca.key keys
cp /etc/openvpn/easy-rsa/../pki/private/client.key keys
cp /etc/openvpn/easy-rsa/../pki/issued/client.crt keys

# 4 权限

chmod 777 *

# 5.将keys中的文件远程拷贝到windows中,共6个文(pfs.key dh.pem ca.crt ca.key client.key client.crt) openvpn客户端安装文件下的config文件夹中

# 6.删除服务器中的ca证书
sudo rm /etc/openvpn/easy-rsa/pki/private/ca.key
sudo rm /etc/openvpn/keys/ca.key
# 7.恢复服务器中keys的权限
cd /etc/openvpn/keys
sudo chmod 600 *
# 8.在客户端windows中创建一个配置文件  client.ovpn
-----------------------------client.opvn----------------------------
client
dev tun
proto udp
remote IP地址 1194  #ip地址替换亚马逊的公网ip
ca ca.crt
cert client.crt
key client.key
tls-version-min 1.2
tls-cipher TLS-ECDHE-RSA-WITH-AES-128-GCM-SHA256:TLS-ECDHE-ECDSA-WITH-AES-128-GCM-SHA256:TLS-ECDHE-RSA-WITH-AES-256-GCM-SHA384:TLS-DHE-RSA-WITH-AES-256-CBC-SHA256
cipher AES-256-CBC
auth SHA512
resolv-retry infinite
auth-retry none
nobind
persist-key
persist-tun
ns-cert-type server
comp-lzo
verb 3
tls-client
tls-auth pfs.key
--------------------------------------------------------------------
```

