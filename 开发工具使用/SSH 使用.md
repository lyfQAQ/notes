[https://zhuanlan.zhihu.com/p/423066660](https://zhuanlan.zhihu.com/p/423066660)
## ssh免密登录

1. 在本地生成ssh公钥id_rsa.pub
2. 把本地的id_rsa.pub拷贝到远程服务器上的authorized_keys
3. 修改authorized_keys的权限为600

```
#本地生成公钥，一路回车
ssh-keygen -t rsa -C "你的github账号邮箱"
# 打开本地id_rsa.pub全部拷贝
cat id_rsa.pub
# 拷贝到远程服务器的authorized_keys，粘贴上去
vim ~/.ssh/authorized_keys
# 修改权限为600
chmod 600 authorized_keys

```

## config配置

```
# vim ~/.ssh/config 在这里打开文件进行IP和用户名修改
Host Ubuntu_Tencent
  HostName 49.235.66.200 # 改成自己的IP
  User ubuntu #改成自己的Linux用户

# Mac上可能需要下面这些内容
Host *
  HostKeyAlgorithms +ssh-rsa
  PubkeyAcceptedKeyTypes +ssh-rsa

```

## vscode连接不上远程服务器

> # vscode远程连接开放

# 在root用户下执行

vi /etc/ssh/sshd_config

```
AllowTcpForwarding yes
AllowAgentForwarding yes
GatewayPorts yes
PermitTunnel yes
CLientAliveCountMax 10

```

## 重启ssh

```
systemctl restart sshd

```

## 快速将本地公钥拷贝到远程服务器