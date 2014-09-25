# 在 CentOS 5.8 上安装 Graphite

标签（空格分隔）： 监控 monitor CentOS 5.8 Graphite 

---

首先说一句，在 CentOS 5.8 上安装真的很坑爹。。。


## 安装基础环境

 - 操作系统内核

```
uname -a

Linux cloud-test-slave-001 2.6.18-308.el5 #1 SMP Tue Feb 21 20:06:06 EST 2012 x86_64 x86_64 x86_64 GNU/Linux
```

 - 操作系统版本

```
[root@cloud-test-slave-001 ~]# cat /etc/redhat-release 
CentOS release 5.8 (Final)
```

 - Python 版本

```
[root@cloud-test-slave-001 ~]# python  --version
Python 2.7
```
 
 
## 必备软件

按照 Graphite 官方文档的要求，需要如下软件：

```
python2.4 或者更高版本（必选）【建议 2.6 或 2.7 版本】
pycairo (需要 PNG 包支持，即 libpng)（必选）
mod_python（必选）
django（必选，需要 django 1.4 版本，为什么必须 django 1.4 在后面会说明）
python-ldap (可选 - needed for ldap-based webapp authentication)
python-memcached (可选 - needed for webapp caching, big performance boost)
python-sqlite2 (可选 - a django-supported database module is required)
bitmap and bitmap-fonts required on some systems, 尤其是在 Red Hat 系统上
```

下面是我目前安装的一些涉及到的软件的版本（可能有多余的，是因为我在虚拟环境安装，没有安装好，只好混装，所以拉出来的有点多）
 
执行命令 `pip freeze` 拉出如下清单：
 
```
Django==1.4
Dozer==0.4
Jinja2==2.7.3
MarkupSafe==0.23
PyYAML==3.10
SQLAlchemy==0.9.7
Twisted==14.0.2
WebOb==1.4
Werkzeug==0.9.6
alembic==0.6.5
configobj==5.0.6
croniter==0.3.3
cssselect==0.9.1
daemonize==2.3.1
dagobah==0.2.3
distribute==0.6.49
django-filter==0.7
django-tagging==0.3.1
lxml==3.3.5
micawber==0.3.0
paramiko==1.11.0
peewee==2.2.5
premailer==1.13
pycrypto==2.6.1
pysqlite==2.6.3
python-dateutil==2.2
python-memcached==1.53
txAMQP==0.6.2
uWSGI==2.0.7
virtualenv==1.11.6
whisper==0.9.12
wsgiref==0.1.2
zope.interface==4.1.1
```

## 开始安装

 1. 安装 yum EPEL 源
 
    ```
    wget "http://mirrors.ustc.edu.cn/fedora/epel/5/i386/epel-release-5-4.noarch.rpm"
    rpm -Uvh epel-release-5-4.noarch.rpm
    ```
    
 2. 安装依赖包
   
    ```
    yum install gcc bitmap bitmap-fonts zope
    ```

    ```
    yum install openldap openldap24-libs openldap-clients openldap-devel openssl-devel
    ```
    ```
    pip install pyOpenSSL python-memcached pycrypto  python-ldap  pysqlite uwsgi nginx
    ```
    
    > 注：因为我这个目前是测试，所以 nginx 就直接 yum 安装了，版本比较低，只有 0.8，如果是生产，建议源码安装最新的稳定版本。
    
    ```
    pip install django==1.4
    ```
    
    安装 django-tagging ，默认的 `pip install tagging` 的这个不行，版本是 0.2.1 ，会报错。
    
    ```
    pip install django-tagging==0.3.1 或 pip install tagging==0.3.1
    ```
    
    如果上面这个方法无法安装 `django-tagging=0.3.1` 那就用源码安装的方法。
    
    ```
    wget https://pypi.python.org/packages/source/d/django-tagging/django-tagging-0.3.1.tar.gz --no-check-certificate
    tar -zxvf django-tagging-0.3.1.tar.gz 
    cd django-tagging-0.3.1
    python setup.py install
    ```
    
 3. 安装 pycairo

  开始好几次就死在安装 pycairo 这个上面，python 2.7 版本的通过 `pip   install pycairo` 命令安装的不行，对应 python 2.7 版本的 pycairo   包的名字叫做 `py2cairo`。     
  
   安装 py2cairo 包的方法如下：

   1）下载 py2cairo 包:
    
    ```
    wget http://www.cairographics.org/releases/py2cairo-1.10.0.tar.bz2
    tar -xvf py2cairo-1.10.0.tar.bz2
    ```

    如果上面的版本不行就用下面的方式获取源码包： 

    ```
     git clone git://git.cairographics.org/git/py2cairo
    ```
    
    要安装 py2cairo 必须要 cairo   包，而且这个包的版本有规定，我这边下载的是 cairo-1.12.2。
    
    ```
    wget http://cairographics.org/releases/cairo-1.12.2.tar.xz

    ```
    cairo 依赖 `libpng` 和 `pixman`，所以需要安装这两个

    ```
    yum -y install libpng*  pixman*
    ```

   > 注，有可能这样安装的 pixman 的版本不对，需要需要源码安装

   源码安装 pixman：

   ```
   wget http://cairographics.org/releases/pixman-0.22.0.tar.gz
   tar -zxvf pixman-0.22.0.tar.gz
   ./configure --prefix=/usr/local
   make && make install
   ```

   > 注：这里 pixman 的安装必需要加个    `prefix=/usr/local`，不然的话，cairo 会找不到再次表示蛋疼。 

   安装 cairo 的正确姿势：

   ```
   tar -zxvf cairo-1.12.2.tar.xz
   cd cairo-1.12.2
   ./configure
   make && make install
   ```

   安装 py2cairo 的正确姿势：

   安装的时候需要明确下 pkgconfig 的位置和 Python 的包路径：

   ```
   cd py2cairo
   export PKG_CONFIG_PATH=/usr/local/lib/pkgconfig:$PKG_CONFIG_PATH
   PYTHONPATH="/usr/local/lib/python2.7/site-packages/:$PYTHONPATH"
   LDFLAGS="-lm -ldl -lutil" ./waf configure --prefix=/usr/local
   ./waf build
   ./waf install
   ```

   经历九九八十一难，终于安装完成。

   > 注：真的是最麻烦的就是 py2cairo 这个家伙，开始折腾了我一天没有安装好，没有安装好就无法出图。。。

  
 4. 安装 Graphite 三大件
 
    ```
    pip install whisper carbon graphite-web
    ```
 5. 安装数据采集工具 Diamond 
    
    ```
    git clone https://github.com/BrightcoveOS/Diamond
    cd Diamond
    ```
    
    修改 Diamond 的配置文件：
    
    ```
    cd Diamond
    cp conf/diamond.conf.example conf/diamond.conf
    ```
    
    `diamond.conf` 配置文件如下：
    
    ```
    #
    #Diamond Configuration File
    ### Options for the server
    [server]
    # Handlers for published metrics.
handlers = diamond.handler.graphite.GraphiteHandler, diamond.handler.archive.ArchiveHandler
    # User diamond will run as
    # Leave empty to use the current user
    user =
    # Group diamond will run as
    # Leave empty to use the current group
    group =
    # Pid file
    pid_file = /var/run/diamond.pid
    # Directory to load collector modules from
    # 这个和你的 Diamond 安装路径有关
    collectors_path = /root/Diamond/src/collectors/
    # Directory to load collector configs from
    # 这个和你的 Diamond 安装路径有关
    collectors_config_path = /root/Diamond/src/collectors/
    # Directory to load handler configs from
    # 这个和你的 Diamond 安装路径有关
    handlers_config_path = /root/Diamond/src/diamond/handler
    handlers_path = /root/Diamond/src/diamond/handler
    # Interval to reload collectors
    collectors_reload_interval = 3600
    ### Options for handlers
    [handlers]
    # daemon logging handler(s)
    keys = rotated_file
    ### Defaults options for all Handlers
    [[default]]
    [[ArchiveHandler]]
    # File to write archive log files
    #需要创建 /var/log/diamond/ 目录
    log_file = /var/log/diamond/archive.log
    # Number of days to keep archive log files
    days = 7
    [[GraphiteHandler]]
    ### Options for GraphiteHandler
    # Graphite server host
    host = 127.0.0.1
    # Port to send metrics to
    port = 2003
    # Socket timeout (seconds)
    timeout = 15
    # Batch size for metrics
    batch = 1
    [[GraphitePickleHandler]]
    ## Options for GraphitePickleHandler
    # Graphite server host
    host = 127.0.0.1
    # Port to send metrics to
    port = 2004
    # Socket timeout (seconds)
    timeout = 15
    # Batch size for pickled metrics
    batch = 256
    [[MySQLHandler]]
    ### Options for MySQLHandler
    # MySQL Connection Info
    hostname    = 127.0.0.1
    port        = 3306
    username    = root
    password    =
    database    = diamond
    table       = metrics
    # INT UNSIGNED NOT NULL
    col_time    = timestamp
    # VARCHAR(255) NOT NULL
    col_metric  = metric
    # VARCHAR(255) NOT NULL
    col_value   = value
    [[StatsdHandler]]
    host = 127.0.0.1
    port = 8125
    [[TSDBHandler]]
    host = 127.0.0.1
    port = 4242
    timeout = 15
    [[LibratoHandler]]
    user = user@example.com
    apikey = abcdefghijklmnopqrstuvwxyz0123456789abcdefghijklmnopqrstuvwxyz01
    [[HostedGraphiteHandler]]
    apikey = abcdefghijklmnopqrstuvwxyz0123456789abcdefghijklmnopqrstuvwxyz01
    timeout = 15
    batch = 1
    # And any other config settings from GraphiteHandler are valid here
    [[HttpPostHandler]]
    ### Urp to post the metrics
    url = http://localhost:8888/
    ### Metrics batch size
    batch = 100
    ### Options for collectors
    [collectors]
    [[default]]
    ### Defaults options for all Collectors
    # Uncomment and set to hardcode a hostname for the collector path
    # Keep in mind, periods are seperators in graphite
    # hostname = my_custom_hostname
    # If you prefer to just use a different way of calculating the hostname
    # Uncomment and set this to one of these values:
    # smart             = Default. Tries fqdn_short. If that's
    localhost, uses hostname_short
    # fqdn_short        = Default. Similar to hostname -s
    # fqdn              = hostname output
    # fqdn_rev          = hostname in reverse (com.example.www)
    # uname_short       = Similar to uname -n, but only the first part
    # uname_rev         = uname -r in reverse (com.example.www)
    # hostname_short    = `hostname -s`
    # hostname          = `hostname`
    # hostname_rev      = `hostname` in reverse (com.example.www)
    # shell             = Run the string set in hostname as a shell command and use its
    #                     output(with spaces trimmed off from both ends) as the hostname.
    # hostname_method = smart
    # Path Prefix and Suffix
    # you can use one or both to craft the path where you want to put metrics
    # such as: %(path_prefix)s.$(hostname)s.$(path_suffix)s.$(metric)s
    # path_prefix = servers
    # path_suffix =
    # Path Prefix for Virtual Machines
    # If the host supports virtual machines, collectors may report per
    # VM metrics. Following OpenStack nomenclature, the prefix for
    # reporting per VM metrics is "instances", and metric foo for VM
    # bar will be reported as: instances.bar.foo...
    # instance_prefix = instances
    # Default Poll Interval (seconds)
    # interval = 300
    ### Options for logging
    # for more information on file format syntax:
    # http://docs.python.org/library/logging.config.html#configuration-file-format
    [loggers]
    keys = root
    # handlers are higher in this config file, in:
    # [handlers]
    # keys = ...
    [formatters]
    keys = default
    [logger_root]
    # to increase verbosity, set DEBUG
    level = INFO
    handlers = rotated_file
    propagate = 1
    [handler_rotated_file]
    class = handlers.TimedRotatingFileHandler
    level = DEBUG
    formatter = default
    # rotate at midnight, each day and keep 7 days
    args = ('/var/log/diamond/diamond.log', 'midnight', 1, 7)
    [formatter_default]
    format = [%(asctime)s] [%(threadName)s] %(message)s
    datefmt =
    ### Options for config merging
    # [configs]
    # path = "/etc/diamond/configs/"
    # extension = ".conf"
    #Example:
    # /etc/diamond/configs/net.conf
    # [collectors]
    #
    # [[NetworkCollector]]
    # enabled = True
    ```
    
    启动 Diamond：
    
    ```
    python diamond -c ../conf/diamond.conf
    ```
    
    
## 启动 Grapgite 三大件

初始化 Graphite 数据库：

```
cd /opt/graphite/webapp/graphite
mv local_settings.py.example local_settings.py
```
修改 local_settings.py 里面的数据库配置，把下面一段的注释取消：

```
DATABASES = {
    'default': {
        'NAME': '/opt/graphite/storage/graphite.db',
        'ENGINE': 'django.db.backends.sqlite3',
        'USER': 'root',
        'PASSWORD': '123456',
        'HOST': '',
        'PORT': ''
    }
}
```

初始化 Graphite 数据库：

```
python manage.py syncdb
```

> 注：在这个初始化的过程中就可能遇到的错误是 tagging 版本太低，或者是 django 的版本问题，所以才有前面的 django 的版本必须为 1.4，tagging 的版本必须为 0.3.1

启动 carbon-cache：

```
cd /opt/graphite/bin
./carbon-cache.py start
```

> 注：这个在启动过程中，可能遇到如下错误：

```
from carbon.util import run_twistd_plugin
File “/opt/graphite/lib/carbon/util.py”, line 19, in <module>
from twisted.scripts._twistd_unix import daemonize
ImportError: cannot import name daemonize
```

解决方法是修改 `/opt/graphite/lib/carbon/util.py` 文件，把其 `from twisted.scripts._twistd_unix import daemonize` 替换成 `import daemonize`

> 注：需要安装 daemonize，`pip install daemonize`

然后再次启动即可

启动 graphite，使用 wsgi 启动，在 `/opt/graphite/webapp` 下新建一个文件 `wsgi.py` 和一个配置文件 ` django.xml`：

**wsgi.py**

```
import os,sys
 
if not os.path.dirname(__file__) in sys.path[:1]:
    sys.path.insert(0, os.path.dirname(__file__))
os.environ['DJANGO_SETTINGS_MODULE'] = 'settings'
 
from django.core.handlers.wsgi import WSGIHandler
application = WSGIHandler()
```

**django.xml**

```
<uwsgi>
    <socket>ip:port</socket>
        <chdir>/opt/graphite/webapp/graphite</chdir> 
        <pythonpath>..</pythonpath>
    <module>wsgi</module>
</uwsgi>
```

启动：

```
uwsgi -x django.xml
```

现在可以愉快的访问了 http://ip:port

>建议：不作死就不会死，要安装这个还是不要用 CentOS 5.8 了，太痛苦。。使用 Python 的 virtualenv 环境也会遇到很多问题。

比如使用 virtualenv 最新版本的时候，会遇到这个错误：

```
  OSError: Command /Users/bgbb/Developer/django/vnv/bin/python -c "import sys, pip;   sys...d\"] + sys.argv[1:]))" setuptools pip failed with error code 2
```
解决办法是降级到  virtualenv 1.10.1 版本：

```
  pip uninstall virtualenv
  wget  https://pypi.python.org/packages/source/v/virtualenv/virtualenv-1.10.1.tar.gz --no-check-certificate
  python setup.py install
```

如果大家安装过程中遇到问题，欢迎交流！
>注：该安装方法适用于 CentOS 5.8，其他的版本应该不会遇到我这么多问题。

## 参考资料

- http://blog.chinaunix.net/uid-28769783-id-3652037.html
- http://www.dongwm.com/archives/shi-yong-grafanahe-diamondgou-jian-graphitejian-kong-xi-tong/
- http://www.linuxsysadmintutorials.com/install-graphite-on-a-centosrhel-server
- http://blog.csdn.net/crazyhacking/article/details/8464235
- http://viewsby.wordpress.com/2014/05/13/carbon-cache-py-cannot-import-name-daemonize
- http://www.tecmint.com/how-to-enable-epel-repository-for-rhel-centos-6-5/
- http://graphite.wikidot.com/
