## SSL/TLS

基于TCP/IP，提供一个安全的握手来初始化一个TCP/IP连接，完成客户端和服务器之间关于安全等级、密码算法、通信密钥的协商，以及执行对连接端身份的认证工作。在连接建立之后，SSL连接所传送的应用层协议数据否会被加密，从而摆正通信的机密性。协议结构上包括：握手协议、告警协议、修改密文协议和记录协议，其中最复杂和最重要的是握手协议。

![img](./assets/SSL-protocol-5.png)

1. 先通过三次握手，建立 TCP 连接
2. 客户端发送 client_hello 请求报文，其中包括客户端支持的 `SSL` 版本号、产生的随机数、会话ID、密码算法列表、压缩算法列表等信息。
3. 服务端响应 server_hello 报文，其中包括服务端支持 `SSL` 版本号、产生随机数、会话ID、客户端列表中选出的密码算法、选定的压缩算法。
4. 服务器发送 CA 中心签发的 `X509` 证书，包括服务器信息和公钥，公钥和后续的密钥交换有关
5. 服务器发送 certificate_request 请求客户端提供证书进行双向认证，但由于客户端一般是匿名的，所以该消息可选
6. 服务器发送 server_hello_done：完成通信，等待客户端应答
7. 如果进行双向认证，客户端响应 certificate_request 报文，发送自己的 `X509` 证书到服务器
8. 客户端发送 client_key_exchange，即客户端密钥交换消息，用于约定后续通信中的对称加密算法，该消息包含一个使用服务器的 `RSA` 公钥加密或 DH 算法运算得到的数值
9. 若用户提供了证书，客户端发送 `certifacate_verify` 消息，作为客户端签名
10. 客户端发送 change_cipher_spec 消息，表示将使用选定的密码算法和参数进行将来的通信。
11. 客户端发送 finished 消息，完成通信
12. 服务端发送 change_cipher_spec 消息，表示将使用与客户端一样的的密码算法和参数进行将来的通信。
13. 服务端发送 finished 消息，完成通信

![img](./assets/ssl_rsa.png)

1. 客户端发送随机数到服务器
2. 服务器发送另一个随机数和携带公钥信息的证书到客户端
3. 客户端产生预主密钥（`pre_master_key`），并用服务器的公钥加密得到`encrypted pre_master_key`
4. 客户端将`encrypted pre_master_key`发送到服务器
5. 服务器用自己的私钥解密`encrypted pre_master_key`得到预主密钥
6. 双方分别使用`pre_master_key`、`ClientHello.random`和`ServerHello.random`生成会话密钥（session_key）
7. 双发使用会话密钥进行通信

## 常见问题

### HTTPS 和HTTP的区别

- 端口
- 安全
- 协议结构