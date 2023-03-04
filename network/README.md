# 常用的网络命令

* `netstat -na |grep 80`  查看网络情况
* `nc host port  -p12345 `  本地使用12345端口连接host:port
* `ss -lnt ` ：查看全连接队列等情况





## TCP 序号绕回

seq 是有4字节组成，理论上会绕回，这就可能导致丢失重传的报文序号刚好会在接收串口里面，被错误的接收

解决方式：

* 开启TCP timestamp. 已能精确计算RTT, 二能防止序列回绕（PAWS）





# TCP 半连接队列&全连接队列

<img src="asset/tcp-syn.png" alt="img" style="zoom:50%;" />

## 半连接满了

后续过来的syn包都丢弃，(攻击者发送大量syn 请求，也即syn攻击)，服务端可以开启 syncookie 功能

> syncookies 是这么做的：服务器根据当前状态计算出一个值，放在己方发出的 SYN+ACK 报文中发出，当客户端返回 ACK 报文时，带上服务端的syncookies,服务端取出该值验证，如果合法，就认为连接建立成功，如下图所示。

应对syn 攻击：

* 开启 syncookie

* 增大半连接队列

  > **要想增大半连接队列，我们不能只单纯增大 tcp_max_syn_backlog 的值，还需一同增大 somaxconn 和 backlog，也就是增大全连接队列**。否则，只单纯增大 tcp_max_syn_backlog 是无效的。

* 减少syn+ack 重传次数

  > 攻击者发送大量的syn,导致服务器大量处于SYN_RECV 状态，服务端会重传SYN_ACK ，到达一定次数后，才会断开连接，减少重传次数，可以快速断开连接

## 全连接队列满了

在服务端并发处理大量请求时，如果 TCP accpet 队列过小，或者应用程序调用 accept() 不及时，就会造成 accpet 队列满了 ，这时后续的连接就会被丢弃，这样就会出现服务端请求数量上不去的现象。

> 通过 ss -lnt 查看accept 队列情况，-t 只显示tcp， -n 不解析服务器名称 ，-l 显示正在listen的socket
>
> - Recv-Q：当前 accpet 队列的大小，也就是当前已完成三次握手并等待服务端 `accept()` 的 TCP 连接个数；
> - Send-Q：当前 accpet 最大队列长度
>
> 当 Recv-Q 的大小超过 Send-Q，就说明发生了 accpet 队列满的情况

解决方式：

* 调大全连接兑现长度。调大 backlog 以及 somaxconn 参数 
* 优化代码，提高accept速率



# 建立连接的TCP 收到SYN 怎么办

 比如客户端宕机重启，和服务端重新建立，此时刚好四元组和原来的一样。且seq(每次建立连接，seq 初始值都是随机的，就是为了避免落到原来连接的窗口内，被接收处理，当然这正是针对非正常关闭连接或宕机的情况，不然正常time wait 等待2MSL ,报文也消亡了 ) 则服务端收到SYN 时，会应答原来期望的seq(即宕机前客户端的seq),称为 **challenge ACK** ,客户端收到发现ack seq 不是自己期望的，就会发RST 释放连接

有两个工具可以关闭TCP连接：

* killcx   原理通过发送SYN 获得服务端期望的client seq ,然后发送RST
* tcpkill  原理监听连接，获取对方的seq,然后发送RST. (不适用与不活跃的连接，及双方没有通讯也获取不到seq)



# TCP TIME_WAIT

TIME_WAIT 发生主动关闭连接的一方。为什么要等待2MSL

1. 防止历史连接中的消息被后面**相同四元组**的连接错误的接收
2. 保证被动关闭的一方正确的关闭

如何减少TIME_WAIT

* tcp_tw_reuse(只适用发起方)

> 在调用 connect() 函数时，**如果内核选择到的端口，已经被相同四元组的连接占用的时候，就会判断该连接是否处于 TIME_WAIT 状态，如果该连接处于 TIME_WAIT 状态并且 TIME_WAIT 状态持续的时间超过了 1 秒，那么就会重用这个连接，然后就可以正常使用该端口了**。所以该选项只适用于连接发起方

* tcp_tw_recycle（服务端客户端都能适用）（4.x 内核已经废弃该选项）( NAT 的网络下是不安全的)

> 对于服务器来说，如果同时开启了recycle 和 timestamps 选项，则会开启一种称之为「 per-host 的 PAWS 机制」
>
> **per-host 是对「对端 IP 做 PAWS 检查」**，而非对「IP + 端口」四元组做 PAWS 检查。
>
> 所以当客户端在路由器后（经过NAT网络），导致服务器在收到请求是由于各个连接的时间戳不一致，会拒绝较小时间戳的连接（相同四元组）

要使得上面这两个参数生效，有一个前提条件，就是要打开 TCP 时间戳，即 net.ipv4.tcp_timestamps=1（默认即为 1)

*  便于精确计算 RTT 
* 防止序列号回绕（PAWS）



# 如何跟踪客户端的IP？

由于后端服务一般部署在反向代理（NGINX）后，如果直接通过`request.GetRemoteAddr()`获取的IP一般是反向代理服务器的地址.
所以我们需要通过HTTP header 中的 `X-Forwarded-For`字段，如：`X-Forwarded-For: client1, proxy1, proxy2`，具体做法如下:
1. 业务侧取Header中的字段`request.getHeader("X-Forwarded-For")`
2. 在NGINX中设置`proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;`,让nginx在header中赋值.

# HTTP常见状态吗

* 304 资源未修改，客户端直接使用缓存

- 301 永久重定向/302 临时重定向
- 401 鉴权失败
- 403 服务器禁止资源访问
- 500 笼统服务器错误
- 502 Bad Gateway 后端服务错误
- 503 Service unavilable 当前服务忙，无法应答客户端



# HTTPS

常用的加密算法有RSA和ECDHE(具备前向安全性，广泛使用)

备注: AES 对称加密 RSA 非对称加密

**前向安全性 **： 是密码学中通讯协议的安全属性，指的是长期使用的主密钥泄漏不会导致过去的会话密钥泄漏 

下面RSA 算法一直使用服务端的同一对公私钥。一旦服务私钥破解，则之前的全部通信都可以解密pre-master,而 客户端和服务端的随机数又是明文传输，则可以算出对称秘钥。也即破解了全部通信。具备前向安全的加密算法，若被破解，应当直接破解当次通信，不能破解以往的加密报文。DH升级版ECDHE算法具备前向安全性

## HTTPS 优化速度

* 计算密集型，提升CPU ,或硬件支持算法的CPU
* 升级操作系统，OpenSSL
* 升级协议，使用ECDHE，选择 x25519 曲线；对称加密秘钥AES_128_GCM（安全性不高不需选择256），升级TLS 1.3(1RTT)，证书算法也选择ECDHE, 使用OCSP Stapling，避免向CA检验证书；
* 复用SessionID ,不用每次都重新计算对称秘钥，在缓存时间内使用上次的秘钥（用sessionID标识）

## TLS1.2 (2RTT)

<img src="asset/tls1.2握手.png" alt="img" style="zoom:50%;" />

1. 客户端生成随机数r1，TLS 版本，密码套件列表
2. 服务端生成随机数r2, 指定密码套件，服务端证书
3. 校验服务端整数，客户端生成pre master(记为r3),用服务器公钥加密返回服务端
4. 服务端解密获得pre master,此时双方就可以生成 对称秘钥，或许加密使用这个对称秘钥



备注：

DH 算法可以简单理解为客户端生成 公私钥，并把公钥发给服务端，服务端也生成公私钥，并把公钥发给客户端，客户端用自己的私钥和服务端的公钥生成对称秘钥，同时服务端用自己的私钥和客户端的公钥生成对称秘钥（这个对称秘钥和客户端生成的一样，DH 算法保证的）

由于DH 算法在第二次握手之后，双方都已经可以生成对称秘钥了，因此真正的数据可以在第三次握手的时候一起传过去（也即实现抢跑，在TLS1.3中协议实现）

## TLS1.3(1RTT)

TLS 1.3 只支持 ECDHE 的密钥协商算法

引入了新的密钥协商机制 — PSK

支持 0-RTT 数据传输(恢复连接)，在建立连接时节省了往返时间

# HTTP1.1

* keep alive 支持长连接
* 支持pipeline ,第一个请求发出后，不需要等待应答，就可以发出第二个请求。但是服务端一定要按照顺序处理并返回客户端

# HTTP2

* 头部压缩。 HPACK 算法，客户端和服务端同时维护一张头信息表，每个传输只需要索引号

* 二进制格式。二进制帧格式，含 header frame 和 data frame

* 并发传输。

  HTTP1.1 虽然支持pipeline,但是存在HTTP协议的队头阻塞问题，HTTP2 引入Stream, 多个Stream 在一个TCP 中。消除了HTTP层的队头阻塞，但是还是存在TCP层的队头阻塞。这里一个Stream可以看成HTTP1 中的一个连接。

* 服务器主动推送资源。服务端可以提前将资源推送客户端（css,js）,后续客户端就不需要向服务端请求

# HTTP3

传输层使用QUIC,有以下特点

* 无（TCP层）队头阻塞。某个Stream数据包到了可以先处理。HTTP2中TCP丢包，会阻塞在操作系统内核，无法将后续到达的Stream  移交上层应用层 

* 更快的建立连接（使用UDP,不需要TCP三次握手）

* 连接迁移（切换网络不需要重新建立连接，直接使用上次的连接ID 即可）

  <img src="asset/quic.png" alt="img" style="zoom:67%;" />

# OAuth2.0
## 背景

首先了解两个单词authentication(认证) and authorization(授权)，OAuth主要解决用户的授权。如云冲印应用想要使用google云相册的功能。有种方式就是直接将 账号密码托管给 云冲印，然后云冲印登录google 云相册获取用户的照片打印。这样有几个安全问题是：

* 用户的账号密码泄露给云冲印
* google 云相册得支持账号密码登录（这种登录方式不安全，现在一般都会加验证码，手机验证等）
* 无法限制云冲印只访问指定的资源，可能会偷偷范问其他资源
* 用户无法收回权限，只能修改账号密码
* 若用户在多个app 想要访问google 云相册，需要多次托管密码，只要有一次泄露，就会导致用户信息泄露

## OAuth是什么

上面的云冲印记为客户端，云相册记为服务提供商，OAuth是让客户端不能直接登录服务提供商，而是用户在服务提供商登录授权个客户端，并颁发一个token给客户端，这个token记录了访问权限和有效期，后续客户端仅通过这个token来服务提供商获取用户的指定资源

## 运行流程

1. 客户端向用户申请授权
2. 用户同意授权
3. 客户端向服务提供商申请授权
4. 服务提供商授权，并发token
5. 客户端通过token向服务提供商获取资源
6. 服务提供商提供资源

## 客户端支持的授权模式

- **授权码模式（authorization code）**(常用)
- 简化模式（implicit）
- 密码模式（resource owner password credentials）
- 客户端模式（client credentials）

## 授权码模式

前提：每个客户端使用授权码模式之前，需要在服务方开发者页面中登记，并获得clientID 和私钥等信息

1. 用户进入客户端网页，跳转到服务提供商的页面进行登录

   如访问服务提供商的授权接口，接口如下，client_id 需要告诉服务方是哪个客户端

   ```http
   GET /authorize?response_type=code&client_id=s6BhdRkqt3&state=xyz
           &redirect_uri=https%3A%2F%2Fclient%2Eexample%2Ecom%2Fcb HTTP/1.1
   Host: server.example.com
   ```

2. 在服务提供商页面登录后，点击授权，并返回客户端（附带返回授权码）

   如服务提供方登录之后，重定向之前客户端的url

   ```http
   HTTP/1.1 302 Found
   Location: https://client.example.com/cb?code=SplxlOBeZQQYbYS6WxSbIA
             &state=xyz
   ```

3. 客户端带上之前的redirect url 和 授权码向服务提供商获取 token

   ```http
   POST /token HTTP/1.1
   Host: server.example.com
   Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
   Content-Type: application/x-www-form-urlencoded
   
   grant_type=authorization_code&code=SplxlOBeZQQYbYS6WxSbIA
   &redirect_uri=https%3A%2F%2Fclient%2Eexample%2Ecom%2Fcb
   ```

4. 服务提供商验证了redirect url 和授权码后，向客户端发token(access token 和refresh token)

   ```http
    HTTP/1.1 200 OK
        Content-Type: application/json;charset=UTF-8
        Cache-Control: no-store
        Pragma: no-cache
   
        {
          "access_token":"2YotnFZFEjr1zCsicMWpAA",
          "token_type":"example",
          "expires_in":3600,
          "refresh_token":"tGzv3JOkF0XG5Qx2TlKWIA",
          "example_parameter":"example_value"
        }
   ```

## 简化模式

相对授权码模式，省略了获取 授权码的过程，直接向服务提供商申请了token

1. 客户端向服务方申请token

```http
GET /authorize?response_type=token&client_id=s6BhdRkqt3&state=xyz
        &redirect_uri=https%3A%2F%2Fclient%2Eexample%2Ecom%2Fcb HTTP/1.1
    Host: server.example.com
```

2. 服务方后的用户登录授权后返回token

```http
 HTTP/1.1 302 Found
     Location: http://example.com/cb#access_token=2YotnFZFEjr1zCsicMWpAA
               &state=xyz&token_type=example&expires_in=3600
```



## 密码模式

1. 用户将服务方的账号密码提供给客户端
2. 客户端将账号密码发给服务方获取token
3. 服务方验证后，返回token

## 客户端模式

备注：这种模式其实不存在服务端授权的问题，通常需要客户端和服务提供商互相信任和签订协议的情况下使用

1. 用户在客户端登录后，客户端直接向服务方申请token
2. 服务方仅校验客户端信息，就把用户的token 返回给客户端

