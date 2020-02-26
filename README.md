# python-web

第一个python练手项目

# 搭建环境

- 安装python

安装python3，进入地址https://www.python.org/downloads/下载安装包，下载完成后点击安装，这里安装3.7版本。

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

`python app.py`运行项目，打开http://127.0.0.1:9000可以看到运行结果



