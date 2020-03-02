# python-web

第一个python练手项目

# 搭建环境

- 安装python

安装python3，进入地址[https://www.python.org/downloads/](https://www.python.org/downloads/)下载安装包，下载完成后点击安装，这里安装3.7版本。

- 更改python对应版本

执行命令`open ~/.bash_profile`，向文件里面添加一行，完整配置如下：

```
# Setting PATH for Python 3.7
# The original version is saved in .bash_profile.pysave
PATH="/Library/Frameworks/Python.framework/Versions/3.7/bin:${PATH}"
export PATH

# 添加或注释，决定使用哪个python版本
# 如果python是通过brew安装的，则地址应该为/usr/local/Cellar/python/3.7.3/Frameworks/Python.framework/Versions/3.7/bin/python3.7
alias python="/Library/Frameworks/Python.framework/Versions/3.7/bin/python3.7"
```

- 安装虚拟环境virtualenv

virtualenv就是用来为一个应用创建一套“隔离”的Python运行环境

```
pip3 install virtualenv

// 创建环境
virtualenv --no-site-packages venv

// 进入环境
source venv/bin/activate

// 退出环境
deactivate
```

- 虚拟环境pipenv，带lock

```python
# 安装pipenv
pip install pipenv

# 创建虚拟环境
# 会在根目录下生成文件Pipfile和Pipfile.lock，类似package.json
pipenv install --python 3.7

#进入虚拟环境
pipenv shell

# 安装依赖
pipenv install aiohttp

# 退出环境
exit
```

> 如果安装Python包太慢，可以使用阿里云提供的镜像源，修改Pipfile前几行的内容为：

```
[[source]]
name = "pypi"
url = "https://mirrors.aliyun.com/pypi/simple"
verify_ssl = true
```

> 如果生成Pipfile.lock太慢，则可以执行pipenv install命令时添加`--skip-lock`选项来跳过lock步骤，最后使用`pipenv lock`命令来统一执行lock操作


- 安装基础包

```python
# 如果使用virtualenv，进入虚拟环境
source venv/bin/activate

pip3 install aiohttp     # 异步框架
pip3 install jinja2      # 前端模板引擎
pip3 install aiomysql    # MySQL的Python异步驱动程序

# 如果使用pipenv
pipenv shell
pipenv install aiohttp
pipenv install jinja2
pipenv install aiomysql
```

一下虚拟环境都使用pipenv，所有代码以pipenv为示例。

- 搭建基本项目目录

```
python-web/              <-- 根目录
|
+- backup/               <-- 备份目录
|
+- conf/                 <-- 配置文件
|
+- dist/                 <-- 打包目录
|
+- www/                  <-- Web目录，存放.py文件
|  |
|  +- static/            <-- 存放静态文件
|  |
|  +- templates/         <-- 存放模板文件
|
+- ios/                  <-- 存放iOS App工程
|
+- LICENSE               <-- 代码LICENSE
```

# 编写骨架

- 写一个基本的app.py

```python
import logging
# 设置log配置
logging.basicConfig(level=logging.INFO)

import asyncio, os, json, time
from datetime import datetime

from aiohttp import web

def index(request):
    return web.Response(content_type="text/html", body=b'<h1>this is a test</h1>')

@asyncio.coroutine
def init(loop):
    app = web.Application(loop=loop)
    app.router.add_route('GET', '/', index)
    srv = yield from loop.create_server(app.make_handler(), '127.0.0.1', 9000)
    logging.info('server started at http://127.0.0.1:9000...')
    return srv

loop = asyncio.get_event_loop()
loop.run_until_complete(init(loop))
loop.run_forever()
```

- 运行

执行`python ./www/app.py`运行项目，打开[http://127.0.0.1:9000](http://127.0.0.1:9000)可以看到运行结果

# 编写骨架

本项目很多地方都需要访问数据库，访问数据库需要:
```
[1]创建数据库连接、游标对象

[2]然后执行sql语句

[3]最后处理异常清理资源。
```
这些访问数据库的代码如果分散到各个函数中，势必无法维护，也不利于代码的复用，因此我们要把常用的select、insert、update、delete操作用函数封装起来。

- 创建连接池

我们将要创建一个全局的连接池，每个http请求都可以从连接池中获取数据库连接，使用连接池的好处是不必频繁的打开和关闭数据库连接，而是能复用就复用。连接池由全局变量__pool存储，缺省情况下编码自动设置为utf-8，且自动提交事务。

```py
async def create_pool(loop, **kw):
    global __pool
    __pool = await aiomysql.create_pool(
        # 获取host，且设置默认值为localhost
        host = kw.get('host', 'localhost'),
        port = kw.get('port', 3306),
        user = kw['user'],
        password = kw['password'],
        db = kw['db'],
        chartset = kw.get('chartset', 'utf-8'),
        autocommit = kw.get('autocommit', True),
        maxsize = kw.get('maxsize', 10),
        minsize = kw.get('minsize', 1),
        loop = loop
    )
```

- 编写select方法

SQL语句的占位符是?，而MySQL的占位符是%s，select()函数在内部自动替换。注意要始终坚持使用带参数的SQL，而不是自己拼接SQL字符串，这样可以防止SQL注入攻击。

```py
async def select(sql, args, size=None):
    global __pool
    async with __pool.get() as conn:
        async with conn.cursor(aiomysql.DictCursor) as cur:
            # SQL语句的占位符是?，而MySQL的占位符是%s，自动替换
            await cur.execute(sql.replace('?', '%s'), args or ())
            # 判断是否有size，决定要取全部还是其他
            if size:
                rs = await cur.fetchmany(size)
            else:
                rs = await cur.fetchall()
        logging.info('rows returned: %s' % len(rs))
        return rs
```

...
...
...


# 编写Model

把Web App需要的3个表用Model表示出来:

```py
import time, uuid

from orm import Model, StringField, BooleanField, FloatField, TextField

def next_id():
    return '%015d%s000' % (int(time.time() * 1000), uuid.uuid4().hex)

class User(Model):
    __table__ = 'users'

    # 给一个Field增加一个default参数可以让ORM自己填入缺省值
    # 缺省值可以作为函数对象传入，在调用save()时自动计算
    # 主键id的缺省值是函数next_id，创建时间created_at的缺省值是函数time.time
    id = StringField(primary_key=True, default=next_id, ddl='varchar(50)')
    email = StringField(ddl='varchar(50)')
    passwd = StringField(ddl='varchar(50)')
    admin = BooleanField()
    name = StringField(ddl='varchar(50)')
    image = StringField(ddl='varchar(500)')
    created_at = FloatField(default=time.time)

class Blog(Model):
    __table__ = 'blogs'

    id = StringField(primary_key=True, default=next_id, ddl='varchar(50)')
    user_id = StringField(ddl='varchar(50)')
    user_name = StringField(ddl='varchar(50)')
    user_image = StringField(ddl='varchar(500)')
    name = StringField(ddl='varchar(50)')
    summary = StringField(ddl='varchar(200)')
    content = TextField()
    created_at = FloatField(default=time.time)

class Comment(Model):
    __table__ = 'comments'

    id = StringField(primary_key=True, default=next_id, ddl='varchar(50)')
    blog_id = StringField(ddl='varchar(50)')
    user_id = StringField(ddl='varchar(50)')
    user_name = StringField(ddl='varchar(50)')
    user_image = StringField(ddl='varchar(500)')
    content = TextField()
    created_at = FloatField(default=time.time)
```

在编写ORM时，给一个Field增加一个default参数可以让ORM自己填入缺省值，非常方便。并且，缺省值可以作为函数对象传入，在调用save()时自动计算。

例如，主键id的缺省值是函数next_id，创建时间created_at的缺省值是函数time.time，可以自动设置当前日期和时间。

日期和时间用float类型存储在数据库中，而不是datetime类型，这么做的好处是不必关心数据库的时区以及时区转换问题，排序非常简单，显示的时候，只需要做一个float到str的转换，也非常容易。

- 初始化数据库表

这里详细介绍了安装以及使用mysql的方法，[点击链接查看](https://www.jianshu.com/p/07a9826898c0)

我们在根目录下创建了一个名为schema.sql的文件，把SQL脚本放到MySQL命令行里执行：`mysql -u root -p < schema.sql`

- 编写代码访问数据库

```py
import orm
from models import User, Blog, Comment

def test():
    yield from orm.create_pool(user='www-data', password='www-data', database='awesome')

    u = User(name='Test', email='test@example.com', passwd='1234567890', image='about:blank')

    yield from u.save()

for x in test():
    pass
```

执行如上命令，查看是否正确向mysql里面插入数据。

# 编写Web框架

aiohttp已经是一个Web框架了，但是我们还需要封装一个，原因是从使用者的角度来说，aiohttp相对比较底层，我们编写框架处理重复工作。