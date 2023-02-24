# 如何跟踪客户端的IP？
由于后端服务一般部署在反向代理（NGINX）后，如果直接通过`request.GetRemoteAddr()`获取的IP一般是反向代理服务器的地址.
所以我们需要通过HTTP header 中的 `X-Forwarded-For`字段，如：`X-Forwarded-For: client1, proxy1, proxy2`，具体做法如下:
1. 业务侧取Header中的字段`request.getHeader("X-Forwarded-For")`
2. 在NGINX中设置`proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;`,让nginx在header中赋值.


# HTTP1.1

# HTTP2

# HTTP3



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

