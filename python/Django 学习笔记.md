# Django 学习笔记

## 安装

```bash
# 安装指定版本
pip3 install Django==3.1.1

# 安装最新版本
pip3 install Django

# 查看安装的版本
python -m django --version
```

## 手动创建项目

```bash
# 这行代码将会在当前目录下创建一个 mysite 目录
$ django-admin startproject mysite
```

mysite的文件结构如下

```bash
├── manage.py
└── mysite
    ├── __init__.py
    ├── asgi.py
    ├── settings.py
    ├── urls.py
    └── wsgi.py
```

这些目录和文件的用处是：

- 最外层的 `mysite/` 根目录只是你项目的容器， 根目录名称对Django没有影响，你可以将它重命名为任何你喜欢的名称。
- `manage.py`: 一个让你用各种方式管理 Django 项目的命令行工具。阅读 [django-admin and manage.py](https://docs.djangoproject.com/zh-hans/3.1/ref/django-admin/) 
- 里面一层的 `mysite/` 目录包含你的项目，它是一个纯 Python 包。它的名字就是当你引用它内部任何东西时需要用到的 Python 包名。 (比如 `mysite.urls`).
- `mysite/__init__.py`：一个空文件，告诉 Python 这个目录应该被认为是一个 Python 包
- `mysite/settings.py`：Django 项目的配置文件。查看 [Django 配置](https://docs.djangoproject.com/zh-hans/3.1/topics/settings/) 
- `mysite/urls.py`：Django 项目的 URL 声明，就像你网站的“目录”。阅读 [URL调度器](https://docs.djangoproject.com/zh-hans/3.1/topics/http/urls/) 
- `mysite/asgi.py`：作为你的项目的运行在 ASGI 兼容的Web服务器上的入口。阅读 [如何使用 ASGI 来部署](https://docs.djangoproject.com/zh-hans/3.1/howto/deployment/asgi/) 
- `mysite/wsgi.py`：作为你的项目的运行在 WSGI 兼容的Web服务器上的入口。阅读 [如何使用 WSGI 进行部署](https://docs.djangoproject.com/zh-hans/3.1/howto/deployment/wsgi/) 

### 启动项目

进入项目根目录

```bash
# 启动项目
python manage.py runserver

# 指定启动端口
python manage.py runserver 8080
```

用于开发的服务器在需要的情况下会对每一次的访问请求重新载入一遍 Python 代码。所以你不需要为了让修改的代码生效而频繁的重新启动服务器。然而，一些动作，比如添加新文件，将不会触发自动重新加载，这时你得自己手动重启服务器。

### 创建应用

在 `Django` 中，每一个应用都是一个 `Python` 包

**项目和应用的区别**

**应用**是一个专门做某件事的网络应用程序——比如博客系统，或者公共记录的数据库，或者小型的投票程序

**项目**则是一个网站使用的配置和应用的集合。项目可以包含很多个应用。应用可以被多个项目使用

```python
# 在项目根目录执行下面命令，将创建一个 polls 目录(polls 应用)
python manage.py startapp polls
```

## Django 配置

Django 的配置文件包含 Django 应用的所有配置项

1. 如果将 [`DEBUG`](https://docs.djangoproject.com/zh-hans/3.1/ref/settings/#std:setting-DEBUG) 设置为 `False`，同时你需要正确的设置 [`ALLOWED_HOSTS`](https://docs.djangoproject.com/zh-hans/3.1/ref/settings/#std:setting-ALLOWED_HOSTS)。

```python
# 只允许列表中的地址访问
ALLOWED_HOSTS = ['www.example.com']

# 任何地址都可以访问，失去了防护，测试项目可这样配置
ALLOWED_HOSTS = ['*']
```

2. `INSTALLED_APPS` 注册应用

   创建的应用必须在这里注册，否则该应用不起作用

   ```python
   INSTALLED_APPS = [
       'django.contrib.admin',
       'django.contrib.auth',
       'django.contrib.contenttypes',
       'django.contrib.sessions',
       'django.contrib.messages',
       'django.contrib.staticfiles',
     	'polls'	# 注册应用
   ]
   ```

3. `ROOT_URLCONF` 根路由配置

4. `TEMPLATES` 模板配置

5. `WSGI_APPLICATION` 程序入口

6. `[DATABASES` 数据库配置](https://docs.djangoproject.com/zh-hans/3.1/ref/settings/#databases)

  字典，其中包含所有数据库的设置

  必须配置一个`default`数据库。也可以指定任意数量的其他数据库

  `SQLite` 配置如下

  ```python
  DATABASES = {
      'default': {
          'ENGINE': 'django.db.backends.sqlite3',
          'NAME': 'mydatabase',
      }
  }
  ```

  `Mysql` 配置如下

  ```python
  DATABASES = {
      'default': {
          'ENGINE': 'django.db.backends.mysql',
          'NAME': 'mydatabase',
          'USER': 'mydatabaseuser',
          'PASSWORD': 'mypassword',
          'HOST': '127.0.0.1',
          'PORT': '3306',
      }
  }
  ```

  [其他可用配置项](https://docs.djangoproject.com/zh-hans/3.1/ref/settings/#databases)

2. `LANGUAGE_CODE` 语言环境配置

   ```python
   # 英文
   LANGUAGE_CODE = 'en-us'
   
   # 中文
   LANGUAGE_CODE = 'zh-hans'
   ```

3. `TIME_ZONE` 数据库连接时区设置

   ```python
   # 中国时区
   TIME_ZONE='Asia/Shanghai'
   ```

4. `USE_TZ`

   是否使用默认时区，设置为 `True` 则系统使用默认的时区，此时设置的 `TIME_ZONE` 将不起作用

   设置`USE_TZ = False, TIME_ZONE = 'Asia/Shanghai'`, 则使用上海的UTC时间

5. [静态资源配置](https://docs.djangoproject.com/zh-hans/3.1/howto/static-files/)

  1. 配置文件 `settings.py` 中确保 [`INSTALLED_APPS`](https://docs.djangoproject.com/zh-hans/3.1/ref/settings/#std:setting-INSTALLED_APPS) 包含了 `django.contrib.staticfiles` 

  2. 配置文件 `settings.py` 中配置 `STATICFILES_DIRS` , `Django` 会从中寻找静态文件
    
```python
     STATICFILES_DIRS = [
      BASE_DIR / "oo/static",
         '/var/www/static/',
       	# 项目根目录下的 static 文件夹
       	os.path.join(BASE_DIR, 'static'),
  ]
```

   3. 模板中使用
   
      ```django
      <!-- 第一种 -->
      <link rel="stylesheet" type="text/css" href="/static/style/register-login.css">
      
      ```

   <!-- 第二种 -->
      <!-- 1.html头部添加 -->
      {% load staticfiles %}
      <html>
      <head>
      <title></title>
      </head>
      <body>
      <!-- 2. 使用 -->
      <img src="{% static 'images/logo.gif' %}" alt=""/>
      <img src="/static/images/acer.gif" alt=""/>
      ```

## 路由

**进行`URL`匹配时 `Django` 不包括 `GET` 或 `POST` 请求方式的参数以及域名。即不检查是 `GET` 还是 `POST` 请求，对同一个 `URL` 无论是 `POST请求` 、 `GET请求` 、或 `HEAD请求` 等都路由到相同的函数**

### Django 如何处理一个请求

当一个用户请求 `Django` 站点的一个页面时：

1. `Django` 查找根 `URL` 配置模块。 [`ROOT_URLCONF`](https://docs.djangoproject.com/zh-hans/3.1/ref/settings/#std:setting-ROOT_URLCONF) （项目同名的目录的 `settings.py` 中）设置的值，根路由默认是 `项目名/项目名/urls.py`

2. `Django` 加载该 `Python` 模块并寻找可用的 `urlpatterns` 。它是 [`django.urls.path()`](https://docs.djangoproject.com/zh-hans/3.1/ref/urls/#django.urls.path) 和(或) [`django.urls.re_path()`](https://docs.djangoproject.com/zh-hans/3.1/ref/urls/#django.urls.re_path) 实例的序列

3. `Django` 会**按顺序遍历**每个 `URL` 模式，然后找到第一个匹配项后停止匹配，并与 [`path_info`](https://docs.djangoproject.com/zh-hans/3.1/ref/request-response/#django.http.HttpRequest.path_info) 匹配

4. 一旦有 `URL` 匹配成功，`Djagno` 导入并调用相关的视图，这个视图是一个Python 函数（或基于类的视图 [class-based view](https://docs.djangoproject.com/zh-hans/3.1/topics/class-based-views/) ）。视图会获得如下参数：

   - 一个 [`HttpRequest`](https://docs.djangoproject.com/zh-hans/3.1/ref/request-response/#django.http.HttpRequest) 实例
   - 如果匹配的 URL 包含未命名组，那么来自正则表达式中的匹配项将作为位置参数提供。
   - 关键字参数由路径表达式匹配的任何命名部分组成，并由 [`django.urls.path()`](https://docs.djangoproject.com/zh-hans/3.1/ref/urls/#django.urls.path) 或 [`django.urls.re_path()`](https://docs.djangoproject.com/zh-hans/3.1/ref/urls/#django.urls.re_path) 的可选 `kwargs` 参数中指定的任何参数覆盖

5. 如果没有 `URL` 被匹配，或者匹配过程中出现了异常，`Django` 会调用一个适当的错误处理视图

### path 路径函数

```python
path(route, view, kwargs=None, name=None)
# route: 字符串类型的 url
# view: 视图函数
# kwargs: 允许您将其他参数传递给视图函数或方法
# name: 给当前的url起个名字，用于反向解析URL
```

```python
from django.urls import path

from . import views

urlpatterns = [
    path('articles/2003/', views.special_case_2003),
    path('articles/<int:year>/', views.year_archive),
    path('articles/<int:year>/<int:month>/', views.month_archive),
    path('articles/<int:year>/<int:month>/<slug:slug>/', views.article_detail),
]
```

注意：

- 要从 `URL` 中取值，使用尖括号
- 捕获的值可以选择性地包含转换器类型。比如，使用 `<int:year>` 来捕获整型参数。如果不包含转换器，则会匹配除了 `/` 外的任何字符
- 开头不需要添加反斜杠`/`，因为每个 `URL` 都有。比如，应该是 `articles` 而不是 `/articles` 

#### 路径转换器

下面的路径转换器在默认情况下是有效的：

- `str` - 匹配除了 `/` 之外的非空字符串。如果表达式内不包含转换器，则会默认匹配字符串
- `int` - 匹配 `0` 或任何正整数。返回一个 `int` 
- `slug` - 匹配任意由 ASCII 字母或数字以及`-`和`_`组成的短标签。比如，`building-your-1st-django-site` 
- `uuid` - 匹配一个格式化的 UUID
- `path` - 匹配任何非空字符，包括路径分隔符 `/` 。它允许你匹配完整的 URL 路径而不是像 `str` 那样匹配 URL 的一部分。如果有多个参数 `path` 必须是最后一个，否则其后的所有 `URL` 都会被 `path` 接收

#### 注册自定义的路径转换器

对于更复杂的匹配需求，你能定义你自己的路径转换器。

转换器是一个类，包含如下内容：

- 字符串形式的 `regex` 类属性。

- `to_python(self, value)` 方法，用来处理匹配的字符串转换为传递到函数的类型。如果没有转换为给定的值，它应该会引发 `ValueError` 。`ValueError` 说明没有匹配成功，因此除非另一个 URL 模式匹配成功，否则会向用户发送404响应。

- `to_url(self, value)` 方法， 该方法处理将`Python`类型转换为要在`URL`中使用的字符串。如果无法转换给定值，则应引发`ValueError`。`ValueError`被解释为不匹配

```python
class FourDigitYearConverter:
    regex = '[0-9]{4}'

    def to_python(self, value):
        return int(value)

    def to_url(self, value):
        return '%04d' % value
```

在 `URLconf` 中使用 [`register_converter()`](https://docs.djangoproject.com/zh-hans/3.1/ref/urls/#django.urls.register_converter) 来注册自定义的转换器类：

```python
from django.urls import path, register_converter

from . import converters, views

register_converter(converters.FourDigitYearConverter, 'yyyy')

urlpatterns = [
    path('articles/2003/', views.special_case_2003),
    path('articles/<yyyy:year>/', views.year_archive),
    ...
]
```

### re_path 正则表达式路径函数

```python
re_path(route, view, kwargs=None, name=None)
# route: 原始字符串的正则表达式, 例如：r'^articles/(?P<year>[0-9]{4})/$'
# view: 视图函数
# kwargs: 允许您将其他参数传递给视图函数或方法
# name: 给当前的url起个名字，用于反向解析URL
```

在 `Python` 正则表达式中，命名正则表达式组的语法是 `(?P<name>pattern)` ，其中 `name` 是组名，`pattern` 是要匹配的模式

#### 示例

```python
from django.urls import path, re_path

from . import views

urlpatterns = [
    path('articles/2003/', views.special_case_2003),
    re_path(r'^articles/(?P<year>[0-9]{4})/$', views.year_archive),
    re_path(r'^articles/(?P<year>[0-9]{4})/(?P<month>[0-9]{2})/$', views.month_archive),
    re_path(r'^articles/(?P<year>[0-9]{4})/(?P<slug>[\w-]+)/$', views.article_detail),
]
```

#### 未命名正则表达式组

命名组语法，例如 `(?P<year>[0-9]{4})` ，你也可以使用更短的未命名组，例如 `([0-9]{4})` 。

不推荐这个用法，因为它容易引发错误

推荐在给定的正则表达式里只使用一个样式。当混杂两种样式时，任何未命名的组会被忽略，只有命名的组才会传递给视图函数

### include() 函数

`include()` 函数用于导入其他应用（包）里面的 `urlpatterns` 路由列表，并将其他应用的 `URL` 追加到后面

```python
from django.urls import include, path

urlpatterns = [
    # ... snip ...
    path('community/', include('aggregator.urls')),
    path('contact/', include('contact.urls')),
    # ... snip ...
]
```

每当 `Django` 遇到 `include()`，它会将匹配到那部分 `URL` 切掉，并将剩余的字符串发送到包含的 `URLconf`进行进一步处理

使用 `path()` 实例的列表来包含其他 `URL` 模式

```python
from django.urls import include, path

from apps.main import views as main_views
from credit import views as credit_views

extra_patterns = [
    path('reports/', credit_views.report),
    path('reports/<int:id>/', credit_views.report),
    path('charge/', credit_views.charge),
]

urlpatterns = [
    path('', main_views.homepage),
    path('help/', include('apps.help.urls')),
    path('credit/', include(extra_patterns)),
]
```

在这个例子中， `/credit/reports/` URL将被 `credit.views.report()` 这个Django 视图处理

### 给视图传递额外的参数

`path()` 的第三个字典参数可用于将额外的参数传递给视图

```python
from django.urls import path
from . import views

urlpatterns = [
    path('blog/<int:year>/', views.year_archive, {'foo': 'bar'}),
]
```

在这个例子里，当请求到 `/blog/2005/` 时，`Django` 将调用 `views.year_archive(request, year=2005, foo='bar')` 

### URL 的反向解析

在 `Django` 项目中，一个常见需求是获取最终形式的 URL，比如用于嵌入生成的内容中（视图和资源网址，给用户展示网址等）或用户服务器端的导航处理（重定向等）

`Django` 提供了一个解决方案，使得 `URL` 映射是 `URL` 设计唯一的仓库。你使用 `URLconf` 来设置它，然后可以双向使用它：

- 从用户/浏览器请求的 `URL` 开始，它调用正确的 `Django` 视图，并从 `URL `中提取相应参数
- 从相应的 `Django` 视图以及要传递给它的参数来获取相关联的 `URL`

第二条就是`反向解析 URL`

`Django` 提供反向解析 `URL` 的工具:

- 在模板里：使用 [`url`](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#std:templatetag-url) 模板标签
- 在 Python 编码：使用 [`reverse()`](https://docs.djangoproject.com/zh-hans/3.1/ref/urlresolvers/#django.urls.reverse) 函数
- 在与 Django 模型实例的 URL 处理相关的高级代码中： [`get_absolute_url()`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/instances/#django.db.models.Model.get_absolute_url) 方法

```python
from django.urls import path

from . import views

urlpatterns = [
    #...
    path('articles/<int:year>/', views.year_archive, name='news-year-archive'),
    #...
]
```

根据这个设计，与 year *nnnn* 相对应的 URL 是 `/articles/<nnnn>/` 。

你可以使用以下方式在模板代码中来获取`URL`：

```html
<a href="{% url 'news-year-archive' %}">不带参数</a>
<a href="{% url 'news-year-archive' 2200 2201 %}">带位置参数</a>
<a href="{% url 'namespace:news-year-archive' year=2200 name='xxx' %}">带命名参数, 还有命名空间</a>
```

在 `python` 代码中获取 `URL`

```python
from django.http import HttpResponseRedirect
from django.urls import reverse

def redirect_to_year(request):
    # ...
    year = 2006
    # 如果name有同名的话就会出错，此时需要 URL命名空间
    return HttpResponseRedirect(reverse('news-year-archive', args=(year,)))
```

#### URL 命名空间

`URL 命名空间` 允许你使用唯一的反向命名URL模式，即便不同应用程序使用相同的 `URL 名称` 

使用了 `URL命名空间` 需要使用 `:` 操作符。比如，使用 `admin:index` 引用 `admin` 应用的首页。这表明命名空间为 `admin` ，命名 URL 为`index` 

`urls.py` 文件中，加上 `app_name` 设置命名空间

**使用**

```python
# polls/urls.py
from django.urls import path

from . import views

# 设定 URL命名空间 为 polls
# 如果没有特殊要求，则命名空间的名字就是他所在包的名称
app_name = 'polls'
urlpatterns = [
    path('', views.IndexView.as_view(), name='index'),
    path('<int:pk>/', views.DetailView.as_view(), name='detail'),
    ...
]
```

按照上面的设置，则可使用下面方式反向解析`url` 

```python
reverse('polls:index', current_app=self.request.resolver_match.namespace)
```

## 视图

一个视图函数（或简称为视图）是一个 `Python` 函数，它接受 `Web` 请求并返回一个 `Web` 响应。这个响应可以是 `Web` 页面的 `HTML` 内容，或者重定向，或者404错误，或者 `XML` 文档，一个图片或是任何内容。视图本身包含返回响应所需的任何逻辑。这个代码可以存在任何地方，只要它在你的 `Python` 路径上就行。`Django`中约定将视图放在名为 `views.py` 的文件里，这个文件在项目或应用目录里

[render()](https://docs.djangoproject.com/zh-hans/3.1/topics/http/shortcuts/#django.shortcuts.render) 渲染模板

可快捷创建响应对象 `HttpResponse`

```python
render(request, template_name, context=None, content_type=None, status=None, using=None)
```

将给定的模板与给定的文字典组合在一起，并以渲染的文本返回一个 [`HttpResponse`](https://docs.djangoproject.com/zh-hans/3.1/ref/request-response/#django.http.HttpResponse) 对象

必选参数

- `request`

  `HttpRequest`请求对象

- `template_name`

  要使用的模板的全名或模板名称的序列
  

可选参数

- `context`

  要添加到模板的字典。 默认情况下，这是一个空的字典。 如果字典中的值是可调用的，则视图将在渲染模板之前调用它

- `content_type`

  用于结果文档的MIME 类型。默认 `text/html` 

- `status`

  响应的状态代码默认`200`

- `using`

  用于加载模板的模板引擎

[redirect()](https://docs.djangoproject.com/zh-hans/3.1/topics/http/shortcuts/#django.shortcuts.redirect) 重定向

将返回 [`HttpResponseRedirect`](https://docs.djangoproject.com/zh-hans/3.1/ref/request-response/#django.http.HttpResponseRedirect) 对象，该对象是 `HttpResponse` 的子类，表示重定向

```python
redirect(to, *args, permanent=False, **kwargs)
```

`permanent=True` 会返回一个永久重定向

`redirect()` 的使用

1. 传递对象，对象的 [`get_absolute_url()`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/instances/#django.db.models.Model.get_absolute_url) 方法将被调用来指向重定向地址

   ```python
   from django.shortcuts import redirect
   
   def my_view(request):
       ...
       obj = MyModel.objects.get(...)
       return redirect(obj)
   ```

2. 传递视图名和一些可选的位置或关键字参数；URL 将使用 [`reverse()`](https://docs.djangoproject.com/zh-hans/3.1/ref/urlresolvers/#django.urls.reverse) 方法来反向解析

   ```python
   def my_view(request):
       ...
       return redirect('some-view-name', foo='bar')
   ```

3. 传递硬编码 URL 来重定向

   ```python
   def my_view(request):
       ...
       return redirect('/some/url/')
   ```

### 一个简单的视图

```python
from django.http import HttpResponse
import datetime

def current_datetime(request):
    now = datetime.datetime.now()
    html = "<html><body>It is now %s.</body></html>" % now
    return HttpResponse(html)
```

说明：

- 我们从 [`django.http`](https://docs.djangoproject.com/zh-hans/3.1/ref/request-response/#module-django.http) 模块导入类 [`HttpResponse`](https://docs.djangoproject.com/zh-hans/3.1/ref/request-response/#django.http.HttpResponse) ，以及 `Python` 的 `datetime` 库
- 然后，我们定义一个名为 `current_datetime` 的函数。这是一个视图函数。**每个视图函数都将 [`HttpRequest`](https://docs.djangoproject.com/zh-hans/3.1/ref/request-response/#django.http.HttpRequest) 对象作为第一个参数，通常名为 `request`** 
- 视图返回一个包含生成的响应的 [`HttpResponse`](https://docs.djangoproject.com/zh-hans/3.1/ref/request-response/#django.http.HttpResponse) 对象。**每个视图函数都要返回 [`HttpResponse`](https://docs.djangoproject.com/zh-hans/3.1/ref/request-response/#django.http.HttpResponse) 对象**

#### 传递所有参数

如果需要一次将函数中的所有局部变量传递给另外的函数，则可使用内置函数 `locals()` 

`locals()` 函数会以字典类型返回当前位置的全部局部变量

```python
def print_vars():
    name = "aaa"
    age = 12
    # locals() 函数会将 name 和 age 以字典的类型一次性都传递过去
    print(locals())

print_vars()
# {'name': 'aaa', 'age': 12}
```

### 返回错误信息

[`HttpResponse`](https://docs.djangoproject.com/zh-hans/3.1/ref/request-response/#django.http.HttpResponse) 的子类除了200外，还有很多常见的 HTTP 状态代码。你可以在 [request/response](https://docs.djangoproject.com/zh-hans/3.1/ref/request-response/#ref-httpresponse-subclasses) 文档中找到所有可用子类的列表。返回这些子类中某个子类的实例而不是 [`HttpResponse`](https://docs.djangoproject.com/zh-hans/3.1/ref/request-response/#django.http.HttpResponse) 来表示错误

```python
from django.http import HttpResponse, HttpResponseNotFound

def my_view(request):
    # ...
    if foo:
        return HttpResponseNotFound('<h1>Page not found</h1>')
    else:
        return HttpResponse('<h1>Page was found</h1>')
```

可以将 `HTTP` 状态代码传递给 [`HttpResponse`](https://docs.djangoproject.com/zh-hans/3.1/ref/request-response/#django.http.HttpResponse) 的构造函数，这样就可以为任何状态代码创建返回类

```python
from django.http import HttpResponse

def my_view(request):
    # ...

    # Return a "created" (201) response code.
    return HttpResponse(status=201)
```

### `Http404` 异常

当你返回错误，例如 [`HttpResponseNotFound`](https://docs.djangoproject.com/zh-hans/3.1/ref/request-response/#django.http.HttpResponseNotFound) ，你需要定义错误页面的 `HTML` 

`Django` 提供 `Http404` 异常。如果你在视图的任何地方引发了 `Http404` ，`Django` 会捕捉到它并且返回标准的错误页面，连同 `HTTP` 错误代码 `404` 

```python
from django.http import Http404
from django.shortcuts import render
from polls.models import Poll

def detail(request, poll_id):
    try:
        p = Poll.objects.get(pk=poll_id)
    except Poll.DoesNotExist:
        raise Http404("Poll does not exist")
    return render(request, 'polls/detail.html', {'poll': p})
```

为了在 `Django` 返回`404`时显示自定义的 `HTML`，你可以创建名为 `404.html` 的`HTML`模板，并将其放置在你的模板根目录中。

 [`DEBUG`](https://docs.djangoproject.com/zh-hans/3.1/ref/settings/#std:setting-DEBUG) = `False` 时自定义的错误 `HTML` 模板生效

 [`DEBUG`](https://docs.djangoproject.com/zh-hans/3.1/ref/settings/#std:setting-DEBUG) = `True` 时，自定义的错误 `HTML` 模板不会生效，你可以提供 `Http404` 信息，并且在标准的 404 调试模板里显示

### 自定义报错处理

1. 在 `views.py` 中（任意app的都可以），定义自己的错误处理函数

   ```python
   def bad_request(request,exception):
     # ...
     return render(request,'400.html', values_for_template, status=400)
   ```

2. 在项目的 `urls.py` 中添加 `handler400 = 'banners.views.bad_request'`，即将错误代码与自定义函数绑定

### [HttpRequest](https://docs.djangoproject.com/zh-hans/3.1/ref/request-response/#django.http.HttpRequest)

前台传递的所有数据都在 [HttpRequest](https://docs.djangoproject.com/zh-hans/3.1/ref/request-response/#django.http.HttpRequest) 对象中（视图函数的第一个参数）

#### HttpRequest.method

字符串类型，表示请求中使用的`HTTP`方法。并且是大写的

```python
if request.method == 'GET':
    do_something()
elif request.method == 'POST':
    do_something_else()
```

#### HttpRequest.path

一个字符串，表示请求的页面的完整路径，不包含域名

#### HttpRequest.encoding

一个字符串，表示提交的数据的编码方式

#### HttpRequest.GET

包含 `get` 请求方式的所有参数

[django.http.QueryDict](https://docs.djangoproject.com/zh-hans/3.1/ref/request-response/#django.http.QueryDict) 类型的数据，类似字典，并且一个键对应多个值，`QueryDict` 是不可变的

`QueryDict.dict()` 函数返回 `Python` 的字典类型，即将 `QueryDict` 类型转换成 `dict` 类型

- 如果一个键同时拥有多个值将获取最后一个值
- 如果键不存在则返回 `None` 值，可以设置默认值进行后续处理

```python
# 例如请求：http://localhost/user/list?age=12&sh=abc
# QueryDict.get(key, default=None)
# 返回给定键的值。如果键有多个值，则返回最后一个值
# 获取单个请求参数值
age = HttpRequest.GET.get("age")

# 获取所有键的所有值
sh = HttpRequest.GET.getlist("sh")
```

#### HttpRequest.POST

使用 `form` 表单请求时，`method` 方式为`post`则会发起`post`方式的请求，需要使用`HttpRequest`对象的`POST`属性接收参数，`POST`属性是`QueryDict`类型的对象

`POST`不包含文件上传信息

```python
# 获取单个值
name = request.POST.get('uname') #获取name
# 获取多个值
genderList = request.POST.getlist('gender') #获取gender
```

#### HttpRequest.COOKIES

包含所有`cookie`的字典。键和值是字符串

#### HttpRequest.FILES

包含所有上传文件的类字典对象

`FILES`仅当请求方法为 `POST`且为`<form>`的请求方法时，才会包含数据 `enctype="multipart/form-data"`。否则，`FILES`将是一个空白的类似于字典的对象

#### HttpRequest.META

包含了所有本次`HTTP`请求的`Header`信息

#### HttpRequest.headers

获取请求headers里面的内容，不区分大小写，类似字典的对象

```python
request.headers['User-Agent']
request.headers['user-agent']
```

在`Django`模板中使用时，也可以使用`_`代替`-` 来查找

```jinja2
{{ request.headers.user_agent }}
```

#### HttpRequest.user

一个 `django.contrib.auth.models.User` 对象，表示当前登录用户。 若当前用户尚未登录，`user` 会设为 `django.contrib.auth.models.AnonymousUser` 的一个实例。 

可以使用`is_authenticated()` 函数判断用户是否登录：

```python
if request.user.is_authenticated:
    ... # Do something for logged-in users.
else:
    ... # Do something for anonymous users.
```

#### HttpRequest.session

一个可读写的类字典对象，表示当前`session`

### [HttpResponse](https://docs.djangoproject.com/zh-hans/3.1/ref/request-response/#django.http.HttpResponse)

```python
HttpResponse(content=b'', content_type=None, status=200, reason=None, charset=None)
```

每个视图函数必须返回一个响应对象，`HttpResponse` 由程序员创建并返回

#### 属性

- `HttpResponse.content`

  表示内容的字节字节串，必要时从字符串编码。

- `HttpResponse.charset`

  一个字符串，字符编码

- `HttpResponse.status_code`

  响应的[**HTTP状态代码**](https://tools.ietf.org/html/rfc7231.html#section-6)

- `HttpResponse.content_type`

  指定输出的`MINE`类型

#### 用法

##### 传递字符串

典型的用法是将页面的内容以字符串传递给[`HttpResponse`](https://docs.djangoproject.com/zh-hans/3.1/ref/request-response/#django.http.HttpResponse)构造函数

```python
response = HttpResponse("Text only, please.", content_type="text/plain")
```

如果要增量添加内容，可以将其`response`用作类似文件的对象：

```python
>>> response = HttpResponse()
>>> response.write("<p>Here's the text of the Web page.</p>")
>>> response.write("<p>Here's another paragraph.</p>")
```

##### 设置响应头字段

```python
response = HttpResponse()
>>> response['Age'] = 120
>>> del response['Age']
```

注意，与字典不同，如果标头字段不存在，`del`则不会引发`KeyError`

##### 告诉浏览器将响应视为文件附件

要告诉浏览器将响应视为文件附件，请设置 `content_type` 和 `Content-Disposition`响应头。例如，这是您可能返回Microsoft Excel电子表格的方式：

```python
>>> response = HttpResponse(my_data, content_type='application/vnd.ms-excel')
>>> response['Content-Disposition'] = 'attachment; filename="foo.xls"'
```

`Content-Disposition`头没有特定于Django的内容

#### 子类

##### HttpResponseRedirect

重定向

##### JsonResponse

返回 `json` 字符串

```python
JsonResponse(data, encoder=DjangoJSONEncoder, safe=True, json_dumps_params=None, **kwargs)
```

`safe=True` 时 `data` 必须是字典类型

`safe=False` 时 `data` 可以是任意类型

特点：

- 其默认`Content-Type` 是 `application/json` 
- 第一个参数 `data` 是一个`dict`实例

##### StreamingHttpResponse

不推荐使用

以流的形式响应

`Django`专为短暂的请求而设计。流式响应将在整个响应期间绑定一个工作进程。这可能会导致性能下降。

一般而言，您应该在请求-响应周期之外执行昂贵的任务，而不是借助流式响应

##### FileResponse

以小块流式传输文件

```python
FileResponse(open_file, as_attachment=False, filename='', **kwargs)
```

- 如果`as_attachment=True`，则`Content-Disposition`标题设置为 `attachment`，要求浏览器将文件提供给用户下载。否则，仅当文件名可用时，才会设置`Content-Disposition`值`inline`（浏览器默认值）的标头 。

- 如果`open_file`没有名称或者不合适，请使用`filename` 参数提供自定义文件名。请注意，如果您将类似文件的对象传递给`io.BytesIO`，则将`seek()`其传递给之前是您的任务`FileResponse`。

- 当`Content-Length`和`Content-Type`标题可以从内容中猜测出来时，将会自动设置`open_file`。

```python
response = FileResponse(open('myfile.png', 'rb'))
# 该文件将自动关闭
```

### 视图装饰器

`Django` 提供很多装饰器，它们可以为视图提供多种 `HTTP` 特性

#### 限制 HTTP 方法

在 [`django.views.decorators.http`](https://docs.djangoproject.com/zh-hans/3.1/topics/http/decorators/#module-django.views.decorators.http) 中的装饰器可以用来根据请求方法来限制对视图的访问。如果条件不满足，这些装饰器将返回 [`django.http.HttpResponseNotAllowed`](https://docs.djangoproject.com/zh-hans/3.1/ref/request-response/#django.http.HttpResponseNotAllowed) 

**require_http_methods(request_method_list)**

装饰器可以要求视图只接受特定的请求方法

```python
from django.views.decorators.http import require_http_methods

# 这里只接受 GET POST 请求
# 请求方法必须大写
@require_http_methods(["GET", "POST"])
def my_view(request):
    # I can assume now that only GET or POST requests make it this far
    # ...
    pass
```

**require_GET()**

装饰器要求视图只接受 `GET` 方法

**require_POST()**

装饰器要求视图只接受 `POST` 方法

**require_safe()**

装饰器要求视图只接收 `GET` 和 `HEAD` 方法。这些方法通常被认为是安全的，因为它们除了检索请求的资源外，没有特别的操作

## 模板

`Django` 使用默认自己的模板引擎，也可以配置一个或多个模板引擎

### 模板渲染

#### loader 渲染

使用此方法可以只加载一次模板，然后进行多次渲染

```python
from django.template import loader

def index(request):
  template = loader.get_template('index.html')
  res = template.render(context={'name': 'xxx'})
  return HttpResponse(res)
```

#### render 渲染

```python
from django.shortcuts import render

def index(request):
  return render(request, 'index.html', context={'name': 'xxx'})
```

### 模板的位置

配置文件 `settings.py` 中配置了模板的搜索路径

```python
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
      	# 定义目录列表，引擎应在目录中按搜索顺序查找模板源文件
	      # 此处配置了，模板在项目根目录的 templates 中
        'DIRS': [os.path.join(BASE_DIR, 'templates')],
      	# True 表示在应用的目录下的 templates 目录中查找，找不到在到 DIRS 配置的路径中查找
      	# False 表示不在应用目录下查找
        'APP_DIRS': True,
        ...
    },
]
```

### [Django模板语言](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/)

主要的是变量和标签

#### [注释(Comments)](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/language/#comments)

要注释掉行的一部分,在模板中使用注释语法: `{# #}` 

```django
{# greeting #}hello
```

注释可以包含任何模板代码, 使其生效或无效

```django
{# {% if foo %}bar{% else %} #}
```

这个语法只能用于单行注释 (不允许在 `{#` 和 `#}` 之间换行). 如果您需要注释掉多行部分的模板, 参见 [`comment`](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#std:templatetag-comment) 标签.

#### 变量

模板中包含的 **变量** 被替换为变量的值

变量从 `context` 文中输出一个值，这是一个类似于dict的对象，将键映射到值

如果变量解析为可调用对象，则模板系统将在不带任何参数的情况下调用它，并使用其结果代替可调用对象

变量在 `{{` 和 `}}` 之间

```django
My first name is {{ first_name }}. My last name is {{ last_name }}.
```

如果 `context = {'first_name': 'John', 'last_name': 'Doe'}` 则渲染的结果是：

```
My first name is John. My last name is Doe.
```

字典、属性、列表索引都以**点**来访问其属性：

```django
{{ my_dict.key }}
{{ my_object.attribute }}

{# 列表的第一个元素 #}
{{ my_list.0 }}
```

#### 标签(Tags)

标签提供渲染处理逻辑

标签放在 `{%` 和 `%}` 之间:

```django
{% csrf_token %}
```

大多数标签可以接受参数:

```django
{% cycle 'odd' 'even' %}
```

##### 常用标签

###### [autoescape](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#autoescape)

控制当前使用的自动转义行为

这个标签带有 `on` 或 `off` 参数, 决定了块内是否自动转义。该块由 `endautoescape` 标签结束

默认自动转义是生效的, 所有变量的内容将被自动转义成`HTML`字面值后输出(在这之前,其他的过滤器均被执行)

唯一的例外是已被标记为"安全"的转义, 如由代码生成的变量, 或使用了 [`safe`](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#std:templatefilter-safe) , [`escape`](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#std:templatefilter-escape) 过滤器

```django
{% autoescape on %}
    {{ body }}
{% endautoescape %}
```

###### [comment](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#comment)

忽略 `{％ comment ％}` 和 `{％ endcomment ％}` 之间的所有内容

```django
<p>Rendered text with {{ pub_date|date:"c" }}</p>
{% comment "Optional note" %}
    <p>Commented out text with {{ create_date|date:"c" }}</p>
{% endcomment %}
```

###### [extends](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#extends)

标记此模板继承的父模板的标签

这个标签有两种使用方式：

- `{% extends "base.html" %}`（使用引号）`Django` 将使用字面值`"base.html"`作为继承的父模板的名字
- `{% extends variable %}`使用变量`variable`。如果变量是一个字符串，则 `Django` 会使用此字符串作为继承的父模板的名字。如果变量是一个`Template`对象，则Django会使用这个对象作为父模板

通常，模板名称是相对于模板加载器的根目录而言的。字符串参数也可以是以`./`或开头的相对路径`../`。例如，假定以下目录结构：

```
dir1/
    template.html
    base2.html
    my/
        base3.html
base1.html
```

在`template.html`中，以下路径将有效：

```django
{% extends "./base2.html" %}
{% extends "../base1.html" %}
{% extends "./my/base3.html" %}
```

###### [filter](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#filter)

通过一个或多个过滤器过滤块的内容。就像使用可变语法一样，可以使用管道指定多个过滤器，并且过滤器可以具有参数

```django
{% filter force_escape|lower %}
    This text will be HTML-escaped, and will appear in all lowercase.
{% endfilter %}
```

###### [firstof](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#firstof)

输出不为 `false` 的第一个参数变量（即存在，不为空，不是错误的布尔值并且不是零数值）。如果所有传递的变量均为 `false`，则不输出任何内容

```django
{% firstof var1 var2 var3 %}
```

这等效于：

```django
{% if var1 %}
    {{ var1 }}
{% elif var2 %}
    {{ var2 }}
{% elif var3 %}
    {{ var3 }}
{% endif %}
```

如果所有传递的变量均为False，则还可以使用文字字符串作为后备值：

```django
{% firstof var1 var2 var3 "fallback value" %}
```

该标签会自动转义变量值。您可以使用以下方法禁用自动转义：

```django
{% autoescape off %}
    {% firstof var1 var2 var3 "<strong>fallback value</strong>" %}
{% endautoescape %}
```

或者，如果仅应转义某些变量，则可以使用：

```django
{% firstof var1 var2|safe var3 "<strong>fallback value</strong>"|safe %}
```

可以将输出存储在变量中: `{% firstof var1 var2 var3 as value %}`

###### [for](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#for)

循环访问数组中的每项

```django
<ul>
{% for athlete in athlete_list %}
    <li>{{ athlete.name }}</li>
{% endfor %}
</ul>
```

还可以反向遍历列表 : `{% for obj in list reversed %}`

如果您需要遍历列嵌套表列表，则可以将每个子列表中的值解包为单独的变量。例如，如果您的上下文包含名为的（x，y）坐标列表`points`，则可以使用以下命令输出点列表：

```django
{% for x, y in points %}
    There is a point at {{ x }},{{ y }}
{% endfor %}
```

如果您需要访问字典中的项目，这也很有用。例如，如果您的上下文包含一个dictionary `data`，则以下内容将显示该字典的键和值：

```django
{% for key, value in data.items %}
    {{ key }}: {{ value }}
{% endfor %}
```

`for` 循环设置了一些可以在循环体内直接使用的变量：

| 变量名                | 描述                                                |
| --------------------- | --------------------------------------------------- |
| `forloop.counter`     | 循环计数器，表示当前循环的索引（从“ 1”开始）。      |
| `forloop.counter0`    | 循环计数器，表示当前循环的索引（从“ 0”开始）。      |
| `forloop.revcounter`  | 反向循环计数器（以最后一次循环为``1''，反向计数）。 |
| `forloop.revcounter0` | 反向循环计数器（以最后一次循环为“ 0”，反向计数）。  |
| `forloop.first`       | 当前循环为首个循环时，该变量为True                  |
| `forloop.last`        | 当前循环为最后一个循环时，该变量为True              |
| `forloop.parentloop`  | 在嵌套循环中，指向当前循环的上级循环                |

###### [for... empty](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#for-empty)

当传递到`for`标签中变量不存在或者为空，可以使用标签`{% empty %}` 来指定输出的内容

```django
<ul>
{% for athlete in athlete_list %}
    <li>{{ athlete.name }}</li>
{% empty %}
  	{# athlete_list 不存在或者为空时执行 #}
    <li>Sorry, no athletes in this list.</li>
{% endfor %}
</ul>
```

上面的代码与下面的代码是等同的

```django
<ul>
  {% if athlete_list %}
    {% for athlete in athlete_list %}
      <li>{{ athlete.name }}</li>
    {% endfor %}
  {% else %}
    <li>Sorry, no athletes in this list.</li>
  {% endif %}
</ul>
```

###### [if](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#if)

`{% if %}` 标签会判断给定的变量，当变量为True时（存在，非空，非布尔值False），就会输出块内的内容：

```django
{% if athlete_list %}
    Number of athletes: {{ athlete_list|length }}
{% elif athlete_in_locker_room_list %}
    Athletes should be out of the locker room soon!
{% else %}
    No athletes.
{% endif %}
```

在上面的例子中，如果`athlete_list`不是空的，那么变量就会被显示出来`{{ athlete_list|length }}`

[布尔操作](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#boolean-operators)

[`if`](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#std:templatetag-if)标签可以使用`and`，`or`或者`not`测试多个变量或取反给定变量：

```django
{% if athlete_list and coach_list %}
    Both athletes and coaches are available.
{% endif %}

{% if not athlete_list %}
    There are no athletes.
{% endif %}

{% if athlete_list or coach_list %}
    There are some athletes or some coaches.
{% endif %}
```

在[`if`](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#std:templatetag-if)标记中使用小括号是无效的语法。如果需要它们来指示优先级，则应使用嵌套[`if`](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#std:templatetag-if)标签

[== 运算符](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#operator)

```django
{% if somevar == "x" %}
  This appears if variable somevar equals the string "x"
{% endif %}
```

[!= 运算符](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#id1)

[< 运算符](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#id2)

[> 运算符](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#id3)

[<= 运算符](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#id4)

[>= 运算符](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#id5)

[in 运算符](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#in-operator)

```django
{% if "bc" in "abcdef" %}
  This appears since "bc" is a substring of "abcdef"
{% endif %}
```

[not in 运算符](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#not-in-operator)

[is 运算符](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#is-operator)

[is not 运算符](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#is-not-operator)

[过滤器](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#filters)

您也可以在[`if`](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#std:templatetag-if)表达式中使用过滤器。例如：

```django
{% if messages|length >= 100 %}
   You have lots of messages today!
{% endif %}
```

混合使用时，运算符的优先级从低到高依次为：

- `or`
- `and`
- `not`
- `in`
- `==`，`!=`，`<`，`>`，`<=`，`>=`

###### [include](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#include)

加载模板并使用当前 `context` 呈现它。这是在模板中“包含”其他模板的一种方式

模板名称可以是变量，也可以是硬编码（带引号）的字符串，用单引号或双引号引起来

```django
{% include "foo/bar.html" %}
```

通常，模板名称是相对于模板加载器的根目录而言的。字符串参数也可以是以标记开头`./`或`../` 在[`extends`](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#std:templatetag-extends)标记中所述的相对路径

```django
{% include template_name %}
```

该变量也可以是带有`render()`接受上下文的方法的任何对象。这使您可以引用`Template`上下文中的已编译

###### [load](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#load)

加载自定义模板标签

```django
{% load somelibrary package.otherlibrary %}
```

您还可以使用`from`参数有选择地从库中加载单个过滤器或标签

```django
{% load foo bar from somelibrary %}
```

###### [url](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#url)

返回与给定视图和可选参数匹配的绝对路径引用（不带域名的URL)

```django
{% url 'some-url-name' v1 v2 %}
```

第一个参数是[URL模式名称](https://docs.djangoproject.com/zh-hans/3.1/topics/http/urls/#naming-url-patterns)。它可以是带引号的文字或任何其他上下文变量。其他参数是可选的，并且应为以空格分隔的值，这些值将用作URL中的参数

您可以使用关键字语法：

```django
{% url 'some-url-name' arg1=v1 arg2=v2 %}
```

不要在一次调用中同时使用位置和关键字语法

```python
path('client/<int:id>/', app_views.client, name='app-views-client')
```

然后，您可以在模板中创建指向该视图的链接，如下所示：

```django
{% url 'app-views-client' client.id %}
```

`template` 标签将输出字符串`/client/123/`

如果您想检索一个URL但不显示它，则可以使用一个稍微不同的调用：

```django
{% url 'some-url-name' arg arg2 as the_url %}

<a href="{{ the_url }}">I'm linking to {{ the_url }}</a>
```

```django
{% url 'some-url-name' as the_url %}
{% if the_url %}
  <a href="{{ the_url }}">Link to optional stuff</a>
{% endif %}
```

如果要检索命名空间的URL，请指定全限定名：

```django
{% url 'myapp:view-name' %}
```

###### [with](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#with)

定义一个变量保存值

```django
{% with total=business.employees.count %}
    {{ total }} employee{{ total|pluralize }}
{% endwith %}
```

变量（示例中`total`）仅在 `{% with %}` 和 `{% endwith %}` 标记之间可用

您可以分配多个上下文变量：

```django
{% with alpha=1 beta=2 %}
    ...
{% endwith %}
```

#### 过滤器

通过使用 **过滤器** 修改变量的显示

通过管道符 (`|`) 间隔变量和过滤器来使用过滤器: `{{ name|lower }}`

过滤器可以链式调用，一个过滤器的输出作为下一个过滤器的输入，`{{ text|escape|linebreaks }}` 是一种常用的转换方式

有些过滤器是带有参数的，一个带参数的过滤器看起来像这样: `{{ bio|truncatewords:30 }}` 他会显示 `bio` 变量的前30个字符

[所有内置过滤器](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#ref-templates-builtins-filters)

##### 常用过滤器

[`default`](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#std:templatefilter-default)

如果变量为 `False` 或为空，则使用给定的默认值。否则，使用变量的值

```django
{# 如果 value 没有提供或者为空，那么他将显示为 "nothing" #}
{{ value|default:"nothing" }}
```

[`length`](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#std:templatefilter-length)

返回值的长度。这适用于字符串和列表

```django
{{ value|length }}
```

[`filesizeformat`](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#std:templatefilter-filesizeformat)

将值格式化为`人类可读`的文件大小(即“ 13 KB”，“ 4.1 MB”，“ 102字节”等)

```django
{# If value is 123456789, the output would be 117.7 MB. #}
{{ value|filesizeformat }}
```

[capfirst](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#capfirst)

将值的第一个字符大写。如果第一个字符不是字母，则此过滤器无效。

```django
{{ value|capfirst }}
```

[lower](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#lower)

将字符串转换为小写字母

```django
{{ value|lower }}
```

如果`value="Totally LOVING this Album!"`，则输出`totally loving this album!` 

[upper](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#upper)

将字符串转换为全部大写

```django
{{ value|upper }}
```

[center](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#center)

将值居中在给定宽度的字段中

```django
"{{ value|center:"15" }}"
{# If value is "Django", the output will be "     Django    " #}
```

[ljust](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#ljust)

将值在给定宽度的字段中左对齐

```django
"{{ value|ljust:"10" }}"
```

如果`value`是`Django`，则输出为`"Django  "` 

[rjust](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#rjust)

在给定宽度的字段中将值右对齐

```django
"{{ value|rjust:"10" }}"
```

如果`value`是`Django`，则输出为`"  Django"` 

[date](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#date)

根据给定的格式格式化日期

可用格式化字符串：

| Format character | 描述                                                         | Example output                                               |
| ---------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **Day**          |                                                              |                                                              |
| `d`              | Day of the month, 2 digits with leading zeros.               | `'01'` to `'31'`                                             |
| `j`              | Day of the month without leading zeros.                      | `'1'` to `'31'`                                              |
| `D`              | Day of the week, textual, 3 letters.                         | `'Fri'`                                                      |
| `l`              | Day of the week, textual, long.                              | `'Friday'`                                                   |
| `S`              | English ordinal suffix for day of the month, 2 characters.   | `'st'`, `'nd'`, `'rd'` or `'th'`                             |
| `w`              | Day of the week, digits without leading zeros.               | `'0'` (Sunday) to `'6'` (Saturday)                           |
| `z`              | Day of the year.                                             | `1` to `366`                                                 |
| **Week**         |                                                              |                                                              |
| `W`              | ISO-8601 week number of year, with weeks starting on Monday. | `1`, `53`                                                    |
| **Month**        |                                                              |                                                              |
| `m`              | Month, 2 digits with leading zeros.                          | `'01'` to `'12'`                                             |
| `n`              | Month without leading zeros.                                 | `'1'` to `'12'`                                              |
| `M`              | Month, textual, 3 letters.                                   | `'Jan'`                                                      |
| `b`              | Month, textual, 3 letters, lowercase.                        | `'jan'`                                                      |
| `E`              | Month, locale specific alternative representation usually used for long date representation. | `'listopada'` (for Polish locale, as opposed to `'Listopad'`) |
| `F`              | Month, textual, long.                                        | `'January'`                                                  |
| `N`              | Month abbreviation in Associated Press style. Proprietary extension. | `'Jan.'`, `'Feb.'`, `'March'`, `'May'`                       |
| `t`              | Number of days in the given month.                           | `28` to `31`                                                 |
| **Year**         |                                                              |                                                              |
| `y`              | Year, 2 digits.                                              | `'99'`                                                       |
| `Y`              | Year, 4 digits.                                              | `'1999'`                                                     |
| `L`              | Boolean for whether it's a leap year.                        | `True` or `False`                                            |
| `o`              | ISO-8601 week-numbering year, corresponding to the ISO-8601 week number (W) which uses leap weeks. See Y for the more common year format. | `'1999'`                                                     |
| **Time**         |                                                              |                                                              |
| `g`              | Hour, 12-hour format without leading zeros.                  | `'1'` to `'12'`                                              |
| `G`              | Hour, 24-hour format without leading zeros.                  | `'0'` to `'23'`                                              |
| `h`              | Hour, 12-hour format.                                        | `'01'` to `'12'`                                             |
| `H`              | Hour, 24-hour format.                                        | `'00'` to `'23'`                                             |
| `i`              | Minutes.                                                     | `'00'` to `'59'`                                             |
| `s`              | Seconds, 2 digits with leading zeros.                        | `'00'` to `'59'`                                             |
| `u`              | Microseconds.                                                | `000000` to `999999`                                         |
| `a`              | `'a.m.'` or `'p.m.'` (Note that this is slightly different than PHP's output, because this includes periods to match Associated Press style.) | `'a.m.'`                                                     |
| `A`              | `'AM'` or `'PM'`.                                            | `'AM'`                                                       |
| `f`              | Time, in 12-hour hours and minutes, with minutes left off if they're zero. Proprietary extension. | `'1'`, `'1:30'`                                              |
| `P`              | Time, in 12-hour hours, minutes and 'a.m.'/'p.m.', with minutes left off if they're zero and the special-case strings 'midnight' and 'noon' if appropriate. Proprietary extension. | `'1 a.m.'`, `'1:30 p.m.'`, `'midnight'`, `'noon'`, `'12:30 p.m.'` |
| **Timezone**     |                                                              |                                                              |
| `e`              | Timezone name. Could be in any format, or might return an empty string, depending on the datetime. | `''`, `'GMT'`, `'-500'`, `'US/Eastern'`, etc.                |
| `I`              | Daylight Savings Time, whether it's in effect or not.        | `'1'` or `'0'`                                               |
| `O`              | Difference to Greenwich time in hours.                       | `'+0200'`                                                    |
| `T`              | Time zone of this machine.                                   | `'EST'`, `'MDT'`                                             |
| `Z`              | Time zone offset in seconds. The offset for timezones west of UTC is always negative, and for those east of UTC is always positive. | `-43200` to `43200`                                          |
| **Date/Time**    |                                                              |                                                              |
| `c`              | ISO 8601 format. (Note: unlike others formatters, such as "Z", "O" or "r", the "c" formatter will not add timezone offset if value is a naive datetime (see [`datetime.tzinfo`](https://docs.python.org/3/library/datetime.html#datetime.tzinfo)). | `2008-01-02T10:30:00.000123+02:00`, or `2008-01-02T10:30:00.000123` if the datetime is naive |
| `r`              | [**RFC 5322**](https://tools.ietf.org/html/rfc5322.html#section-3.3) formatted date. | `'Thu, 21 Dec 2000 16:01:07 +0200'`                          |
| `U`              | Seconds since the Unix Epoch (January 1 1970 00:00:00 UTC).  |                                                              |

可以使用预定义的格式：

[`DATETIME_FORMAT`](https://docs.djangoproject.com/zh-hans/3.1/ref/settings/#std:setting-DATETIME_FORMAT)
[`SHORT_DATE_FORMAT`](https://docs.djangoproject.com/zh-hans/3.1/ref/settings/#std:setting-SHORT_DATE_FORMAT)
[`SHORT_DATETIME_FORMAT`](https://docs.djangoproject.com/zh-hans/3.1/ref/settings/#std:setting-SHORT_DATETIME_FORMAT)

或使用在上面表中的自定义格式

[default_if_none](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#default-if-none)

只有当值为`None`，则使用给定的默认值。否则，使用该值。

请注意，如果是一个空字符串，将不使用默认值

```django
{# 如果value是None，则输出为nothing #}
{{ value|default_if_none:"nothing" }}
```

[escape](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#escape)

转义字符串的HTML。具体来说，它将进行以下替换：

- `<` 被替换为 `<`
- `>` 被替换为 `>`
- `'` （单引号）转换为 `'`
- `"` （双引号）被替换为 `"`
- `&` 被替换为 `&`

[first](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#first)

返回列表中的第一项

```django
{{ value|first }}
```

如果`value=['a', 'b', 'c']`，则输出`a`

[last](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#last)

返回列表中的最后一项

```django
{{ value|last }}
```

如果`value=['a', 'b', 'c']`，则输出`c`

[floatformat](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#floatformat)

不带参数使用时，将浮点数四舍五入到小数点后一位-但仅当要显示小数部分时才如此

| `value`    | 模板                      | 输出量 |
| ---------- | ------------------------- | ------ |
| `34.23234` | `{{ value|floatformat }}` | `34.2` |
| `34.00000` | `{{ value|floatformat }}` | `34`   |
| `34.26000` | `{{ value|floatformat }}` | `34.3` |

如果与数字整数参数一起使用，则将数字四舍五入到小数点后指定的位数

| `value`    | 模板                        | 输出量   |
| ---------- | --------------------------- | -------- |
| `34.23234` | `{{ value|floatformat:3 }}` | `34.232` |
| `34.00000` | `{{ value|floatformat:3 }}` | `34.000` |
| `34.26000` | `{{ value|floatformat:3 }}` | `34.260` |

传递0（零）作为参数将浮点数舍入为最接近的整数

| `value`    | 模板                          | 输出量 |
| ---------- | ----------------------------- | ------ |
| `34.23234` | `{{ value|floatformat:"0" }}` | `34`   |
| `34.00000` | `{{ value|floatformat:"0" }}` | `34`   |
| `39.56000` | `{{ value|floatformat:"0" }}` | `40`   |

如果传递给的参数为负数，它将把数字四舍五入到小数点后指定的位数，如果小数点后都是0，则不显示小数点后的0

| `value`    | 模板                           | 输出量   |
| ---------- | ------------------------------ | -------- |
| `34.23234` | `{{ value|floatformat:"-3" }}` | `34.232` |
| `34.00000` | `{{ value|floatformat:"-3" }}` | `34`     |
| `34.26000` | `{{ value|floatformat:"-3" }}` | `34.260` |

[join](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#join)

将列表用字符串连接起来

```django
{{ value|join:" // " }}
```

如果`value=['a', 'b', 'c']`，则输出字符串 `a // b // c`

[linenumbers](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#linenumbers)

显示带有行号的文本

```django
{{ value|linenumbers }}
```

如果`value`是：

```
one
two
three
```

输出将是：

```
1. one
2. two
3. three
```

[timesince](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#timesince)

将日期格式化为自该日期以来的时间（例如:  4天6小时）

接受一个可选参数，该参数是一个包含要用作比较点的日期的变量（不带参数，则比较的是现在）。例如，如果`blog_date`是2006年6月1日 00:00:00，并且`comment_date`是2006年6月1日08:00，则以下内容将返回“ 8 hours”：

```django
{{ blog_date|timesince:comment_date }}
```

分钟是最小单位

##### [创建自定义模板过滤器](https://docs.djangoproject.com/zh-hans/3.1/howto/custom-template-tags/)

#### [模板继承](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/language/#template-inheritance)

`Django` 模板中最强大和最复杂的部分就是模板继承. 模板继承允许你建立一个基本的"骨架"模板, 它包含了你网站所有常见的元素,并定义了可以被子模板覆盖的 **块(blocks)** 

```django
<!-- 骨架模板 -->
<html lang="en">
<head>
    <link rel="stylesheet" href="style.css">
    <title>{% block title %}My amazing site{% endblock %}</title>
</head>

<body>
    <div id="sidebar">
        {% block sidebar %}
        <ul>
            <li><a href="/">Home</a></li>
            <li><a href="/blog/">Blog</a></li>
        </ul>
        {% endblock %}
    </div>

    <div id="content">
        {% block content %}{% endblock %}
    </div>
</body>
</html>
```

[`block`](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#std:templatetag-block) 标签告诉了模板系统哪些地方将被子模板覆盖

```django
{% extends "base.html" %}

{% block title %}My amazing blog{% endblock %}

{% block content %}
{% for entry in blog_entries %}
    <h2>{{ entry.title }}</h2>
    <p>{{ entry.body }}</p>
{% endfor %}
{% endblock %}
```

[`extends`](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#std:templatetag-extends) 标签告诉模板系统这个模板继承了另外的模板

需要注意的是, 因为子模板没有定义 `sidebar` 块, 那么将使用父模板的内容

模板继承注意事项：

- 如果你在模板中使用 [`{% extends %}`](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#std:templatetag-extends) 继承父模板, 他必须位于模板的最开始

- 在你的基础模板中使用更多的 [`{% block %}`](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#std:templatetag-block) 是一个好的方法

- 如果你发现自己在许多模板中有重复内容的了, 这可能意味着你要移动这些内容到父模板的 `{% block %}` 中

- 如果你需要得到父模板块中的内容, 可以用 `{{ block.super }}` 变量

- 通过使用与您继承的模板名称相同的模板，`{％ entends ％}` 可用于在覆盖模板的同时继承模板。

- 在 `{％ block ％}` 外部使用模板标记创建的变量不能在该块内使用

  ```django
  {% translate "Title" as title %}
  {# 下面是错误使用， title不能在 block 标记块中使用 #}
  {% block content %}{{ title }}{% endblock %}
  ```

- 给 `{% endblock %}` 标签定义一个名字以增加可读性

  ```django
  {% block content %}
  ...
  {% endblock content %}
  ```

### 使用 Jinja2 模板

安装 Jinja2 包

```shell
pip install jinja2
```

在项目同名的目录下创建文件 `jinja2.py`

```python
from django.templatetags.static import static
from django.urls import reverse

from jinja2 import Environment


def environment(**options):
    env = Environment(**options)
    env.globals.update({
        'static': static,
        'url': reverse,
    })
    return env
```

修改配置 `settings.py` 文件如下

```python
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.jinja2.Jinja2',
        'DIRS': [os.path.join(BASE_DIR, 'templates')],
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
          	'environment': 'myproject.jinja2.environment'
        },
    },
]
```

## [CSRF 跨站请求伪造保护](https://docs.djangoproject.com/zh-hans/3.1/ref/csrf/#module-django.middleware.csrf)

### form 中使用

1. 默认情况下，`settings.py` 文件中 [`MIDDLEWARE`](https://docs.djangoproject.com/zh-hans/3.1/ref/settings/#std:setting-MIDDLEWARE) 设置已激活了CSRF中间件

   如果禁用了，则可以在要保护的特定视图上使用 [`csrf_protect()`](https://docs.djangoproject.com/zh-hans/3.1/ref/csrf/#django.views.decorators.csrf.csrf_protect) 

2. 在 `POST` 表单的模板中，如果表单用于内部 `URL`，请在`<form>`元素中使用 [`csrf_token`](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/builtins/#std:templatetag-csrf_token) 标记

   ```django
   <form method="post">
     {% csrf_token %}
     ...
   </form>
   ```

   对于以外部 `URL` 为目标的 `POST` 表单，不应这样做，因为这会导致CSRF令牌泄漏，从而导致漏洞

3. 在相应的视图功能中，确保 [`RequestContext`](https://docs.djangoproject.com/zh-hans/3.1/ref/templates/api/#django.template.RequestContext)使用来呈现响应，以使其正常工作。如果您使用的是函数、通用视图或Contrib应用程序，那么这些内容都已被使用

### ajax 中使用

#### 获取 csrftoken

##### [`CSRF_USE_SESSIONS`](https://docs.djangoproject.com/zh-hans/3.1/ref/settings/#std:setting-CSRF_USE_SESSIONS)和[`CSRF_COOKIE_HTTPONLY`](https://docs.djangoproject.com/zh-hans/3.1/ref/settings/#std:setting-CSRF_COOKIE_HTTPONLY)是`False` 

这个是默认设置

使用 [JavaScript Cookie库](https://github.com/js-cookie/js-cookie/) 获取  `cookies` 中的 `csrftoken`

```js
const csrftoken = Cookies.get('csrftoken');
```

##### [`CSRF_USE_SESSIONS`](https://docs.djangoproject.com/zh-hans/3.1/ref/settings/#std:setting-CSRF_USE_SESSIONS)和[`CSRF_COOKIE_HTTPONLY`](https://docs.djangoproject.com/zh-hans/3.1/ref/settings/#std:setting-CSRF_COOKIE_HTTPONLY)是`True` 

这个时候需要从会话中获取

则必须在 `HTML` 中包含 `CSRF` 令牌，并使用 `JavaScript` 从 `DOM` 中读取该令牌

```django
{% csrf_token %}
<script>
const csrftoken = document.querySelector('[name=csrfmiddlewaretoken]').value;
</script>
```

#### 设置请求头

```
# 在请求头中添加
'X-CSRFToken': csrftoken
```

一个完整的例子

```js
<script>
        var csrftoken = Cookies.get('csrftoken');
        
        function csrfSafeMethod(method) {
            // these HTTP methods do not require CSRF protection
            return (/^(GET|HEAD|OPTIONS|TRACE)$/.test(method));
        }

        $.ajaxSetup({
            beforeSend: function(xhr, settings) {
                if (!csrfSafeMethod(settings.type) && !this.crossDomain) {
                    xhr.setRequestHeader("X-CSRFToken", csrftoken);
                }
            }
        });
 
        $(function () {
            $('#btn').click(function () {
                $.ajax({
                    url:'/login/',
                    type:"POST",
                    data:{'username':'root','pwd':'123123'},
                    success:function (arg) {
                        
                    }
                })
            })
        });
</script>
```

#### ajax 请求参数添加 `CSRF`

```js
// 将 CSRF 增加到请求参数中（请求参数和请求头中的CSRF的key是不同的）
$.post("/login/", {name: 'xxx', pwd: 'aa', csrfmiddlewaretoken: 'lsdjfljdfl'})
```

### 排除不需要保护的视图

在不需要保护的视图函数上使用 `@csrf_exempt` 装饰器关闭 `CSRF` 验证

在需要保护的视图函数上使用 `@csrf_protect` 装饰器开启 `CSRF` 验证

```python
from django.views.decorators.csrf import csrf_exempt, csrf_protect

@csrf_exempt
def my_view(request):

    @csrf_protect
    def protected_path(request):
        do_something()

    if some_condition():
       return protected_path(request)
    else:
       do_something_else()
```

### 前后端分离项目使用

需要手动将 `csrftoken` 写入 `cookie` 的方法

1. 手动设置，在 `view` 中添加

```python
request.META["CSRF_COOKIE_USED"] = True
```

2. 手动调用 `csrf` 中的 `get_token(request)` 或 `rotate_token(request)` 方法

```python
from django.middleware.csrf import get_token ,rotate_token
 
def server(request):
    # get_token(request)       // 两者选一
    # rotate_token(request)   // 此方法每次设置新的cookies
    return render(request, ‘server.html‘)
```

3. 在 `HTML` 模板中添加 `{% csrf_token %}`

4. 在需要设置 `cookie` 的视图上加装饰器 `ensure_csrf_cookie()`

```python
from django.views.decorators.csrf import ensure_csrf_cookie
 
@ensure_csrf_cookie
def server(request):
    return render(request,'server.html')
```

##  [部署静态资源文件](https://docs.djangoproject.com/zh-hans/3.1/howto/static-files/deployment/) 

## [模型](https://docs.djangoproject.com/zh-hans/3.1/topics/db/models/)

模型是数据的字段和行为的集合

一般来说，每一个模型都对应一张数据库表

创建模型比较好的方式是：先创建表，然后反向生成模型

模型基础

- 每个模型都是一个 `Python` 类，并且继承 [`django.db.models.Model`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/instances/#django.db.models.Model)
- 模型类的每个属性都相当于一个数据库的字段
- `Django` 提供了一个自动生成访问数据库的 API；请参阅 [执行查询](https://docs.djangoproject.com/zh-hans/3.1/topics/db/queries/)。
- 主键 `id` 字段默认自动添加

### 简单模型

```python
from django.db import models

class Person(models.Model):
    first_name = models.CharField(max_length=30)
    last_name = models.CharField(max_length=30)
```

### [字段](https://docs.djangoproject.com/zh-hans/3.1/topics/db/models/#fields)

字段在类属性中定义

#### [字段类型](https://docs.djangoproject.com/zh-hans/3.1/topics/db/models/#field-types)

模型中每一个字段都应该是某个 [`Field`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/fields/#django.db.models.Field) 类的实例，

`Django` 内置了数十种字段类型:  [模型字段参考](https://docs.djangoproject.com/zh-hans/3.1/ref/models/fields/#model-field-types) 

默认情况下， `Django` 会给每一个模型添加自增主键：

```python
id = models.AutoField(primary_key=True)
```

##### 字段类型参数:

###### verbose_name

字段的可读名称，主要用于后台管理的显示

##### 常用字段类型:

###### AutoField()

自增id(32位整型)int(32)

###### BigAutoField()

自增id(64位整型)int(64)

###### BooleanField()

boolean 类型（True、False），不能为null, 默认值为`None`tinyint(1)

###### NullBooleanField()

支持Null、True、False三种值

###### BigIntegerFieldCharField(max_length=字符长度)

字符串字段，必须接收一个`max_length`参数来指定其长度

###### varcharDateField()

日期字段，使用的是 `python` 的 `datetime.date` 实例

参数说明：

`DateField.auto_now`: 每次保存对象时自动将字段设置为现在, <del>该字段只有在调用 `Model.save()` 方法是有效</del>

`DateField.auto_now_add`: 插入时将字段自动设置为当前时间，设置该属性为 `True` 后，则即便手动设置了该字段的值也不起作用

###### DateTimeField()

日期和时间，在 `Python` 中以`datetime.datetime`实例表示。接受与[`DateField`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/fields/#django.db.models.DateField) 相同的参数

###### DecimalField()

`DecimalField(max_digits =None，decimal_places =None，**options)` 

固定精度的十进制浮点数，在 `Python` 中由 [`Decimal`](https://docs.python.org/3/library/decimal.html#decimal.Decimal)实例表示

有两个必传参数：

- `DecimalField.max_digits` 

  数字中允许的最大位数。请注意，此数字必须大于或等于`decimal_places`。

- `DecimalField.decimal_places` 

  小数位数

```python
# 浮点数总位数是5，其中小数位数是2，即xxx.xx
models.DecimalField(..., max_digits=5, decimal_places=2)
```

###### FloatField()

`FloatField(**options)` 

在 `Python` 中由`float`实例表示的浮点数

###### IntegerField

`IntegerField(`**options)` 

整数

###### JSONField()

`JSONField(encoder=None, decoder=None, **options)`

Django 3.1新增

用于存储 `JSON` 数据。在 `Python` 中，数据以其Python本机格式表示：字典，列表，字符串，数字，布尔值和 `None`。

- `JSONField.encoder`

  一个可选的[`json.JSONEncoder`](https://docs.python.org/3/library/json.html#json.JSONEncoder)子类，用于序列化标准JSON序列化器（例如`datetime.datetime` 或[`UUID`](https://docs.python.org/3/library/uuid.html#uuid.UUID)）不支持的数据类型。默认为`json.JSONEncoder` 

- `JSONField.decoder` 

  一个可选的[`json.JSONDecoder`](https://docs.python.org/3/library/json.html#json.JSONDecoder)子类，用于反序列化从数据库检索的值。默认为`json.JSONDecoder` 

###### PositiveIntegerField()

类似于[`IntegerField`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/fields/#django.db.models.IntegerField)，但必须为正数或零（`0`）。从`0`到的值`2147483647`在Django支持的所有数据库中都是安全的

###### PositiveBigIntegerField()

Django 3.1的新功能

类似于[`PositiveIntegerField`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/fields/#django.db.models.PositiveIntegerField)，从`0`到的值`9223372036854775807`在Django支持的所有数据库中都是安全的

###### TextField()

大文本字段，超过4000个字符时使用

###### TimeField()

时间，在Python中以`datetime.time`实例表示。接受与[`DateField`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/fields/#django.db.models.DateField)相同的参数

##### 关联关系字段类型

###### ForeignKey()

`ForeignKey(to, on_delete, **options)`

外键，多对一的关系。需要两个位置参数：被关联的类和[`on_delete`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/fields/#django.db.models.ForeignKey.on_delete)选项

如果要创建一个递归关系-一个自身本身有多对一关系的对象-则使用。`models.ForeignKey('self', on_delete=models.CASCADE)`

参数说明：

to:  被关联的模型，可以使用字符串表示

```python
models.ForeignKey('Manufacturer', on_delete=models.CASCADE)
```

on_delete: 指定当被关联的对象被删除时的行为

- models.CASCADE: 级联删除，删除包含ForeignKey的对象
- models.PROTECT: 通过引发[`ProtectedError`](https://docs.djangoproject.com/zh-hans/3.1/ref/exceptions/#django.db.models.ProtectedError)的子类来 防止删除引用的对象
- models.RESTRICT: 通过引发[`RestrictedError`](https://docs.djangoproject.com/zh-hans/3.1/ref/exceptions/#django.db.models.RestrictedError)（的子类 [`django.db.IntegrityError`](https://docs.djangoproject.com/zh-hans/3.1/ref/exceptions/#django.db.IntegrityError)）防止删除引用的对象 。与不同[`PROTECT`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/fields/#django.db.models.PROTECT)，如果引用的对象还引用了在同一操作中通过[`CASCADE`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/fields/#django.db.models.CASCADE) 关系删除的另一个对象，则允许删除该对象。
- models.SET_NULL: 设置为ForeignKeynull；这只有在null存在的情况下才有可能 True。
- models.SET_DEFAULT: 将ForeignKey其设置为默认值；ForeignKey必须设置默认值 。
- models.SET(): 将设置为ForeignKey传递给的值 SET()，或者如果传递了callable，则调用它的结果。在大多数情况下，有必要传递一个callable以避免在导入models.py时执行查询
- models.DO_NOTHING: 不做任何操作

###### ManyToManyField()

`ManyToManyField(to，**options)`

多对多关系

###### OneToOneField()

`OneToOneField(to，on_delete，parent_link = False，**options)` 

一对一关系

#### 字段备注

任何字段类型都接收一个可选的位置参数 [`verbose_name`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/fields/#django.db.models.Field.verbose_name)，如果未指定该参数值， `Django` 会自动使用字段的属性名作为该参数值，并且把下划线转换为空格

```python
first_name = models.CharField("person's first name", max_length=30)
```

#### 关联关系

`Django` 提供了三种常见的数据库关联关系：多对一，多对多，一对一

##### 多对一

定义一个多对一的关联关系，使用 [`django.db.models.ForeignKey`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/fields/#django.db.models.ForeignKey) 类

[`ForeignKey`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/fields/#django.db.models.ForeignKey) 类需要添加一个位置参数，即你想要关联的模型类名

例如：如果一个 `Car` 模型有一个制造者 `Manufacturer` --就是说一个 `Manufacturer` 制造许多辆车，但是每辆车都仅有一个制造者-- 那么使用下面的方法定义这个关系：

```python
from django.db import models

class Manufacturer(models.Model):
    # ...
    pass

class Car(models.Model):
    manufacturer = models.ForeignKey(Manufacturer, on_delete=models.CASCADE)
    # ...
```

你也可以创建 [自关联关系](https://docs.djangoproject.com/zh-hans/3.1/ref/models/fields/#recursive-relationships) （一个模型与它本身有多对一的关系）

```python
# 自关联
models.ForeignKey('self', on_delete=models.CASCADE)
```

##### 多对多关联

定义一个多对多的关联关系，使用 [`django.db.models.ManyToManyField`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/fields/#django.db.models.ManyToManyField) 类

[`ManyToManyField`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/fields/#django.db.models.ManyToManyField) 类需要一个位置参数，即你想要关联的模型类名

可以在任何一个模型中添加 [`ManyToManyField`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/fields/#django.db.models.ManyToManyField) 字段，但只能选择一个模型设置该字段，即不能同时在两模型中添加该字段

例如：如果 `Pizza` 含有多种 `Topping` （配料） -- 也就是一种 `Topping` 可能存在于多个 `Pizza` 中，并且每个 `Pizza` 含有多种 `Topping` --那么可以这样表示这种关系：

```python
from django.db import models

class Topping(models.Model):
    # ...
    pass

class Pizza(models.Model):
    # ...
    toppings = models.ManyToManyField(Topping)
```

##### 一对一关联

使用 [`OneToOneField`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/fields/#django.db.models.OneToOneField) 来定义一对一关系

[`OneToOneField`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/fields/#django.db.models.OneToOneField) 需要一个位置参数：与模型相关的类

### [`Meta` 类](https://docs.djangoproject.com/zh-hans/3.1/topics/db/models/#meta-options)

使用内部 `Meta类` 给模型赋予元数据

模型的元数据即`所有不是字段的东西`

```python
from django.db import models

class Ox(models.Model):
    horn_length = models.IntegerField()

    class Meta:
        ordering = ["horn_length"]
        verbose_name_plural = "oxen"
```

#### [Meta可用选项](https://docs.djangoproject.com/zh-hans/3.1/ref/models/options/)

##### abstract

- `abstract = True`: 此模型将是 [抽象基类](https://docs.djangoproject.com/zh-hans/3.1/topics/db/models/#abstract-base-classes) , 即此模型不和数据库中的表对应
- `abstract = False`: 默认值，与数据库中的表一一对应

##### app_label

如果模型是在 [`INSTALLED_APPS`](https://docs.djangoproject.com/zh-hans/3.1/ref/settings/#std:setting-INSTALLED_APPS) 配置的应用外部定义的，则必须声明其所属的应用程序

```python
app_label = 'myapp'
```

##### db_table

用于指定模型对应数据库表的名称：`db_table = 'music_album' `

##### managed

默认值：`True`，表示如果模型变更了，那么 `Django` 将在[`migrate`](https://docs.djangoproject.com/zh-hans/3.1/ref/django-admin/#django-admin-migrate)迁移过程中修改数据库

`False`: 不会对此模型执行数据库表创建、修改或删除操作

##### ordering

查询结果集时使用的排序字段，该值是一个字符串元组或者列表

其中每个字符串都是一个字段名称，带有一个 `-` 前缀，表示降序。没有 `-` 的字段将按升序排列

```python
# 降序
ordering = ['-pub_date']

# 升序
ordering = ['pub_date']

# 通过 pub_date 降序然后 author 升序进行排序
ordering = ['-pub_date', 'author']
```

##### indexes

在模型上定义索引列表

```python
from django.db import models

class Customer(models.Model):
    first_name = models.CharField(max_length=100)
    last_name = models.CharField(max_length=100)

    class Meta:
        indexes = [
            models.Index(fields=['last_name', 'first_name']),
            models.Index(fields=['first_name'], name='first_name_idx'),
        ]
```

##### verbose_name

表的可读名称，后台显示

### 模型属性

- `objects`

  模型当中最重要的属性是 [`Manager`](https://docs.djangoproject.com/zh-hans/3.1/topics/db/managers/#django.db.models.Manager)。它是 Django 模型和数据库查询操作之间的接口，并且它被用作从数据库当中 [获取实例](https://docs.djangoproject.com/zh-hans/3.1/topics/db/queries/#retrieving-objects)，如果没有指定自定义的 `Manager` 默认名称是 [`objects`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/class/#django.db.models.Model.objects)。Manager 只能通过模型类来访问，不能通过模型实例来访问。
  
- pk

  每个模型都有一个名为 `pk` 的属性，他是模型主键字段的别名

### [操作数据库](https://docs.djangoproject.com/zh-hans/3.1/topics/db/queries/)

#### save()

插入或者修改

如果 `id` 字段的值存在，则插入，否则更新

如果要自定义保存行为，则可以覆盖此`save()` 方法

```python
User.objects.save()
```

**指定哪些字段保存**

如果给 `save()` 方法传递了关键字字段名称列表 `update_fields`，则仅更新该列表中命名的字段

```python
product.name = 'Name changed again'
product.save(update_fields=['name'])
```

该`update_fields`参数可以是任何可迭代的包含字符串的参数

指定`update_fields`将强制进行更新

#### 查询对象

要从数据库检索对象，要通过模型类的 [`Manager`](https://docs.djangoproject.com/zh-hans/3.1/topics/db/managers/#django.db.models.Manager) 构建一个 [`QuerySet`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/querysets/#django.db.models.query.QuerySet) 

一个 [`QuerySet`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/querysets/#django.db.models.query.QuerySet) 代表来自数据库中对象的一个集合。它可以有 0 个，1 个或者多个，可以根据给定参数缩小查询结果。在 SQL 的层面上， [`QuerySet`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/querysets/#django.db.models.query.QuerySet) 对应 `SELECT` 语句，而*filters*对应类似 `WHERE` 或 `LIMIT` 的限制子句

可以通过模型的 [`Manager`](https://docs.djangoproject.com/zh-hans/3.1/topics/db/managers/#django.db.models.Manager) 获取 [`QuerySet`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/querysets/#django.db.models.query.QuerySet)。每个模型至少有一个 [`Manager`](https://docs.djangoproject.com/zh-hans/3.1/topics/db/managers/#django.db.models.Manager)，默认名称是 [`objects`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/class/#django.db.models.Model.objects)  

##### 查询全部

方法 [`all()`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/querysets/#django.db.models.query.QuerySet.all) 返回一个包含数据库中所有对象的 [`QuerySet`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/querysets/#django.db.models.query.QuerySet) 对象

```python
User.objects.all()
```

##### 添加查询条件

给 `QuerySet` 添加过滤条件

```python
filter(**kwargs)
```

返回一个新的 [`QuerySet`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/querysets/#django.db.models.query.QuerySet)，包含的对象满足给定查询条件

```python
exclude(**kwargs)
```

返回一个新的 [`QuerySet`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/querysets/#django.db.models.query.QuerySet)，包含的对象不满足给定查询参数

```python
User.objects.filter(pub_date__year=2006)
User.objects.all().filter(pub_date__year=2006)
```

##### 链式调用

只要是返回结果是 `QuerySet` 的都可链式调用

```python
User.objects.filter(name__eq='alix').filter(age__lt=30)
```

##### `QuerySet` 是惰性的

`QuerySet` 是惰性的 —— 创建 [`QuerySet`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/querysets/#django.db.models.query.QuerySet) 并不会引发任何数据库活动

一般来说， [`QuerySet`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/querysets/#django.db.models.query.QuerySet) 的结果到你 `要使用` 时才会从数据库中拿出

##### 限制 `QuerySet` 的数量

利用 `Python` 的数组切片语法将 [`QuerySet`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/querysets/#django.db.models.query.QuerySet) 切成指定长度。这等价于 SQL 的 `LIMIT` 和 `OFFSET` 子句

**不支持负索引**

```python
# 返回前 5 个对象 (LIMIT 5)
Entry.objects.all()[:5]
```

#### `get()` 查询单个对象

返回单个结果对象

```python
one_entry = User.objects.get(pk=1)
```

 [`get()`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/querysets/#django.db.models.query.QuerySet.get) 和 [`filter()`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/querysets/#django.db.models.query.QuerySet.filter) 有类似的查询表达式，即查询条件

如果没有满足查询条件的结果， [`get()`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/querysets/#django.db.models.query.QuerySet.get) 会抛出 `DoesNotExist` 异常

#### [查询条件](https://docs.djangoproject.com/zh-hans/3.1/topics/db/queries/#field-lookups)

如何制定 SQL `WHERE` 子句。它们以关键字参数的形式传递给 [`QuerySet`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/querysets/#django.db.models.query.QuerySet) 的 [`filter()`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/querysets/#django.db.models.query.QuerySet.filter) 方法， [`exclude()`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/querysets/#django.db.models.query.QuerySet.exclude) 和 [`get()`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/querysets/#django.db.models.query.QuerySet.get)。

基本的查询关键字参数是： `field__lookuptype=value`（有个双下划线）

这里的 `field` 是模型的一个字段名

```python
Entry.objects.filter(pub_date__lte='2006-01-01')
# 对应 SQL: SELECT * FROM blog_entry WHERE pub_date <= '2006-01-01';
```

[**常用查询条件**](https://docs.djangoproject.com/zh-hans/3.1/ref/models/querysets/#field-lookups)

`MySql` 中尽量少使用嵌套查询

##### [`exact`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/querysets/#std:fieldlookup-exact)

等于查询

如果值是`None`，它将被解释为SQL `NULL`

```python
Entry.objects.get(headline__exact="Cat bites dog")
# 对应sql：SELECT ... WHERE headline = 'Cat bites dog';
```

如果关键字参数没有双下划线(`__`)，查询类型会被指定为 `exact`

```python
# 两条语句是等价的
Blog.objects.get(id__exact=14)  # Explicit form
Blog.objects.get(id=14)         # __exact is implied
```

##### [`iexact`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/querysets/#std:fieldlookup-iexact)

不分大小写的匹配

```python
Blog.objects.get(name__iexact="beatles blog")
```

##### [`contains`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/querysets/#std:fieldlookup-contains)

是否包含 - 大小写敏感

`icontains` 是否包含 - 不区分大小写

```python
Entry.objects.get(headline__contains='Lennon')
# sql: SELECT ... WHERE headline LIKE '%Lennon%';
```

##### [`startswith`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/querysets/#std:fieldlookup-startswith), [`endswith`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/querysets/#std:fieldlookup-endswith)

`以……开头`和`以……结尾`的查找 - 大小写敏感

 [`istartswith`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/querysets/#std:fieldlookup-istartswith) 和 [`iendswith`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/querysets/#std:fieldlookup-iendswith)：大小写不敏感

##### in

查询在给定的迭代中(通常是列表，元组或查询集)。可以接受字符串（可迭代）

```python
# SELECT ... WHERE id IN (1, 3, 4);
Entry.objects.filter(id__in=[1, 3, 4])

# SELECT ... WHERE headline IN ('a', 'b', 'c');
Entry.objects.filter(headline__in='abc')
```

还可以使用查询集来动态查询

```python
inner_qs = Blog.objects.filter(name__contains='Cheddar')
entries = Entry.objects.filter(blog__in=inner_qs)
# SELECT ... WHERE blog.id IN (SELECT id FROM ... WHERE NAME LIKE '%Cheddar%')
```

##### gt gte lt lte

gt: 大于 >

gte: 大于等于>=

lt: 小于 <

lte: 小于等于 <=

```python
Entry.objects.filter(id__gt=4)
# SELECT ... WHERE id > 4;
```

##### range

范围查询

```python
import datetime
start_date = datetime.date(2005, 1, 1)
end_date = datetime.date(2005, 3, 31)
Entry.objects.filter(pub_date__range=(start_date, end_date))
# SELECT ... WHERE pub_date BETWEEN '2005-01-01' and '2005-03-31';
# SELECT ... WHERE pub_date BETWEEN '2005-01-01 00:00:00' and '2005-03-31 00:00:00';
```

您可以在SQL中使用的`range`任何位置使用`BETWEEN`日期和数字甚至字符

##### date

对于日期时间字段，将值强制转换为日期，并允许链接其他字段查找

```python
Entry.objects.filter(pub_date__date=datetime.date(2005, 1, 1))
Entry.objects.filter(pub_date__date__gt=datetime.date(2005, 1, 1))
```

##### year month day week

对于日期和日期时间字段，精确匹配年份。允许链接其他字段查找。需要一个整数年。

```python
Entry.objects.filter(pub_date__year=2005)
# SELECT ... WHERE pub_date BETWEEN '2005-01-01' AND '2005-12-31';

Entry.objects.filter(pub_date__year__gte=2005)
# SELECT ... WHERE pub_date >= '2005-01-01';
```

##### hour minute second

对于日期时间和时间字段，精确小时匹配。允许链接其他字段查找

hour: 取0到23之间的整数

minute: 取0到59之间的整数

second: 取0到59之间的整数

```python
Event.objects.filter(timestamp__hour=23)
# SELECT ... WHERE EXTRACT('hour' FROM timestamp) = '23';

Event.objects.filter(time__hour=5)
# SELECT ... WHERE EXTRACT('hour' FROM time) = '5';

Event.objects.filter(timestamp__hour__gte=12)
# SELECT ... WHERE EXTRACT('hour' FROM timestamp) >= '12';
```

##### isnull

采用任一`True`或`False`，其对应于SQL查询的分别: `IS NULL` 和 `IS NOT NULL`

```python
Entry.objects.filter(pub_date__isnull=True)
# SELECT ... WHERE pub_date IS NULL;
```

##### regex

区分大小写的正则表达式匹配

`iregex`: 不区分大小写的正则表达式匹配

正则表达式语法是所使用的数据库后端的语法。对于没有内置正则表达式支持的SQLite，此功能由（Python）用户定义的REGEXP函数提供，因此正则表达式语法与Python `re`模块的语法相同。

```python
Entry.objects.get(title__regex=r'^(An?|The) +')
```

#### 聚合查询

`Django` 提供了两种生成聚合的方法

1. `QuerySet` 后添加 `aggregate()` 子句

   ```python
   from django.db.models import Avg
   
   # 计算 price 字段的平均值
   Book.objects.all().aggregate(Avg('price'))
   # 可简化成下面的写法
   Book.objects.aggregate(Avg('price'))
   # 结果：{'price__avg': 34.35}
   # 这里的 price__avg 是根据字段名和聚合函数而自动生成的
   # 如果你想指定一个聚合值的名称，你可以在指定聚合子句的时候提供指定的名称
   Book.objects.aggregate(average_price=Avg('price'))
   # 结果：{'average_price': 34.35}
   
   # 多个聚合值
   Book.objects.aggregate(Avg('price'), Max('price'), Min('price'))
   # 结果：{'price__avg': 34.35, 'price__max': Decimal('81.20'), 'price__min': Decimal('12.99')}
   ```

   传递给 `aggregate()` 的参数描述了我们想要计算的聚合值

   可用的聚合函数列表 [QuerySet reference](https://docs.djangoproject.com/zh-hans/3.1/ref/models/querysets/#aggregation-functions) 

2. 分组聚合 `annotate()` 

   先分组然后再聚合，返回 `QuerySet` 对象

   使用 [`annotate()`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/querysets/#django.db.models.query.QuerySet.annotate) 子句生成每一个对象的汇总

   当指定 `annotate()` 子句，`QuerySet` 中的每一个对象将对指定值进行汇总。

   名称是根据聚合函数和被聚合的字段名自动生成的，可以通过提供一个别名重写这个默认名

   ```python
   # 按照 authors 分组统计数量
   Book.objects.annotate(num_authors=Count('authors'))
   ```

   指定查询哪个字段

   ```python
   Student.objects.values('name').annotate(Count('hobbies'))
   ```

##### 连接(Joins)和聚合

关联表进行聚合查询

在聚合函数里面指定聚合的字段时，`Django` 允许你在过滤相关字段的时候使用双下划线表示法。`Django` 将处理任何需要检索和聚合的关联值的表连接(table joins)

```python
from django.db.models import Max, Min
Store.objects.annotate(min_price=Min('books__price'), max_price=Max('books__price'))
```

检索 `Store` 模型，连接（通过多对多关系） `Book` 模型，并且聚合书籍模型的价格字段来获取最大最小值

##### order_by()

可以对聚合查询进行排序

```python
Book.objects.annotate(num_authors=Count('authors')).order_by('num_authors')
```

##### values()

当使用 `values()` 时，先根据定义在 `values()` 子句中的字段组合先对结果进行分组，再对每个单独的分组进行注解，这个注解值是根据分组中所有的对象计算得到的

```python
Author.objects.annotate(average_rating=Avg('book__rating'))
```

这段代码返回的是数据库中的所有作者及其所著书的平均评分

但是如果你使用 `values()` 子句，结果会稍有不同：

```python
Author.objects.values('name').annotate(average_rating=Avg('book__rating'))
```

作者会按名字分组，所以你只能得到不重名的作者分组的注解值。这意味着如果你有两个作者同名，那么他们原本各自的查询结果将被合并到同一个结果中；两个作者的所有评分都将被计算为一个平均分

- 如果 `values()` 子句在 `annotate()` 之前，就会根据 `values()` 子句产生的分组来进行分组统计

- 如果 `annotate()` 子句在 `values()` 之前，就会对整个查询集分组统计。这种情况下，`values()` 子句只能限制输出的字段

#### [QuerySet API](https://docs.djangoproject.com/zh-hans/3.1/ref/models/querysets/#queryset-api) 

```python
QuerySet(model=None, query=None, using=None, hints=None)
```

`QuerySet`方法返回新的查询集

##### QuerySet 常用方法

##### delete()

删除

```python
b = Blog.objects.get(pk=1)

# Delete all the entries belonging to this Blog.
Entry.objects.filter(blog=b).delete()
# (4, {'weblog.Entry': 2, 'weblog.Entry_authors': 2})
```

##### bulk_create()

```python
bulk_create(objs, batch_size=None, ignore_conflicts=False)
```

批量插入

```python
Entry.objects.bulk_create([
     Entry(headline='This is a test'),
     Entry(headline='This is only a test'),
])
```

##### update()

```python
update(**kwargs)
```

更新指定字段，并返回操作的行数

```python
Entry.objects.filter(id=10).update(comments_on=False)
```

##### bulk_update()

批量更新

##### filter()

```python
filter(**kwargs)
```

返回一个`QuerySet`包含与给定查找参数匹配的新对象

##### exclude()

```python
exclude(**kwargs)
```

返回一个新的`QuerySet`包含对象，这些对象与给定的查找参数*不*匹配

```python
Entry.objects.exclude(pub_date__gt=datetime.date(2005, 1, 3), headline='Hello')
# SELECT ... WHERE NOT (pub_date > '2005-1-3' AND headline = 'Hello')
```

##### order_by()

```python
order_by(**fields)
```

默认情况下，`QuerySet` 使用模型元数据`Meta` 的 `ordering` 给出列表进行排序

但是可以使用 `order_buy()` 方法覆盖默认排序

```python
Entry.objects.filter(pub_date__year=2005).order_by('-pub_date', 'headline')
```

上面的结果将按`pub_date`降序排列，然后按 `headline`升序排列。前面的负号`"-pub_date"`表示降序

随机排序使用`"?"`

```python
Entry.objects.order_by('?')
```

注意：`order_by('?')`查询可能很慢

使用不同模型中的字段排序，则模型名称后跟双下划线（`__`），然后是新模型中的字段名称

```python
Entry.objects.order_by('blog__name', 'headline')
```

如果未指定关联模型的排序字段，则 `Django` 将在相关模型上使用默认的排序，如果`Meta.ordering` 未指定，则按相关模型的主键进行排序

```python
Entry.objects.order_by('blog')
# 等同于：Entry.objects.order_by('blog__id')
```

##### distinct()

```python
distinct(*fields)
```

从查询结果中消除重复的行

##### values()

```python
values(*fields，**expressions)
```

返回一个包含字典的 `QuerySet`，而不是包含模型的 `QuerySet`

参数说明：

`fields`: 指定结果中包含哪些字段。如果指定字段，每个字典将只包含指定的字段的键/值。如果没有指定字段，则查询表中的所有字段

`expressions`: 可选的关键字参数，这些参数将传递给[`annotate()`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/querysets/#django.db.models.query.QuerySet.annotate)

##### values_list()

```python
values_list(*fields, flat=False, named=False)
```

返回包含元组的 `QuerySet` 对象，元组中的元素按照参数指定的顺讯排列

```python
Entry.objects.values_list('id', 'headline')
# <QuerySet [(1, 'First entry'), ...]>
```

如果仅传递单个字段，可以传递`flat` 参数。如果为`True`，则表示返回的结果是单个值

```python
Entry.objects.values_list('id').order_by('id')
# <QuerySet[(1,), (2,), (3,), ...]>

Entry.objects.values_list('id', flat=True).order_by('id')
# <QuerySet [1, 2, 3, ...]>
```

通过`named=True`来获得结果命名元组结果

```python
Entry.objects.values_list('id', 'headline', named=True)
# <QuerySet [Row(id=1, headline='First entry'), ...]>
```

如果您没有将任何值传递给`values_list()`，它将按照模型中的声明它们的顺序返回模型中的所有字段

需要获取某个模型实例的特定字段值。则先使用`values_list()`然后再`get()`调用：

```python
Entry.objects.values_list('headline', flat=True).get(pk=1)
# 'First entry'
```

##### all()

返回包含所有结果的 `QuerySet`

##### union()

```python
union(*other_qs, all=False)
```

合并两个或多个`QuerySet`的结果

`all = False`: 不允许重复的值

`all = True`: 允许重复的值

```python
qs1.union(qs2, qs3)
```

##### defer()

```python
defer(*fields)
```

延迟查询指定的字段，当访问该字段时才从数据库中获取，即告诉 `Django` 不要从数据库中检索它们

```python
Entry.objects.defer("headline", "body")
```

如果要清除一组延迟字段，请`None`作为参数传递给`defer()`：

```python
# Load all fields immediately.
my_queryset.defer(None)
```

##### only()

与 `defer()` 相反，用来指定立即查询的字段，其他的字段都延迟查询

```python
# 下面两个相同
Person.objects.defer("age", "biography")
Person.objects.only("name")
```

##### select_for_update()

```python
select_for_update(nowait=False, skip_locked=False, of=())
```

返回一个查询集，该查询集将锁定行直到事务结束，相当于：SELECT ... FOR UPDATE`

```python
from django.db import transaction

entries = Entry.objects.select_for_update().filter(author=request.user)
with transaction.atomic():
    for entry in entries:
        ...
```

##### raw()

```python
raw(raw_query, params=None, translations=None)
```

进行原始SQL查询，返回一个 `django.db.models.query.RawQuerySet`实例

##### get()

```python
get(**kwargs)
```

返回符合条件的单个模型对象结果

```python
Entry.objects.get(id=1)
Entry.objects.get(blog=blog, entry_number=1)
```

如果`get()`找不到任何对象，则会引发[`Model.DoesNotExist`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/class/#django.db.models.Model.DoesNotExist)异常

如果`get()`发现多个对象，则会引发 [`Model.MultipleObjectsReturned`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/class/#django.db.models.Model.MultipleObjectsReturned)异常

捕获异常

```python
from django.core.exceptions import ObjectDoesNotExist

try:
    blog = Blog.objects.get(id=1)
    entry = Entry.objects.get(blog=blog, entry_number=1)
except ObjectDoesNotExist:
    print("Either the blog or entry doesn't exist.")
```

##### create()

便捷方法，可一步创建对象并将其全部保存

```python
p = Person.objects.create(first_name="Bruce", last_name="Springsteen")
```

##### update_or_create()

```python
update_or_create(defaults=None, **kwargs)
```

查找并跟新，`kwargs` 要查找的条件，`defaults` （字典）要更新到数据库的值

尝试根据给定的 `kwargs` 的值查询数据库，如果找到就根据 `defaults` 字典给定的值更新字段

```python
obj, created = Person.objects.update_or_create(
    first_name='John', last_name='Lennon',
    defaults={'first_name': 'Bob'},
)
```

##### count()

返回一个整数，该整数表示数据库中匹配条件的数量

```python
# Returns the total number of entries in the database.
Entry.objects.count()

# Returns the number of entries whose headline contains 'Lennon'
Entry.objects.filter(headline__contains='Lennon').count()
```

##### latest()

根据给定的字段返回表中的最新对象

```python
Entry.objects.latest('pub_date')
```

##### first()

返回与查询集匹配的第一个对象，或者`None`没有匹配的对象。如果`QuerySet`未定义排序，则按主键进行排序

```python
p = Article.objects.order_by('title', 'pub_date').first()

# 也可这样写
try:
    p = Article.objects.order_by('title', 'pub_date')[0]
except IndexError:
    p = None
```

##### last()

返回最后一个对象

##### exists()

返回 `True/False`，有结果则返回 `True`，否则返回 `False`

##### explain()

返回执行 `SQL` 的**执行计划**字符串

#### QuerySet 运算符

将两个相同模型的结果合并

##### AND (&)

使用 `&` 合并两个 `QuerySet` 结果

```python
Model.objects.filter(x=1) & Model.objects.filter(y=2)
Model.objects.filter(x=1, y=2)

from django.db.models import Q
Model.objects.filter(Q(x=1) & Q(y=2))

# SELECT ... WHERE x=1 AND y=2
```

##### OR(|)

使用 `|` 合并两个 `QuerySet` 结果

```python
Model.objects.filter(x=1) | Model.objects.filter(y=2)

from django.db.models import Q
Model.objects.filter(Q(x=1) | Q(y=2))

# SELECT ... WHERE x=1 OR y=2
```

### Q对象和F对象

#### Q对象

一个 [`Q 对象`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/querysets/#django.db.models.Q) (`django.db.models.Q`) 用于压缩关键字参数集合

```python
from django.db.models import Q
Q(question__startswith='What')
```

`Q` 对象能通过 `&` 和 `|` 操作符连接起来。当操作符被用于两个 `Q` 对象之间时会生成一个新的 `Q` 对象

```python
Q(question__startswith='Who') | Q(question__startswith='What')
# WHERE question LIKE 'Who%' OR question LIKE 'What%'
```

`Q` 对象也可通过 `~` 操作符取反，允许在组合查询中组合普通查询或反向 (`NOT`) 查询:

```python
Q(question__startswith='Who') | ~Q(pub_date__year=2005)
```

每个接受关键字参数的查询函数 (例如 [`filter()`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/querysets/#django.db.models.query.QuerySet.filter)， [`exclude()`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/querysets/#django.db.models.query.QuerySet.exclude)， [`get()`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/querysets/#django.db.models.query.QuerySet.get)) 也同时接受一个或多个 `Q` 对象作为位置（未命名的）参数

若为查询函数提供了多个 `Q` 对象参数，这些参数会通过 "AND" 连接

```python
Poll.objects.get(
    Q(question__startswith='Who'),
    Q(pub_date=date(2005, 5, 2)) | Q(pub_date=date(2005, 5, 6))
)

# SELECT * from polls WHERE question LIKE 'Who%'
#    AND (pub_date = '2005-05-02' OR pub_date = '2005-05-06')
```

查询函数能混合使用 `Q` 对象和关键字参数，此时 `Q` 对象必须位于所有关键字参数之前

```python
Poll.objects.get(
    Q(pub_date=date(2005, 5, 2)) | Q(pub_date=date(2005, 5, 6)),
    question__startswith='Who',
)
```

#### [F对象](https://docs.djangoproject.com/zh-hans/3.1/ref/models/expressions/#django.db.models.F)

一个`F()`对象表示一个模型字段或注释的列的值引用。它使得引用模型字段值并使用它们执行数据库操作成为可能，而实际上不必将它们从数据库中拉出到Python内存中

```python
from django.db.models import F

reporter = Reporters.objects.get(name='Tintin')
reporter.stories_filed = F('stories_filed') + 1
reporter.save()
```

上面操作只是操作一次数据库

如果使用下面的将进行两次数据库操作

```python
reporter = Reporters.objects.get(name='Tintin')
reporter.stories_filed += 1
reporter.save()
```

### 比较对象

比较两个模型实例，使用标准的 `Python` 比较操作符： `==`

实际上，这比较了两个模型实例的主键值

```python
# 下面等效
some_entry == other_entry
some_entry.id == other_entry.id
```

### 缓存 和 QuerySet

每个 [`QuerySet`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/querysets/#django.db.models.query.QuerySet) 都带有缓存，尽量减少数据库访问

新创建的 [`QuerySet`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/querysets/#django.db.models.query.QuerySet) 缓存是空的。一旦要计算 [`QuerySet`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/querysets/#django.db.models.query.QuerySet) 的值，就会执行数据查询，随后，Django 就会将查询结果保存在 [`QuerySet`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/querysets/#django.db.models.query.QuerySet) 的缓存中，并返回这些显式请求的缓存（例如，下一个元素，若 [`QuerySet`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/querysets/#django.db.models.query.QuerySet) 正在被迭代）。后续针对 [`QuerySet`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/querysets/#django.db.models.query.QuerySet) 的计算会复用缓存结果

### [执行原生SQL](https://docs.djangoproject.com/zh-hans/3.1/topics/db/sql/#performing-raw-queries)

Django 允许你用两种方式执行原生 SQL 查询

####  [Manager.raw()](https://docs.djangoproject.com/zh-hans/3.1/topics/db/sql/#django.db.models.Manager.raw) 

```python
Manager.raw(raw_query, params=None, translations=None)
```

该方法接受一个原生 `SQL` 查询语句，并返回一个 `django.db.models.query.RawQuerySet` 实例。这个 `RawQuerySet` 能像普通的 [`QuerySet`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/querysets/#django.db.models.query.QuerySet) 一样被迭代获取对象实例

```python
Person.objects.raw('SELECT * FROM myapp_person')
```

##### 将查询字段映射为模型字段

`raw()` 字段将查询语句中的字段映射至模型中的字段

可以使用 `SQL` 的 `AS` 子句将查询语句中的字段映射至模型中的字段，`AS` 后面的名字必须和模型中的名字相同才可映射到模型

```python
Person.objects.raw('''SELECT first AS first_name,
...                              last AS last_name,
...                              bd AS birth_date,
...                              pk AS id,
...                       FROM some_other_table''')
```

可以用 `raw()` 的 `translations` 参数将查询语句中的字段映射至模型中的字段

```python
>>> name_map = {'first': 'first_name', 'last': 'last_name', 'bd': 'birth_date', 'pk': 'id'}
>>> Person.objects.raw('SELECT * FROM some_other_table', translations=name_map)
```

##### 索引查询

`raw()` 支持索引

```python
# 查询第一个结果
first_person = Person.objects.raw('SELECT * FROM myapp_person')[0]
```

不过，索引和切片不是在数据库层面上实现的，这里只是 `Python` 中的集合索引操作

##### 延迟模型字段

省略字段

```python
people = Person.objects.raw('SELECT id, first_name FROM myapp_person')
```

该查询返回的 `Person` 对象即延迟模型实例。这意味着查询语句中省略的字段按需加载

```
>>> for p in Person.objects.raw('SELECT id, first_name FROM myapp_person'):
...     print(p.first_name, # This will be retrieved by the original query
...           p.last_name) # This will be retrieved on demand
...
John Smith
Jane Jones
```

表面上，看起来该查询同时检出了 first name 和 last name。然而，这个例子实际上执行了三次查询。只有 first names 是由 raw() 查询检出的 —— last names 是在它们被打印时按需检出。

**主键字段不能省略**。`Django` 用主键来区分模型实例，所以必须在原生查询语句中包含主键。若你忘了包含主键会抛出 [`FieldDoesNotExist`](https://docs.djangoproject.com/zh-hans/3.1/ref/exceptions/#django.core.exceptions.FieldDoesNotExist) 异常

##### 给 `raw()` 传递参数

如果你需要执行参数化的查询，可以使用 `raw()` 的 `params` 参数:

```python
>>> lname = 'Doe'
>>> Person.objects.raw('SELECT * FROM myapp_person WHERE last_name = %s', [lname])
```

`params` 是一个参数字典。你将用一个列表替换查询字符串中 `%s` 占位符，或用字典替换 `%(key)s` 占位符（`key` 被字典 key 替换）

#### [直接执行自定义 SQL](https://docs.djangoproject.com/zh-hans/3.1/topics/db/sql/#executing-custom-sql-directly)

直接访问数据库，完全绕过模型层

对象 `django.db.connection` 代表默认数据库连接。使用这个数据库连接，调用 `connection.cursor()` 来获取一个指针对象。然后，调用 `cursor.execute(sql, [params])` 来执行该 SQL 和 `cursor.fetchone()`，或 `cursor.fetchall()` 获取结果数据

```python
from django.db import connection

def my_custom_sql(self):
    with connection.cursor() as cursor:
        cursor.execute("UPDATE bar SET foo = 1 WHERE baz = %s", [self.baz])
        cursor.execute("SELECT foo FROM bar WHERE baz = %s", [self.baz])
        row = cursor.fetchone()

    return row
```

### 调用存储过程

```python
CursorWrapper.callproc(procname, params=None, kparams=None)
```

以给定名称调用数据库存储过程。要提供一个序列 (`params`) 或字典 (`kparams`) 作为输入参数。大多数数据库不支持 `kparams`

```python
from django.db import connection

with connection.cursor() as cursor:
    cursor.callproc('test_procedure', [1, 'test'])
```

### [模型继承](https://docs.djangoproject.com/zh-hans/3.1/topics/db/models/#model-inheritance)

所有模型继承自 [`django.db.models.Model`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/instances/#django.db.models.Model)

#### 抽象基类

抽象基类在你要将公共信息放入很多模型时会很有用。编写你的基类，并在 [Meta](https://docs.djangoproject.com/zh-hans/3.1/topics/db/models/#meta-options) 类中填入 `abstract=True`。该模型将不会对应任何数据表。当其用作其它模型类的基类时，它的字段会自动添加至子类

```python
from django.db import models

class CommonInfo(models.Model):
    name = models.CharField(max_length=100)
    age = models.PositiveIntegerField()

    class Meta:
        abstract = True

class Student(CommonInfo):
    home_group = models.CharField(max_length=5)
```

从抽象基类继承来的字段可被其它字段或值重写，或用 `None` 删除

#### 多表继承

继承层次结构中的每个模型都是一个单独的模型。每个模型都指向分离的数据表，且可被独立查询和创建

```python
from django.db import models

class Place(models.Model):
    name = models.CharField(max_length=50)
    address = models.CharField(max_length=80)

class Restaurant(Place):
    serves_hot_dogs = models.BooleanField(default=False)
    serves_pizza = models.BooleanField(default=False)
```

多表继承情况下，子类不会继承父类的 [Meta](https://docs.djangoproject.com/zh-hans/3.1/topics/db/models/#meta-options)

#### 代理模型

使用 [多表继承](https://docs.djangoproject.com/zh-hans/3.1/topics/db/models/#multi-table-inheritance) 时，每个子类模型都会创建一张新表。这一般是期望的行为，因为子类需要一个地方存储基类中不存在的额外数据字段。不过，有时候你只想修改模型的 Python 级行为——可能是修改默认管理器，或添加一个方法。

这是代理模型继承的目的：为原模型创建一个 *代理*。你可以创建，删除和更新代理模型的实例，所以的数据都会存储的像你使用原模型（未代理的）一样。不同点是你可以修改代理默认的模型排序和默认管理器，而不需要修改原模型。

代理模型就像普通模型一样申明。你需要告诉 Django 这是一个代理模型，通过将 `Meta` 类的 [`proxy`](https://docs.djangoproject.com/zh-hans/3.1/ref/models/options/#django.db.models.Options.proxy) 属性设置为 `True`

```python
from django.db import models

class Person(models.Model):
    first_name = models.CharField(max_length=30)
    last_name = models.CharField(max_length=30)

class MyPerson(Person):
    class Meta:
        proxy = True

    def do_something(self):
        # ...
        pass
```

### [数据库事务](https://docs.djangoproject.com/zh-hans/3.1/topics/db/transactions/#module-django.db.transaction)  

#### 管理数据库事务

##### [Django 默认的事务行为](https://docs.djangoproject.com/zh-hans/3.1/topics/db/transactions/#django-s-default-transaction-behavior)

`Django` 默认的事务行为是自动提交

##### [连结事务与 HTTP 请求](https://docs.djangoproject.com/zh-hans/3.1/topics/db/transactions/#tying-transactions-to-http-requests)

在 Web 里，处理事务比较常用的方式是将每个请求封装在一个事务中。 在你想启用事物的数据库中，把配置中 `settings.py -> DATABASES` 的参数 [`ATOMIC_REQUESTS`](https://docs.djangoproject.com/zh-hans/3.1/ref/settings/#std:setting-DATABASE-ATOMIC_REQUESTS) 设置为 `True`，这里开启的是全局事物

```python
non_atomic_requests(using=None)
```

取消事物

```python
from django.db import transaction

@transaction.non_atomic_requests
def my_view(request):
    do_stuff()

@transaction.non_atomic_requests(using='other')
def my_other_view(request):
    do_stuff_on_the_other_database()
```

##### [显式控制事务](https://docs.djangoproject.com/zh-hans/3.1/topics/db/transactions/#controlling-transactions-explicitly)

```python
atomic(using=None, savepoint=True)
```

`atomic` 允许创建代码块来保证数据库的原子性。如果代码块成功创建，这个变动会提交到数据库。如果有异常，变动会回滚。

`atomic` 块可以嵌套

```python
from django.db import transaction

# 开启事物
@transaction.atomic
def viewfunc(request):
    # This code executes inside a transaction.
    do_stuff()
```

#### 提交后

有时你需要执行与当前数据库事务相关的操作，但前提是事务成功提交。例子可能包含 [Celery](https://pypi.org/project/celery/) 任务，邮件提醒或缓存失效

```python
on_commit(func, using=None)
```

该回调函数将在事物成功执行后被调用

```python
from django.db import transaction

def do_something():
    pass  # send a mail, invalidate a cache, fire off a Celery task, etc.

transaction.on_commit(do_something)
```

### 数据库迁移

迁移是 `Django` 将你对模型的修改（例如增加一个字段，删除一个模型）应用至数据库表结构对方式

`Django` 处理数据库表结构的方式：

- [`migrate`](https://docs.djangoproject.com/zh-hans/3.1/ref/django-admin/#django-admin-migrate)，负责将迁移文件应用到数据库和撤销迁移
- [`makemigrations`](https://docs.djangoproject.com/zh-hans/3.1/ref/django-admin/#django-admin-makemigrations)，将模型修改打包进独立的迁移文件中
- [`sqlmigrate`](https://docs.djangoproject.com/zh-hans/3.1/ref/django-admin/#django-admin-sqlmigrate)，展示迁移使用的 SQL 语句
- [`showmigrations`](https://docs.djangoproject.com/zh-hans/3.1/ref/django-admin/#django-admin-showmigrations)，列出项目的迁移和迁移的状态

 `makemigrations` 负责将模型修改打包进独立的迁移文件中——类似提交修改，而 `migrate` 负责将其应用至数据库

每个应用的迁移文件位于该应用的 `migrations` 目录中

迁移步骤：

1. 生成迁移文件

   ```shell
   $ python manage.py makemigrations
   ```

2. 将迁移文件应用于数据库

   ```shell
   $ python manage.py migrate
   ```

### [使用已有数据库](https://docs.djangoproject.com/zh-hans/3.1/howto/legacy-databases/)

按照已经存在的数据库创建对应模型

```shell
$ python manage.py inspectdb
```

通过标准 `Unix` 输出重定向将其保存到指定的文件

```shell
$ python manage.py inspectdb > models.py
```

默认情况下， [`inspectdb`](https://docs.djangoproject.com/zh-hans/3.1/ref/django-admin/#django-admin-inspectdb) 创建未托管的模型。也就是说，模型 `Meta` 类中的 `managed = False` ，也就是说对模型的修改不影响数据库

### [分页](https://docs.djangoproject.com/zh-hans/3.1/topics/pagination/)

分页使用 `Paginator` 类实现

```python
Paginator(object_list, per_page, orphans=0, allow_empty_first_page=True)
```

参数说明：

- `Paginator.object_list`

  必须参数。其值可以是：`QuerySet`、列表、元组、或其他可切片对象 

- `Paginator.per_page`

  必须参数。每页数量（pageSize）

- `Paginator.orphans`

  可选。如果您不希望最后一页包含很少的项目，请使用此选项。如果最后一页通常包含少于或等于的项目`orphans`，那么这些项目将被添加到上一页（即成为最后一页），而不是将这些项目自己留在页面上

- `Paginator.allow_empty_first_page`

  可选。第一页是否允许为空。如果 `False`和`object_list`为空，则将引发`EmptyPage`错误。

#### Paginator 方法:

- `Paginator.get_page(页码)`

  返回指定页码的 [`Page`](https://docs.djangoproject.com/zh-hans/3.1/ref/paginator/#django.core.paginator.Page) 对象（即第几页）

  同时还处理超出范围和无效的页码。如果页面不是数字，则返回第一页。如果页码为负或大于页数，则返回最后一页。[`EmptyPage`](https://docs.djangoproject.com/zh-hans/3.1/ref/paginator/#django.core.paginator.EmptyPage)仅当您指定且`object_list`为空时才引发异常

- `Paginator.page(页码)`

  返回指定页码的 [`Page`](https://docs.djangoproject.com/zh-hans/3.1/ref/paginator/#django.core.paginator.Page) 对象（即第几页）
  
  如果给定的页码不存在，则引发 [`InvalidPage`](https://docs.djangoproject.com/zh-hans/3.1/ref/paginator/#django.core.paginator.InvalidPage) 异常

#### Paginator 属性

- `Paginator.count`

  总记录条数

- `Paginator.num_pages`

  总页数

- `Paginator.page_range`

  可迭代页码列表，例如：`[1, 2, 3, 4]`

#### 使用

```python
from django.core.paginator import Paginator

objects = ['john', 'paul', 'george', 'ringo']
all_user = User.objects.all()

# 创建分页器
p = Paginator(objects, 2)
page_user = Paginator(all_user, 10)

# 获取第一页数据
first_page = p.page(1)
first_page_user = page_user.page(1)

# 获取第一个所有数据列表
data_list = first_page.object_list
```

## session









## form 表单







