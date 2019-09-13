## [原创] 中国科学技术大学 ustc.edu.cn 域增加DNSSEC功能过程

本文原创：**中国科学技术大学 张焕杰**

修改时间：2019.09.13

清华大学段海新老师在2011年就写过有关DNSSEC的技术文章[DNSSEC 原理、配置与布署简介](https://blog.csdn.net/syh_486_007/article/details/50990973)，详细介绍了DNSSEC。本文记录 ustc.edu.cn 域增加DNSSEC功能的过程。


## ustc.edu.cn 域名情况

ustc.edu.cn 分了5个view，每个view有一个zone文件，因此有5个zone文件。bind 软件为CentOS 6中内置，使用chroot模式运行。

由于使用的bind软件并不是最新，因此本文使用的方式不一定是最简单的方式。

bind 大部分文件存放在 /var/named/chroot/var/named 目录，zone文件存放在 /var/named/chroot/var/named/zone 目录。

## 步骤一：生成ZSK和KSK密钥

```
cd /var/named/chroot/var/named
dnssec-keygen -r /dev/urandom -3 ustc.edu.cn
dnssec-keygen -f ksk -r /dev/urandom -3 ustc.edu.cn
cp *.key zone
```

生成了4个文件：
```
Kustc.edu.cn.+007+32747.key
Kustc.edu.cn.+007+32747.private
Kustc.edu.cn.+007+19065.key
Kustc.edu.cn.+007+19065.private
```

由于zone文件放在/var/named/chroot/var/named/zone 目录下，为了后续方便，把 *.key 文件cp到zone目录下

## 步骤二：在zone文件中增加key文件

在zone文件最后面增加：
```
$INCLUDE Kustc.edu.cn.+007+19065.key
$INCLUDE Kustc.edu.cn.+007+32747.key
```

## 步骤三：对原有zone文件进行签名

科大有5个zone文件，因此对5个文件签名，签名后的zone文件自动添加后缀.signed，如ustc.edu.cn.cernet签名后生成文件ustc.edu.cn.cernet.signed

```
cd /var/named/chroot/var/named
dnssec-signzone -A -3 $(head -c 1000 /dev/urandom | sha1sum | cut -b 1-16) -N INCREMENT -o ustc.edu.cn -t zones/ustc.edu.cn.cernet
dnssec-signzone -A -3 $(head -c 1000 /dev/urandom | sha1sum | cut -b 1-16) -N INCREMENT -o ustc.edu.cn -t zones/ustc.edu.cn.chinanet
dnssec-signzone -A -3 $(head -c 1000 /dev/urandom | sha1sum | cut -b 1-16) -N INCREMENT -o ustc.edu.cn -t zones/ustc.edu.cn.cncnet
dnssec-signzone -A -3 $(head -c 1000 /dev/urandom | sha1sum | cut -b 1-16) -N INCREMENT -o ustc.edu.cn -t zones/ustc.edu.cn.cmcc
dnssec-signzone -A -3 $(head -c 1000 /dev/urandom | sha1sum | cut -b 1-16) -N INCREMENT -o ustc.edu.cn -t zones/ustc.edu.cn.other
```

## 步骤四：修改bind配置，使用签名后域文件

如原来配置为
``` 
zone "ustc.edu.cn" in{type master; file "zones/ustc.edu.cn.cernet";};
``` 
修改为
``` 
zone "ustc.edu.cn" in{type master; file "zones/ustc.edu.cn.cernet.signed";};
``` 

同时，确保options中有启用dnssec功能的配置
```
	dnssec-enable yes;
```

## 步骤五：重启bind，生效配置

```
service named restart
```

重启后，访问`https://dnssec-analyzer.verisignlabs.com/ustc.edu.cn`能看到有DNSSEC相关记录，但有个警告是还没有DS记录。

## 步骤六：上级服务器增加DS记录

文件`/var/named/chroot/var/named/dsset-ustc.edu.cn.`中内容如下：
```
ustc.edu.cn.		IN DS 19065 7 1 4EBE527CCF84FC2DD62ACFCD464BE008E0FEAF68
ustc.edu.cn.		IN DS 19065 7 2 EBD1C6420F893D8FF9950ADBF896075D059006439419634128566709 8EBD74F1
```

将这些内容发给edu.cn服务器管理员，添加后，访问`https://dnssec-analyzer.verisignlabs.com/ustc.edu.cn`能看到DNSSEC工作正常。

## 修改域名服务器的签名步骤

一旦修改了域名zone文件，均要重复步骤三的过程，并重启bind。我们使用git pre-commit hook自动这个过程。

默认的域签名有效期30天，30天内要重复步骤三的过程，并重启bind。30天内我们的DNS一定会有修改，不担心这个问题。


***
欢迎 [加入我们整理资料](https://github.com/bg6cq/ITTS)
