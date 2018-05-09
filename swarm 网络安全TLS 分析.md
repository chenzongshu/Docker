
# TLS 基础

## 数字签名过程

发送人将发送内容使用HASH算法生成一个内容摘要(digest) 
---> 内容摘要(digest)使用私钥进行加密
---> 生成数字签名(signature) 
---> 数字签名(signature)附在信件下面 
---> 将信件发送给收件人 ----> 收件人收到信件，并使用发送人的公钥进行解密：证明信件来自于发件人 
---> 收件人使用同种Hash算法计算发送内容的摘要(digest)，并和解密得到的digest进行对比：一致则证明信件内容没有别修改

一次数字签名涉及到一个哈希函数、发送者的公钥、发送者的私钥（不在签名里）。
数字签名使用私钥对内容摘要用私钥进行加密；加密通信过程中，使用公钥加密，私钥进行解密

## 数字中心(CA)

验证公钥的正确性，为公钥做认证，使用自己的私钥为其他公钥和一些信息进行加密；  生成"数字证书"Digital Certificate。

## 签名证书

CA进行对公钥和其他一些内容进行数字签名的信件或消息。

常用扩展名：

- .crt 证书文件 ，可以是DER（二进制）编码的，也可以是PEM（ ASCII (Base64) ）编码的 ，在类unix系统中比较常见
- .cer 也是证书 常见于Windows系统 编码类型同样可以是DER或者PEM的，windows 下有工具可以转换crt到cer
- .csr 证书签名请求 一般是生成请求以后发送给CA，然后CA会给你签名并发回证书.key 一般公钥或者密钥都会用这种扩展名，可以是DER编码的或者是PEM编码的 查看DER编码的（公钥或者密钥）的文件的命令为 openssl rsa -inform DER -noout -text -in xxx
- .key 查看PEM编码的（公钥或者密钥）的文件的命令为 openssl rsa -inform PEM -noout -text -in xxx.key

# swarm TLS加固分析

Docker中内置的集群模式公钥基础设施（PKI）系统使得安全部署容器编排系统变得非常简单。
集群中的节点使用传输层安全性（TLS）来认证，授权和加密与集群中其他节点的通信。

运行docker swarm init来创建swarm时，Docker将自己指定为管理节点。
默认情况下，管理节点会生成一个新的根证书颁发机构（CA）和一个密钥对，用于保护与加入集群的其他节点的通信。

也可以使用docker swarm init命令的--external-ca标志指定外部生成的根CA.

当其他节点加入到集群时，管理节点还会生成两个令牌：一个工作节点令牌和一个管理节点令牌。
每个令牌都包含根CA证书的摘要和随机生成的密钥。当节点加入群集时，加入节点使用摘要来验证来自远程管理节点的根CA证书。

远程管理节点使用该秘钥来确保加入节点是批准的节点。

每当新节点加入群时，管理节点就会向节点发出证书。证书包含一个随机生成的节点ID，用于标识证书公用名称（CN）下的节点和组织单位（OU）下的角色。

节点ID用作当前集群中节点生存期的密码安全节点标识(swarm-node.crt)。
当一个节点加入到集群的时候，首先要检查本地是否存在根证书CA，如果存在则用TLS文件进行校验。如果不存在则调用grpc 接口，从远端获取ca证书，并保存到本地。
ca证书名称为sarm-root-ca.crt。

下一步加载本地 TLS证书，如果存在则获取TLS中的NodeID;如果证书不存在，则向加入到的管理节点请求TLS证书。

针对加载TLS失败场景分两种情况：
（1）rootca 满足rca.Cert == nil || rca.Pool == nil || rca.Signer == nil 则需要从远端请求
新的TLS证书，证书中包含新的nodeID
（2）不满足1的条件，则在本地 生成cn := identity.NewID()、org := identity.NewID()
cn为NodeID。

swarm-root-ca.crt 和swarm-root-ca.key是leader节点生成的。其他节点请求swarm-root-ca.crt。

每个节点，生成秘钥对，私钥保存为swarm-node.key，然后使用公钥经过一系列处理，生成证书请求，leader 产生证书（证书包含一个随机生成的节点ID，用于标识证书公用名称（CN）下的节点和组织单位（OU）下的角色），用swarm-root-ca.key加密返回给节点。

节点用swarm-root-ca.crt解密验证完整性和安全性。此证书保存为swarm-node.crt。


## 概述

swarm 集群间个节点通信，先进行身份验证，再进行加密通信。
执行命令：

```
docker swarm init --advertise-addr
```

在 `${docker-root}/ /swarm/certificates/` 目录下生成四个文件

```
[root@localhost certificates]# ls
swarm-node.crt swarm-node.key swarm-root-ca.crt swarm-root-ca.key
```

文件说明：
> - swarm-root-ca.crt：CA的自签名证书（执行docker swarm init 节点会启动CA(证书中心)线程，为其他节点颁发签名证书）；
- swarm-root-ca.key：CA的私钥；
- swarm-node.crt：这个文件不是一个证书而是一个证书集合，包含：当前节点证书和证书中心CA的证书;包含CA的证书的目的是为了
- swarm-node.key：当前节点的私钥；


docker swarm (LINUX下)的证书使用pem格式，查看证书信息：

```
[root@localhost certificates]# openssl x509 -in swarm-node.crt -inform pem -noout -text
Certificate:
Data:
Version: 3 (0x2)
Serial Number:
24:45:68:8a:07:b7:e2:e1:2c:51:c0:be:43:da:1f:54:fe:09:4e:47
Signature Algorithm: ecdsa-with-SHA256
Issuer: CN=swarm-ca //证书颁发结构
Validity
Not Before: Feb 11 00:24:00 2018 GMT
Not After : May 12 01:24:00 2018 GMT
Subject: O=cvp2afdvt625g5ddhds5t9ldq, OU=swarm-manager, CN=83lp2qgjndo09mgpf96k4jkom //拥有者信息
Subject Public Key Info:
Public Key Algorithm: id-ecPublicKey
Public-Key: (256 bit)
pub: 
04:9e:27:3c:81:2c:ea:99:f4:0c:45:c8:96:a4:7d:
94:5f:58:14:a9:98:61:a3:31:0a:ab:16:b5:7c:31:
31:a1:57:57:60:47:71:77:64:e6:99:53:25:63:02:
c0:b7:7f:1a:eb:78:92:9b:4b:94:a8:50:a2:f5:c2:
8a:1b:81:d1:77
ASN1 OID: prime256v1
```

由上信息，我们可以看出签名证书中保护多种信息，比如证书持有者、证书颁发者、证书的公钥的散列值等，具体证书格式和内容参考文章《OpenSSL 与 SSL 数字证书概念贴》；

docker swarm .key文件保留着当前节点的私钥，查看私钥：

```
[root@localhost certificates]# cat swarm-node.key 
-----BEGIN EC PRIVATE KEY-----
MHcCAQEEIEyyW3De4ix3KOdfv/DtxwtvykdTT9Wla7uUxlus4DjEoAoGCCqGSM49
AwEHoUQDQgAEnic8gSzqmfQMRciWpH2UX1gUqZhhozEKqxa1fDExoVdXYEdxd2Tm
mVMlYwLAt38a63iSm0uUqFCi9cKKG4HRdw==
-----END EC PRIVATE KEY-----
```

docker 加入swarm 集群，执行命令：

```
docker swarm join --token ${token-string} host-ip:port
```

命令执行成功后，在当前节点的 `${docker-root}/ /swarm/certificates/`  目录下出现三个文件

```
[root@localhost certificates]# ls
swarm-node.crt swarm-node.key swarm-root-ca.crt
```

>文件说明：
- swarm-root-ca.crt：从执行swarm init命令的节点下载的CA证书文件，使用来验证其他节点的签名证书的完整性、安全性
- swarm-node.key：当前节点的x509私钥
- swarm-node.crt：当前节点的公钥签名证书，颁发机构为 swarm-root-ca -→ 执行swarm init 的节点

# docker swarm certificates 生成

## 执行 docker swarm init 的节点各证书生成

swarm-root-ca.crt swarm-root-ca.key
证书中心(CA)的自签名证书文件，由leader节点生成。


- 执行swarm init 时，会在leader节点启动ca.server协程，响应其他节点的证书请求，颁发证书；
- 若不适用外部CA，生成的swarm-root-ca.key会保留在本地，而swarm-root-ca.crt会分发给其他节点（其他节点join的时候），所以集群的每个node节点上的swarm-root-ca.crt都是一样的；

## 执行swarm join 节点证书

1. swarm-root-ca.crt
swarm-root-ca.crt文件不是当前节点生成的，从CA.server下载（默认leader节点会启动CA.server进程服务）。由于是第一次连接，没有TLS安全机制进行检测，为了保证根CA证书的安全、完整性，使用 JoinToken 散列验证证书；
      
2. swarm-node.crt swarm-node.key
这两个证书文件和leader节点的同名的两个证书生成过程类似，只不过此节点的证书请求CSR通过TLS加密通道发送个leader节点（通信前，会验证leader节点的身份），由leader节点的 Ca.server 使用私钥（swarm-root-ca.key）签名生成签名证书，再有leader节点发送给新加入节点；

## docker swarm 节点间TLS加密通信

swarm 集群多个集群通信之前，像正常加密通信方式一样，需要先使用证书验证对方身份，然后发送端使用公钥对内容进行加密，收取端使用私钥进行解密；

TLS验证对方身份:
golang 已经为我们封装好了很多加密方法，放在crypto库中。
为了减少持有互相的公钥，docker swarm 使用了golang的MutableTLS机制，此部分涉及到MutableTL验证和很多库引用，待以后分析；


# swarm TLS安全加固源码分析

## swarm JoinToken

### JoinToken产生

JoinToken产生调用链（JoinToken由Leader产生）：

```
/manager/manager.go: Run
/ca/certificates.go: GenerateJoinToken
JoinToken查询：
docker swarm join-token work/manager
work节点和manager节点的JoinToken值不同，其他节点加入集群时，凭借JoinToken值区分work或是manager
```

### JoinToken使用

当我们将一个节点加入一个集群时，需执行命令：
`docker swarm join --token ${token-string} host-ip:port`

swarm token的作用：在加入集群前，当前节点没有swarm集群的根证书中心(root-ca)的证书，无法根据TLS安全认证验证数据的root-ca的证书完整、安全性，token的作用就是使用散列来验证根CA证书的完整、安全性；

token-string -> Digest(split(token)[2]) -> digest.NewDigestVerifier() 获取verifer算法：获取相应的hash算法：
SHA256 Algorithm = "sha256" // sha256 with hex encoding
SHA384 Algorithm = "sha384" // sha384 with hex encoding
SHA512 Algorithm = "sha512" // sha512 with hex encoding
根据相应的 hash 算法，获取token中的hash值于证书从新计算出的值进行对比验证。
