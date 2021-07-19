---
title: CURL 与证书链问题
author: fangfeng
date: 2021-05-19
tags:
  - curl
  - ssl
  - certificate
categories:
  - 案例分析
---

```shell
[ffutop@desktop ~] $ curl https://saas.ffutop.com
curl: (60) SSL certificate problem: unable to get local issuer certificate
More details here: https://curl.haxx.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
```

被 SSL 证书反复折磨，终于在某天灵光一现，发现了问题的根本原因。

<!--more-->

## 背景

最近为公司处理域名新证书，受限于流程无法直接操作，被交付了一套经过签署的 SSL 证书（及其私钥）。按照预期，证书丢到 NGINX 服务器上，写两行配置，就能够正常对外提供 HTTPS 服务了。

结果，为先行验证，拿测试机试跑，篇首的错误立刻就出现了。借助 curl 提供的文档，反复对比都没有找到匹配的错误描述。

## 原因

curl 在处理 SSL 握手流程的证书验证( `CertificateVerify` ) 发生了错误。 

```plain
Client                                                Server

ClientHello                   -------->
                                                 ServerHello
                                                Certificate*
                                          ServerKeyExchange*
                                         CertificateRequest*
                              <--------      ServerHelloDone
Certificate*
ClientKeyExchange
CertificateVerify*
[ChangeCipherSpec]
Finished                      -------->
                                          [ChangeCipherSpec]
                              <--------             Finished
Application Data              <------->     Application Data
```

一般来说终端用户证书是三级证书，它由中间证书签署，并可以基于对中间证书的信任，验证用户证书的合法性。比如 `saas.ffutop.com` 的证书由中间证书(`Encryption Everywhere DV TLS CA - G1`) 签署。这份中间证书的合法性由顶级证书 (`DigiCert` ) 验证。至于顶级证书，没有上级证书来验证，它们被有限枚举、直接内置在设备中，并由各大浏览器厂商和操作系统厂商来对其进行合规验证。

本次错误，是 curl 在证书验证过程中，只拿到了 `saas.ffutop.com` 的证书，它的签发方 `Encryption Everywhere DV TLS CA - G1` 并没有内置在设备中，无法形成一个合理的证书链。

```shell
[ffutop@desktop ~] $ # 通过 openssl 查看 saas.ffutop.com 提供的证书
[ffutop@desktop ~] $ openssl s_client -showcerts saas.ffutop.com:443
CONNECTED(00000003)
depth=0 CN = saas.ffutop.com
verify error:num=20:unable to get local issuer certificate
verify return:1
depth=0 CN = saas.ffutop.com
verify error:num=21:unable to verify the first certificate
verify return:1
---
Certificate chain
 0 s:CN = saas.ffutop.com
   i:C = US, O = DigiCert Inc, OU = www.digicert.com, CN = Encryption Everywhere DV TLS CA - G1
-----BEGIN CERTIFICATE-----
MIIF8zCCBNugAwIBAgIQAjsd7UkoEGgdd1kdA14xUjANBgkqhkiG9w0BAQsFADBu
MQswCQYDVQQGEwJVUzEVMBMGA1UEChMMRGlnaUNlcnQgSW5jMRkwFwYDVQQLExB3
d3cuZGlnaWNlcnQuY29tMS0wKwYDVQQDEyRFbmNyeXB0aW9uIEV2ZXJ5d2hlcmUg
                  -----SOMETHING OMITTED---
yCJyKqPT3sNqdKu0Qj7nwOgiVApbCwPV+Cnh3Rvq+esLXmcf3GswhYWSqpiOCMlX
HzaFomA/hMVeft3WODD2e5lgBsZopbSK8eHToY+7E89m8LZuc7Zi
-----END CERTIFICATE-----
---
Server certificate
subject=CN = saas.ffutop.com

issuer=C = US, O = DigiCert Inc, OU = www.digicert.com, CN = Encryption Everywhere DV TLS CA - G1

---
No client certificate CA names sent
Peer signing digest: SHA256
Peer signature type: RSA-PSS
Server Temp Key: X25519, 253 bits
---
SSL handshake has read 2188 bytes and written 400 bytes
Verification error: unable to verify the first certificate
---
# Something omitted
```

## 解决

### 方案1：完善证书链

问题的根本原因是验证过程无法构建完整的证书链，使得 `saas.ffutop.com` 证书的合法性无法得到验证。那么完善证书链即可通过握手流程。

MySSL.com 提供了[证书链修复工具](https://myssl.com/chain_download.html)，可以基于终端用户证书，结合网络上的公开信息，构建一条完整的证书链。

完善证书链后，重新加载 NGINX 服务，再次尝试使用 curl 请求获取了正常的响应。

```shell
[ffutop@desktop ~] $ # 再次通过 openssl 查看 saas.ffutop.com 提供的证书
[ffutop@desktop ~] $ openssl s_client -showcerts saas.ffutop.com:443
CONNECTED(00000003)
depth=2 C = US, O = DigiCert Inc, OU = www.digicert.com, CN = DigiCert Global Root CA
verify return:1
depth=1 C = US, O = DigiCert Inc, OU = www.digicert.com, CN = Encryption Everywhere DV TLS CA - G1
verify return:1
depth=0 CN = saas.ffutop.com
verify return:1
---
Certificate chain
 0 s:CN = saas.ffutop.com
   i:C = US, O = DigiCert Inc, OU = www.digicert.com, CN = Encryption Everywhere DV TLS CA - G1
-----BEGIN CERTIFICATE-----
MIIF8zCCBNugAwIBAgIQAjsd7UkoEGgdd1kdA14xUjANBgkqhkiG9w0BAQsFADBu
MQswCQYDVQQGEwJVUzEVMBMGA1UEChMMRGlnaUNlcnQgSW5jMRkwFwYDVQQLExB3
d3cuZGlnaWNlcnQuY29tMS0wKwYDVQQDEyRFbmNyeXB0aW9uIEV2ZXJ5d2hlcmUg
                  -----SOMETHING OMITTED---
yCJyKqPT3sNqdKu0Qj7nwOgiVApbCwPV+Cnh3Rvq+esLXmcf3GswhYWSqpiOCMlX
HzaFomA/hMVeft3WODD2e5lgBsZopbSK8eHToY+7E89m8LZuc7Zi
-----END CERTIFICATE-----
 1 s:C = US, O = DigiCert Inc, OU = www.digicert.com, CN = Encryption Everywhere DV TLS CA - G1
   i:C = US, O = DigiCert Inc, OU = www.digicert.com, CN = DigiCert Global Root CA
-----BEGIN CERTIFICATE-----
MIIEqjCCA5KgAwIBAgIQAnmsRYvBskWr+YBTzSybsTANBgkqhkiG9w0BAQsFADBh
MQswCQYDVQQGEwJVUzEVMBMGA1UEChMMRGlnaUNlcnQgSW5jMRkwFwYDVQQLExB3
d3cuZGlnaWNlcnQuY29tMSAwHgYDVQQDExdEaWdpQ2VydCBHbG9iYWwgUm9vdCBD
                  -----SOMETHING OMITTED---
sNE2DpRVMnL8J6xBRdjmOsC3N6cQuKuRXbzByVBjCqAA8t1L0I+9wXJerLPyErjy
rMKWaBFLmfK/AHNF4ZihwPGOc7w6UHczBZXH5RFzJNnww+WnKuTPI0HfnVH8lg==
-----END CERTIFICATE-----
---
Server certificate
subject=CN = saas.ffutop.com

issuer=C = US, O = DigiCert Inc, OU = www.digicert.com, CN = Encryption Everywhere DV TLS CA - G1

---
No client certificate CA names sent
Peer signing digest: SHA256
Peer signature type: RSA-PSS
Server Temp Key: X25519, 253 bits
---
SSL handshake has read 3389 bytes and written 400 bytes
Verification: OK
---
# Something omitted
```

### 方案2：跳过验证流程（测试流程可用）

为了能够顺利完成 SSL 握手，`CertificateVerify`  作为可选过程，通过 curl 的 `--insecure` 参数直接跳过，即可正常完成握手。

不过不推荐这个方案，证书链事实上并不完整，虽然在终端用户证书正确的情况下，在一些浏览器也能够正常使用，但不符合标准，并且可能被严格的浏览器禁止。

```shell
[ffutop@desktop ~] $ curl --insecure -v https://saas.ffutop.com
curl --insecure https://saas.ffutop.com -v
*   Trying 172.16.0.10:443...
* TCP_NODELAY set
* Connected to saas.ffutop.com (172.16.0.10) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* successfully set certificate verify locations:
*   CAfile: /etc/ssl/certs/ca-certificates.crt
  CApath: /etc/ssl/certs
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS handshake, Server key exchange (12):
* TLSv1.2 (IN), TLS handshake, Server finished (14):
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
* TLSv1.2 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.2 (OUT), TLS handshake, Finished (20):
* TLSv1.2 (IN), TLS handshake, Finished (20):
* SSL connection using TLSv1.2 / ECDHE-RSA-AES128-GCM-SHA256
* ALPN, server accepted to use http/1.1
* Server certificate:
*  subject: CN=saas.ffutop.com
*  start date: Apr 30 00:00:00 2021 GMT
*  expire date: Apr 30 23:59:59 2022 GMT
*  issuer: C=US; O=DigiCert Inc; OU=www.digicert.com; CN=Encryption Everywhere DV TLS CA - G1
*  SSL certificate verify result: unable to get local issuer certificate (20), continuing anyway.
> GET / HTTP/1.1
> Host: saas.ffutop.com
> User-Agent: curl/7.68.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: nginx/1.16.1
< Date: Wed, 19 May 2021 08:58:18 GMT
< Content-Type: text/html
< Content-Length: 8025
< Last-Modified: Mon, 17 May 2021 05:47:06 GMT
< Connection: keep-alive
< ETag: "60a2035a-1f59"
< Accept-Ranges: bytes
<
<!doctype html>
<html lang="zh">

<head>
</head>
<body>
</body>
</html>
```



