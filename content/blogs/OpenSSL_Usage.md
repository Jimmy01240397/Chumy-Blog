---
title: "OpenSSL 簽憑證教學"
date: 2024-07-10T11:00:00+08:00
draft: false
github_link: ""
author: "Chumy"
tags:
  - Certificate
  - Infra
image: ""
description: ""
toc: 
---

## OpenSSL
### Generate a private key
``` bash
openssl genrsa -out <private key file> <key length general we used 2048>
```
執行結果
![](https://i.imgur.com/RH3dXBZ.png)
key 的內容
![](https://i.imgur.com/FU8bv1Z.png)

### Generate a certificate request
``` bash
openssl req -new -key <private file> -out <request file> -addext 'subjectAltName=<Alternative Name>'

#addext is optional
#addext 是可選項
```

執行結果
![](https://i.imgur.com/JYYVsH6.png)

### Self sign web certificate
#### Init environment
``` bash
mkdir demoCA
mkdir demoCA/newcerts
mkdir demoCA/private
> demoCA/index.txt
> demoCA/serial
echo 01 > demoCA/crlnumber
> demoCA/cacert.pem
cp <private key> demoCA/private/cakey.pem
```
![](https://i.imgur.com/Ct9Va1h.png)

#### Modify /etc/ssl/openssl.cnf
Add copy_extensions = copy at /etc/ssl/openssl.cnf under [ CA_default ]
``` bash
vi /etc/ssl/openssl.cnf
```
![](https://i.imgur.com/4VIRu4e.png)

#### Write extensions file
``` bash
vi <extensions file>
```
![](https://i.imgur.com/WXybn8U.png)
![](https://i.imgur.com/uNGts39.png)

#### Sign certificate
``` bash
openssl ca -in <request file> -out <certificate> -days <Validity period (day)> -batch -rand_serial -extfile <extensions file> -selfsign
```

### Sign a Root CA
``` bash
openssl x509 -req -days <Validity period (day)> -sha256 -extfile /etc/ssl/openssl.cnf -extensions v3_ca -signkey <your private key> -in <request file> -out <certificate file>
```

執行結果
![](https://i.imgur.com/B7LJ2Sa.png)
![](https://i.imgur.com/A9Jrbka.png)


### Sign a sub CA certificate
#### Init environment
``` bash
mkdir demoCA
mkdir demoCA/newcerts
mkdir demoCA/private
> demoCA/index.txt
> demoCA/serial
echo 01 > demoCA/crlnumber
cp <CA certificate> demoCA/cacert.pem
cp <CA private key> demoCA/private/cakey.pem
```
![](https://i.imgur.com/67klQBu.png)

#### Write extensions file
``` bash
vi <extensions file>
```
![](https://i.imgur.com/SR1etbR.png)
![](https://i.imgur.com/e2TVD5Q.png)

crlDistributionPoints is optional
crlDistributionPoints 是可選項

#### Sign Sub CA certificate
``` bash
openssl ca -in <request file> -out <sub CA certificate file> -days <Validity period (day)> -batch -rand_serial -extfile <extensions file>
```

執行結果
![](https://i.imgur.com/KWwZuXb.png)

### Sign a certificate
#### Init environment
``` bash
mkdir demoCA
mkdir demoCA/newcerts
mkdir demoCA/private
> demoCA/index.txt
> demoCA/serial
echo 01 > demoCA/crlnumber
cp <CA certificate> demoCA/cacert.pem
cp <CA private key> demoCA/private/cakey.pem
```
![](https://i.imgur.com/Ct9Va1h.png)

#### Modify /etc/ssl/openssl.cnf
Add copy_extensions = copy at /etc/ssl/openssl.cnf under [ CA_default ]
``` bash
vi /etc/ssl/openssl.cnf
```
![](https://i.imgur.com/4VIRu4e.png)

#### Write extensions file
``` bash
vi <extensions file>
```
![](https://i.imgur.com/WXybn8U.png)
![](https://i.imgur.com/uNGts39.png)

#### Sign certificate
``` bash
openssl ca -in <request file> -out <certificate> -days <Validity period (day)> -batch -rand_serial -extfile <extensions file>
```

執行結果
![](https://i.imgur.com/R7TMRlB.png)

![](https://i.imgur.com/Cj02FuE.png)
![](https://i.imgur.com/9QVNtr8.png)
![](https://i.imgur.com/WU3MpFU.png)

### Revoke certificate
``` bash
openssl ca -revoke <certificate file>
```

執行結果
![](https://i.imgur.com/MRVWR1n.png)

### Generate CRL
``` bash
openssl ca -gencrl -out <crl file>

# Everytime you revoke your certificate, you need to regenerate your CRL.
```

執行結果
![](https://i.imgur.com/uBK8LCz.png)
![](https://i.imgur.com/TXyTUaM.png)


## certificate 的信任 (trust of certificate)
### Untrusted Certificate (Root CA untrusted)
![](https://i.imgur.com/ng4xLgR.png)

### Trusted Certificate (Root CA trusted)
![](https://i.imgur.com/6H04mJB.png)

### How to trust Root CA in windows
1. ![](https://i.imgur.com/MZoNzDe.png)
2. ![](https://i.imgur.com/HRJpxQf.png)
3. ![](https://i.imgur.com/NVTc4FW.png)
4. ![](https://i.imgur.com/1L19EBL.png)
5. ![](https://i.imgur.com/DSEt9SO.png)
6. ![](https://i.imgur.com/17jz2or.png)
7. ![](https://i.imgur.com/Ltm8V6p.png)
8. Reopen Root CA certificate file. ![](https://i.imgur.com/wufNBsO.png)

### How to trust Root CA in linux
#### For openssl
Copy CA certificate to /usr/local/share/ca-certificates/ and run update-ca-certificates
``` bash
cp <Root CA certificate> /usr/local/share/ca-certificates/
update-ca-certificates

# flush all ca use: update-ca-certificates -f
```
![](https://i.imgur.com/5f87Zyh.png)

#### For linux browser
Go to setting of browser. Ex: Firefox
![](https://i.imgur.com/R0Fjtm1.png)
![](https://i.imgur.com/IBKroQi.png)

## Nginx ssl setting
![](https://i.imgur.com/CEIe0RM.png)
Windows
![](https://i.imgur.com/0KYmqkN.png)
Linux
![](https://i.imgur.com/6ebDXMN.png)
