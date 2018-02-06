---
title: OpenSSL生成根证书CA以及签发子证书
date: 2018-02-05 16:15:56
tags:
	- openssl
	- 证书
	- https
---

<!-- more -->

##环境
需要安装openssl，如果没有可以使用homebrew
```
brew install openssl
```

## 操作步骤
**下面所有[]中的内容代表可选项目**
### 生成根证书CA
1. 生成根证书私钥
```
openssl genrsa [-aes256] -out ca_private.pem 1024
```
含义：
	* genrsa 表示使用rsa算法产生私钥
	* 1024 表示私钥长度
	* -aes256表示使用256位的AES算法对密钥进行加密

2. 生成根证书请求文件
```
openssl req -new -key ca_private.pem -out ca.csr -subj "/C=CN/ST=Guangdong/L=Shenzhen/O=Zhenai/OU=Java/CN=HttpProxy"
```
含义:
	* req 表示执行证书签发命令
	* new 表示新证书签发请求
	* key 指定证书私钥路径
	* out表示输出的证书请求文件的位置
	* subj表示证书相关的用户信息

3. 自签发根证书
```
openssl x509 -req -days 365 [-sha1] [-extfile openssl.cnf] -extensions v3_ca -signkey ca_private.pem -in ca.csr -out ca.crt
```
含义:
	* x509表示生成x509格式证书
	* req 表示输入csr文件
	* days 表示证书有效期(天)
	* sha1表示证书的摘要使用sha1
	* extensions 表示安装openssl.conf文件中配置的v3_ca项添加拓展
	* -extfile 表示配置文件
	* signkey 表示签发证书使用的私钥

### 用根证书签发server端证书
1. 生成服务端私钥
```
openssl genrsa -out server_private.pem
```

2. 生成证书请求文件
```
openssl req -new -key server_private.pem -out server.csr -subj "/C=CN/ST=Guangdong/L=Shenzhen/O=Zhenai/OU=Java/CN=HttpProxy"
```

3. 使用根证书签发服务端证书
```
openssl x509 -req -days 365 [-sha256] [-extfile openssl.conf] -extensions v3_req -CA ca.crt -CAkey ca_private.pem -CAserial ca.srl -CAcreateserial -in server.csr -out server.crt
```
含义：
	* CA 指定ca证书
	* CAkey指定证书的私钥
	* CAserial 指定证书序列号文件
	* 表示创建证书序列号文件(即上方提到的serial文件)，创建的序列号文件默认名称为-CA，指定的证书名称后加上.srl后缀
同时，注意这里的是**-extensions v3_req** 


## 注意要点
1. 上面过程中生成的私钥是不能直接被java读取的，只能通过下述命令转换一下：
```
openssl pkcs8 -topk8 -nocrypt -inform PEM -outform PEM -in ca_private.pem  -out ca_private_pkcs8.pem
```

2. curl使用证书的时候必须是pem格式的，crt到pem格式转换如下
```
openssl x509 -in ca.crt -out ca.pem -text
```

## 附录
openssl文件：

[下载文件](/assets/file/openssl.conf)
