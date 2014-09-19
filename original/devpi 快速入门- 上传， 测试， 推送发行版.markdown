# devpi 快速入门: 上传， 测试， 推送发行版

标签（空格分隔）： pypi devpi python

---

该快速入门文档将引导你为你的 Python 包设置完成一个独立的 pypi 发布上传，测试和 staging 系统。

## 安装 devpi 客户端和服务器端

我想在我的笔记本上运行完整的 devpi 系统：

```
pip install -U devpi
```

这将安装 `devpi-client` 和 `devpi-server` 这两个 pypi 包。

## devpi quickstart：初始化基本场景

`devpi quickstart` 命令在你本地机器上执行一些基础的初始化步骤：

 - 启动一个后台的 devpi-server，通过 `http://localhost:3141` 访问
 - 配置客户端工具 `devpi` 来连接最新启动的 devpi 服务
 - 创建和登陆一个用户，使用你当前默认的登陆名和一个空的密码
 - 创建一个索引然后直接使用它

让我们运行 quickstart 命令来触发一系列的其他 devpi 命令：

```
$ devpi quickstart
--> $ devpi-server --start
2014-09-04 15:12:19,311 INFO  NOCTX DB: Creating schema
2014-09-04 15:12:19,353 INFO  [Wtx-1] setting password for user u'root'
2014-09-04 15:12:19,353 INFO  [Wtx-1] created user u'root' with email None
2014-09-04 15:12:19,353 INFO  [Wtx-1] created root user
2014-09-04 15:12:19,353 INFO  [Wtx-1] created root/pypi index
2014-09-04 15:12:19,367 INFO  [Wtx-1] fswriter0: committed: keys: u'.config',u'root/.config'
starting background devpi-server at http://localhost:3141
/tmp/home/.devpi/server/.xproc/devpi-server$ /home/hpk/venv/0/bin/devpi-server
process u'devpi-server' started pid=841
devpi-server process startup detected
logfile is at /tmp/home/.devpi/server/.xproc/devpi-server/xprocess.log
--> $ devpi use http://localhost:3141
using server: http://localhost:3141/ (not logged in)
no current index: type 'devpi use -l' to discover indices
~/.pydistutils.cfg     : no config file exists
~/.pip/pip.conf        : no config file exists
~/.buildout/default.cfg: no config file exists
always-set-cfg: no

--> $ devpi user -c testuser password=
user created: testuser

--> $ devpi login testuser --password=
logged in 'testuser', credentials valid for 10.00 hours

--> $ devpi index -c dev
http://localhost:3141/testuser/dev:
  type=stage
  bases=root/pypi
  volatile=True
  uploadtrigger_jenkins=None
  acl_upload=testuser
  pypi_whitelist=

--> $ devpi use dev
current devpi index: http://localhost:3141/testuser/dev (logged in as testuser)
~/.pydistutils.cfg     : no config file exists
~/.pip/pip.conf        : no config file exists
~/.buildout/default.cfg: no config file exists
always-set-cfg: no
COMPLETED!  you can now work with your 'dev' index
  devpi install PKG   # install a pkg from pypi
  devpi upload        # upload a setup.py based project
  devpi test PKG      # download and test a tox-based project
  devpi PUSH ...      # to copy releases between indexes
  devpi index ...     # to manipulate/create indexes
  devpi use ...       # to change current index
  devpi user ...      # to manipulate/create users
  devpi CMD -h        # help for a specific command
  devpi -h            # general help
docs at http://doc.devpi.net
```

显示版本：

```
$ devpi --version
2.0.2
```

## devpi install：安装一个包

我们现在可以使用 `devpi` 命令行客户端来触发一个从已经运行的服务器上使用索引的 pypi 包的 `pip install`。

```
$ devpi install pytest
--> $ /tmp/docenv/bin/pip install -U -i http://localhost:3141/testuser/dev/+simple/ pytest  [PIP_USE_WHEEL=1,PIP_PRE=1]
Downloading/unpacking pytest
  http://localhost:3141/testuser/dev/+simple/pytest/ uses an insecure transport scheme (http). Consider using https if localhost:3141 has it available
  Running setup.py (path:/tmp/docenv/build/pytest/setup.py) egg_info for package pytest

http://localhost:3141/testuser/dev/+simple/py/ uses an insecure transport scheme (http). Consider using https if localhost:3141 has it available
Requirement already up-to-date: py>=1.4.22 in /tmp/docenv/lib/python2.7/site-packages (from pytest)
Installing collected packages: pytest
  Running setup.py install for pytest

    Installing py.test-2.7 script to /tmp/docenv/bin
    Installing py.test script to /tmp/docenv/bin
Successfully installed pytest
Cleaning up...
```
 
`devpi install` 命令配置一个 pip 调用，使用在 `testuser/dev`  索引上的与 pypi 兼容的  `+simple/` 页来寻找和下载包。`pip` 执行的是在 `PATH` 搜索和在 `docenv/bin/pip` 寻找。

让我们检查 `pytest` 是否被正确安装：

```
$ py.test --version
This is pytest version 2.6.1, imported from /tmp/docenv/local/lib/python2.7/site-packages/pytest.pyc
```

你可以第二次调用 `devpi install` 命令，即使在没有网络的情况下，它也能正常工作。

## devpi upload：上传一个或多个包

我们将使用 `devpi` 命令行工具来执行上传（你也可以使用 [plain setup.py][1]）。

让我们检查我们已经登录进了正确的索引：

```
$ devpi use
current devpi index: http://localhost:3141/testuser/dev (logged in as testuser)
~/.pydistutils.cfg     : no config file exists
~/.pip/pip.conf        : no config file exists
~/.buildout/default.cfg: no config file exists
always-set-cfg: no
```

现在进入你其中一个项目的 `setup.py` 文件的目录（我们假设名字为 `example`） 来构建和上传你的包到 `testuser/dev` 索引。

```
example $ devpi upload
using workdir /tmp/devpi0
--> $ /usr/bin/hg st -nmac .
hg-exported project to /tmp/devpi0/upload/example -> new CWD
--> $ /tmp/docenv/bin/python setup.py sdist --formats gztar
built: /home/hpk/p/devpi/doc/example/dist/example-1.0.tar.gz [SDIST.TGZ] 0kb
register example-1.0 to http://localhost:3141/testuser/dev/
file_upload of example-1.0.tar.gz to http://localhost:3141/testuser/dev/
```

这里有三个触发动作：

 - 检测到一个 mercurial 库，导致拷贝所有的文件版本到一个临时工作目录。如果你没有使用 mercurial，拷贝动作会跳过，会直接在你的源码树上做上传操作。
 - 如我们在 `setup.py` 中定义的，把 `example` 版本注册进我们当前的索引
 - 从工作目录，构建和上传一个 `gztar` 格式的文件到当前索引（使用一个在钩子下面的 `setup.py` 调用）
 

我们现在可以安装刚上传好的包：

```
$ devpi install example
--> $ /tmp/docenv/bin/pip install -U -i http://localhost:3141/testuser/dev/+simple/ example  [PIP_USE_WHEEL=1,PIP_PRE=1]
Downloading/unpacking example
  http://localhost:3141/testuser/dev/+simple/example/ uses an insecure transport scheme (http). Consider using https if localhost:3141 has it available
  Downloading example-1.0.tar.gz
  Running setup.py (path:/tmp/docenv/build/example/setup.py) egg_info for package example

Installing collected packages: example
  Running setup.py install for example

Successfully installed example
Cleaning up...
```

这个你刚刚从 `testuser/dev` 索引安装的上传包是我们开始上传好的包。

> 注意：devpi upload 允许同时上传你的发行文件的多个不同格式，比如：`sdist.zip` 或是 `bdist_egg`，默认是 `sdist.tgz`。

## devpi test：测试一个已经上传的包

如果你有一个包使用 [tox][2] 做测试，你或许要调用：

```
$ devpi test example  # package needs to contain tox.ini
received http://localhost:3141/testuser/dev/+f/5bc/44b5ac34ff65b/example-1.0.tar.gz#md5=5bc44b5ac34ff65b6b16b332f9ccc22c
unpacking /tmp/devpi-test0/downloads/example-1.0.tar.gz to /tmp/devpi-test0
/tmp/devpi-test0/example-1.0$ tox --installpkg /tmp/devpi-test0/downloads/example-1.0.tar.gz -i ALL=http://localhost:3141/testuser/dev/+simple/ --result-json /tmp/devpi-test0/toxreport.json -c /tmp/devpi-test0/example-1.0/tox.ini
python create: /tmp/devpi-test0/example-1.0/.tox/python
python installdeps: pytest
python inst: /tmp/devpi-test0/downloads/example-1.0.tar.gz
python runtests: PYTHONHASHSEED='2866296898'
python runtests: commands[0] | py.test
___________________________________ summary ____________________________________
  python: commands succeeded
  congratulations :)
wrote json report at: /tmp/devpi-test0/toxreport.json
posting tox result data to http://localhost:3141/testuser/dev/+f/5bc/44b5ac34ff65b/example-1.0.tar.gz
successfully posted tox result data
```

以上操作发生了这些事：

 - devpi 从当前的索引中获取 `example` 的最新可用版本
 - 把它解压到一个临时目录，发现 `tox.ini` 并且调用 tox，把它指向我们的 `example-1.0.tar.gz`。强迫所有的安装都通过我们当前的 `testuser/dev/+simple/` 索引和命令它创建一个 `json` 格式的报告
 - 在所有的测试运行完成后，我们发送 `toxreport.json` 到 devpi 服务器去，它将被精确的连接到我们的发行文件

我们可以校验测试状态是否被记录：

```
$ devpi list example
http://localhost:3141/testuser/dev/+f/5bc/44b5ac34ff65b/example-1.0.tar.gz
  cobra linux2 python 2.7.6 tests passed
```


## devpi push: staging a release to another index

一旦你觉得发行文件没有问题，你可以把它推送到一个 devpi-managed 索引和一个外部的 pypi 索引服务器。

让我们创建另外一个 `staging` 索引：

```
$ devpi index -c staging volatile=False
http://localhost:3141/testuser/staging:
  type=stage
  bases=root/pypi
  volatile=False
  uploadtrigger_jenkins=None
  acl_upload=testuser
  pypi_whitelist=
```

我们创建了一个非易失性的索引，这意味着不能覆盖和删除发行文件，看  [Non Volatile Indexes][3] 来获取更多信息。
 

我们现在可以把 `example-1.0.tar.gz` 从上面的所有推送到 `staging` 索引：

```
$ devpi push example-1.0 testuser/staging
   200 register example 1.0 -> testuser/staging
   200 store_releasefile testuser/staging/+f/5bc/44b5ac34ff65b/example-1.0.tar.gz
   200 store_toxresult testuser/staging/+f/5bc/44b5ac34ff65b/example-1.0.tar.gz.toxresult0
```

这将决定所有在 `testuser/dev` 索引上的文件属于指定的 `example-1.0` 发行版并且把它们拷贝进 `testuser/staging` 索引。

## devpi push: 发行到一个外部的索引

让我们检查我们当前的索引：

```
$ devpi use
current devpi index: http://localhost:3141/testuser/dev (logged in as testuser)
~/.pydistutils.cfg     : no config file exists
~/.pip/pip.conf        : no config file exists
~/.buildout/default.cfg: no config file exists
always-set-cfg: no
```

让我们使用 `testuser/staging` 索引：

```
$ devpi use testuser/staging
current devpi index: http://localhost:3141/testuser/staging (logged in as testuser)
~/.pydistutils.cfg     : no config file exists
~/.pip/pip.conf        : no config file exists
~/.buildout/default.cfg: no config file exists
always-set-cfg: no
```

并且再次检查测试结果：

```
$ devpi list example
http://localhost:3141/testuser/staging/+f/5bc/44b5ac34ff65b/example-1.0.tar.gz
  cobra linux2 python 2.7.6 tests passed
```

从最后一步推送后，测试状态依然是可用的。

我们现在可以决定推送这个发行版到外部的 pypi 索引，我们已经在 `.pypirc` 文件中配置了：

```
$ devpi push example-1.0 pypi:testrun
no pypirc file found at: /tmp/home/.pypirc
```

这将推送 `example-1.0` 发行版的所有的文件到一个外部的 `testrun` 索引服务器。使用资格证书和 发现在 `pypi` 章节 `.pypirc` 文件中的 URL。


## 索引继承重新配置

现在，我们已经有 `example-1.0` 发行版和发行版文件在 `testuser/dev` 和 `testuser/staging` 索引，如果我们想在我们的开发索引上一直使用 staging 包，我们可以重新配置 `testuser/dev` 的继承基线。

```
$ devpi index testuser/dev bases=testuser/staging
/testuser/dev changing bases: ['testuser/staging']
http://localhost:3141/testuser/dev:
  type=stage
  bases=testuser/staging
  volatile=True
  uploadtrigger_jenkins=None
  acl_upload=testuser
  pypi_whitelist=
```

现在我们切回 `testuser/dev`：

```
$ devpi use testuser/dev
current devpi index: http://localhost:3141/testuser/dev (logged in as testuser)
~/.pydistutils.cfg     : no config file exists
~/.pip/pip.conf        : no config file exists
~/.buildout/default.cfg: no config file exists
always-set-cfg: no
```

然后寻找我们的 example 发行文件：

```
$ devpi list example
http://localhost:3141/testuser/dev/+f/5bc/44b5ac34ff65b/example-1.0.tar.gz
  cobra linux2 python 2.7.6 tests passed
http://localhost:3141/testuser/staging/+f/5bc/44b5ac34ff65b/example-1.0.tar.gz
  cobra linux2 python 2.7.6 tests passed
```

我们看到 `example-1.0.tar.gz` 被包含在两个索引中，让我们移除 `testuser/dev` 的  `example` 发行版：

```
$ devpi remove -y example
About to remove the following releases and distributions
version: 1.0
  - http://localhost:3141/testuser/dev/+f/5bc/44b5ac34ff65b/example-1.0.tar.gz
  - http://localhost:3141/testuser/dev/+f/5bc/44b5ac34ff65b/example-1.0.tar.gz.toxresult0
Are you sure (yes/no)? yes (autoset from -y option)
deleting release 1.0 of example
```

如果你没有指定 `-y` 选项，你将被询问确认删除的交互式操作。

`example-1.0` 发行版本仍然可以通过 `testuser/dev` 访问，因为他从 `testuser/staging` 基线继承了所有的发行版。

```
$ devpi list example
http://localhost:3141/testuser/staging/+f/5bc/44b5ac34ff65b/example-1.0.tar.gz
  cobra linux2 python 2.7.6 tests passed
```

```
$ devpi-server --stop
killed server pid=841
```

## 永久的运行 devpi-server

如果你想配置安装一个永久的 `devpi-server`。你也可以通过  [Quickstart: permanent install on server/laptop][4] 获取一些帮助。
  


  [1]: http://doc.devpi.net/latest/userman/devpi_misc.html#configure-pypirc
  [2]: http://tox.testrun.org/
  [3]: http://doc.devpi.net/latest/userman/devpi_concepts.html#non-volatile-indexes
  [4]: http://doc.devpi.net/latest/quickstart-server.html#quickstart-server