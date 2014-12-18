# 创建一个网站

**警告**

* 这个向导需要基本的Python知识
* 此向导基于安装好的odoo
* 对于生产部署，请参考专门的部署指南

# 创建一个简单模块

Odoo中的功能都是通过新的模块来完成。

模块通过添加新的功能或修改已存在的模块来达到自定义odoo的功能。

首先让我们创建一个目录，里面可以包含一个或多个相关或不相关的功能模块。

  $ mkdir my-modules

然后创建一个模块:

  $ mkdir my-modules/academy

一个模块是一个合法的python包，所以需要添加一个空的**__init__.py**文件。还需要添加一个清单的文件，里面包含一个python字典类，描述此模块的元数据。academy/__openerp__.py:

  {
    'name':'Academy',
    'description':"""
    """,
    'depends':['base'],
  }

# 一个示例模块

我们已经拥有一个可安装的完整模块，尽管它几乎什么也没做，但仍然可以安装它。

* 启动odoo服务

  $ ./odoo.py --addons-path addons,my-modules

* 访问localhost:8069
* 创建一个包含demo数据的数据库
* 到设置－》模块－》已安装模块
* 在右上角的搜索框中移除搜索条件，再输入academy
* 点击academy的安装按钮

**另请参阅** : 在开发的时候通常会通过odoo提供的脚手架程序创建模块，而不是直接通过手动的方式。


# To the browser

controllers 拦截浏览器请求并返回结果。添加一个简单的controller并import它（这样odoo才能找到它）。在academy/__init__.py文件中：
  
  from . import controllers

academy/controllers.py：

  # -*- coding: utf-8 -*-
  from openerp import http
  
  class Academy(http.Controller):
    @http.route('/academy/',auth='public')
    def index(self):
      return "Hello,world!"
      
重启服务：

  $ ./odoo.py --addons-path addons,my-modules
  
然后在浏览器中打开http://localhost:8069/academy/，你应该能看到"Hello world!“的内容。

# 模板


