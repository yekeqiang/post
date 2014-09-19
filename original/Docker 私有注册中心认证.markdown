# Docker 私有注册中心认证

标签（空格分隔）： Docker registry private authentication API

---

安全已经融入了我们的生活。我们锁门，使用密码保护我们的银行信息，但是通常密码如此复杂以至于造成我们很容易忘记它。用常识来保护系统的安全是良好的实践。这真的很容易呈现，因为它是一个内部的系统，没有必要启用身份认证以及安全传输，但在我们当前的远程工作时代，内部网络可能十分广泛。

考虑到这一点，我们花费了一些时间审查了我们维护的多种系统，并在这星期，我给我们的私人 Docker 注册中心设置了添加身份认证的目标。你也许知道，Docker 注册没有提供身份认证的方法。因此我们决定早期的方案就是在我们的镜像仓库前面加一个身份认证的代理。在我们的案例中，我们决定通过 SSL 使用 Nginx 并加上一个内部身份认证 API。

![此处输入图片的描述][1]


该解决方案的几个优势：

  - 允许我们使用我们内部的身份验证 API
  - 可以给其他系统重复使用身份认证
  - 可以通过 Docker 容器实现（我们使用的容器数量小于 3 个）

我把一个简单的身份认证服务和一个 Nginx 容器放在一起，我们可以在 github 上获取可用的版本（https://registry.hub.docker.com/u/opendns）。


## 简单的基础身份认证服务

作为 Nginx 代理容器的一个参考，我们使用 NodeJS 构建了一个身份认证 API，非常容易的创建了一个基础身份认证服务。所有我们所需要做的就是创建一个真正简单的 [server.js][2]，使用 [htpasswd utility][3] 生成一个资格证书文件，并且在一个 Docker 容器中封装，我们可以创建如下 [Dockerfile][4]：

```
FROM google/nodejs 
ADD . /app 
WORKDIR /app 
RUN npm install http-auth 
EXPOSE 8000 
ENV NODE_PATH /data/node_modules/ 
CMD ["node", "server.js"]
```

然后我们测试并部署我们的服务：

```
ubuntu@trusty-64:/basic-auth# docker build -t opendns/basic-auth-service . 

ubuntu@trusty-64:/basic-auth# docker run --name simple-auth opendns/basic-auth-service 

ubuntu@trusty-64:/basic-auth# docker inspect --format '{{ .NetworkSettings.IPAddress }}' simple-auth 
172.17.0.40 

ubuntu@trusty-64:/basic-auth# curl 172.17.0.40:8000 
401 Unauthorized 

ubuntu@trusty-64:/basic-auth# curl -u testuser:testpassword 172.17.0.40:8000 
User authenticated successfully
```

你可以在[这里][5]找到基础身份认证服务的所有示例代码，可用的容器在[这里][6]。

## Nginx 身份认证代理

Nginx 代理的关键部分是配置文件：

```
# define an /auth section to send the request to an authentication service 

location = /auth { 
    proxy_pass {{auth_backend}}; 
    proxy_pass_request_body off; 
    proxy_set_header Content-Length ""; 
    proxy_set_header X-Original-URI $request_uri;
    proxy_set_header X-Docker-Token ""; 
} 

# use the auth_request directive to redirect all requests to the /auth section above 

location / { 
    proxy_pass {{backend}}; 
    auth_request /auth; 
    proxy_next_upstream error timeout invalid_header http_500     http_502 http_503 http_504; proxy_buffering off; 
}
```

使用了 [http_auth_request][7] 模块发送用户，使用 proxy_pass 指令转发请求到我们简单身份认证服务处理，返回 200 或是 401。 401 授权响应触发 Docker 客户端回应一组使用基本身份验证的凭据。一旦证书被接受，API 将返回 200。Nginx 可以发送请求到私有注册中心，把这两个容器放在一起：


```
ubuntu@trusty-64:/nginx-auth-proxy# docker run -d --name hello-world hello-world  # run a simple web server that prints out “Hello world” 

ubuntu@trusty-64:/nginx-auth-proxy# docker inspect --format '{{ .NetworkSettings.IPAddress }}' hello-world 
172.17.0.41 

ubuntu@trusty-64:/nginx-auth-proxy# docker run -d -e AUTH_BACKEND=http://172.17.0.40:8000 -e BACKEND=http://172.17.0.41:8081 -p 0.0.0.0:8080:80 nginx-auth 

ubuntu@trusty-64:/nginx-auth-proxy# curl 0.0.0.0:8080 
<html> 
<head><title>401 Authorization Required</title></head> 
<body bgcolor="white"> 
<center><h1>401 Authorization Required</h1></center> <hr><center>nginx/1.6.1</center> 
</body> 
</html> 

ubuntu@trusty-64:/nginx-auth-proxy# curl -u testuser:testpassword 0.0.0.0:8080 
Hello world 
```

唯一剩下的事情是给这些容器添加 SSL 证书，然后你就可以好好的玩耍了。你可以在[这里][8]找到 Nginx 身份认证代理的代码，在[这里][9]是可用的容器。使用这个方案需要注意几件事：

 - 使用基础认证意味着每个请求都将访问身份认证 API
 - 基础认证意味着你的证书不受阻碍的发送，除非你使用 SSL 加密了你的连接。注意：永远不要在没有 SSL 的情况下发送基础认证证书。Docker 的客户端和注册中心不会接受 HTTP 方式的基础认证
 - 访问私有注册中心需要对到身份认证代理的请求做一定限制

该解决方案另外一个有趣的附带影响就是，在私有注册中心启用 SSL 已经降低了每个 pull 请求花费的时间，比如，除非在注册中心的 URL 上指定了端口，否则在连接 80 端口之前，客户端首先初始化尝试连接 443 端口。
 
  [1]: http://d1uwwgb7urm13v.cloudfront.net/wp-content/uploads/2014/08/Blog-post1.png
  [2]: https://github.com/opendns/basic-auth-service/blob/master/server.js
  [3]: https://github.com/opendns/basic-auth-service#simple-basic-authentication-service
  [4]: https://github.com/opendns/basic-auth-service/blob/master/Dockerfile
  [5]: https://github.com/opendns/basic-auth-service
  [6]: https://registry.hub.docker.com/u/opendns/basic-auth-service
  [7]: http://nginx.org/en/docs/http/ngx_http_auth_request_module.html
  [8]: https://github.com/opendns/nginx-auth-proxy
  [9]: https://registry.hub.docker.com/u/opendns/nginx-auth-proxy