---
title: "HTTPS 单双向认证及证书相关"
slug: "HTTPS-TLS-and-mTLS-and-certificate-related"
date: 2024-06-09T16:47:46+08:00
author: CapriWits
image: mtls-cover.png
tags:
  - Network
  - Cryptography
categories:
  - Backend
hidden: false
math: true
comments: true
draft: false
---

<!-- TOC -->
* [HTTPS 单双向认证及证书相关](#https-单双向认证及证书相关)
  * [一些常识](#一些常识)
    * [CA 证书和私钥](#ca-证书和私钥)
    * [服务端和客户端SSL证书](#服务端和客户端ssl证书)
    * [证书格式](#证书格式)
    * [证书请求](#证书请求)
    * [证书链](#证书链)
  * [HTTPS 单向认证（SSL/TLS)](#https-单向认证ssltls)
  * [HTTPS 双向认证（mTLS）](#https-双向认证mtls)
  * [自签名证书 mTLS 实战](#自签名证书-mtls-实战)
    * [自签名根证书 CA](#自签名根证书-ca)
    * [自签名服务端证书](#自签名服务端证书)
    * [自签名客户端证书](#自签名客户端证书)
    * [验证](#验证)
      * [带证书的成功调用](#带证书的成功调用)
      * [不带证书的调用](#不带证书的调用)
  * [Reference](#reference)
<!-- TOC -->

# HTTPS 单双向认证及证书相关

## 一些常识

### CA 证书和私钥

- **CA（Certificate Authority）证书**：由权威认证机构（CA）签发的数字证书，用于证明证书持有者的身份。CA证书可以自签名，也可以由更上级的CA签名。根CA证书通常是自签名的，用 root.key 签名一个 root.crt 根证书。CA 证书可能是一个根证书（root.crt），也可能是一个中间证书（子 CA 证书）。
- **CA 私钥**：用于签发和验证其他证书。CA私钥非常重要，必须妥善保管，确保不会泄露。
- 公钥是存储在证书里的，key 是私钥，用于签发证书

常见 CA 机构，可以签发服务端 SSL 证书

1. Let's Encrypt：Let's Encrypt 是一个免费的、自动化的证书颁发机构，致力于推动全球网站加密化。
2. DigiCert：DigiCert 是一家知名的商业证书颁发机构，提供各种 `SSL/TLS` 证书和数字证书解决方案。
3. Sectigo（原名Comodo）：Sectigo 是一家全球领先的数字证书颁发机构，提供 `SSL` 证书、代码签名证书等安全解决方案。
4. GlobalSign：GlobalSign 是一家全球性的数字证书颁发机构，提供 `SSL` 证书、代码签名证书、身份验证等服务。
5. GoDaddy：GoDaddy 是一家知名的域名注册商，也提供 `SSL` 证书和其他网络安全服务。
6. Entrust Datacard：Entrust Datacard 是一家提供数字证书、身份验证和加密解决方案的领先供应商。

### 服务端和客户端SSL证书

- **服务端SSL证书**：安装在服务器上的数字证书，用于加密客户端和服务器之间的通信，确保数据的安全性和完整性。通常由 `CA` 签发，包含服务器的公钥信息。（`server.crt`
- **客户端SSL证书**：安装在客户端上的数字证书，用于验证客户端的身份，通常在双向认证（mTLS）中使用。(`client.crt`

### 证书格式

- `X.509`

`X.509` 格式证书是一种标准的公钥证书格式，`X.509` 证书的主要特点和组成部分包括：

1. 标准化格式：`X.509` 定义了一个结构化的证书格式，确保了不同系统之间能够互操作和理解。
2. 公钥信息：证书包含了证书持有者的公钥，这是用于加密数据或验证数字签名的关键部分。
3. 身份信息：证书还包括证书持有者的身份信息，如组织名称、组织单元、国家/地区等。这些信息帮助验证证书持有者的身份。
4. 签名：证书由证书颁发机构（CA）进行数字签名，以确保其完整性和真实性。这意味着如果证书被篡改，签名将不再有效。
5. 有效期：每个 `X.509` 证书都有一个有效的时间范围，由“开始日期”和“结束日期”定义。这确保了证书的时效性和定期更新需求。
6. 证书链和信任：`X.509` 证书通常是证书链的一部分，其中每个证书都由上一级证书颁发机构签名，最终追溯到一个受信任的根证书。这种结构建立了信任链，使得终端用户可以验证证书的合法性。
7. 扩展性：`X.509` 证书支持扩展字段，允许添加额外信息，如密钥用途、证书策略等，以满足不同应用场景的需求

`.crt` 后缀证书，可能是 `PEM` 编码，也可能是 `DER` 编码

- `PEM` (Privacy Enhanced Mail)

`PEM` 是一种广泛使用的编码格式，特别是安全领域的数据，如证书、密钥等。`PEM` 格式使用 `Base64` 编码的 `ASCII` 文本表示，并带有 `"-----BEGIN CERTIFICATE-----"` 和 `"-----END CERTIFICATE-----"` 的边界标记。可以以 ASCII 的格式存储 `X.509` 格式的证书 crt。

证书可以存储**公钥、私钥、证书、证书链、证书请求**

公钥：通常以 `“-----BEGIN PUBLIC KEY-----”` 和 `“-----END PUBLIC KEY-----”` 为边界

私钥：通常以 `“-----BEGIN PRIVATE KEY-----”` 和 `“-----END PRIVATE KEY-----”` 为边界，或者对于某些类型的私钥（如RSA私钥），可能是 `“-----BEGIN RSA PRIVATE KEY-----”` 和 `“-----END RSA PRIVATE KEY-----”`

证书：包括服务器证书、中间证书等，通常以 `“-----BEGIN CERTIFICATE-----”` 和 `“-----END CERTIFICATE-----”` 为边界

证书请求（如CSR，Certificate Signing Request）：通常以 `“-----BEGIN CERTIFICATE REQUEST-----”` 和 `“-----END CERTIFICATE REQUEST-----”` 为边界。

```bash
# 查看 PEM 证书信息
openssl x509 -in certificate.pem -text -noout
# PEM 转 DER
openssl x509 -in cert.crt -outform der -out cert.der
```

- `DER` (Distinguished Encoding Rules)

`X.509` 证书在二进制形式下通常使用 `DER` 格式。虽然 `DER` 本身不是证书格式，但它是 `X.509` 证书 **二进制编码** 的基础。存证书，没有私钥

```bash
# 查看 DER 证书信息
openssl x509 -in certificate.der -inform der -text -noout
# DER 转 PEM
openssl x509 -in cert.crt -inform der -outform pem -out cert.pem
```

- `PFX/PKCS12`（Public-Key Cryptography Standards #12）

常用于存储私钥和相关的公钥证书链，以 `.pfx` 或 `.p12` 为文件扩展名。这种格式可以转换为 `PEM` 格式，从而提取出私钥和证书。通常包含私钥、证书和证书链的组合

PKCS12 可以包含 **公钥、私钥、证书和证书链** ，并且有 **密码保护** ，一般便于不同平台传输，常用于客户端证书（`client.crt → client.p12`

```bash
# 查看 p12 证书信息
openssl pkcs12 -info -in your_certificate.p12
# p12 转 pem
openssl pkcs12 -in your_certificate.p12 -nocerts -out private_key.pem
```

- `JKS`（Java KeyStore）

`JKS` 是 Java 中用于存储密钥和证书的专有格式，二进制格式。`JKS` 文件通常以 `.jks` 为文件扩展名。在 Java 应用程序中管理密钥和证书，包括私钥、证书、证书链等信息。`JKS` 文件通常需要密码来保护存储在其中的密钥和证书。

生成和转换需要用到 `Java keytool`

### 证书请求

`CSR`（Certificate Signing Request）是一种包含有关组织或个人信息的加密文本，用于向证书颁发机构（CA）申请数字证书。`CSR`包含了将包含在数字证书中的 **公钥**，以及与该公钥相关联的组织信息，如组织名称、域名等。

```bash
# 先创建一个私钥 key
openssl genrsa -out root.key 4096
# 再根据 key 生成一个证书请求
openssl req -new -out root.csr -key root.key
# 下面是提示出来需要填写到 csr 的信息，包括国家、省、城市、公司、单位、域名、密码等信息
Country Name (2 letter code) [XX]:cn
State or Province Name (full name) []:bj
Locality Name (eg, city) [Default City]:bj
Organization Name (eg, company) [Default Company Ltd]:alibaba
Organizational Unit Name (eg, section) []:test
Common Name (eg, your name or your servers hostname) []:root
Email Address []:a.alibaba.com
A challenge password []:
An optional company name []:

# 用 key 和 csr 生成 10 年期限的根证书 root.crt
openssl x509 -req -in root.csr -out root.crt -signkey root.key -CAcreateserial -days 3650
```

生成的 `root.csr` 证书请求，一方面包含公司信息，另一方面包含 key 对应的公钥。

用 key 和 csr 就可以签发一个证书 `root.crt`。假设公司信息不改变，key 也未泄露，由于公私钥是对应的，理论上等到 `root.crt` 过期后，可以再次签发一个 `root.crt`，达到续签的目的。

### 证书链

证书链是指从根证书（Root Certificate）开始，通过中间证书（Intermediate Certificate）逐级签发，直到最终的终端用户证书（End-Entity Certificate）

- **根证书（Root Certificate）**：自签名的证书，位于证书链的最顶端。根证书由权威认证机构（CA）发行并保存在受信任的存储区中。
- **中间证书（Intermediate Certificate）**：由根证书或其他中间证书签发，形成证书链。中间证书用于签发更下级的证书。
- **终端用户证书（End-Entity Certificate）**：由根证书或中间证书签发，最终用于服务器或客户端。

`root.crt → intermediate1.crt → intermediate2.crt → server.crt`

以上过程 `root.crt` 是根证书，以 `root.crt` 为 CA 签发中间证书 `intermediate1.crt`，再用 `intermediate1.crt` 为 CA 签发 `intermediate2.crt` 中间证书，最后 `intermediate2.crt` 签发终端用户证书 `server.crt`

```bash
openssl genrsa -out root.key 4096  # 根私钥
# 根证书
openssl req -new -x509 -days 3650 -key root.key -out root.crt -subj "/CN=Root CA"

openssl genrsa -out intermediate1.key 4096 # 中间证书 1 私钥
# 中间证书请求 1
openssl req -new -key intermediate1.key -out intermediate1.csr -subj "/CN=Intermediate CA 1"
# 根证书签发中间证书 1
openssl x509 -req -in intermediate1.csr -CA root.crt -CAkey root.key -CAcreateserial -out intermediate1.crt -days 3650

openssl genrsa -out intermediate2.key 4096 # 中间证书 2 私钥
# 中间证书请求 2
openssl req -new -key intermediate2.key -out intermediate2.csr -subj "/CN=Intermediate CA 2"
# 中间证书 1 做 CA 签发中间证书 2
openssl x509 -req -in intermediate2.csr -CA intermediate1.crt -CAkey intermediate1.key -CAcreateserial -out intermediate2.crt -days 3650

openssl genrsa -out server.key 4096 # 终端用户证书私钥
# 终端用户证书请求
openssl req -new -key server.key -out server.csr -subj "/CN=server.example.com"
# 中间证书 2 做 CA 签发终端用户证书
openssl x509 -req -in server.csr -CA intermediate2.crt -CAkey intermediate2.key -CAcreateserial -out server.crt -days 365
```

**根证书（root.crt）**：通常保存在操作系统或浏览器的受信任证书存储中。作为信任链的根，客户端无需直接访问它。

**中间证书一（intermediate1.crt）和中间证书二（intermediate2.crt）**：保存在服务器上，用于建立完整的证书链。客户端会通过服务器传递的证书链验证服务器证书的可信度。

**终端用户证书（server.crt）和私钥（server.key）**：安装在服务器上，用于加密通信和验证服务器身份。

**完整的证书链文件**：服务器需要提供一个包含所有中间证书的文件，以便客户端可以验证证书链。可以将中间证书和终端证书合并成一个文件。

一般来说，证书链的顺序是：**SSL 证书 → 中间证书 → 根证书**，依次向上游，一般客户端会默认信任根证书，可以省略

```bash
cat server.crt intermediate2.crt intermediate1.crt > fullchain.crt
```

**服务器配置示例（以Nginx为例）**：

```bash
server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate /path/to/fullchain.crt;
    ssl_certificate_key /path/to/server.key;
}
```

**客户端验证证书链**：

- 客户端验证终端用户证书是否由中间证书二签发。
- 客户端验证中间证书二是否由中间证书一签发。
- 客户端验证中间证书一是否由根证书签发。
- 客户端检查根证书是否在受信任的根证书列表中。

```bash
# 验证中间证书一是否由根证书签发
openssl verify -CAfile root.crt intermediate1.crt # intermediate1.crt: OK

# 接下来，验证中间证书二是否由中间证书一签发。
# 为了验证中间证书二，先创建一个证书链文件，包含根证书和中间证书一。
cat root.crt intermediate1.crt > chain1.pem
openssl verify -CAfile chain1.pem intermediate2.crt # intermediate2.crt: OK

# 最后，验证终端用户证书是否由中间证书二签发。
# 创建一个证书链文件，包含根证书和所有中间证书。
cat root.crt intermediate1.crt intermediate2.crt > chain2.pem
openssl verify -CAfile chain2.pem server.crt # server.crt: OK

# 为了完整验证整个证书链，你可以使用以下命令验证整个证书链文件 fullchain.crt：
cat server.crt intermediate2.crt intermediate1.crt > fullchain.crt
openssl verify -CAfile root.crt fullchain.crt # fullchain.crt: OK
```

## HTTPS 单向认证（SSL/TLS)

![TLS 单向认证](TLS.png)

1. 客户端发起建立HTTPS连接请求，将SSL协议版本的信息发送给服务器端；
2. 服务器端将本机的公钥证书（`server.crt`）发送给客户端；
3. 客户端读取公钥证书（`server.crt`），取出了服务端公钥；
4. 客户端生成一个随机数（密钥 `R`），用刚才得到的服务器公钥去加密这个随机数形成密文，发送给服务端；
5. 服务端用自己的私钥（`server.key`）去解密这个密文，得到了密钥 `R`
6. 服务端和客户端在后续通讯过程中就使用这个密钥 `R` 进行通信了。

服务端部署时，将 `server.crt` 和 `server.key` 上传至服务器即可。如果服务端用云服务 LoadBalancer，则将两个文件上传至 LB 绑定的地方，例如阿里云 CLB 界面有证书管理，AWS 有 ACM（AWS Certificate Manager）专门管理证书等；如果服务端用 K8S Ingress 配置 SSL，则将两个文件 base64 到 secret，再将 secret 配置到 ingress 实现 SSL 单向认证。如果用 Nginx，Apache 等服务器，则找到对应的方式上传证书即可。

key 私钥用于解密客户端公钥加密的数据，也可以在服务端生成签名。crt 证书配合客户端已经信任的 CA，验证服务端的身份（crt 带有服务端的信息，包括国家、城市、公司、域名、证书起始和过期时间等；

一般公网环境，客户端默认在本机都会安装操作系统时，默认信任一些 CA，例如微软系统 IIS 专门管理证书，浏览器自动读取，用户无感；

## HTTPS 双向认证（mTLS）

![mTLS 双向认证](mTLS.png)

1. 客户端发起建立 `HTTPS` 连接请求，将 `SSL` 协议版本的信息发送给服务端；
2. 服务器端将本机的公钥证书（`server.crt`）发送给客户端；
3. 客户端读取公钥证书（`server.crt`），取出了服务端公钥；
4. 客户端将客户端公钥证书（`client.crt`）发送给服务器端；
5. 服务器端使用根证书（`root.crt`）解密客户端公钥证书，拿到客户端公钥；
6. 客户端发送自己支持的加密方案给服务器端；
7. 服务器端根据自己和客户端的能力，选择一个双方都能接受的加密方案，使用客户端的公钥加密后发送给客户端；
8. 客户端使用自己的私钥解密加密方案，生成一个随机数 `R`，使用服务器公钥加密后传给服务器端；
9. 服务端用自己的私钥去解密这个密文，得到了密钥 `R`
10. 服务端和客户端在后续通讯过程中就使用这个密钥 `R` 进行通信了。

证书准备：

- 服务器端公钥证书：`server.crt`
- 服务器端私钥文件：`server.key`
- 根证书：`root.crt`
- 客户端公钥证书：`client.crt`
- 客户端私钥文件：`client.key`
- 客户端集成证书（包括公钥和私钥，用于浏览器访问场景）：`client.p12`

所有证书都可以找 CA 机构签发，如果内网使用，非公众，则可以自建 `PKI`（Public Key Infrastructure） 系统自签发。

服务器除了单向认证步骤里，上传 `server.crt` 和 `server.key` 两个文件外，还需要额外添加 CA 证书 `root.crt`（假设 CA 证书就是根证书，不是子 CA 证书，否则需要上传整个证书链），这个 `client-CA`（`root.crt`) 证书相当于服务端也是一个客户端，需要提前信任一个签发 `client.crt` 的 CA 证书，便于客户端发送带客户端公钥的 `client.crt` 时，验明客户端身份，实现双向认证（即额外增加客户端的身份验证）。

因此客户端本地除了已经信任签发 `server.crt` 的 CA 证书，也需要上传 `client.crt` 和 `client.key`，就像服务端一样，为了可以安装浏览器，还需要将 `client.crt` 转为 `client.p12` 便于安装。

## 自签名证书 mTLS 实战

找一台机器有 openssl 的 linux 机器模拟自签名

### 自签名根证书 CA

（1）创建根证书私钥：

```bash
openssl genrsa -out root.key 1024
```

（2）创建根证书请求文件：

```bash
openssl req -new -out root.csr -key root.key
```

后续参数请自行填写，下面是一个例子：

注意：CA 证书 `Common Name` 需要保证唯一性，不要与服务端证书或者客户端证书的 `Common Name` 相同。

```bash
Country Name (2 letter code) [XX]:cn
State or Province Name (full name) []:bj
Locality Name (eg, city) [Default City]:bj
Organization Name (eg, company) [Default Company Ltd]:alibaba
Organizational Unit Name (eg, section) []:test
Common Name (eg, your name or your servers hostname) []:root
Email Address []:a.alibaba.com
A challenge password []:
An optional company name []:
```

（3）创建根证书：

```jsx
openssl x509 -req -in root.csr -out root.crt -signkey root.key -CAcreateserial -days 3650
```

在创建证书请求文件的时候需要注意三点，下面生成服务器请求文件和客户端请求文件均要注意这三点： 根证书的 `Common Name` 填写 `root` 就可以，所有客户端和服务器端的证书这个字段需要填写域名，一定要注意的是，根证书的这个字段和客户端证书、服务器端证书不能一样； 其他所有字段的填写，根证书、服务器端证书、客户端证书需保持一致最后的密码可以直接回车跳过。

经过上面三个命令行，我们最终可以得到一个签名有效期为 10 年的根证书 `root.crt`，后面我们可以用这个根证书去颁发服务器证书和客户端证书。

### 自签名服务端证书

（1）生成服务器端证书私钥：

```bash
openssl genrsa -out server.key 1024
```

（2） 生成服务器证书请求文件，过程和注意事项参考根证书，本节不详述：

```bash
openssl req -new -out server.csr -key server.key
```

（3） 生成服务器端公钥证书

```bash
openssl x509 -req -in server.csr -out server.crt -signkey server.key -CA root.crt -CAkey root.key -CAcreateserial -days 3650
```

经过上面的三个命令，我们得到：

`server.key`：服务器端的密钥文件 `server.crt`：有效期十年的服务器端公钥证书，使用根证书和服务器端私钥文件一起生成

### 自签名客户端证书

（1）生成客户端证书密钥：

```bash
openssl genrsa -out client.key 1024
openssl genrsa -out client2.key 1024
```

（2） 生成客户端证书请求文件，过程和注意事项参考根证书，本节不详述：

```bash
openssl req -new -out client.csr -key client.key
openssl req -new -out client2.csr -key client2.key
```

（3） 生客户端证书

```bash
openssl x509 -req -in client.csr -out client.crt -signkey client.key -CA root.crt -CAkey root.key -CAcreateserial -days 3650
openssl x509 -req -in client2.csr -out client2.crt -signkey client2.key -CA root.crt -CAkey root.key -CAcreateserial -days 3650
```

（4） 生客户端p12格式证书，需要输入一个密码，选一个好记的，比如123456

```bash
openssl pkcs12 -export -clcerts -in client.crt -inkey client.key -out client.p12
openssl pkcs12 -export -clcerts -in client2.crt -inkey client2.key -out client2.p12
```

重复使用上面的命令，我们得到两套客户端证书：

- client.key / client2.key：客户端的私钥文件
- client.crt / client2.key：有效期十年的客户端证书

使用根证书和客户端私钥一起生成 `client.p12/client2.p12`，这个证书文件包含客户端的公钥和私钥，主要用来给浏览器访问使用

### 验证

使用 curl 加上证书路径，可以直接测试 Nginx 的 HTTPS 双向认证是否配置成功。下面我们测试三个用例：

- 使用 `client.crt / client.key` 这一套客户端证书来调用服务器端
- 使用 `client2.crt / client2.key` 这一套客户端证书来调用服务器端
- 不使用证书来调用服务器端

#### 带证书的成功调用

```bash
#--cert指定客户端公钥证书的路径
#--key指定客户端私钥文件的路径
#-k 使用本参数不校验证书的合法性，因为我们用的是自签名证书
#可以使用-v来观察具体的SSL握手过程
curl --cert ./client.crt --key ./client.key https://integration-fred2.fredhuang.com -k -v
* Rebuilt URL to: https://47.93.XX.XX/
*   Trying 47.93.XX.XX...
* TCP_NODELAY set
* Connected to 47.93.XX.XX (47.93.XX.XX) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* Cipher selection: ALL:!EXPORT:!EXPORT40:!EXPORT56:!aNULL:!LOW:!RC4:@STRENGTH
* successfully set certificate verify locations:
*   CAfile: /etc/ssl/cert.pem
  CApath: none
* TLSv1.2 (OUT), TLS handshake, Client hello (1):
* TLSv1.2 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS handshake, Server key exchange (12):
* TLSv1.2 (IN), TLS handshake, Request CERT (13):
* TLSv1.2 (IN), TLS handshake, Server finished (14):
* TLSv1.2 (OUT), TLS handshake, Certificate (11):
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
* TLSv1.2 (OUT), TLS handshake, CERT verify (15):
* TLSv1.2 (OUT), TLS change cipher, Client hello (1):
* TLSv1.2 (OUT), TLS handshake, Finished (20):
* TLSv1.2 (IN), TLS change cipher, Client hello (1):
* TLSv1.2 (IN), TLS handshake, Finished (20):
* SSL connection using TLSv1.2 / ECDHE-RSA-AES256-GCM-SHA384
* ALPN, server accepted to use http/1.1
* Server certificate:
*  subject: C=CN; ST=BJ; L=BJ; O=Alibaba; OU=Test; CN=integration-fred2.fredhuang.com; emailAddress=a@alibaba.com
*  start date: Nov  2 01:01:34 2019 GMT
*  expire date: Oct 30 01:01:34 2029 GMT
*  issuer: C=CN; ST=BJ; L=BJ; O=Alibaba; OU=Test; CN=root; emailAddress=a@alibaba.com
*  SSL certificate verify result: unable to get local issuer certificate (20), continuing anyway.
> GET / HTTP/1.1
> host:integration-fred2.fredhuang.com
> User-Agent: curl/7.54.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Server: nginx/1.17.5
< Date: Sat, 02 Nov 2019 02:39:43 GMT
< Content-Type: text/html
< Content-Length: 612
< Last-Modified: Wed, 30 Oct 2019 11:29:45 GMT
< Connection: keep-alive
< ETag: "5db97429-264"
< Accept-Ranges: bytes
<
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>
<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>
<p><em>Thank you for using nginx.</em></p>
</body>
</html>
* Connection #0 to host 47.93.XX.XX left intact
```

使用 `client2.crt / client2.key` 这一套客户端证书来调用服务器端

```bash
curl --cert ./client2.crt --key ./client2.key https://integration-fred2.fredhuang.com -k
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>
<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>
<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

#### 不带证书的调用

```bash
curl https://integration-fred2.fredhuang.com -k
<html>
<head><title>400 No required SSL certificate was sent</title></head>
<body>
<center><h1>400 Bad Request</h1></center>
<center>No required SSL certificate was sent</center>
<hr><center>nginx/1.17.5</center>
</body>
</html>
```

三个用例都符合预期，从第一个测试日志中可以看到，整个通信过程较长，客户端验证服务器端的证书，客户端也将自己的证书上传到服务器端进行验证。使用根证书颁发的两个客户端证书都可以正常发起双向HTTPS认证的调用。没有带客户端证书的调用会被服务器端拒绝服务。

## Reference

[什么是HTTPS双向认证(Mutual TLS authentication)_API 网关(API Gateway)-阿里云帮助中心](https://help.aliyun.com/zh/api-gateway/user-guide/mutual-tls-authentication)

[使用CLB部署HTTPS业务（双向认证）_负载均衡(SLB)-阿里云帮助中心](https://help.aliyun.com/zh/slb/classic-load-balancer/use-cases/configure-mutual-authentication-on-an-https-listener-1)