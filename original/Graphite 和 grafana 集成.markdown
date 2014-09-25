# Graphite 和 grafana 集成

标签（空格分隔）： Graphite grafana 监控部署 可视化

---

由于 Graphite 自带的界面太难看，赞伟大的开源世界，于是我们有了 grafana 可用

## 安装 grafana

下载 grafana：

```
wget http://grafanarel.s3.amazonaws.com/grafana-1.8.0.zip
unzip grafana-1.8.0.zip
mv grafana-1.8.0 grafana
```

## 和 Graphite 集成

修改 grafana 的 config.js 配置文件：

```
cd grafana
cp -av config.sample.js  config.js
```

把 config.js 的以下内容做配置：

```
    datasources: {
      graphite: {
        type: 'graphite',
        url: "http://graphite.test.com",
        render_method: 'GET',
      },
      elasticsearch: {
        type: 'elasticsearch',
        url: "http://elas.test.com:9200",
        index: 'grafana-dash',
        grafanaDB: true,
      }
    },

```

> 注：为了方便，定义一个域名，然后自己配置 /etc/hosts 做解析，在生产环境自然用 DNS 做解析。

## 配置 Nginx

> 注：这个尤其需要的是注意跨域问题了

下面是我的测试的 nginx 的配置文件，供参考：


```
server {
       listen 80;
       server_name graphite.test.com;   

       location / {
           uwsgi_pass 192.168.1.6:8080;
           include uwsgi_params;
           if ($http_origin ~* (http://[^/]*\.test\.com)){
                 set $cors "true";
           }

           if ($cors = 'true') {
               add_header  Access-Control-Allow-Origin $http_origin;
               add_header Access-Control-Allow-Headers X-Requested-With;
               add_header  "Access-Control-Allow-Credentials" "true";
               add_header  "Access-Control-Allow-Methods" "GET, POST, OPTIONS";
               add_header  "Access-Control-Allow-Headers" "Authorization, origin, accept";
           }
        }       
    }
    

    server {
        listen       80;
        server_name  grafana.test.com;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;


        location / {
           auth_basic            "Restricted";
           auth_basic_user_file  /opt/graphite/grafana/htpasswd/pass;

           if ($http_origin ~* (http://[^/]*\.test\.com)){
                 set $cors "true";
           }

           if ($cors = 'true') {
               add_header  Access-Control-Allow-Origin $http_origin;
               add_header Access-Control-Allow-Headers X-Requested-With;
               add_header  "Access-Control-Allow-Credentials" "true";
               add_header  "Access-Control-Allow-Methods" "GET, POST, OPTIONS";
               add_header  "Access-Control-Allow-Headers" "Authorization, origin, accept";
           }
           root /opt/graphite/grafana;
        }

    }

```

然后启动 Nginx，启动成功后，就可以使用愉快的访问了：

```
http://grafana.test.com
```

## 配置 grafana 

这个后续讲解



## 参考资料

主要是参考了官方文档

- http://grafana.org/docs/

