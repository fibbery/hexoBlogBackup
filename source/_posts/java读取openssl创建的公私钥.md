---
title: java读取openssl创建的公私钥
date: 2018-01-19 18:38:12
tags:
	- openssl
	- rsa
---
最近在考虑使用Netty搭建HTTP/HTTPS代理服务器，遇到了openssl密钥读取问题，下面概述步骤。
选用openssl生成密钥的初衷是netty的SslContextBuilder对其支持比较直接，相较于使用keytool更为容易读取。

## 公私钥以及证书的生成
openssl工具生成十分简单，只概叙其命令步骤
```shell
# 生成私钥
1. openssl genrsa -out rsa_private_key.pem 1024
# 根据私钥生成公钥
2. openssl rsa -in rsa_private_key.pem -out rsa_public_key.pem -outform PEM -pubout  
# java不能直接读取该私钥，需要转换成pkcs8格式
3. openssl pkcs8 -topk8 -inform PEM -in rsa_private_key.pem -outform PEM -nocrypt -out rsa_private_key_pkcs8.pem
# 根据私钥创建证书请求
4. openssl req -new -key rsa_private_key.pem -out rsa_public_key.csr
# 生成证书并且签名
5. openssl x509 -req -days 3650 -in rsa_public_key.csr -signkey rsa_private_key.pem -out rsa_public_key.crt

```

### java读取私钥以及公钥
文本打开公私钥，可以发现格式都是如下类似格式：
```
-----BEGIN PUBLIC KEY-----
MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQC5M2njgC84dLEvMHNHEmZMhtYx
w7ncLjZQIyAHyGfruJqTksdJ16V1eRNN3pEn17QAOk6Y3pPjyH/+Meir2iK9GGyB
L9Yvli0eo5xMcLv1XmLeLgQ1EFetRj6i4jTtvaIAIb0uAG3XHHqmQRD6+nLnqV5E
f3iHyXNfo12Y7bjbbwIDAQAB
-----END PUBLIC KEY-----

```
可以看出导出的文本都是Base64编码的字符串，我们只需要读入有效字符串，然后实例化成具体实体即可。

1. 读取文本字符串
```java
private static String readKeyStr(InputStream in) throws Exception {
        BufferedReader br = new BufferedReader(new InputStreamReader(in));
        StringBuilder buffer = new StringBuilder();
        String line = null;
        while ((line = br.readLine()) != null) {
            if (line.charAt(0) != '-') {
                buffer.append(line);
            }
        }
        return buffer.toString();
    }
```
2. base64解码成真正的密钥字符
```java
 byte[] bytes = Base64.getDecoder().decode(readKeyStr(in));
```
3. 实例化成具体密钥实体
```java
 public static RSAPublicKey loadPublicKey(InputStream in) throws Exception {
        byte[] bytes = Base64.getDecoder().decode(readKeyStr(in));
        KeyFactory factory = KeyFactory.getInstance("rsa");
        X509EncodedKeySpec spec = new X509EncodedKeySpec(bytes);
        return (RSAPublicKey) factory.generatePublic(spec);
    }
    
public static RSAPrivateKey loadPrivateKey(InputStream in) throws Exception {
        byte[] bytes = Base64.getDecoder().decode(readKeyStr(in));
        KeyFactory factory = KeyFactory.getInstance("rsa");
        PKCS8EncodedKeySpec spec = new PKCS8EncodedKeySpec(bytes);
        return (RSAPrivateKey) factory.generatePrivate(spec);
    }
```

