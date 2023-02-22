# 如何跟踪客户端的IP？
由于后端服务一般部署在反向代理（NGINX）后，如果直接通过`request.GetRemoteAddr()`获取的IP一般是反向代理服务器的地址.
所以我们需要通过HTTP header 中的 `X-Forwarded-For`字段，如：`X-Forwarded-For: client1, proxy1, proxy2`，具体做法如下:
1. 业务侧取Header中的字段`request.getHeader("X-Forwarded-For")`
2. 在NGINX中设置`proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;`,让nginx在header中赋值.


# HTTP1.1

# HTTP2

# HTTP3
