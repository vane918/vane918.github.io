---
layout: post
title:  "安卓PMS模块之APK签名及验证流程"
date:   2020-03-29 00:00:00
catalog:  true
tags:
    - PMS
    - Android

---



## 1. APK签名基础知识

在进行通信时，必须至少解决两个问题：一是确保消息来源的真实性，二是确保消息不会被第三方篡改。在安装Apk时，同样需要确保Apk来源的真实性以及Apk没有被第三方篡改。如何解决这两个问题呢？方法就是开发者对Apk进行签名：在Apk中写入一个具有唯一性的“指纹”。指纹写入以后，Apk中有任何修改，都会导致这个指纹无效，Android系统在安装Apk进行签名校验时就会不通过，从而保证了安全性。 
在讲解Apk签名前，先简单讲解下签名涉及到的概念。

### 1.1 非对称加密算法

非对称加密算法需要两个密钥：公开密钥（简称公钥）和私有密钥（简称私钥）。公钥与私钥是一对，如果用公钥对数据进行加密，只有用对应的私钥才能解密；如果用私钥对数据进行加密，那么只有用对应的公钥才能解密。因为加密和解密使用的是两个不同的密钥，所以这种算法叫作非对称加密算法。
非对称加密算法是数字签名和数字证书的基础，我们熟悉的apk签名文件RSA就是非对称加密算法的一种实现。

### 1.2 消息摘要算法

消息摘要算法（Message Digest Algorithm） 是将任意长度的消息变成固定长度的短消息，它类似于一个自变量是消息的函数，也就是Hash函数。数字摘要就是采用单向Hash函数将需要加密的明文“摘要”成一串固定长度的密文，这一串密文又称为数字指纹，它有固定的长度，而且不同的明文摘要成密文，其结果总是不同的，而同样的明文其摘要必定一致。著名的摘要算法有RSA公司的MD5算法和SHA-1算法及其大量的变体。
消息摘要具有下面的几个特征：

1. 唯一性

   在不考虑碰撞的情况下，不同的数据的计算出的摘要是不同的。

2. 固定长度

   不同的Hash算法计算的长度是不一样的，但对同一个算法来说是一样的。比较常用的Hash算法  有MD5和SHA1，MD5的长度是128拉，SHA1的长度是160位。

3. 不可逆性

   即从正向计算的摘要不可能逆向推导出原始数据。

### 1.3 数字签名和数字证书

所谓数字签名，就是为了解决在通信时要确保消息来源的真实性和确保消息不会被第三方篡改这两个问题而产生的，它是对前面提到的非对称加密技术与数字摘要技术的一个具体的应用。
对于消息的发送者来说，先要生成一对公私钥对，将公钥给消息的接收者。
如果消息的发送者有一天想给消息接收者发消息，在发送的信息中，除了要包含原始的消息外，还要加上另外一段消息。这段消息通过如下两步生成：

1）对要发送的原始消息提取消息摘要；

2）对提取的信息摘要用自己的私钥加密。

通过这两步得出的消息，就是所谓的原始信息的数字签名。
而对于信息的接收者来说，他所收到的信息，将包含两个部分，一是原始的消息内容，二是附加的那段数字签名。他将通过以下三步来验证消息的真伪：

1）对原始消息部分提取消息摘要，注意这里使用的消息摘要算法要和发送方使用的一致；

2）对附加上的那段数字签名，使用预先得到的公钥解密；

3）比较前两步所得到的两段消息是否一致。如果一致，则表明消息确实是期望的发送者发的，且内容没有被篡改过；相反，如果不一致，则表明传送的过程中一定出了问题，消息不可信。

通过这种所谓的数字签名技术，确实可以有效解决可靠通信的问题。如果原始消息在传送的过程中被篡改了，那么在消息接收者那里，对被篡改的消息提取的摘要肯定和原始的不一样。并且，由于篡改者没有消息发送方的私钥，即使他可以重新算出被篡改消息的摘要，也不能伪造出数字签名。
所以，综上所述，数字签名其实就是只有信息的发送者才能产生的别人无法伪造的一段数字串，这段数字串同时也是对信息的发送者发送信息真实性的一个有效证明。
前面讲的这种数字签名方法，有一个前提，就是消息的接收者必须要事先得到正确的公钥。如果一开始公钥就被别人篡改了，那坏人就会被你当成好人，而真正的消息发送者给你发的消息会被你视作无效的。而且，很多时候根本就不具备事先沟通公钥的信息通道。那么如何保证公钥的安全可信呢？这就要靠数字证书来解决了。
所谓数字证书，一般包含以下一些内容：

- 证书的发布机构（Issuer）
- 证书的有效期（Validity）
- 消息发送方的公钥
- 证书所有者（Subject）
- 数字签名所使用的算法
- 数字签名

可以看出，数字证书其实也用到了数字签名技术。只不过要签名的内容是消息发送方的公钥，以及一些其它信息。但与普通数字签名不同的是，数字证书中签名者不是随随便便一个普通的机构，而是要有一定公信力的机构，即数字证书是身份认证机构（Certificate Authority）颁发的 。一般来说，这些有公信力机构的根证书已经在设备出厂前预先安装到设备上了。所以，数字证书可以保证数字证书里的公钥确实是这个证书的所有者的，或者证书可以用来确认对方的身份。数字证书主要是用来解决公钥的安全发放问题。
需要注意的是，Apk的证书通常的自签名的，也就是由开发者自己制作，没有向CA机构申请。Android在安装Apk时并没有校验证书本身的合法性，只是从证书中提取公钥和加密算法，这也正是对第三方Apk重新签名后，还能够继续在没有安装这个Apk的系统中继续安装的原因。
完整的签名和校验过程如下：（图片来源：[维基百科](https://upload.wikimedia.org/wikipedia/commons/2/2b/Digital_Signature_diagram.svg)） 

![sign-verify](/images/pms/apk-sign/sign-verify.png)

### 1.4 ZIP文件结构

apk也是一个zip文件，zip文件的结构如下：

![zip](/images/pms/apk-sign/zip.png)

zip文件分为3部分：

1. 数据区
   此区块包含了zip中所有文件的记录，是一个列表，每条记录包含：文件名、压缩前后size、压缩后的数据等；
2. 中央目录
   存放目录信息，也是一个列表，每条记录包含：文件名、压缩前后size、本地文件头的起始偏移量等。通过本地文件头的起始偏移量即可找到压缩后的数据；
3. 中央目录结尾记录
   标识中央目录结尾，包含：中央目录条目数、size、起始偏移量、zip文件注释内容等。
   通过中央目录起始偏移量和size即可定位到中央目录，再遍历中央目录条目，根据本地文件头的起始偏移量即可在数据区中找到相应的压缩数据。

### 1.5 证书格式

Apk签名时并没有直接指定私钥、公钥和数字证书，而是使用keystore文件，这些信息都包含在了keystore文件中。根据编码不同，keystore文件分为很多种，Android使用的是Java标准keystore格式JKS(Java Key Storage)，所以通过Android Studio导出的keystore文件是以.jks结尾的。
keystore使用的证书标准是X.509，X.509标准也有多种编码格式，常用的有两种：pem（Privacy Enhanced Mail）和der（Distinguished Encoding Rules）。jks使用的是der格式，Android也支持直接使用pem格式的证书进行签名，Android系统源码中对系统apk进行签名的正是采用pem格式的证书：platform.x509.pem。
两种证书编码格式的区别：

- DER（Distinguished Encoding Rules）

  二进制格式，所有类型的证书和私钥都可以存储为der格式。

- PEM（Privacy Enhanced Mail）

  base64编码，内容以—–BEGIN xxx—– 开头，以—–END xxx—– 结尾，比如platform.x509.pem：

  ```
  -----BEGIN CERTIFICATE-----
  MIIDzzCCAregAwIBAgIJALtcDKQRTtIcMA0GCSqGSIb3DQEBBQUAMH4xCzAJBgNV
  BAYTAkNOMRIwEAYDVQQIDAlHdWFuZ0RvbmcxETAPBgNVBAcMCFNoZW5aaGVuMQww
  CgYDVQQKDANKU1IxDDAKBgNVBAsMA0pTUjEMMAoGA1UEAwwDSlNSMR4wHAYJKoZI
  hvcNAQkBFg9pbm5vc0Bpbm5vcy5jb20wHhcNMTYwNDE5MTExMDA4WhcNNDMwOTA1
  MTExMDA4WjB+MQswCQYDVQQGEwJDTjESMBAGA1UECAwJR3VhbmdEb25nMREwDwYD
  VQQHDAhTaGVuWmhlbjEMMAoGA1UECgwDSlNSMQwwCgYDVQQLDANKU1IxDDAKBgNV
  BAMMA0pTUjEeMBwGCSqGSIb3DQEJARYPaW5ub3NAaW5ub3MuY29tMIIBIjANBgkq
  hkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAwYZYnD7S/ZxsNPecVoeq64/a+FqonrX5
  H7I71pLhTZFp0QmNd8+Y8x14robXCukoVgTTvxBEho1zDHzPJTXWa4/r4m0oRapD
  BX5N3uEqzsN5yE6SOBkhBggVFdqb/UuyF0orLvpEUVULgjeqOChxbrXxVKb5K2wC
  9GKbto36+DcR3wEziZTmjQBQEkaFV8sDN6AhVOYDObuFtNcT5//+AzqPzELPOtzT
  tftDo9xhV2FbiLSlZm2eD8DZ4v0J1HIQIOObcxt/vfXjGA4mS9/1RvNoAQRBn5MI
  LiLDaGVTvqWXf6VtoovUyb2WL5HYRWMrd3lSvNVPk1ko+87yTwlraQIDAQABo1Aw
  TjAdBgNVHQ4EFgQUeg7kI7METfsf7vjcJ5Wcs2lI24kwHwYDVR0jBBgwFoAUeg7k
  I7METfsf7vjcJ5Wcs2lI24kwDAYDVR0TBAUwAwEB/zANBgkqhkiG9w0BAQUFAAOC
  AQEAOmjTRkS2vBcJtadN6TWGdQSReXXgfA88l120Jms9BBVIgrRS0igKLXpQZhEN
  UVSTJFpEmNdS6GYEjg02kqwdaca3/8+Fb86u8ucqXowO+G2w5wtti/F5isxZ+kvh
  A8/c/PB/vQWZAG3gGvaK/LjB1d4T30EpGkIXUWzlrLCx61opKI3fpz/0ogm++0zU
  4Sh7KEHiqP57PD2AshXFFsLyoFLbbtazUZcMLUiKIfms5o3I78d4QkGzF7Rsw3ON
  9jyqmrgdN8xRuSBP43pGUz+LjlNY6kaAY/ZKC9JaUk+zceRsMVG1sEgfpUkUpxCk
  h3UkfQ5g2dUPEfB6o80F2uarkA==
  -----END CERTIFICATE-----
  ```

  使用下面的指令解析platform.x509.pem得到以下信息：

![platform.x509](/images/pms/apk-sign/platform.x509.png)

### 1.6 V1和V2签名

在Android Studio中点击菜单 Build->Generate signed apk打包签名过程中， 可以看到两种签名选项 V1(Jar Signature)  V2(Full APK Signature)。 从Android 7.0开始，谷歌增加新签名方案 V2 Scheme (APK Signature) ， 但Android 7.0以下版本，只能用旧签名方案 V1 scheme (JAR signing) 。

**V1签名:**

- 对zip压缩包的每个文件进行验证, 签名后还能对压缩包修改(移动/重新压缩文件)

- 对V1签名的apk/jar解压,在META-INF存放签名文件(MANIFEST.MF, CERT.SF, CERT.RSA)

- 其中MANIFEST.MF文件保存所有文件的SHA1指纹(除了META-INF文件)，由此可知: V1签名是对压缩包中单个文件签名验证

 **V2签名:** 

- 对zip压缩包的整个文件验证, 签名后不能修改压缩包(包括zipalign)

- 对V2签名的apk解压,没有发现签名文件,重新压缩后V2签名就失效, 由此可知: V2签名是对整个APK签名验证

**V2签名优点:**

- 签名更安全(不能修改压缩包)

- 签名验证时间更短(不需要解压验证),因而安装速度加快

### 1.7 APK签名工具

jarsigner是JDK提供的针对jar包签名的通用工具， 位于JDK/bin/jarsigner.exe。apksigner是Google官方提供的针对Android apk签名及验证的专用工具，位于Android SDK/build-tools/SDK版本/apksigner.bat。源码在build/tools/signapk。 不管是apk包还是jar包，本质都是zip格式的压缩包，所以它们的签名过程都差不多。

**jarsigner与apksigner的区别：**

- jarsigner只能对apk进行V1签名，apksigner可以对apk进行V1和V2签名

- jarsigner使用keystore文件进行签名，apksigner除了支持使用keystore文件进行签名外，还支持直接指定pem证书文件和私钥进行签名

### 1.8 签名相关命令

**jarsigner签名**

jarsigner签名只支持V1签名。

`jarsigner -keystore keystore_file -signedjar signed.apk unsigned.apk alias_name -storepass pwd`

keystore_file是签名证书，signed.apk是签名后的apk，unsigned.apk是待签名的apk，alias_name是签名证书的alias属性，用来区分不同证书的，pwd是签名证书的密码。

如：

`jarsigner -keystore test.jks -signedjar Test-sign.apk Test-unsigned.apk KEY0 -storepass 123456`

**使用 `apksigner` 工具为 APK 签名** 

apksigner 签名默认同时使用V1和V2签名。

`apksigner sign --ks 密钥库名 --in app.apk --out app-signed.apk`

 **若密钥库中有多个密钥对,则必须指定密钥别名**

`apksigner sign --ks 密钥库名 --ks-key-alias 密钥别名 --in app.apk --out app-signed.apk`

**使用两个签名对APK进行签名**

`apksigner sign --ks 密钥库名1 --ks-key-alias 密钥别名1 --next-signer --ks 密钥库名2 --ks-key-alias 密钥别名2 --in app.apk --out app-signed.apk`

**如果想禁用V2签名** 

`apksigner sign --v2-signing-enabled false --ks 密钥库名 --in app.apk --out app-signed.apk`

`apksigner sign --key release.pk8 --cert release.x509.pem --in app-unsign.apk --out app-signed.apk`

使用 `apksigner` 工具为 APK 签名时，必须提供签名者的私钥和证书。可以通过两种不同的方式添加此信息：

- 使用 `--ks` 选项指定密钥库文件。
- 使用 `--key` 和 `--cert` 选项分别指定私钥文件和证书文件。私钥文件必须使用 PKCS #8 格式，证书文件必须使用 X.509 格式。

 通常情况下，只需要使用一个签名者为 APK 签名即可。如果需要使用多个签名者为 APK 签名，使用 `--next-signer` 选项： 

`apksigner sign --ks release.jks --next-signer --ks magic.jks --in  app-unsign.apk --out app-signed.apk`

**查看keystore文件**
`keytool -list  -v -keystore keystore_file -storepass pwd`

```
keytool -list  -v -keystore test.jks -storepass 123456

密钥库类型: JKS
密钥库提供方: SUN

您的密钥库包含 1 个条目

别名: key0
创建日期: 2020-3-26
条目类型: PrivateKeyEntry
证书链长度: 1
证书[1]:
所有者: CN=xxxx, OU=xxxx, O=xxxx
发布者: CN=xxxx, OU=xxxx, O=xxxx
序列号: 70b2e38e
有效期开始日期: Thu Mar 26 11:16:12 CST 2020, 截止日期: Mon Mar 20 11:16:12 CST 2045
证书指纹:
         MD5: DA:D3:ED:21:FF:6D:8D:64:CA:B6:5E:C8:C6:BD:12:1C
         SHA1: 03:4C:6F:F6:8A:02:25:0F:DB:E5:00:66:3E:5E:59:12:CE:C6:B3:63
         SHA256: 72:52:D0:BD:B8:64:87:F2:0E:CB:04:2F:16:DE:91:8E:A5:A9:9E:D7:1F:F8:10:70:67:7F:18:0D:2A:46:4C:D2
         签名算法名称: SHA256withRSA
         版本: 3
```

**查看apk证书**
`keytool -printcert -jarfile apk`

得到的信息与查看keystore文件指令一样。

**查看DER格式证书(META-INF/CERT.RSA)**
`openssl pkcs7 -inform DER -in CERT.RSA -noout -print_certs -print -text`

该指令会列出详细的参数，如Public-Key和Signature。

**查看PEM格式证书**
`openssl x509 -in cert.x509.pem -text -noout`

**apksigner检查apk是否签名，以及查看证书SHA值**
`apksigner verify -v --print-certs apk`

```
λ apksigner verify -v --print-certs app-release.apk
Verifies
Verified using v1 scheme (JAR signing): true
Verified using v2 scheme (APK Signature Scheme v2): true
Number of signers: 1
Signer #1 certificate DN: CN=vane, OU=vane
Signer #1 certificate SHA-256 digest: 9c120bd076437911258aed99469b5383f11d4e8c63cbbd05355b81c65ca77a2d
Signer #1 certificate SHA-1 digest: 60ea35b6d774cd6fac650a3982132cbec0ca467e
Signer #1 certificate MD5 digest: 3f0d490aff93a8526733a05cd3c8d6b0
Signer #1 key algorithm: RSA
Signer #1 key size (bits): 2048
Signer #1 public key SHA-256 digest: 02e998d3d4ce4f7dbb8b2ff75dc480e8e9bf5a3ff34393263bf3e89b5bd2b9d9
Signer #1 public key SHA-1 digest: 5ccaa3929f2d14757abc260b9725ee7064ce73f5
Signer #1 public key MD5 digest: a3da2f89fb1ea07223cc00664079daa4
```

## 2. APK签名方案

Android 的签名方案，发展到现在，不是一蹴而就的。Android 现在已经支持三种应用签名方案：

- v1 方案：基于 JAR 签名。
- v2 方案：APK 签名方案 v2，在 Android 7.0 引入。
- v3 方案：APK 签名方案v3，在 Android 9.0 引入。

v1 到 v2 是颠覆性的，为了解决 JAR 签名方案的安全性问题，而到了 v3 方案，其实结构上并没有太大的调整，可以理解为 v2 签名方案的升级版，有一些资料也把它称之为 v2+ 方案。因为这种签名方案的升级，就是向下兼容的，所以只要使用得当，这个过程对开发者是透明的。

### 2.1 APK 签名方案 V1

经过 apksigner或jarsigner工具签名的apk，解压apk后在根目录多了一个META-INF文件夹，文件夹里面有MANIFEST.MF、xxx.SF和xxx.RSA三个文件。分别看下这三个文件的详细内容和作用。

#### 2.1.1 MANIFEST.MF

```xml
Manifest-Version: 1.0
Built-By: Generated-by-ADT
Created-By: Android Gradle 3.5.2

Name: AndroidManifest.xml
SHA1-Digest: VDUFjzpBsRw+GPjmT9sNN9MAt5k=
Name: res/anim/abc_fade_in.xml
SHA1-Digest: L0IgyYapku8uZtPqOfviagHTIEk=
...
```

 该文件中保存的内容其实就是逐一遍历 APK 中的所有条目，如果是目录就跳过，如果是一个文件，就用 SHA1（或者 SHA256）消息摘要算法提取出该文件的摘要然后进行 BASE64 编码后，作为「SHA1-Digest」属性的值写入到 MANIFEST.MF 文件中的一个块中。该块有一个「Name」属性， 其值就是该文件在 APK 包中的路径。 

注意：如果重复签名，不会对MANIFEST.MF、xxx.SF和xxx.RSA这三个文件进行摘要。

#### 2.1.2 xxx.SF

采用pk8、x509.pem 文件进行签名后生成的签名文件统一使用“CERT”命名， 使用 keystore 文件进行签名。生成的签名文件默认使用 keystore 的别名命名。 

```xml
Signature-Version: 1.0
Created-By: 1.0 (Android SignApk)
SHA1-Digest-Manifest: UqrnpIV7xJRwGslI3Fo+m9rOT6I=
X-Android-APK-Signed: 2

Name: AndroidManifest.xml
SHA1-Digest: oq+mJ4ZRKQlxJ2d2robOvNEvmQs=
Name: res/anim/abc_fade_in.xml
SHA1-Digest: XXaSktKiWqcfhbVCAylwxGU9L2s=
...
```

- SHA1-Digest-Manifest：对整个 MANIFEST.MF 文件做 SHA1（或者 SHA256）后再用 Base64 编码
- SHA1-Digest：对 MANIFEST.MF 的各个条目做 SHA1（或者 SHA256）后再用 Base64 编码

#### 2.1.3 xxx.RSA

 这里会把之前生成的 xxx.SF 文件，用私钥计算出签名, 然后将签名以及包含公钥信息的数字证书一同写入 xxx.RSA 中保存。这里要注意的是，Android APK 中的 CERT.RSA 证书是自签名的，并不需要这个证书是第三方权威机构发布或者认证的，用户可以在本地机器自行生成这个自签名证书，如pk8、x509.pem 文件和 keystore 文件。

采用openssl pkcs7 -inform DER -in CERT.RSA -noout -print_certs -print -text查看：

![RSA-1](/images/pms/apk-sign/RSA-1.PNG)

![RSA-2](/images/pms/apk-sign/RSA-2.PNG)

包含了公钥 public_key、签名signature和摘要加密enc_digest等信息。

其中的enc_digest加密摘要就是对xxx.SF文件进行摘要后再用私钥进行加密的值，等到apk安装的时候进行签名验证，就会用xxx.RSA文件中的公钥对enc_digest进行解密，取出xxx.RSA文件摘要，与计算得到的xxx.RSA文件摘要相比较，如果不相同，意味着xxx.RSA文件被篡改了。

#### 2.1.4 APK 签名方案 V1总结

签名过程会生成三个文件，都在META-INF目录下

- MANIFEST.MF：对apk中所有内容的摘要记录
- CERT.SF：对MANIFEST.MF整个内容的摘要记录，以及对MANIFEST.MF中每个内容摘要的二次摘要记录
- CERT.RSA：对CERT.SF内容进行私钥加密生成数字签名，并连同用户公钥证书等信息生成的加密文件

apk签名流程概括如下：

1. 将apk除了META-INF/之外的所有文件，计算摘要生成MANIFEST.MF文件
2. 将MANIFEST.MF文件整个做一次摘要计算，并把MANIFEST.MF内每个文件的摘要再做一次摘要计算，生成CERT.SF文件
3. 将CERT.SF文件用开发者私钥加密生成数字签名，连同开发者公钥证书一同，加密生成CERT.RSA签名文件。

### 2.2 APK 签名方案 V2

 APK 签名方案 v2 是一种全文件签名方案，该方案能够发现对 APK 的受保护部分进行的所有更改，从而有助于加快验证速度并增强完整性保证。 

#### 2.2.1 V2 的改进

从 Android 7.0 开始，Android 支持了一套全新的 V2 签名机制，为什么要推出新的签名机制呢？ v1 签名有两个地方可以改进：

- 签名校验速度慢

​       校验过程中需要对apk中所有文件进行摘要计算，在 APK 资源很多、性能较差的机器上签名校验会花费较长时 
​       间，导致安装速度慢。

- 完整性保障不够

​       META-INF 目录用来存放签名，自然此目录本身是不计入签名校验过程的，可以随意在这个目录中添加文件，
​       比如一些快速批量打包方案就选择在这个目录中添加渠道文件。

为了解决这两个问题，在 Android 7.0 Nougat 中引入了全新的 APK Signature Scheme v2。

由于在 v1 仅针对单个 ZIP 条目进行验证，因此，在apk 签名后可进行许多修改：可以移动甚至重新压缩文件。事实上，编译过程中要用到的 ZIPalign 工具就是这么做的，它用于根据正确的字节限制调整 ZIP 条目，以改进运行时性能。而且我们也可以利用这个东西，在打包之后修改 META-INF 目录下面的内容，或者修改 ZIP 的注释来实现多渠道的打包，在 v1 签名中都可以校验通过。
v2 签名将验证apk中的所有字节，而不是单个 ZIP 条目，因此，在签名后无法再运行 ZIPalign（必须在签名之前执行）。

#### 2.2.2 V2 签名的原理

 V2方案为加强数据完整性保证，不在数据区和中央目录中插入数据，选择在 数据区和中央目录 之间插入一个APK签名分块，存储了签名，摘要，签名算法，证书链，额外属性等信息，从而保证了原始zip（apk）数据的完整性。具体如下所示： 

![v2](/images/pms/apk-sign/v2.png)

为了保护 APK 内容，整个 apk（ZIP 文件格式）被分为以下 4 个区块：

- ZIP 条目的内容（从偏移量 0 处开始一直到“apk签名分块”的起始位置）
- apk签名分块
- ZIP 中央目录
- ZIP 中央目录结尾

其中，应用签名方案的签名信息会被保存在 区块 2 (APK Signing Block)中， 区块 2负责保护第 1、3、4 部分的完整性，以及第 2 部分包含的`APK 签名方案 v2分块`中的`signed data`分块的完整性。 

在解析 APK 时，首先要通过以下方法找到“ZIP 中央目录”的起始位置：在文件末尾找到“ZIP 中央目录结尾”记录，然后从该记录中读取“中央目录”的起始偏移量。通过 `magic` 值，可以快速确定“中央目录”前方可能是“APK 签名分块”。然后，通过 `size of block` 值，可以高效地找到该分块在文件中的起始位置。 

#### 2.2.3 APK 签名分块

为了保持与 v1 APK 格式向后兼容，v2 及更高版本的 APK 签名会存储在“APK 签名分块”内，该分块是为了支持 APK 签名方案 v2 而引入的一个新容器。在 APK 文件中，“APK 签名分块”位于“ZIP 中央目录”（位于文件末尾）之前并紧邻该部分。

该分块包含多个“ID-值”对，所采用的封装方式有助于更轻松地在 APK 中找到该分块。APK 的 v2 签名会存储为一个“ID-值”对，其中 ID 为 0x7109871a。

**格式**

“APK 签名分块”的格式如下（所有数字字段均采用小端字节序）：

- `size of block`，以字节数（不含此字段）计 (uint64)
- 带 uint64 长度前缀的“ID-值”对序列：
  - `ID` (uint32)
  - `value`（可变长度：“ID-值”对的长度 - 4 个字节）
- `size of block`，以字节数计 - 与第一个字段相同 (uint64)
- `magic`“APK 签名分块 42”（16 个字节）

![find-v2](/images/pms/apk-sign/find-v2.png)

在解析 APK 时，首先要通过以下方法找到“ZIP 中央目录”的起始位置：在文件末尾找到“ZIP 中央目录结尾”记录，然后从该记录中读取“中央目录”的起始偏移量。通过 `magic` 值，可以快速确定“中央目录”前方可能是“APK 签名分块”。然后，通过 `size of block` 值，可以高效地找到该分块在文件中的起始位置。

在解译该分块时，应忽略 ID 未知的“ID-值”对。

#### 2.2.4 APK 签名方案 v2 分块

APK 由一个或多个签名者/身份签名，每个签名者/身份均由一个签名密钥来表示。该信息会以“APK 签名方案 v2 分块”的形式存储。对于每个签名者，都会存储以下信息：

- （签名算法、摘要、签名）元组。摘要会存储起来，以便将签名验证和 APK 内容完整性检查拆开进行。
- 表示签名者身份的 X.509 证书链。
- 采用键值对形式的其他属性。

对于每位签名者，都会使用收到的列表中支持的签名来验证 APK。签名算法未知的签名会被忽略。如果遇到多个支持的签名，则由每个实现来选择使用哪个签名。这样一来，以后便能够以向后兼容的方式引入安全系数更高的签名方法。建议的方法是验证安全系数最高的签名。

**格式**

APK 签名方案 v2分块是一个签名序列，说明可以使用多个签名者对同一个APK进行签名。每个签名信息中均包含了三个部分的内容：

- 带长度前缀的signed data


​       其中包含了通过一系列算法计算的摘要列表、证书信息，以及extra信息（可选）；

- 带长度前缀的signatures序列

​       通过一系列算法对signed data的签名列表。签名时使用了多个签名算法，在签名校验时会是选择系统支持的
​       安全系数最高的签名进行校验；

- 证书公钥

![V2-sign](/images/pms/apk-sign/V2-sign.png)

“APK 签名方案 v2 分块”存储在“APK 签名分块”内，ID 为 `0x7109871a`。

**签名算法 ID**

- 0x0101 - 采用 SHA2-256 摘要、SHA2-256 MGF1、32 个字节的盐且尾部为 0xbc 的 RSASSA-PSS 算法
- 0x0102 - 采用 SHA2-512 摘要、SHA2-512 MGF1、64 个字节的盐且尾部为 0xbc 的 RSASSA-PSS 算法
- 0x0103 - 采用 SHA2-256 摘要的 RSASSA-PKCS1-v1_5 算法。此算法适用于需要确定性签名的构建系统。
- 0x0104 - 采用 SHA2-512 摘要的 RSASSA-PKCS1-v1_5 算法。此算法适用于需要确定性签名的构建系统。
- 0x0201 - 采用 SHA2-256 摘要的 ECDSA 算法
- 0x0202 - 采用 SHA2-512 摘要的 ECDSA 算法
- 0x0301 - 采用 SHA2-256 摘要的 DSA 算法

Android 平台支持上述所有签名算法。签名工具可能只支持其中一部分算法。

**支持的密钥大小和 EC 曲线：**

- RSA：1024、2048、4096、8192、16384
- EC：NIST P-256、P-384、P-521
- DSA：1024、2048、3072

#### 2.2.5 受完整性保护的内容

为了保护 APK 内容，APK 包含以下 4 个部分：

1. ZIP 条目的内容（从偏移量 0 处开始一直到“APK 签名分块”的起始位置）
2. APK 签名分块
3. ZIP 中央目录
4. ZIP 中央目录结尾

![v2](/images/pms/apk-sign/v2.png)

APK 签名方案 v2 负责保护第 1、3、4 部分的完整性，以及第 2 部分包含的“APK 签名方案 v2 分块”中的 `signed data` 分块的完整性。

第 1、3 和 4 部分的完整性通过其内容的一个或多个摘要来保护，这些摘要存储在 `signed data` 分块中，而这些分块则通过一个或多个签名来保护。

第 1、3 和 4 部分的摘要采用以下计算方式，类似于两级 [Merkle 树](https://en.wikipedia.org/wiki/Merkle_tree)。每个部分都会被拆分成多个大小为 1 MB（220 个字节）的连续块。每个部分的最后一个块可能会短一些。每个块的摘要均通过字节 `0xa5` 的串联、块的长度（采用小端字节序的 uint32 值，以字节数计）和块的内容进行计算。顶级摘要通过字节 `0x5a` 的串联、块数（采用小端字节序的 uint32 值）以及块的摘要的连接（按照块在 APK 中显示的顺序）进行计算。摘要以分块方式计算，以便通过并行处理来加快计算速度。摘要流程如下：

1. 将第1、3、4部分拆分为多个块(chunk)
   将每个部分拆分成多个大小为 1 MB大小的chunk，最后一个chunk可能小于1M。之所以分块，是为了可以通过并行计算摘要以加快计算速度；
2. 分别计算块(chunk)摘要
   字节 0xa5 + 块的长度（字节数） + 块的内容 进行计算；
3. 计算整体摘要
   字节 0x5a + chunk数 + 块的摘要的连接（按块在 APK 中的顺序）进行计算。
   这里要注意的是：中央目录结尾记录中包含了中央目录的起始偏移量，插入APK签名分块后，中央目录的起始偏移量将发生变化。故在校验签名计算摘要时，需要把中央目录的起始偏移量当作APK签名分块的起始偏量。

![v2-digest](/images/pms/apk-sign/v2-digest.png)

#### 2.2.6 防回滚保护

攻击者可能会试图在支持对带 v2 签名的 APK 进行验证的 Android 平台上将带 v2 签名的 APK 作为带 v1 签名的 APK 进行验证。为了防范此类攻击，带 v2 签名的 APK 如果还带 v1 签名，其 META-INF/.SF 文件的主要部分中必须包含 X-Android-APK-Signed 属性。该属性的值是一组以逗号分隔的 APK 签名方案 ID（v2 方案的 ID 为 2）。在验证 v1 签名时，对于此组中验证程序首选的 APK 签名方案（例如，v2 方案），如果 APK 没有相应的签名，APK 验证程序必须要拒绝这些 APK。此项保护依赖于内容 META-INF/.SF 文件受 v1 签名保护这一事实。请参阅 [JAR 已签名的 APK 的验证](https://source.android.google.cn/security/apksigning/v2?hl=zh-cn#v1-verification)部分。

攻击者可能会试图从“APK 签名方案 v2 分块”中删除安全系数较高的签名。为了防范此类攻击，对 APK 进行签名时使用的签名算法 ID 的列表会存储在通过各个签名保护的 `signed data` 分块中。

### 2.3 APK 签名方案总结

apk签名是为了确认apk开发者身份和防止内容的篡改，设计了一套apk签名的方案保证apk的安全性，即在打包时由开发者进行apk的签名，在安装apk时Android系统会有相应的开发者身份和内容正确性的验证，只有验证通过才可以安装apk，签名过程和验证的设计就是基于非对称加密的思想。 
Android在7.0以前使用的一套签名方案：在apk根目录下的META-INF/文件夹下生成签名文件，然后在安装时在系统的PackageManagerService里进行签名文件的验证。
从7.0开始，Android提供了新的V2签名方案：利用apk(zip)压缩文件的格式，在几个原始内容区之外增加了一块用于存放签名信息的数据区，然后同样在安装时在系统的PackageManagerService里进行V2版本的签名验证，V2方案会更安全、使校验更快安装更快。

## 3. APK安装时签名验证

google为Android系统设计了一套apk安装时验证apk签名的方案，目的是保证apk的安全性，避免被不怀好意的攻击者恶意篡改。接下来分别分析apk的两种签名方案在apk安装时是如何验证的。

### 3.1 APK签名方案V1验证流程

已签名的apk 是一种[标准的已签名 JAR](https://docs.oracle.com/javase/8/docs/technotes/guides/jar/jar.html#Signed_JAR_File) 。其中包含的条目必须与 META-INF/MANIFEST.MF 中列出的条目完全相同，并且所有条目都必须已由同一组签名者签名。其完整性按照以下方式进行验证：

1. 每个签名者均由一个包含 META-INF/<signer>.SF 和 META-INF/<signer>.(RSA|DSA|EC) 的 JAR 条目来表示。
2. <signer>.(RSA|DSA|EC) 是[具有 SignedData 结构的 PKCS #7 CMS ContentInfo](https://tools.ietf.org/html/rfc5652)，其签名通过 <signer>.SF 进行验证。
3. <signer>.SF 文件包含 META-INF/MANIFEST.MF 的全文件摘要和 META-INF/MANIFEST.MF 各个部分的摘要。需要验证 MANIFEST.MF 的全文件摘要。如果该验证失败，则改为验证 MANIFEST.MF 各个部分的摘要。
4. 对于每个受完整性保护的 JAR 条目，META-INF/MANIFEST.MF 都包含一个具有相应名称的部分，其中包含相应条目未压缩内容的摘要。所有这些摘要都需要验证。
5. 如果 APK 包含未在 MANIFEST.MF 中列出且不属于 JAR 签名一部分的 JAR 条目，则 APK 验证将会失败。

因此，保护链是每个受完整性保护的 JAR 条目的 <signer>.(RSA|DSA|EC) -> <signer>.SF -> MANIFEST.MF -> 内容。

接下来结合源码分析apk V1签名验证的流程。

安装apk时验证签名的入口在： frameworks\base\core\java\android\content\pm\PackageParser.java

PackageParser.java是解析apk的工具类，验证签名时，调用 `collectCertificates(Package pkg, File apkFile, int parseFlags)`方法进行签名信息的验证。

```java
private static void collectCertificates(Package pkg, File apkFile, int parseFlags)
        throws PackageParserException {
    final String apkPath = apkFile.getAbsolutePath();
    boolean verified = false;//V2签名验证标记位
    {
        Certificate[][] allSignersCerts = null;
        Signature[] signatures = null;
        try {
            Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "verifyV2");
            //第一步：V2签名验证
            allSignersCerts = ApkSignatureSchemeV2Verifier.verify(apkPath);
            signatures = convertToSignatures(allSignersCerts);
            // APK verified using APK Signature Scheme v2.
            verified = true;
        } catch (ApkSignatureSchemeV2Verifier.SignatureNotFoundException e) {
          ...
                Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
        }
        // 如果V2签名验证通过  
        if (verified) {
            if (pkg.mCertificates == null) {
                pkg.mCertificates = allSignersCerts;
                pkg.mSignatures = signatures;
                pkg.mSigningKeys = new ArraySet<>(allSignersCerts.length);
                for (int i = 0; i < allSignersCerts.length; i++) {
                    Certificate[] signerCerts = allSignersCerts[i];
                    Certificate signerCert = signerCerts[0];
                    pkg.mSigningKeys.add(signerCert.getPublicKey());
                }
            }
        }

        int objectNumber = verified ? 1 : NUMBER_OF_CORES;
        StrictJarFile[] jarFile = new StrictJarFile[objectNumber];
        final ArrayMap<String, StrictJarFile> strictJarFiles = new ArrayMap<String, StrictJarFile>();
        try {
            // 第二步：进行.SF和.RSA文件的验证工作
            Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "strictJarFileCtor");
            ...
            for (int i = 0; i < objectNumber; i++) {
                //这里做了.SF和.RSA的验证工作
                jarFile[i] = new StrictJarFile(
                        apkPath,
                        !verified, // whether to verify JAR signature
                        signatureSchemeRollbackProtectionsEnforced);
            }
            Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);

            // Always verify manifest, regardless of source
            // 第三步：验证AndroidManifest.xml是否存在，不存在抛出异常
            final ZipEntry manifestEntry = jarFile[0].findEntry(ANDROID_MANIFEST_FILENAME);
            if (manifestEntry == null) {
                throw new PackageParserException(INSTALL_PARSE_FAILED_BAD_MANIFEST,
                        "Package " + apkPath + " has no manifest");
            }

            // Optimization: early termination when APK already verified
            // 第四步 ：V2签名验证通过的话，就不需要继续下面的工作了
            if (verified) {
                return;
            }

            // 第五步：对apk文件进行验证，主要是通过MANIFEST.MF文件记录的消息摘要与源文件生成的对比
            Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "verifyV1");
            final List<ZipEntry> toVerify = new ArrayList<>();
            toVerify.add(manifestEntry);

            // 非系统apk需要验证apk的文件,除了AndroidManifest.xml和META-INF文件夹下的文件都不需要验证
            if ((parseFlags & PARSE_IS_SYSTEM_DIR) == 0) {
                final Iterator<ZipEntry> i = jarFile[0].iterator();
                while (i.hasNext()) {
                    final ZipEntry entry = i.next();

                    if (entry.isDirectory()) continue;

                    final String entryName = entry.getName();
                    if (entryName.startsWith("META-INF/")) continue;
                    if (entryName.equals(ANDROID_MANIFEST_FILENAME)) continue;

                    toVerify.add(entry);
                }
            }

            final StrictJarFile[] sJarFiles = jarFile;
            class VerificationData {
                public Exception exception;
                public int exceptionFlag;
                public boolean wait;
                public int index;
                public Object objWaitAll;
                public boolean shutDown;
            }
            VerificationData vData = new VerificationData();
            vData.objWaitAll = new Object();
            // 开启线程池进行验证，提高验证效率
            final ThreadPoolExecutor verificationExecutor = new ThreadPoolExecutor(
                    NUMBER_OF_CORES,
                    NUMBER_OF_CORES,
                    1,/*keep alive time*/
                    TimeUnit.SECONDS,
                    new LinkedBlockingQueue<Runnable>());
            // 遍历验证文件
            for (ZipEntry entry : toVerify) {
                Runnable verifyTask = new Runnable(){
                    public void run() {
                        try {
                            ...
                            // 文件验证通过则返回证书
                            final Certificate[][] entryCerts = loadCertificates(tempJarFile, entry);
                            if (ArrayUtils.isEmpty(entryCerts)) {
                                throw new PackageParserException(INSTALL_PARSE_FAILED_NO_CERTIFICATES,
                                        "Package " + apkPath + " has no certificates at entry "
                                        + entry.getName());
                            }
                            final Signature[] entrySignatures = convertToSignatures(entryCerts);
    // 第六步：认证开发者身份
    synchronized (pkg) {
        if (pkg.mCertificates == null) {
            pkg.mCertificates = entryCerts;
            pkg.mSignatures = entrySignatures;
            pkg.mSigningKeys = new ArraySet<PublicKey>();
            for (int i=0; i < entryCerts.length; i++) {
                pkg.mSigningKeys.add(entryCerts[i][0].getPublicKey());
            }
        } else {
            if (!Signature.areExactMatch(pkg.mSignatures, entrySignatures)) {
                    throw new PackageParserException(
                                                INSTALL_PARSE_FAILED_INCONSISTENT_CERTIFICATES, "Package " + apkPath
                                                + " has mismatched certificates at entry "
                                                + entry.getName());
             }
        }
    }
    ...
}
```

在这个方法中主要做了6个操作，流程概括如下：

1. 在这个函数中，首先对签名apk做了v2方式的签名校验。也就是说首先用针对v2方式的签名方式来做签名校验，如果校验成功verified  = true。如果在校验的过程中抛出了异常，那么有两种可能：

   - apk没有用v2签名方式进行签名；
   - apk用了v2签名方式进行签名，但是签名检验没有成功。

2. 进行.SF和.RSA文件的验证工作

3. 验证AndroidManifest.xml是否存在，不存在抛出异常

4. V2签名验证通过的话，直接返回

5. 验证MANIFEST.MF文件

   主要是通过MANIFEST.MF文件记录的消息摘要与源文件生成的对比

6. 认证开发者身份

验证apk中的每一个entry(如AndroidManifest.xml)是否证书都一致，不一致则抛出异常。

#### 3.1.1 进行.RSA和.SF文件的验证工作

这两个文件的验证是在StrictJarFile的构造函数中的，接下来看下StrictJarFile构造函数的代码。

```java
private StrictJarFile(String name,FileDescriptor fd,boolean verify,
        boolean signatureSchemeRollbackProtectionsEnforced)
        throws IOException, SecurityException {
    this.nativeHandle = nativeOpenJarFile(name, fd.getInt$());
    this.fd = fd;

    try {
        if (verify) {
            // 将MANIFEST.MF、CERT.SF和CERT.RSA缓存到metaEntries中，key为文件名字，如META-INF/MANIFEST.MF
            HashMap<String, byte[]> metaEntries = getMetaEntries();
            // 创建MANIFEST.MF的StrictJarManifest对象，StrictJarManifest的构造器会检查MANIFEST.MF中是否有
            // 相同的Name条目，如Name: AndroidManifest.xml，如果有相同的条目，则抛出异常
            this.manifest = new StrictJarManifest(metaEntries.get(JarFile.MANIFEST_NAME), true);
            this.verifier =
                    new StrictJarVerifier(
                            name,
                            manifest,
                            metaEntries,
                            signatureSchemeRollbackProtectionsEnforced);
            Set<String> files = manifest.getEntries().keySet();
            for (String file : files) {
                if (findEntry(file) == null) {
                    throw new SecurityException("File " + file + " in manifest does not exist");
                }
            }
            // isSignedJar方法只要有认证过的签名文件就为true
            isSigned = verifier.readCertificates() && verifier.isSignedJar();
        } 
        ...
    }
```

最后会调用StrictJarVerifier的readCertificates方法进行.SF和.RSA文件的验证工作。

```java
synchronized boolean readCertificates() {
    if (metaEntries.isEmpty()) {
        return false;
    }

    Iterator<String> it = metaEntries.keySet().iterator();
    while (it.hasNext()) {
        String key = it.next();
        if (key.endsWith(".DSA") || key.endsWith(".RSA") || key.endsWith(".EC")) {
            verifyCertificate(key);
            it.remove();
        }
    }
    return true;
}
```

签名认证时，会寻找META-INF/下.RSA或者.DSA或者.EC文件，而我们打包时通常是使用RSA算法生成.RSA文件，所以这里会找到CERT.RSA文件，调用verifyCertificate方法进行验证 。

```java
private void verifyCertificate(String certFile) {
    String signatureFile = certFile.substring(0, certFile.lastIndexOf('.')) + ".SF";
    // .SF
    byte[] sfBytes = metaEntries.get(signatureFile);
    if (sfBytes == null) {
        return;
    }
    // MANIFEST.MF
    byte[] manifestBytes = metaEntries.get(JarFile.MANIFEST_NAME);
    // Manifest entry is required for any verifications.
    if (manifestBytes == null) {
        return;
    }
    // .RSA文件
    byte[] sBlockBytes = metaEntries.get(certFile);
    try {
        // 第一步：验证.SF文件，主要是通过.RSA文件与.SF文件的对比
        Certificate[] signerCertChain = verifyBytes(sBlockBytes, sfBytes);
        if (signerCertChain != null) {
            certificates.put(signatureFile, signerCertChain);
        }
    } catch (GeneralSecurityException e) {
      throw failedVerification(jarName, signatureFile, e);
    }

    // Verify manifest hash in .sf file
    Attributes attributes = new Attributes();
    HashMap<String, Attributes> entries = new HashMap<String, Attributes>();
    try {
        StrictJarManifestReader im = new StrictJarManifestReader(sfBytes, attributes);
        im.readEntries(entries, null);
    } catch (IOException e) {
        return;
    }
    //第二步：检验是否有V2签名标记
    if (signatureSchemeRollbackProtectionsEnforced) {
        String apkSignatureSchemeIdList =
                attributes.getValue(
                ApkSignatureSchemeV2Verifier.SF_ATTRIBUTE_ANDROID_APK_SIGNED_NAME);
        if (apkSignatureSchemeIdList != null) {
            boolean v2SignatureGenerated = false;
            StringTokenizer tokenizer = new StringTokenizer(apkSignatureSchemeIdList, ",");

            //如果CERT.SF头部有X-Android-APK-Signed: 2，证明是用V2签名方案的，此流程是验证V1方案，
            //因此抛出异常
            if (id == ApkSignatureSchemeV2Verifier.SF_ATTRIBUTE_ANDROID_APK_SIGNED_ID) {
                    v2SignatureGenerated = true;
                    break;
                }
            }

            if (v2SignatureGenerated) {
                throw new SecurityException(signatureFile + " indicates " + jarName
                        + " is signed using APK Signature Scheme v2, but no such signature was"
                        + " found. Signature stripped?");
            }
        }
    }

    // 第三步：使用.SF的"-Digest-Manifest-Main-Attributes"摘要验证.MF
    if (mainAttributesEnd > 0 && !createdBySigntool) {
        String digestAttribute = "-Digest-Manifest-Main-Attributes";
        if (!verify(attributes, digestAttribute, manifestBytes, 0, mainAttributesEnd, false, true)) {
            throw failedVerification(jarName, signatureFile);
        }
    }

    // 第四步：使用.SF分别验证.MF各个条目
    String digestAttribute = createdBySigntool ? "-Digest" : "-Digest-Manifest";
    if (!verify(attributes, digestAttribute, manifestBytes, 0, manifestBytes.length, false, false)) {
        Iterator<Map.Entry<String, Attributes>> it = entries.entrySet().iterator();
        while (it.hasNext()) {
            Map.Entry<String, Attributes> entry = it.next();
            StrictJarManifest.Chunk chunk = manifest.getChunk(entry.getKey());
            if (chunk == null) {
                return;
            }
            if (!verify(entry.getValue(), "-Digest", manifestBytes,
                    chunk.start, chunk.end, createdBySigntool, false)) {
                throw invalidDigest(signatureFile, entry.getKey(), jarName);
            }
        }
    }
    // 第五步：缓存.SF文件的entries
    metaEntries.put(signatureFile, null);
    signatures.put(signatureFile, entries);
}
```

流程概括如下：

1. 验证.RSA文件

   就是拿.RSA文件内容和.SF文件内容进行认证，具体认证调用verifyBytes方法 

2. 检验是否有V2签名标记

   因为在经过V2签名的APK中同时带有V1签名，攻击者可能将APK的V2签名删除，使得Android系统只校验V1签名。为防范此类攻击，V2方案规定:V2签名的APK如果还带V1签名，其 META-INF/.SF 文件的首部中必须包含 X-Android-APK-Signed 属性，如`X-Android-APK-Signed: 2`。该属性的值是一组以英文逗号分隔的 APK 签名方案 ID（v2 方案的 ID 为 2）。在验证 v1 签名时，对于此组中验证程序首选的 APK 签名方案（例如，v2 方案），如果 APK 没有相应的签名，APK 验证程序必须要拒绝这些 APK。此项保护依赖于内容 META-INF/.SF 文件受 v1 签名保护这一事实。

3. 验证.SF文件

4. 缓存.SF文件的entries

**(1) 验证.RSA文件**

 验证.RSA文件就是拿.RSA文件内容和.SF文件内容进行认证，具体认证调用verifyBytes方法。

[\frameworks\base\core\java\android\util\jar\StrictJarVerifier.java]

```java
static Certificate[] verifyBytes(byte[] blockBytes, byte[] sfBytes)
    throws GeneralSecurityException {
    Object obj = null;
    try {

        obj = Providers.startJarVerification();
        PKCS7 block = new PKCS7(blockBytes);
        // 提取.RSA中的消息摘要验证.SF整个文件
        SignerInfo[] verifiedSignerInfos = block.verify(sfBytes);
        if ((verifiedSignerInfos == null) || (verifiedSignerInfos.length == 0)) {
            throw new GeneralSecurityException(
                    "Failed to verify signature: no verified SignerInfos");
        }
        SignerInfo verifiedSignerInfo = verifiedSignerInfos[0];
        List<X509Certificate> verifiedSignerCertChain =
                verifiedSignerInfo.getCertificateChain(block);
        if (verifiedSignerCertChain == null) {
            throw new GeneralSecurityException(
                "Failed to find verified SignerInfo certificate chain");
        } else if (verifiedSignerCertChain.isEmpty()) {
            throw new GeneralSecurityException(
                "Verified SignerInfo certificate chain is emtpy");
        }
        
        return verifiedSignerCertChain.toArray(
                new X509Certificate[verifiedSignerCertChain.size()]);
        } catch (IOException e) {
            throw new GeneralSecurityException("IO exception verifying jar cert", e);
        } finally {
            Providers.stopJarVerification(obj);
        }
}
```

.RSA文件的验证在block.verify(sfBytes)。

```java
public SignerInfo[] verify(byte[] bytes)
    throws NoSuchAlgorithmException, SignatureException {

    Vector<SignerInfo> intResult = new Vector<SignerInfo>();
    for (int i = 0; i < signerInfos.length; i++) {

        SignerInfo signerInfo = verify(signerInfos[i], bytes);
        if (signerInfo != null) {
            intResult.addElement(signerInfo);
        }
    }
    ...
}
```

最后会调用到SignerInfo.java的`verify(PKCS7 block, InputStream inputStream)`

[\libcore\ojluni\src\main\java\sun\security\pkcs\SignerInfo.java]

```java
SignerInfo verify(PKCS7 block, InputStream inputStream)
    throws NoSuchAlgorithmException, SignatureException, IOException {

   try {
        ...

            // first, check content type
            // 检查内容类型
            ObjectIdentifier contentType = (ObjectIdentifier)
                   authenticatedAttributes.getAttributeValue(
                     PKCS9Attribute.CONTENT_TYPE_OID);
            if (contentType == null ||
                !contentType.equals((Object)content.contentType))
                return null;  // contentType does not match, bad SignerInfo

            // now, check message digest
       		// 检查消息摘要
            byte[] messageDigest = (byte[])
                authenticatedAttributes.getAttributeValue(
                     PKCS9Attribute.MESSAGE_DIGEST_OID);

            if (messageDigest == null) // fail if there is no message digest
                return null;

            // check that algorithm is not restricted
            if (!JAR_DISABLED_CHECK.permits(DIGEST_PRIMITIVE_SET,
                    digestAlgname, null)) {
                throw new SignatureException("Digest check failed. " +
                        "Disabled algorithm used: " + digestAlgname);
            }

           // 对.SF文件计算消息摘要
            MessageDigest md = MessageDigest.getInstance(digestAlgname);

            byte[] buffer = new byte[4096];
            int read = 0;
            while ((read = inputStream.read(buffer)) != -1) {
               md.update(buffer, 0 , read);
            }
            byte[] computedMessageDigest = md.digest();

            if (messageDigest.length != computedMessageDigest.length)
                return null;
            // 计算出来的.SF文件的消息摘要和保存在.RSA文件中的消息摘要进行对比
            for (int i = 0; i < messageDigest.length; i++) {
                if (messageDigest[i] != computedMessageDigest[i])
                    return null;
            }

            ...
}
```

 该方法的大体意思就是，使用.RSA文件中保存的公钥解密消息摘要，然后与计算出来的.SF文件消息摘要进行对比，只有相等的情况才可以证明.SF原始内容没有被篡改，认证通过，就可以拿到用户的数字证书，后续取出公钥对比用户信息了。

可以用`openssl pkcs7 -inform DER -in CERT.RSA -noout -print_certs -print -text`查看.RSA文件中保存的.SF文件消息摘要，字段为 enc_digest。该摘要是在apk签名的时候用私钥加密过的，因此在验证签名时需要用.RSA文件中的公钥进行解密。由于私钥只有开发者才拥有，因此该消息摘要其他人伪造不了。一旦其他人修改了.SF文件，那么与用公钥解密.RSA文件中的加密消息摘要enc_digest对比肯定不相同，因此签名验证失败。

**(2) 验证.SF文件**

 在上述verifyCertificate方法后半段，还有一个认证过程。

```java
// 第四步：使用.SF分别验证.MF各个条目
    String digestAttribute = createdBySigntool ? "-Digest" : "-Digest-Manifest";
    if (!verify(attributes, digestAttribute, manifestBytes, 0, manifestBytes.length, false, false)) {
        Iterator<Map.Entry<String, Attributes>> it = entries.entrySet().iterator();
        while (it.hasNext()) {
            Map.Entry<String, Attributes> entry = it.next();
            StrictJarManifest.Chunk chunk = manifest.getChunk(entry.getKey());
            if (chunk == null) {
                return;
            }
            if (!verify(entry.getValue(), "-Digest", manifestBytes,
                    chunk.start, chunk.end, createdBySigntool, false)) {
                throw invalidDigest(signatureFile, entry.getKey(), jarName);
            }
        }
    }
```

.SF文件就是记录了整个MANIFEST.MF内容的编码摘要，以及每个内容的编码摘要，所以这个方法就是在验证MANIFEST.MF文件是否被改动，如何验证的呢？无论是整个内容还是每个内容，都是调用的verify方法 。

```java
private boolean verify(Attributes attributes, String entry, byte[] data,
        int start, int end, boolean ignoreSecondEndline, boolean ignorable) {
           
    for (int i = 0; i < DIGEST_ALGORITHMS.length; i++) {
        String algorithm = DIGEST_ALGORITHMS[i];
        String hash = attributes.getValue(algorithm + entry);
        if (hash == null) {
            continue;
        }

        MessageDigest md;
        try {
            md = MessageDigest.getInstance(algorithm);
        } catch (NoSuchAlgorithmException e) {
            continue;
        }
        if (ignoreSecondEndline && data[end - 1] == '\n' && data[end - 2] == '\n') {
            md.update(data, start, end - 1 - start);
        } else {
            md.update(data, start, end - start);
        }
        // 相同的算法提取apk中MANIFEST.MF内容的摘要
        byte[] b = md.digest();
        // 获取.SF文件中对应内容
        byte[] encodedHashBytes = hash.getBytes(StandardCharsets.ISO_8859_1);
        // 将b进行base64编码直接与.SF的内容进行相等比较
        return verifyMessageDigest(b, encodedHashBytes);
    }
    return ignorable;
}
```

verify方法主要是将.SF中保存的消息摘要编码的内容和计算出来的MANIFEST.MF对应的条目消息摘要编码内容进行比较，从而验证MANIFEST.MF是否被有篡改。

```java
private static boolean verifyMessageDigest(byte[] expected, byte[] encodedActual) {
    byte[] actual;
    try {
        actual = java.util.Base64.getDecoder().decode(encodedActual);
    } catch (IllegalArgumentException e) {
        return false;
    }
    return MessageDigest.isEqual(expected, actual);
}
```

该方法主要是对进过Base64编码的消息摘要进行Base64解码，然后比较消息摘要。

#### 3.1.2 验证MANIFEST.MF

验证MANIFEST.MF文件的目的是验证apk各个文件是否有篡改。MANIFEST.MF文件记录的是apk的各个文件的消息摘要编码。回到collectCertificates方法的第五步，是调用`loadCertificates(tempJarFile, entry)`方法完成MANIFEST.MF的验证工作的。

[\frameworks\base\core\java\android\content\pm\PackageParser.java]

```java
private static Certificate[][] loadCertificates(StrictJarFile jarFile, ZipEntry entry)
        throws PackageParserException {
    InputStream is = null;
    try {
        // We must read the stream for the JarEntry to retrieve
        // its certificates.
        is = jarFile.getInputStream(entry);// 第一步：获取apk的指定的源文件
        readFullyIgnoringContents(is);// 第二步：读取内容并验证
        return jarFile.getCertificateChains(entry);// 第三步：返回认证信息
    } catch (IOException | RuntimeException e) {
        throw new PackageParserException(INSTALL_PARSE_FAILED_UNEXPECTED_EXCEPTION,
                "Failed reading " + entry.getName() + " in " + jarFile, e);
    } finally {
        IoUtils.closeQuietly(is);
    }
}
```

 **(1) 第一步：获取apk的指定的源文件**

读取文件是通过StrictJarFile.getInputStream获取apk的指定的源文件。

```java
public InputStream getInputStream(ZipEntry ze) {
    final InputStream is = getZipInputStream(ze);

    // apk签名已验证成功
    if (isSigned) {
        // 获取VerifierEntry实例
        StrictJarVerifier.VerifierEntry entry = verifier.initEntry(ze.getName());
        if (entry == null) {
            return is;
        }

        return new JarFileInputStream(is, ze.getSize(), entry);
    }

    return is;
}
```

 

```java
VerifierEntry initEntry(String name) {
    if (manifest == null || signatures.isEmpty()) {
        return null;
    }
    // 获取MANIFEST.MF中相应内容(文件)的内容
    Attributes attributes = manifest.getAttributes(name);
    ...

    for (int i = 0; i < DIGEST_ALGORITHMS.length; i++) {
        final String algorithm = DIGEST_ALGORITHMS[i];
        final String hash = attributes.getValue(algorithm + "-Digest");
        if (hash == null) {
            continue;
        }
        //获取对应的编码摘要
        byte[] hashBytes = hash.getBytes(StandardCharsets.ISO_8859_1);

        try {
            // 构建VerifierEntry对象
            return new VerifierEntry(name, MessageDigest.getInstance(algorithm), hashBytes,
                    certChainsArray, verifiedEntries);
        } catch (NoSuchAlgorithmException ignored) {
        }
    }
    return null;
}
```

该方法找到MANIFEST.MF文件中对于内容的摘要值，构建了一个VerifierEntry对象。

**(2) 第二步：读取内容并验证**

 readFullyIgnoringContents(is)

```java
public static long readFullyIgnoringContents(InputStream in) throws IOException {
    byte[] buffer = sBuffer.getAndSet(null);
    if (buffer == null) {
        buffer = new byte[4096];
    }

    int n = 0;
    int count = 0;
    while ((n = in.read(buffer, 0, buffer.length)) != -1) {
        count += n;
    }

    sBuffer.set(buffer);
    return count;
}
```

注意的是上面的getInputStream方法如果apk签名已验证返回的是JarFileInputStream输入流，因此readFullyIgnoringContents里面的in.read(buffer, 0, buffer.length))是JarFileInputStream的read。

[\frameworks\base\core\java\android\util\jar\StrictJarFile.java]

```java
@Override
public int read() throws IOException {
    if (done) {
        return -1;
    }
    if (count > 0) {
        int r = super.read();
        if (r != -1) {
            entry.write(r);// 写入到VerifierEntry中
            count--;
        } else {
            count = 0;
        }
        if (count == 0) {// 读取完毕
            done = true;
            entry.verify();// 使用VerifierEntry进行认证
        }
        return r;
    } else {
        done = true;
        entry.verify();
        return -1;
    }
}
```

 首先不断的将文件读取到VerifierEntry中，读完后VerifierEntry进行认证 。

```java
@Override
public void write(int value) {
    digest.update((byte) value);
}
```

 构建VerifierEntry时传入的摘要算法，如SHA1或SHA256。

```java
void verify() {
    byte[] d = digest.digest();
    // 文件摘要与MANIFEST.MF中记录对应的文件摘要进行比较
    if (!verifyMessageDigest(d, hash)) {
        throw invalidDigest(JarFile.MANIFEST_NAME, name, name);
    }
    verifiedEntries.put(name, certChains);
}
```

```java
private static boolean verifyMessageDigest(byte[] expected, byte[] encodedActual) {
    byte[] actual;
    try {
        actual = java.util.Base64.getDecoder().decode(encodedActual);
    } catch (IllegalArgumentException e) {
        return false;
    }
    return MessageDigest.isEqual(expected, actual);
}
```

write方法就是使用构建VerifierEntry时传入的摘要算法，对apk的文件进行摘要计算，随后的verify方法，将apk文件的摘要进行base64编码，与MANIFEST.MF中对应的值进行比较，如果相等则说明文件没有被篡改，否则认证不通过抛异常。 

#### 3.1.3 认证开发者身份

 当上面的校验都通过后，就拿到了apk的开发者身份证书，最后还要做一步，就是要比较新老apk的开发者身份是否为同一人。 回到collectCertificates方法的最后：

 

```java
// 第六步：认证开发者身份
synchronized (pkg) {
    // 如果没有旧的apk，则将认证信息填入
    if (pkg.mCertificates == null) {
        pkg.mCertificates = entryCerts;
        pkg.mSignatures = entrySignatures;
        pkg.mSigningKeys = new ArraySet<PublicKey>();
        for (int i=0; i < entryCerts.length; i++) {
            pkg.mSigningKeys.add(entryCerts[i][0].getPublicKey());
        }
    // 如果有旧的apk，则要比较新旧apk的开发者签名信息，相同才能认证成功
    } else {
        if (!Signature.areExactMatch(pkg.mSignatures, entrySignatures)) {
                throw new PackageParserException(
                                                INSTALL_PARSE_FAILED_INCONSISTENT_CERTIFICATES, "Package " + apkPath
                                           + " has mismatched certificates at entry "
                                            + entry.getName());
         }
    }
}
```

#### 3.1.4 V1签名方案验证总结

V1签名方案概括如下：

1. 通过.RSA文件获取数字签名(enc_digest)和开发者公钥，使用公钥解密数字签名，与.SF文件做对比，如果不等则校验失败
2. 计算MANIFEST.MF文件整个内容的摘要，与.SF文件中记录的值做对比，如果不等则校验失败；计算MANIFEST.MF文件中每个内容的摘要值，与.SF文件中记录的对应值做对比，如果不等则校验失败
3. 计算apk中除了META-INF之外每个文件的摘要值，与.SF文件中记录的值做对比，如果不等则校验失败
4. 比较待安装的apk开发者信息与原apk开发者信息，如果不等则校验失败

这样的流程为什么会保证apk的安全呢，和非对称加密的原理类似：

下面来结合实例来说明上面三个文件在V1签名方案中的应用。

1. 篡改apk的文件内容

   校验apk中每个文件的完整性时失败；如果是添加新文件，因为此文件的hash值在MANIFEST.MF和CERT.SF中无记录，同样校验失败；

2. 篡改apk的文件内容，同时篡改MANIFEST.MF文件相应的摘要

   校验MANIFEST.MF文件的摘要会失败；

3. 篡改apk的文件内容，同时篡改MANIFEST.MF文件相应的摘要，以及CERT.SF文件的内容

   校验CERT.SF文件的签名会失败；

4. 把apk的文件内容和签名信息一同全部篡改

   这相当于对apk进行了重新签名，在此apk没有安装到系统中的情况下，是可以正常安装的，这相当于是一个新的app；但如果进行覆盖安装，则证书不一样，安装失败。

从这里可以看出只要篡改了apk中的任何内容，都会使得签名校验失败。

另外还有一点，我们一直在提到META-INF/下的文件在安装时是不会算在校验里的，所以我们可以在生成的apk中，修改这个文件夹里面的任意内容而不用重新打包签名，这也是通常我们渠道包使用该文件夹来实现的原因。

### 3.2 APK签名方案V2验证流程

由APK的V2签名方案可知， 签名数据会存放到apk文件的单独区块内，那么安装时系统对于V2签名是如何验证的呢？  在 Android 7.0 及更高版本中，可以根据 APK 签名方案 v2+ 或 JAR 签名（v1 方案）验证 APK。更低版本的平台会忽略 v2 签名，仅验证 v1 签名。 

![v1v2](/images/pms/apk-sign/v1v2.png)

#### 3.2.1 APK 签名方案 v2 验证

1. 找到“APK 签名分块”并验证以下内容：
   1. “APK 签名分块”的两个大小字段包含相同的值。
   2. “ZIP 中央目录结尾”紧跟在“ZIP 中央目录”记录后面。
   3. “ZIP 中央目录结尾”之后没有任何数据。
2. 找到“APK 签名分块”中的第一个“APK 签名方案 v2 分块”。如果 v2 分块存在，则继续执行第 3 步。否则，回退至[使用 v1 方案](https://source.android.google.cn/security/apksigning/v2?hl=zh-cn#v1-verification)验证 APK。
3. 对“APK 签名方案 v2 分块”中的每个signer执行以下操作：
   1. 从 `signatures` 中选择安全系数最高的受支持 `signature algorithm ID`。安全系数排序取决于各个实现/平台版本。
   2. 使用 `public key` 并对照 `signed data` 验证 `signatures` 中对应的 `signature`。（现在可以安全地解析 `signed data` 了。）
   3. 验证 `digests` 和 `signatures` 中的签名算法 ID 列表（有序列表）是否相同。（这是为了防止删除/添加签名。）
   4. 使用签名算法所用的同一种摘要算法[计算 APK 内容的摘要](https://source.android.google.cn/security/apksigning/v2?hl=zh-cn#integrity-protected-contents)。
   5. 验证计算出的摘要是否与 `digests` 中对应的 `digest` 一致。
   6. 验证 `certificates` 中第一个 `certificate` 的 SubjectPublicKeyInfo 是否与 `public key` 相同。
4. 如果找到了至少一个 `signer`，并且对于每个找到的 `signer`，第 3 步都取得了成功，APK 验证将会成功。

接下来结合源码详细分析V2签名的验证流程。

首先回到collectCertificates方法的开头，安装apk验证签名是默认先进行V2方案签名的，注意V2方案是Android 7.0之后引入的，也就是说，在Android 7.0之前只有V1签名方案验证，7.0及以后的版本默认先进行V2方案验证，如果apk没用V2方案对apk进行签名，则走V1签名验证流程。

```java
private static void collectCertificates(Package pkg, File apkFile, int parseFlags)
        throws PackageParserException {
    final String apkPath = apkFile.getAbsolutePath();
    boolean verified = false;//V2签名验证标记位
    {
        Certificate[][] allSignersCerts = null;
        Signature[] signatures = null;
        try {
            Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "verifyV2");
            //第一步：V2签名验证
            allSignersCerts = ApkSignatureSchemeV2Verifier.verify(apkPath);
            signatures = convertToSignatures(allSignersCerts);
            // APK verified using APK Signature Scheme v2.
            verified = true;
        } catch (ApkSignatureSchemeV2Verifier.SignatureNotFoundException e) {
          ...
                Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
        }
        // V2认证通过后，继续认证新老apk的开发者信息，和V1一样；如果V2没认证，则使用V1
        if (verified) {
            if (pkg.mCertificates == null) {
                pkg.mCertificates = allSignersCerts;
                pkg.mSignatures = signatures;
                pkg.mSigningKeys = new ArraySet<>(allSignersCerts.length);
                for (int i = 0; i < allSignersCerts.length; i++) {
                    Certificate[] signerCerts = allSignersCerts[i];
                    Certificate signerCert = signerCerts[0];
                    pkg.mSigningKeys.add(signerCert.getPublicKey());
                }
            }
        }
        ...
```

V2签名验证是在`ApkSignatureSchemeV2Verifier.verify(apkPath)`里面完成的。

[\frameworks\base\core\java\android\util\apk\ApkSignatureSchemeV2Verifier.java]

```java
public static X509Certificate[][] verify(String apkFile)
        throws SignatureNotFoundException, SecurityException, IOException {
    try (RandomAccessFile apk = new RandomAccessFile(apkFile, "r")) {
        return verify(apk);
    }
}
private static X509Certificate[][] verify(RandomAccessFile apk)
        throws SignatureNotFoundException, SecurityException, IOException {
    // 第一步：验证apk文件格式并获取签名块内容
    SignatureInfo signatureInfo = findSignature(apk);
    // 第二步：认证流程
    return verify(apk.getFD(), signatureInfo);
}
```

#### 3.2.2 验证apk文件格式并获取签名块内容

```java
private static SignatureInfo findSignature(RandomAccessFile apk)
        throws IOException, SignatureNotFoundException {
    // Find the ZIP End of Central Directory (EoCD) record.
    // 第一步：通过EoCD的信息确定CD的位置
    Pair<ByteBuffer, Long> eocdAndOffsetInFile = getEocd(apk);
    ByteBuffer eocd = eocdAndOffsetInFile.first;
    long eocdOffset = eocdAndOffsetInFile.second;
    if (ZipUtils.isZip64EndOfCentralDirectoryLocatorPresent(apk, eocdOffset)) {
        throw new SignatureNotFoundException("ZIP64 APK not supported");
    }

    // Find the APK Signing Block. The block immediately precedes the Central Directory.
    // 第二步：通过CD的信息确定签名块的位置
    long centralDirOffset = getCentralDirOffset(eocd, eocdOffset);
    Pair<ByteBuffer, Long> apkSigningBlockAndOffsetInFile =
            findApkSigningBlock(apk, centralDirOffset);
    ByteBuffer apkSigningBlock = apkSigningBlockAndOffsetInFile.first;
    long apkSigningBlockOffset = apkSigningBlockAndOffsetInFile.second;

    // Find the APK Signature Scheme v2 Block inside the APK Signing Block.
    // 第三步：读取签名块中的签名数据信息
    ByteBuffer apkSignatureSchemeV2Block = findApkSignatureSchemeV2Block(apkSigningBlock);
   // 构建签名数据对象
    return new SignatureInfo(
            apkSignatureSchemeV2Block,
            apkSigningBlockOffset,
            centralDirOffset,
            eocdOffset,
            eocd);
}
```

该方法主要三步：

1. 通过EoCD区域记录的CD位置信息，确定CD区域的存在
2. 通过CD区域记录的位置信息，确定签名区域的位置
3. 从签名区域里读取签名数据的信息

 SignatureInfo中包括整个签名块的内容，签名块的位置信息，核心目录块的的位置信息，目录结束标识块的位置及内容。  这里重点看一下第3步：从签名区域里读取签名数据的信息。

```java
private static ByteBuffer findApkSignatureSchemeV2Block(ByteBuffer apkSigningBlock)
        throws SignatureNotFoundException {
    checkByteOrderLittleEndian(apkSigningBlock);
    // FORMAT:
    // OFFSET       DATA TYPE  DESCRIPTION
    // * @+0  bytes uint64:    size in bytes (excluding this field)
    // * @+8  bytes pairs
    // * @-24 bytes uint64:    size in bytes (same as the one above)
    // * @-16 bytes uint128:   magic
    ByteBuffer pairs = sliceFromTo(apkSigningBlock, 8, apkSigningBlock.capacity() - 24);

    int entryCount = 0;
    while (pairs.hasRemaining()) {
        entryCount++;
        if (pairs.remaining() < 8) {
           throw new SignatureNotFoundException(
                   "Insufficient data to read size of APK Signing Block entry #" + entryCount);
        }
        long lenLong = pairs.getLong();
        if ((lenLong < 4) || (lenLong > Integer.MAX_VALUE)) {
           throw new SignatureNotFoundException(
                    "APK Signing Block entry #" + entryCount
                            + " size out of range: " + lenLong);
        }
        int len = (int) lenLong;
        int nextEntryPos = pairs.position() + len;
        if (len > pairs.remaining()) {
           throw new SignatureNotFoundException(
                    "APK Signing Block entry #" + entryCount + " size out of range: " + len
                           + ", available: " + pairs.remaining());
        }
        int id = pairs.getInt();
        //从ID-value中找到签名数据
        if (id == APK_SIGNATURE_SCHEME_V2_BLOCK_ID) {
            return getByteBuffer(pairs, len - 4);
        }
        pairs.position(nextEntryPos);
    }

    throw new SignatureNotFoundException(
            "No APK Signature Scheme v2 block in APK Signing Block");
}
```

这里我们看到，会遍历所以ID-value数据，找到APK_SIGNATURE_SCHEME_V2_BLOCK_ID对应的值，而APK_SIGNATURE_SCHEME_V2_BLOCK_ID就是我们上面签名时存的ID：0x7109871a。
当找不到时就会抛出异常SignatureNotFoundException，会使得系统认为apk没有使用V2签名，转而使用V1签名认证。
到此，我们拿到了签名数据，然后再来看看最后一个verify方法是如何验证的呢。

#### 3.2.3 认证流程

[\frameworks\base\core\java\android\util\apk\ApkSignatureSchemeV2Verifier.java]

```java
private static X509Certificate[][] verify(
        FileDescriptor apkFileDescriptor,
        SignatureInfo signatureInfo) throws SecurityException {
    int signerCount = 0;
    Map<Integer, byte[]> contentDigests = new ArrayMap<>();
    List<X509Certificate[]> signerCerts = new ArrayList<>();
    CertificateFactory certFactory;
    try {
        certFactory = CertificateFactory.getInstance("X.509");
    } catch (CertificateException e) {
        throw new RuntimeException("Failed to obtain X.509 CertificateFactory", e);
    }
    ByteBuffer signers;
    try {
        // 读取签名块内所有签名信息
        signers = getLengthPrefixedSlice(signatureInfo.signatureBlock);
    } catch (IOException e) {
        throw new SecurityException("Failed to read list of signers", e);
    }
    // 第一步：验证每个签名，一个apk可以有多个签名，但一般只签名一次
    while (signers.hasRemaining()) {
        signerCount++;
        try {
            ByteBuffer signer = getLengthPrefixedSlice(signers);
            X509Certificate[] certs = verifySigner(signer, contentDigests, certFactory);
            signerCerts.add(certs);
        } catch (IOException | BufferUnderflowException | SecurityException e) {
            throw new SecurityException(
                    "Failed to parse/verify signer #" + signerCount + " block",
                    e);
        }
    }

    if (signerCount < 1) {
        throw new SecurityException("No signers found");
    }

    if (contentDigests.isEmpty()) {
        throw new SecurityException("No content digests found");
    }
    // 第二步：验证摘要
    verifyIntegrity(
            contentDigests,
            apkFileDescriptor,
            signatureInfo.apkSigningBlockOffset,
            signatureInfo.centralDirOffset,
            signatureInfo.eocdOffset,
            signatureInfo.eocd);
    // 返回所有验证通过的信息
    return signerCerts.toArray(new X509Certificate[signerCerts.size()][]);
}
```

 这里分为了两步，验证签名信息以及验证摘要信息。

**(1) 验证签名信息**

```java
private static X509Certificate[] verifySigner(
        ByteBuffer signerBlock,
        Map<Integer, byte[]> contentDigests,
        CertificateFactory certFactory) throws SecurityException, IOException {
    // 签名块的三部分数据，可见2.2.4 V2 分块格式小节的图
    ByteBuffer signedData = getLengthPrefixedSlice(signerBlock);
    ByteBuffer signatures = getLengthPrefixedSlice(signerBlock);
    byte[] publicKeyBytes = readLengthPrefixedByteArray(signerBlock);

    int signatureCount = 0;
    int bestSigAlgorithm = -1;
    byte[] bestSigAlgorithmSignatureBytes = null;
    List<Integer> signaturesSigAlgorithms = new ArrayList<>();
    while (signatures.hasRemaining()) {
        signatureCount++;
        try {
            ByteBuffer signature = getLengthPrefixedSlice(signatures);
            if (signature.remaining() < 8) {
                throw new SecurityException("Signature record too short");
            }
            int sigAlgorithm = signature.getInt();
            signaturesSigAlgorithms.add(sigAlgorithm);
            if (!isSupportedSignatureAlgorithm(sigAlgorithm)) {
                continue;
            }
            // 第一步： 从signatures中选择安全系数最高的受支持 signature algorithm ID那个
            if ((bestSigAlgorithm == -1)
                    || (compareSignatureAlgorithm(sigAlgorithm, bestSigAlgorithm) > 0)) {
                bestSigAlgorithm = sigAlgorithm;
                bestSigAlgorithmSignatureBytes = readLengthPrefixedByteArray(signature);
            }
        } catch (IOException | BufferUnderflowException e) {
            throw new SecurityException(
                    "Failed to parse signature record #" + signatureCount,
                    e);
        }
    }
    if (bestSigAlgorithm == -1) {
        if (signatureCount == 0) {
                throw new SecurityException("No signatures found");
        } else {
            throw new SecurityException("No supported signatures found");
        }
    }

    String keyAlgorithm = getSignatureAlgorithmJcaKeyAlgorithm(bestSigAlgorithm);
    Pair<String, ? extends AlgorithmParameterSpec> signatureAlgorithmParams =
           getSignatureAlgorithmJcaSignatureAlgorithm(bestSigAlgorithm);
    String jcaSignatureAlgorithm = signatureAlgorithmParams.first;
    AlgorithmParameterSpec jcaSignatureAlgorithmParams = signatureAlgorithmParams.second;
    boolean sigVerified;
    try {
        PublicKey publicKey =
                KeyFactory.getInstance(keyAlgorithm)
                        .generatePublic(new X509EncodedKeySpec(publicKeyBytes));
        Signature sig = Signature.getInstance(jcaSignatureAlgorithm);
        sig.initVerify(publicKey);
        if (jcaSignatureAlgorithmParams != null) {
            sig.setParameter(jcaSignatureAlgorithmParams);
        }
        sig.update(signedData);
        // 第二步：使用公钥验证签名信息
        sigVerified = sig.verify(bestSigAlgorithmSignatureBytes);
    } catch (NoSuchAlgorithmException | InvalidKeySpecException | InvalidKeyException
            | InvalidAlgorithmParameterException | SignatureException e) {
        throw new SecurityException(
                "Failed to verify " + jcaSignatureAlgorithm + " signature", e);
    }
    if (!sigVerified) {
        throw new SecurityException(jcaSignatureAlgorithm + " signature did not verify");
    }

    // Signature over signedData has verified.
    // 第三步：取出signed data中记录的摘要信息，以便后面做摘要验证
    byte[] contentDigest = null;
    signedData.clear();
    ByteBuffer digests = getLengthPrefixedSlice(signedData);
    List<Integer> digestsSigAlgorithms = new ArrayList<>();
    int digestCount = 0;
    while (digests.hasRemaining()) {
        digestCount++;
        try {
            ByteBuffer digest = getLengthPrefixedSlice(digests);
            if (digest.remaining() < 8) {
                throw new IOException("Record too short");
            }
            int sigAlgorithm = digest.getInt();
            digestsSigAlgorithms.add(sigAlgorithm);
            if (sigAlgorithm == bestSigAlgorithm) {
                contentDigest = readLengthPrefixedByteArray(digest);
            }
        } catch (IOException | BufferUnderflowException e) {
            throw new IOException("Failed to parse digest record #" + digestCount, e);
        }
    }

    if (!signaturesSigAlgorithms.equals(digestsSigAlgorithms)) {
        throw new SecurityException(
                "Signature algorithms don't match between digests and signatures records");
    }
    int digestAlgorithm = getSignatureAlgorithmContentDigestAlgorithm(bestSigAlgorithm);
    byte[] previousSignerDigest = contentDigests.put(digestAlgorithm, contentDigest);
    if ((previousSignerDigest != null)
            && (!MessageDigest.isEqual(previousSignerDigest, contentDigest))) {
        throw new SecurityException(
                getContentDigestAlgorithmJcaDigestAlgorithm(digestAlgorithm)
                + " contents digest does not match the digest specified by a preceding signer");
    }
    // 第四步：取出signed data中的证书信息
    ByteBuffer certificates = getLengthPrefixedSlice(signedData);
    List<X509Certificate> certs = new ArrayList<>();
    int certificateCount = 0;
    while (certificates.hasRemaining()) {
        certificateCount++;
        byte[] encodedCert = readLengthPrefixedByteArray(certificates);
        X509Certificate certificate;
        try {
            certificate = (X509Certificate)
                    certFactory.generateCertificate(new ByteArrayInputStream(encodedCert));
        } catch (CertificateException e) {
            throw new SecurityException("Failed to decode certificate #" + certificateCount, e);
        }
        certificate = new VerbatimX509Certificate(certificate, encodedCert);
        certs.add(certificate);
    }

    if (certs.isEmpty()) {
        throw new SecurityException("No certificates listed");
    }
    // 第五步：验证证书身份和公钥是否为同一个身份
    X509Certificate mainCertificate = certs.get(0);
    byte[] certificatePublicKeyBytes = mainCertificate.getPublicKey().getEncoded();
    if (!Arrays.equals(publicKeyBytes, certificatePublicKeyBytes)) {
        throw new SecurityException(
                "Public key mismatch between certificate and signature record");
    }
    // 返回证书信息
    return certs.toArray(new X509Certificate[certs.size()]);
}
```

该方法流程概括如下：

1. 找到算法等级最高的签名验证

   从 `signatures` 中选择安全系数最高的受支持 `signature algorithm ID` 那个，算法等级为：SHA512 > SHA256。因为V2签名不用SHA1签名，因此不需要比较SHA1。

2. 使用公钥验证签名信息

   先拿公钥验证签名信息，即公钥解密由私钥加密的signed data，与signed data做对比，只有正确的公钥才能解开对应的私钥加密的数据；如果验证不通过说明开发者身份错误或者signed data被篡改

3. 取出signed data中记录的摘要信息，以便后面做摘要验证

   签名验证通过后，就可以安全使用摘要，将摘要记录下来，用以后面的摘要验证

4. 取出signed data中的证书信息

5. 验证 `certificates` 中第一个 `certificate` 的 SubjectPublicKeyInfo 是否与 `public key` 相同。

**(2) 验证摘要信息**

回到verify最后部分：验证摘要信息。

```java
verifyIntegrity(
        contentDigests,
        apkFileDescriptor,
        signatureInfo.apkSigningBlockOffset,
        signatureInfo.centralDirOffset,
        signatureInfo.eocdOffset,
        signatureInfo.eocd);
```

```java
private static void verifyIntegrity(
        Map<Integer, byte[]> expectedDigests,
        FileDescriptor apkFileDescriptor,
        long apkSigningBlockOffset,
        long centralDirOffset,
        long eocdOffset,
        ByteBuffer eocdBuf) throws SecurityException {

    if (expectedDigests.isEmpty()) {
        throw new SecurityException("No digests provided");
    }
    // 获取apk的contents、CD、EoCD三个部分内容
    DataSource beforeApkSigningBlock =
            new MemoryMappedFileDataSource(apkFileDescriptor, 0, apkSigningBlockOffset);
    DataSource centralDir =
            new MemoryMappedFileDataSource(
                    apkFileDescriptor, centralDirOffset, eocdOffset - centralDirOffset);

    eocdBuf = eocdBuf.duplicate();
    eocdBuf.order(ByteOrder.LITTLE_ENDIAN);
    ZipUtils.setZipEocdCentralDirectoryOffset(eocdBuf, apkSigningBlockOffset);
    DataSource eocd = new ByteBufferDataSource(eocdBuf);

    int[] digestAlgorithms = new int[expectedDigests.size()];
    int digestAlgorithmCount = 0;
    for (int digestAlgorithm : expectedDigests.keySet()) {
        digestAlgorithms[digestAlgorithmCount] = digestAlgorithm;
        digestAlgorithmCount++;
    }
    // 计算每个部分内容的实际摘要值
    byte[][] actualDigests;
    try {
        actualDigests =
                computeContentDigests(
                        digestAlgorithms,
                        new DataSource[] {beforeApkSigningBlock, centralDir, eocd});
    } catch (DigestException e) {
        throw new SecurityException("Failed to compute digest(s) of contents", e);
    }
    // 对比每个部分的实际摘要值和signed data中对应的原始摘要值，如果全部相等则说明内容未被篡改，通过验证
    for (int i = 0; i < digestAlgorithms.length; i++) {
        int digestAlgorithm = digestAlgorithms[i];
        byte[] expectedDigest = expectedDigests.get(digestAlgorithm);
        byte[] actualDigest = actualDigests[i];
        if (!MessageDigest.isEqual(expectedDigest, actualDigest)) {
            throw new SecurityException(
                    getContentDigestAlgorithmJcaDigestAlgorithm(digestAlgorithm)
                            + " digest of contents did not verify");
        }
    }
}
```

这个验证很简单，就是计算apk中的contents、CD、EoCD三个部分的真是摘要，与signed data中的原始摘要做对比，全部相同说明内容未被篡改，验证通过。
至此，再加上新老apk的公钥身份的验证，就完成了V2签名的认证过程。

#### 3.2.4 .V2签名方案验证总结

1. 7.0及以上系统，会直接尝试使用V2签名认证过程认证，如果检测apk文件没有使用V2签名方式或7.0一下系统，会直接使用V1签名认证过程，以便向后兼容。
2. 检测apk文件的格式，包括contents区、CD区、EoCD区的存在性
3. 然后根据zip文件格式，很方便的找到并读取其存在的签名区块内的ID为0x7109871a的签名数据；如果没有该区块或没有该ID的值，说明没有使用V2签名，系统会继续走V1认证过程
4. 找到签名数据后，使用公钥认证数字签名与signed data
5. 签名认证通过后，计算apk内三个部分的真实摘要，与signed data内的原始摘要进行对比，相同则证明内容没有被篡改
6. 最后再校验新老apk的公钥，确定是同一开发者开发，这一步与V1认证一致

相比V1签名方案，V2方案有以下优点：

1. 利用zip文件格式做签名数据的存储和验证，可以使得验证过程加快；使用分块方式对数据做摘要计算，加速了签名过程
2. 对于全部原始内容区域做完整性保证，增加了完整性的保证

## 4. APK 签名方案 v3

Android 9 支持 [APK 密钥轮替](https://developer.android.google.cn/about/versions/pie/android-9.0?hl=zh-cn#apk-key-rotation)，这使应用能够在 APK 更新过程中更改其签名密钥。为了实现轮替，APK 必须指示新旧签名密钥之间的信任级别。为了支持密钥轮替，我们将 [APK 签名方案](https://source.android.google.cn/security/apksigning/v2?hl=zh-cn)从 v2 更新为 v3，以允许使用新旧密钥。v3 在 APK 签名分块中添加了有关受支持的 SDK 版本和 proof-of-rotation 结构的信息。

### 4.1 APK 签名分块

为了保持与 v1 APK 格式的向后兼容性，v2 和 v3 APK 签名存储在“APK 签名分块”内紧邻 ZIP Central Directory 前面。

v3 APK 签名分块的格式[与 v2 相同](https://source.android.google.cn/security/apksigning/v2?hl=zh-cn#apk-signing-block-format)。APK 的 v3 签名会存储为一个“ID-值”对，其中 ID 为 0xf05368c0。

### 4.2 APK 签名方案 v3 分块

v3 方案的设计与 [v2 方案](https://source.android.google.cn/security/apksigning/v2?hl=zh-cn#apk-signature-scheme-v2-block)非常相似，它们采用相同的常规格式，并支持相同的[签名算法 ID](https://source.android.google.cn/security/apksigning/v2?hl=zh-cn#signature-algorithm-ids)、密钥大小和 EC 曲线。

但是，v3 方案增添了有关受支持的 SDK 版本和 proof-of-rotation 结构的信息。

**格式**

“APK 签名方案 v3 分块”存储在“APK 签名分块”内，ID 为 `0xf05368c0`。

“APK 签名方案 v3 分块”采用 v2 的格式：

-  带长度前缀的 `signer`（带长度前缀）序列： 
   -  带长度前缀的 `signed data`： 
      -  带长度前缀的 `digests`（带长度前缀）序列： 
         - `signature algorithm ID`（4 个字节）
         - `digest`（带长度前缀）
      -  带长度前缀的 X.509 certificates序列：
         - 带长度前缀的 X.509 `certificate`（ASN.1 DER 格式）
      -  `minSDK` (uint32) - 如果平台版本低于此数字，则应忽略该签名者。
      -  `maxSDK` (uint32) - 如果平台版本高于此数字，则应忽略该签名者。
      -  带长度前缀的additional attributes（带长度前缀）序列：
         - `ID` (uint32)
         - `value`（可变长度：附加属性的长度 - 4 个字节）
         - `ID - 0x3ba06f8c`
         - `value -` Proof-of-rotation 结构
   -  `minSDK` (uint32) - 签名数据部分中 minSDK 值的副本 - 用于在当前平台不在范围内时跳过对此签名的验证。必须与签名数据值匹配。
   -  `maxSDK` (uint32) - 签名数据部分中 maxSDK 值的副本 - 用于在当前平台不在范围内时跳过对此签名的验证。必须与签名数据值匹配。
   -  带长度前缀的signatures（带长度前缀）序列：
      - `signature algorithm ID` (uint32)
      - `signed data` 带长度前缀的 `signature`
   -  带长度前缀的 `public key`（SubjectPublicKeyInfo，ASN.1 DER 格式）

### 4.3 Proof-of-rotation 和 self-trusted-old-certs 结构

proof-of rotation 结构允许应用轮替其签名证书，而不会使这些证书在与这些应用通信的其他应用上被屏蔽。为此，应用签名包含两个新数据块：

- 告知第三方应用的签名证书可信（只要其先前证书可信）的断言
- 应用的旧签名证书（应用本身仍信任这些证书）

签名数据部分中的 proof-of-rotation 属性包含一个单链表，其中每个节点都包含用于为之前版本的应用签名的签名证书。此属性旨在包含概念性 proof-of-rotation 和 self-trusted-old-certs 数据结构。该单链表按版本排序，最旧的签名证书对应于根节点。在构建 proof-of-rotation 数据结构时，系统会让每个节点中的证书为列表中的下一个证书签名，从而为每个新密钥提供证据来证明它应该像旧密钥一样可信。

在构造 self-trusted-old-certs 数据结构时，系统会向每个节点添加标记来指示它在集合中的成员资格和属性。例如，可能存在一个标记，指示给定节点上的签名证书可信，可获得 Android 签名权限。此标记允许由旧证书签名的其他应用仍被授予由使用新签名证书签名的应用所定义的签名权限。由于整个 proof-of-rotation 属性都位于 v3 `signer` 字段的签名数据部分中，因此用于为所含 APK 签名的密钥会保护该属性。

此格式排除了[多个签名密钥](https://source.android.google.cn/security/apksigning/v3?hl=zh-cn#multiple-certificates)的情况和将[不同祖先签名证书](https://source.android.google.cn/security/apksigning/v3?hl=zh-cn#multiple-ancestors)收敛到一个证书的情况（多个起始节点指向一个通用接收器）。

**格式**

proof-of-rotation 存储在“APK 签名方案 v3 分块”内，ID 为 `0x3ba06f8c`。其格式为：

- 带长度前缀的levels（带长度前缀）序列：
  - 带长度前缀的signed data（由上一个证书签名 - 如果存在）
    - 带长度前缀的 X.509 `certificate`（ASN.1 DER 格式）
    - `signature algorithm ID` (uint32) - 上一级证书使用的算法
  - `flags` (uint32) - 这些标记用于指示此证书是否应该在 self-trusted-old-certs 结构中，以及针对哪些操作。
  - `signature algorithm ID` (uint32) - 必须与下一级中的签名数据部分的 ID 一致。
  - 上述 `signed data` 的带长度前缀的 `signature`

**多个证书**

Android 目前将使用多个证书签名的 APK 视为具有与所含证书不同的签名身份。因此，签名数据部分中的 proof-of-rotation 属性构成了一个有向无环图，最好将其视为单链表，其中给定版本的每组签名者都表示一个节点。这使得 proof-of-rotation 结构（下面的多签名者版本）更复杂。排序成为一个特别突出的问题。更重要的是，无法再单独为 APK 签名，因为 proof-of-rotation 结构必须让旧签名证书为新的证书集签名，而不是逐个签名。例如，如果希望由两个新密钥 B 和 C 签名的 APK 是由密钥 A 签名的，则它不能让 B 签名者仅包含 A 或 B 的签名，因为这是与 B 和 C 不同的签名身份。这意味着签名者必须在构建此类结构之前进行协调。

**多个签名者 proof-of-rotation 属性**

- 带长度前缀的sets（带长度前缀）序列：
  - signed data（由上一组证书签名 - 如果存在）
    - 带长度前缀的certificates序列
      - 带长度前缀的 X.509 `certificate`（ASN.1 DER 格式）
    - `signature algorithm IDs `(uint32) 序列 - 上一组证书中的每个证书对应一个序列，且采用相同顺序。
  - `flags `(uint32) - 这些标记用于指示这组证书是否应该在 self-trusted-old-certs 结构中，以及针对哪些操作。
  - 带长度前缀的signatures（带长度前缀）序列：
    - `signature algorithm ID` (uint32) - 必须与签名数据部分中的相应 ID 一致
    - 上述 `signed data` 带长度前缀的 `signature`

**proof-of-rotation 结构中有多个祖先**

v3 方案也无法处理两个不同密钥轮替到同一个应用的同一签名密钥的情形。这不同于收购情形，在收购情形中，收购公司希望转移收购的应用以使用其签名密钥来共享权限。收购被视为受支持的用例，因为新应用将通过其软件包名称来区分，并且可以包含自己的 proof-of-rotation 结构。不受支持的用例是，同一应用有两个不同的路径指向相同的证书，这打破了在密钥轮替设计中做出的许多假设。

**验证**

在 Android 9 及更高版本中，可以根据 APK 签名方案 v3、v2 或 v1 验证 APK。较旧的平台会忽略 v3 签名并尝试验证 v2 签名，然后验证 v1。

![APK 签名验证过程](https://source.android.google.cn/security/images/apk-validation-process.png?hl=zh-cn)

### 4.4 APK 签名方案 v3 验证

1. 找到“APK 签名分块”并验证以下内容：
   1. “APK 签名分块”的两个大小字段包含相同的值。
   2. “ZIP 中央目录结尾”记录紧跟在“ZIP 中央目录”后面。
   3. “ZIP 中央目录结尾”之后没有任何数据。
2. 找到“APK 签名分块”中的第一个“APK 签名方案 v3 分块”。 如果 v3 分块存在，则继续执行第 3 步。否则，回退至[使用 v2 方案](https://source.android.google.cn/security/apksigning/v2?hl=zh-cn#v2-verification)验证 APK。
3. 对“APK 签名方案 v3 分块”中的每个signer（最低和最高 SDK 版本在当前平台的范围内）执行以下操作：
   1. 从 `signatures` 中选择安全系数最高的受支持 `signature algorithm ID`。安全系数排序取决于各个实现/平台版本。
   2. 使用 `public key` 并对照 `signed data` 验证 `signatures` 中对应的 `signature`。（现在可以安全地解析 `signed data` 了。）
   3. 验证签名数据中的最低和最高 SDK 版本是否与为 `signer` 指定的版本匹配。
   4. 验证 `digests` 和 `signatures` 中的签名算法有序 ID 列表是否相同。（这是为了防止删除/添加签名。）
   5. 使用签名算法所用的同一种摘要算法[计算 APK 内容的摘要](https://source.android.google.cn/security/apksigning/v2?hl=zh-cn#integrity-protected-contents)。
   6. 验证计算出的摘要是否与 `digests` 中对应的 `digest` 一致。
   7. 验证 `certificates` 中第一个 `certificate` 的 SubjectPublicKeyInfo 是否与 `public key` 相同。
   8. 如果 `signer` 存在 proof-of-rotation 属性，则验证结构是否有效，以及此 `signer` 是否为列表中的最后一个证书。
4. 如果在当前平台范围内仅找到了一个 `signer`，并且对该 `signer` 成功执行第 3 步，则验证成功。

**注意**：如果第 3 步或第 4 步失败，则不得使用 v1 或 v2 方案验证 APK。

## 5. APK签名方案总结

 Android的应用程序apk的签名是用来确保apk来源的真实性以及apk没有被第三方篡改，目前有两种签名方案：V1和V2。

V1签名方案与V2方案对比如下：

|    对比项    |  V1  |  V2  |
| :----------: | :--: | :--: |
| 签名验证速度 |  慢  |  快  |
|    完整性    |  低  |  高  |

- 签名验证速度

V1签名方案验证过程中需要对apk中所有文件进行摘要计算，在apk资源很多、性能较差的机器上签名校验会花费较长时间，导致安装速度慢；V2签名方案利用zip文件格式做签名数据的存储和验证，可以使得验证过程加快。使用分块方式对数据做摘要计算，加速了签名过程。

- 完整性


V1签名方案META-INF目录用来存放签名，此目录本身是不计入签名验证过程的，可以随意在这个目录中添加文件，比如一些快速批量打包方案就选择在这个目录中添加渠道文件；V2对于全部原始内容区域做完整性保证，增加了完整性的保证。

但是我们注意到，android识别签名块数据，是识别的该区块内ID为0x7109871a的值，其他值是忽略的，因此可以自定义ID值写入这个区域，就可以实现在V2签名方案打渠道包，美团公司实现的V2打渠道包方式就是利用该原理实现的。