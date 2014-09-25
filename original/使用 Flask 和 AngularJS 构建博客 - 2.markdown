# 使用 Flask 和 AngularJS 构建博客 - 2

标签（空格分隔）： Flask AngularJS Python

---

> 注：该文作者是 [John Kevin M. Basco][1]，原文地址是 [Building a blog using Flask and AngularJS Part 2][2]


![此处输入图片的描述][3]

这是这个教程系列的第二部分，如果你还没有都第一部分，请移步到这里：http://blog.john.mayonvolcanosoftware.com/building-a-blog-using-flask-and-angularjs-part-1/

因为我们在该系列的第一部分已经构建好了 REST API ，在这部分我们将专注于构建一个 AngularJS 应用，用来使用我们构建的 REST API。

## 目录结构

AngularJS 应用的目录结构看起来像这样：

![此处输入图片的描述][4]


## 安装必要的包

我们将使用的包如下：

  - angular-route
  - bootstrap
  - restangular
  - angularjs
  - angular-local-storage

如果你不熟悉 Restangular 的话，我建议你首先读下 Restangular 的文档 - https://github.com/mgonto/restangular

或者如果你懒得读它的文档，随时继续阅读本教程，当你遇到如下对你没有意义的 Restangular 代码的时候，你可以参考下 Restangular 文档。


为了使得事情更容易，在 blog/client/ 目录创建一个 bower.json 文件，并且拷贝和粘贴以下内容：

```
{
  "name": "client",
  "version": "0.0.0",
  "authors": [
    "John Kevin Basco <basco.johnkevin@gmail.com>"
  ],
  "license": "MIT",
  "ignore": [
    "**/.*",
    "node_modules",
    "bower_components",
    "test",
    "tests"
  ],
  "dependencies": {
    "angular-route": "~1.2.21",
    "bootstrap": "~3.2.0",
    "restangular": "~1.4.0",
    "angularjs": "~1.2.21",
    "angular-local-storage": "~0.0.7"
  }
}
```

然后进入 blog/client 目录，运行 `bower install` 来安装必要的包。

## 初始化应用设置

首先，在 blog/client 目录下创建一个新文件，并命名为 index.html，然后拷贝和粘贴以下内容：

```
<!DOCTYPE html>
<html ng-app="Blog" ng-controller="ApplicationCtrl">
    <head>
        <meta charset="utf-8" />
        <title>Blog</title>
 
        <link rel="stylesheet" type="text/css" href="./bower_components/bootstrap/dist/css/bootstrap.css">
        <link rel="stylesheet" type="text/css" href="./css/theme.css">
        <link rel="stylesheet" type="text/css" href="./css/styles.css">
    </head>
    <body>
 
        <div class="blog-masthead">
            <div class="container">
                <nav class="blog-nav">
                    <a class="blog-nav-item" href="#" ng-class="{active: isActive('/')}">Home</a>
                    <a class="blog-nav-item" href="#/sessions/create" ng-hide="isLoggedIn" ng-class="{active: isActive('/sessions/create')}">Login</a>
                    <a class="blog-nav-item" href="#/sessions/destroy" ng-show="isLoggedIn">Logout</a>
                    <a class="blog-nav-item" href="#/users/create" ng-class="{active: isActive('/users/create')}">Register</a>
                    <a class="blog-nav-item" href="#/posts/create" ng-show="isLoggedIn" ng-class="{active: isActive('/posts/create')}">Create a post</a>
                </nav>
            </div>
        </div>
 
        <div class="container main-view">
 
            <div ng-view></div>
 
        </div><!-- /.container -->
 
        <div class="blog-footer">
            <p>Blog built by <a href="https://twitter.com/johnkevinmbasco">@johnkevinmbasco</a> using Flask and AngularJS.</p>
            <p>
                <a href="#">Back to top</a>
            </p>
        </div>
 
        <script type="text/javascript" src="./bower_components/angularjs/angular.js"></script>
        <script type="text/javascript" src="./bower_components/angular-route/angular-route.js"></script>
        <script type="text/javascript" src="./bower_components/lodash/dist/lodash.js"></script>
        <script type="text/javascript" src="./bower_components/restangular/dist/restangular.js"></script>
        <script type="text/javascript" src="./bower_components/angular-local-storage/angular-local-storage.js"></script>
        <script type="text/javascript" src="./js/main.js"></script>
        <script type="text/javascript" src="./js/controllers/HomeDetailCtrl.js"></script>
        <script type="text/javascript" src="./js/controllers/ApplicationCtrl.js"></script>
        <script type="text/javascript" src="./js/controllers/SessionCreateCtrl.js"></script>
        <script type="text/javascript" src="./js/controllers/SessionDestroyCtrl.js"></script>
        <script type="text/javascript" src="./js/controllers/UserCreateCtrl.js"></script>
        <script type="text/javascript" src="./js/controllers/PostCreateCtrl.js"></script>
        <script type="text/javascript" src="./js/factories/Session.js"></script>
        <script type="text/javascript" src="./js/factories/User.js"></script>
        <script type="text/javascript" src="./js/factories/Post.js"></script>
        <script type="text/javascript" src="./js/services/AuthService.js"></script>
        <script type="text/javascript" src="./js/directives/match.js"></script>
 
    </body>
</html>
```

在以上代码中，我们只是简单的包含了我们稍后将要创建的 css 和 javascript 文件。我们也为 navigation bar， footer 等等添加了 html markup。你将注意到在以上的 gist 中，它包含了 `<div ng-view></div>` 这个 html 元素。这是一个当前的路由将用来插入模板的地方。你将看到这些事情：`ng-class="{active: isActive('/')}"` 和 `ng-show="isLoggedIn"`，忽略它们现在。我们将在 为 ApplicationCtrl 写代码的时候讨论它们。

## 添加一些样式

现在让我们添加一些样式以便这个 app 看起来更好。为了使我们的工作更轻松，让我们使用一个 Bootstrap 提供的主题 - http://getbootstrap.com/examples/blog/。在 app/client/css 目录下创建一个新文件，并命名为 theme.css，然后拷贝和粘贴以下内容：

```
/*
 * Globals
 */
 
body {
    font-family: Georgia, "Times New Roman", Times, serif;
    color: #555;
}
 
h1, .h1,
h2, .h2,
h3, .h3,
h4, .h4,
h5, .h5,
h6, .h6 {
    margin-top: 0;
    font-family: "Helvetica Neue", Helvetica, Arial, sans-serif;
    font-weight: normal;
    color: #333;
}
 
 
/*
 * Override Bootstrap's default container.
 */
 
@media (min-width: 1200px) {
    .container {
        width: 970px;
    }
}
 
 
/*
 * Masthead for nav
 */
 
.blog-masthead {
    background-color: #428bca;
    -webkit-box-shadow: inset 0 -2px 5px rgba(0,0,0,.1);
                    box-shadow: inset 0 -2px 5px rgba(0,0,0,.1);
}
 
/* Nav links */
.blog-nav-item {
    position: relative;
    display: inline-block;
    padding: 10px;
    font-weight: 500;
    color: #cdddeb;
}
.blog-nav-item:hover,
.blog-nav-item:focus {
    color: #fff;
    text-decoration: none;
}
 
/* Active state gets a caret at the bottom */
.blog-nav .active {
    color: #fff;
}
.blog-nav .active:after {
    position: absolute;
    bottom: 0;
    left: 50%;
    width: 0;
    height: 0;
    margin-left: -5px;
    vertical-align: middle;
    content: " ";
    border-right: 5px solid transparent;
    border-bottom: 5px solid;
    border-left: 5px solid transparent;
}
 
 
/*
 * Blog name and description
 */
 
.blog-header {
    padding-top: 20px;
    padding-bottom: 20px;
}
.blog-title {
    margin-top: 30px;
    margin-bottom: 0;
    font-size: 60px;
    font-weight: normal;
}
.blog-description {
    font-size: 20px;
    color: #999;
}
 
 
/*
 * Main column and sidebar layout
 */
 
.blog-main {
    font-size: 18px;
    line-height: 1.5;
}
 
/* Sidebar modules for boxing content */
.sidebar-module {
    padding: 15px;
    margin: 0 -15px 15px;
}
.sidebar-module-inset {
    padding: 15px;
    background-color: #f5f5f5;
    border-radius: 4px;
}
.sidebar-module-inset p:last-child,
.sidebar-module-inset ul:last-child,
.sidebar-module-inset ol:last-child {
    margin-bottom: 0;
}
 
 
 
/* Pagination */
.pager {
    margin-bottom: 60px;
    text-align: left;
}
.pager > li > a {
    width: 140px;
    padding: 10px 20px;
    text-align: center;
    border-radius: 30px;
}
 
 
/*
 * Blog posts
 */
 
.blog-post {
    margin-bottom: 60px;
}
.blog-post-title {
    margin-bottom: 5px;
    font-size: 40px;
}
.blog-post-meta {
    margin-bottom: 20px;
    color: #999;
}
 
 
/*
 * Footer
 */
 
.blog-footer {
    padding: 40px 0;
    color: #999;
    text-align: center;
    background-color: #f9f9f9;
    border-top: 1px solid #e5e5e5;
}
.blog-footer p:last-child {
    margin-bottom: 0;
}
```

在 app/client/css 目录下创建另外一个文件，命名为 styles.css ，然后拷贝和粘贴一下内容：

```
.main-view {
    margin-top: 20px;
    margin-bottom: 20px;
}
```

现在进入 blog/client 目录，然后运行这个命令：`python -m SimpleHTTPServer 8000`，现在访问 `http://localhost:8000/ `，你应该可以看到应用已经有设计样式。

## 创建必须的 javascript 文件

注意：以下应用使用的配置变量的值，为了假话，被硬编码了。在生产环境的应用中，需要把它放入一个配置文件中，因此你可以根据不同的环境使用不同的值，比如开发环境和生产环境。

现在让我们创建必须的 javascript 文件，在 blog/client/js 目录下创建一个新文件，命名为 main.js 并且以下的代码拷贝和粘贴进文件：

```
window.Blog = angular.module('Blog', ['ngRoute', 'restangular', 'LocalStorageModule'])
 
.run(function($location, Restangular, AuthService) {
    Restangular.setFullRequestInterceptor(function(element, operation, route, url, headers, params, httpConfig) {
        headers['Authorization'] = 'Basic ' + AuthService.getToken();
        return {
            headers: headers
        };
    });
 
    Restangular.setErrorInterceptor(function(response, deferred, responseHandler) {
        if (response.config.bypassErrorInterceptor) {
            return true;
        } else {
            switch (response.status) {
                case 401:
                    AuthService.logout();
                    $location.path('/sessions/create');
                    break;
                default:
                    throw new Error('No handler for status code ' + response.status);
            }
            return false;
        }
    });
})
 
.config(function($routeProvider, RestangularProvider) {
 
    RestangularProvider.setBaseUrl('http://localhost:5000/api/v1');
 
    var partialsDir = '../partials';
 
    var redirectIfAuthenticated = function(route) {
        return function($location, $q, AuthService) {
 
            var deferred = $q.defer();
 
            if (AuthService.isAuthenticated()) {
                deferred.reject()
                $location.path(route);
            } else {
                deferred.resolve()
            }
 
            return deferred.promise;
        }
    }
 
    var redirectIfNotAuthenticated = function(route) {
        return function($location, $q, AuthService) {
 
            var deferred = $q.defer();
 
            if (! AuthService.isAuthenticated()) {
                deferred.reject()
                $location.path(route);
            } else {
                deferred.resolve()
            }
 
            return deferred.promise;
        }
    }
 
    $routeProvider
        .when('/', {
            controller: 'HomeDetailCtrl',
            templateUrl: partialsDir + '/home/detail.html'
        })
        .when('/sessions/create', {
            controller: 'SessionCreateCtrl',
            templateUrl: partialsDir + '/session/create.html',
            resolve: {
                redirectIfAuthenticated: redirectIfAuthenticated('/posts/create')
            }
        })
        .when('/sessions/destroy', {
            controller: 'SessionDestroyCtrl',
            templateUrl: partialsDir + '/session/destroy.html'
        })
        .when('/users/create', {
            controller: 'UserCreateCtrl',
            templateUrl: partialsDir + '/user/create.html'
        })
        .when('/posts/create', {
            controller: 'PostCreateCtrl',
            templateUrl: partialsDir + '/post/create.html',
            resolve: {
                redirectIfNotAuthenticated: redirectIfNotAuthenticated('/sessions/create')
            }
        });
})
```

在以上代码中，我们初始化了我们的应用，配置 Restangular 来包括每个请求的登录用户的用户名和密码，配置 Restangular 来使用一个错误拦截器，它将删除本地存储的所有已经保存的数据，并且如果 REST API 返回一个 401 状态码，则把用户跳转到登录页面。

在配置部分，我们设置了 Restangular 将使用的基准 url，我们也设置了路由。你将注意到 /posts/create 路由有一个解决属性包含了 redirectIfNotAuthenticated promise。是的，像你猜的那样，如果用户未被授权，它将把用户跳转到登录页面。/sessions/create 路由也有一个解决属性，但是它包含了 redirectIfAuthenticated promise 而不是 redirectIfNotAuthenticated promise。如果用户被授权了，它将简单的跳转用户到创建博客的页面。你可以阅读关于这个主题的更多信息 - http://blog.john.mayonvolcanosoftware.com/protecting-routes-in-angularjs/。

## Factories

让我们创建我们的应用将使用到的工厂方法。

## Post

让我们在 blog/client/js/factories 目录下创建一个新文件，命名为 Post.js ，然后拷贝并粘贴一下代码到文件中：

```
Blog.factory('Post', function(Restangular) {
    var Post;
    Post = {
        get: function() {
            return Restangular
                .one('posts')
                .getList();
        },
        create: function(data) {
            return Restangular
                .one('posts')
                .customPOST(data);
        }
    };
    return Post;
})
```

在以上代码中，factory 将创建一个对象，该对象有一个 get 和 create 方法。我们将使用 get 方法来获取博客列表，create 方法来创建一篇博客。


## Session

让我们在 blog/client/js/factories 目录下创建一个新文件，命名为 Session.js  ，然后拷贝并粘贴一下代码到文件中：

```
Blog.factory('Session', function(Restangular) {
    var Session;
    Session = {
        create: function(data, bypassErrorInterceptor) {
            return Restangular
                .one('sessions')
                .withHttpConfig({bypassErrorInterceptor: bypassErrorInterceptor})
                .customPOST(data);
        }
    };
    return Session;
})
```

上面的 Session  工厂有一个 create 方法返回这个对象。我们将用这个 create 方法来授权一个用户的 email 和密码。你将注意到 create 方法有一个参数叫做 bypassErrorInterceptor。这个参数应该是一个布尔类型的值（true or false），当 bypassErrorInterceptor 是 true 的时候，它将绕过我们早先在配置块中定义的 error 拦截器。这不会立即显得有意义，但是在后面我们将遇到一个场景，在那我们需要绕过 error 拦截器。


## User

在 blog/client/js/factories 目录下创建一个新文件，并命名为 User.js，然后把以下代码拷贝和粘贴进文件：

```
Blog.factory('User', function(Restangular) {
    var User;
    User = {
        create: function(user) {
            return Restangular
                .one('users')
                .customPOST(user);
        }
    };
    return User;
}
```

User 工厂将有一个 create 方法返回该对象。我们将使用 create 方法创建一个 user。

## Services

我们的应用需要一个服务，该服务包含登录，登出，和校验该用户是否被授权的业务逻辑。我们把该服务命名为 AuthService。在 blog/cient/js/services 目录下创建一个新文件，并命名为 AuthService.js，然后拷贝和粘贴一下代码到文件中：

```
Blog.service('AuthService', AuthService = function($q, localStorageService, Session) {
 
    this.login = function(credentials) {
        var me = this;
        deferred = $q.defer()
        Session.create(credentials, true).then(function(user) {
            me.setToken(credentials);
            return deferred.resolve(user);
        }, function(response) {
            if (response.status == 401) {
                return deferred.reject(false);
            }
            throw new Error('No handler for status code ' + response.status);
        });
        return deferred.promise
    };
 
    this.logout = function() {
        localStorageService.clearAll();
    };
 
    this.isAuthenticated = function() {
        var token = this.getToken();
        if (token) {
            return true
        }
        return false;
    };
 
    this.setToken = function(credentials) {
        localStorageService.set('token', btoa(credentials.email + ':' + credentials.password));
    };
 
    this.getToken = function() {
        return localStorageService.get('token');
    };
 
    return this;
}
```

以上的 login 方法使用 `Session.create` 认证用户的认证信息。当登录认证信息是有效的，我们将保存 base64  编码格式的 email 和密码到本地存储稍后使用，并授权。当登录认证信息是无效的（当登录认证信息是无效的，REST API 将返回一个 401 状态码），我们简单的拒绝授权，并且让使用该方法的代码处理它。看一下我们传递给 `Session.create` 的第二个参数。我们传递一个布尔类型（true），这个就是我们早先讨论的 bypassErrorInterceptor 的参数。我们需要绕过在配置块中的 error 拦截器，因为如果我们不这样做，处理 401 错误码的 handler 将把用户跳转到登录视图。我们想这个发生在其他场景，但当我们正在登录视图并且用户提交了错误的认证信息的时候，我们想展示一个错误消息，比如“Incorrect login credentials. Please try again.”。这就是为什么我们需要绕过默认的 error 拦截器，使用一个新的，包含了不同的业务逻辑（检查以上代码的 10 - 15 行）。

logout 方法将清理保存在本地存储的数据，isAuthenticated 方法仅仅通过检查 token 是否被呈现来检查当前用户是否被授权。setToken 方法保存一个 token （base64 编码格式的 email 和 password）到本地存储中，getToken 方法获取保存在本地存储中的 token。


## Directives

我们应用的 registration form 将有一个 password 和 一个确认 password  字段，因此我们需要一个方法来校验值是否匹配。为这个我们将使用 directive，在 blog/client/js/directives 目录下创建一个新文件，并命名为 match.js，然后拷贝和粘贴一下代码到文件中：

```
Blog.directive('match', function () {
    return {
        require: 'ngModel',
        restrict: 'A',
        scope: {
            match: '='
        },
        link: function(scope, elem, attrs, ctrl) {
            scope.$watch(function() {
                return (ctrl.$pristine && angular.isUndefined(ctrl.$modelValue)) || scope.match === ctrl.$modelValue;
            }, function(currentValue) {
                ctrl.$setValidity('match', currentValue);
            });
        }
    };
}
```

信任以上 directive 的作者。我是从别处抓取来了以上源码，但是我没有忘记链接到该站点和谁是作者。

我们现在完成了我们 AngularJS 应用程序的一半。我们还没有写的剩下部分是控制层和模板。因为这篇文章太长了，我决定在第三部分写控制层和模板部分。同时，研究目前为止我们已经编写的代码，让我们更熟悉它。再使用  Flask 和 AngularJS 构建博客教程系列的第三部分见。

所有的 REST API 源码都在 Github 上 - https://github.com/basco-johnkevin/building-a-blog-using-flask-and-angularjs ，但是 AngularJS 应用程序的源码还没有放上去，因为我们还没有完成它，等到第三部分写完以后再放全部的源码到 Github 上。










  [1]: http://blog.john.mayonvolcanosoftware.com/
  [2]: http://blog.john.mayonvolcanosoftware.com/building-a-blog-using-flask-and-angularjs-part-2/?utm_source=Python%20Weekly%20Newsletter&utm_campaign=4d7e0202aa-Python_Weekly_Issue_156_September_11_2014&utm_medium=email&utm_term=0_9e26887fc5-4d7e0202aa-312702461
  [3]: http://files.mayonvolcanosoftware.com/flask-angularjs-blog.jpg
  [4]: http://files.mayonvolcanosoftware.com/angularjs-blog-directory-structure.jpg