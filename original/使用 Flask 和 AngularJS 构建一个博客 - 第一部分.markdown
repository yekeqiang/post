# 使用 Flask 和 AngularJS 构建一个博客 - 第一部分

标签（空格分隔）： Flask AngularJS Python REST javascript

---


> 注：原文作者 [John Kevin M. Basco][1]，原文地址为 [Building a blog using Flask and AngularJS Part 1][2]


在这个教程中，我们将使用 Flask 和 AngularJS 构建一个博客。

这是这个系列教程的第一部分，在这部分我们将专注于构建 REST API ，该 API 将被 AngularJS 应用使用。


# 目标

该应用的目标非常简单：

 - 允许任何用户注册
 - 允许注册的用户登录
 - 允许登录的用户创建博客
 - 允许在首页展示博客
 - 允许登录的用户退出

这个应用在该系列教程的最后应该看起来像这样：

![此处输入图片的描述][3]


# 必要条件

该教程假设你能顺畅的写 Python 的代码，知道怎样设置一个虚拟环境以及熟悉终端窗口

# 目录结构

我们将使用的目录结构应该看起来像这样：

![此处输入图片的描述][4]

# 安装必要的 Python 包

我们将使用的包如下：

 - Flask-RESTful - Flask 的 RESTful 扩展
 - Flask-SQLAlchemy - Flask 的 SQLAlchemy 扩展
 - Flask-Bcrypt - Flask 的 一个为你的应用提供 bcrypt 哈希的工具扩展
 - Flask-HTTPAuth - 一个为 Flask 路由提供 Basic and Digest HTTP authentication 的扩展
 - Flask-WTF - http://docs.jinkan.org/docs/flask-wtf/
 - WTForms-Alchemy - 一个 WTForms 扩展，能很简单的基于表单创建模型的工具集
 - marshmallow - 是一个 ORM/ODM/ 的框架，用于转换复杂的数据类型，http://marshmallow.readthedocs.org/en/latest/quickstart.html

为了使事情更加简单，在 blog/server/ 目录下创建一个名称为 requirements.txt 的文件，并且把以下内容拷贝进去：

```
Flask==0.10.1
Flask-Bcrypt==0.6.0
Flask-HTTPAuth==2.2.1
Flask-RESTful==0.2.12
Flask-SQLAlchemy==1.0
Flask-WTF==0.10.0
Jinja2==2.7.3
MarkupSafe==0.23
SQLAlchemy==0.9.7
SQLAlchemy-Utils==0.26.9
WTForms==2.0.1
WTForms-Alchemy==0.12.8
WTForms-Components==0.9.5
Werkzeug==0.9.6
aniso8601==0.83
decorator==3.4.0
infinity==1.3
intervals==0.3.1
itsdangerous==0.24
marshmallow==0.7.0
py-bcrypt==0.4
pytz==2014.4
six==1.7.3
validators==0.6.0
wsgiref==0.1.2
```

进入 blog/server/ 目录，并且运行 `pip install -r requirements.txt` 命令来安装必须的包

# 应用的初始化设置

首先，在 blog/server/app 目录下创建一个文件并命名为 config.py，然后复制和粘贴以下的内容到文件中：

```
DEBUG = True
WTF_CSRF_ENABLED = False
```

然后在 blog/server/app/ 目录下创建一个文件命名为 server.py。这就是我们将放置我们初始的代码的地方。复制和粘贴以下的代码到文件中：

```
import os
 
from flask import Flask
from flask.ext import restful
from flask.ext.restful import reqparse, Api
from flask.ext.sqlalchemy import SQLAlchemy
from flask.ext.bcrypt import Bcrypt
from flask.ext.httpauth import HTTPBasicAuth
 
basedir = os.path.join(os.path.abspath(os.path.dirname(__file__)), '../')
 
app = Flask(__name__)
app.config.from_object('app.config')
 
# flask-sqlalchemy
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///' + os.path.join(basedir, 'app.sqlite')
db = SQLAlchemy(app)
 
# flask-restful
api = restful.Api(app)
 
# flask-bcrypt
flask_bcrypt = Bcrypt(app)
 
# flask-httpauth
auth = HTTPBasicAuth()
 
@app.after_request
def after_request(response):
    response.headers.add('Access-Control-Allow-Origin', '*')
    response.headers.add('Access-Control-Allow-Headers', 'Content-Type,Authorization')
    response.headers.add('Access-Control-Allow-Methods', 'GET,PUT,POST,DELETE')
    return response
 
import views
```

在这个文件中，我们初始化了 Flask，加载了从一个配置文件中加载了配置文件的变量，创建了 flask-sqlalchemy，flask-restful 对象等等。。。而且我们也在 after_request 函数中加了一些响应头，它允许跨域资源共享（CORS），这将允许我们的托管服务器（REST API）和客户端（AngularJS app）在不同的域以及不同的子域（例如：api.johnsblog.com 和 johnsblog.com）。在开发期间，这将允许我们把它们运行在不同的端口（例如：localhost:8000 和 localhost:5000）。

# Models

现在我们已经完成了应用的初始化部分，让我们定义 models，在 blog/server/app/ 目录下创建一个文件并命名为 models.py，然后拷贝和粘贴以下代码到文件中：

```
from flask import g
 
from wtforms.validators import Email
 
from server import db, flask_bcrypt
 
class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    email = db.Column(db.String(120), unique=True, nullable=False, info={'validators': Email()})
    password = db.Column(db.String(80), nullable=False)
    posts = db.relationship('Post', backref='user', lazy='dynamic')
 
    def __init__(self, email, password):
        self.email = email
        self.password = flask_bcrypt.generate_password_hash(password)
 
    def __repr__(self):
        return '<User %r>' % self.email
 
class Post(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(120), nullable=False)
    body = db.Column(db.Text, nullable=False)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'))
    created_at = db.Column(db.DateTime, default=db.func.now())
 
    def __init__(self, title, body):
        self.title = title
        self.body = body
        self.user_id = g.user.id
 
    def __repr__(self):
        return '<Post %r>' % self.title
```

在以上的代码中，我们定义了一个用户和文章的模型，用户模型有一个 `id` 作为它的主键，被定义成 integer 类型，`email` 和 `password` 属性被定义成 strings。通过  `posts` 属性它也和 POST 模型有关联。POST 模型也有一个 `id` 作为它的主键，并也被定义成 integer，`title` 和 `body` 属性被定义成 strings。它也有一个 `user_id` 属性被定义成 integer 并作为 User 模型的 id 属性的外键。它还有一个 `created_at` 属性被定义成 DateTime。

# Forms

现在我们已经完成了模型定义部分，让我们定义 forms，我们将使用 forms 校验我们的用户输入。为了 form 校验，我们将使用是一个名称为 WTForms 的 Flask 扩展，并且我们将使用 WTForms-Alchemy 扩展它 来更快更简单的定义我们的 forms。在 blog/server/app 目录下创建一个新文件并命名为 forms.py，然后拷贝和粘贴以下代码到文件中：

```
from flask.ext.wtf import Form
 
from wtforms_alchemy import model_form_factory
from wtforms import StringField
from wtforms.validators import DataRequired
 
from app.server import db
from models import User, Post
 
BaseModelForm = model_form_factory(Form)
 
class ModelForm(BaseModelForm):
    @classmethod
    def get_session(self):
        return db.session
 
class UserCreateForm(ModelForm):
    class Meta:
        model = User
 
class SessionCreateForm(Form):
    email = StringField('name', validators=[DataRequired()])
    password = StringField('password', validators=[DataRequired()])
 
class PostCreateForm(ModelForm):
    class Meta:
        model = Post
```

为了使得 WTForms-Alchemy 能与 Flask-WTF 一起工作，我们定义了一个类命名为 ModelForm，它继承自 BaseModelForm ，由 WTForms-Alchemy 提供。你可以在这里找到更多的信息 - http://wtforms-alchemy.readthedocs.org/en/latest/advanced.html#using-wtforms-alchemy-with-flask-wtf

以上代码是非常完美的。但是如果你不明白以上代码做了什么，我建议你读 WTForms 的文档 https://wtforms.readthedocs.org/en/latest/#  和 WTForms-Alchemy 的文档 http://wtforms-alchemy.readthedocs.org/en/latest/index.html 。

# Serializers 

为了在我们的 responses 中把我们的 model 实例渲染成 JSON，我们首先需要把它们转换成原生的 Python 数据类型， Flask-RESTful 可以使用 fields 模块 和 marshal_with() 装饰器（更多的详细信息请移步 - http://flask-restful.readthedocs.org/en/latest/quickstart.html#data-formatting）。当我开始构建 REST API 的时候我不知道 Flask-RESTful 支持这个，因此我以 Marshmallow 完成 http://marshmallow.readthedocs.org/en/latest/ 。在 blog/server/app/ 目录下创建一个新文件并命名为 serializers.py，然后拷贝和粘贴以下代码到文件中：

```
from marshmallow import Serializer, fields
 
class UserSerializer(Serializer):
    class Meta:
        fields = ("id", "email")
 
class PostSerializer(Serializer):
    user = fields.Nested(UserSerializer)
 
    class Meta:
        fields = ("id", "title", "body", "user", "created_at")
```

# Views

现在我们已经完成了我们将使用的 models，forms 和 serializers 的定义，让我们定义 views 并且使用。在 blog/server/app/ 目录下创建一个新文件并命名为 views.py ，然后拷贝和粘贴以下代码到文件中：

```
from flask import g
from flask.ext import restful
 
from server import api, db, flask_bcrypt, auth
from models import User, Post
from forms import UserCreateForm, SessionCreateForm, PostCreateForm
from serializers import UserSerializer, PostSerializer
 
@auth.verify_password
def verify_password(email, password):
    user = User.query.filter_by(email=email).first()
    if not user:
        return False
    g.user = user
    return flask_bcrypt.check_password_hash(user.password, password)
 
class UserView(restful.Resource):
    def post(self):
        form = UserCreateForm()
        if not form.validate_on_submit():
            return form.errors, 422
 
        user = User(form.email.data, form.password.data)
        db.session.add(user)
        db.session.commit()
        return UserSerializer(user).data
 
class SessionView(restful.Resource):
    def post(self):
        form = SessionCreateForm()
        if not form.validate_on_submit():
            return form.errors, 422
 
        user = User.query.filter_by(email=form.email.data).first()
        if user and flask_bcrypt.check_password_hash(user.password, form.password.data):
            return UserSerializer(user).data, 201
        return '', 401
 
class PostListView(restful.Resource):
    def get(self):
        posts = Post.query.all()
        return PostSerializer(posts, many=True).data
 
    @auth.login_required
    def post(self):
        form = PostCreateForm()
        if not form.validate_on_submit():
            return form.errors, 422
        post = Post(form.title.data, form.body.data)
        db.session.add(post)
        db.session.commit()
        return PostSerializer(post).data, 201
 
class PostView(restful.Resource):
    def get(self, id):
        posts = Post.query.filter_by(id=id).first()
        return PostSerializer(posts).data
 
api.add_resource(UserView, '/api/v1/users')
api.add_resource(SessionView, '/api/v1/sessions')
api.add_resource(PostListView, '/api/v1/posts')
api.add_resource(PostView, '/api/v1/posts/<int:id>')
```

上面的 verify_password 函数被 auth.verify_password 装饰，并将被 Flask-HTTPAuth 使用来鉴定用户。它基本上通过 email 来获取用户，以及通过校验给出的密码是否与数据库中存储的密码匹配。

UserView 类将处理用户的注册请求，SessionView 类将处理用户的登录请求，PostListView 将处理获取文章列表和创建文章的请求。最后，PostView将处理获取单篇文章的请求。在文件的底部，我们简单的设置了 API 的资源路由。

# 创建数据库

我们也需要创建一个名称为 db_create.py 的文件。我们将运行这个脚本来初始化数据库。现在我们将进入 blog/server/ 目录并使用 `python db_create.py` 运行这个脚本。

```
from app.server import db
 
db.create_all()
```

# 运行 REST API 服务

我们需要创建的另外一个文件是 run.py。这是一个脚本，我们将通过运行它来运行一个 REST API 服务。

```
from app.server import app
 
app.run()
```

如果你正确的遵循了以上步骤，你现在应该可以通过进入 blog/server/ 目录并运行 `python run.py` 来运行 REST API 服务。

# 结束点

## 注册一个用户

为了注册一个用户你需要发送一个 POST 请求给 localhost:5000/api/v1/users。这个请求属性是 email 和 password。这里有 curl 请求来创建一个用户

```
curl --dump-header - -H "Content-Type: application/json" -X POST -d '{"email": "johndoe@gmail.com","password": "admin"}' http://localhost:5000/api/v1/users
```

## 登陆一个用户

为了登陆一个用户，你需要通过使用 POST 请求发送一个用户的登陆资格给 localhost:5000/api/v1/sessions。示例：

```
curl --dump-header - -H "Content-Type: application/json" -X POST -d '{"email": "johndoe@gmail.com","password": "admin"}' http://localhost:5000/api/v1/sessions
```

## 创建一篇文章

为了创建一篇文章，你需要发送一个 POST 请求给 localhost:5000/api/v1/posts。需要的属性是 title 和 body。因为创建文章的时候要求用户是已经登陆的，注意你需要发送一个包含 base64 编码的用户资格的 Authorization header ，它是通过冒号（":"）分离的。示例：

```
curl --dump-header - -H "Content-Type: application/json" -H "Authorization: Basic am9obmRvZUBnbWFpbC5jb206YWRtaW4=" -X POST -d '{"title": "Example post","body": "Lorem ipsum"}' http://localhost:5000/api/v1/posts
```

## 获取文章

为了获取所有保存的文章，你需要发送一个  GET 请求给http://localhost:5000/api/v1/posts 。示例：

```
curl -i http://localhost:5000/api/v1/posts
```

除了使用 curl，你也可以使用更高级的 REST 客户端 Chrome 扩展来测试 API。

# 下一步？

该教程的第一部分到这里就结束了。在教程的第二部分，我们将专注于构建一个 AngularJS web 应用，它将调用我们已经创建的 REST api。

如果你发现了这篇文章的一些错误或是你有一些问题想问，请在下面留下你的评论。

我希望你喜欢该教程的一部分。在教程的第二部分见。

可用的 REST API 源码在 Github 上，如果你喜欢这个教程，并且它对你有帮助，请给星，下载和 fork 它 - https://github.com/basco-johnkevin/building-a-blog-using-flask-and-angularjs

你也可以在 Twitter 上关注我 - https://twitter.com/johnkevinmbasco

  [1]: http://blog.john.mayonvolcanosoftware.com/
  [2]: http://blog.john.mayonvolcanosoftware.com/building-a-blog-using-flask-and-angularjs-part-1/
  [3]: http://files.mayonvolcanosoftware.com/webapp.jpg
  [4]: http://files.mayonvolcanosoftware.com/directory-structure.jpg