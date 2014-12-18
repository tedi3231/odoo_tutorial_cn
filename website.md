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

一个模块是一个合法的python包，所以需要添加一个空的**\_\_init\_\_.py**文件。还需要添加一个清单的文件，里面包含一个python字典类，描述此模块的元数据。

academy/\_\_openerp\_\_.py:

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

controllers 拦截浏览器请求并返回结果。添加一个简单的controller并import它（这样odoo才能找到它）。

academy/__init__.py：
  
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

直接在python中生成html不是一件很爽的事情，常用的解决方案是使用模板，一种使用点位符和显示逻辑的伪文档。odoo允许使用任何一种python模板系统，但是默认使用qweb模板系统以方便与其他odoo特性集成。让我们创建第一个模板，然后在清单文件中指定，并在controllers中使用。

academy/\_\_openerp\_\_.py

      	""",
    	# Which modules must be installed for this one to work
    	'depends': ['base'],
    	# data files which are always installed
    	'data': [
       		'templates.xml',
    	],
    }

academy/controllers.py

	class Academy(http.Controller):
    	@http.route('/academy/',auth='public')
    	def index(self):
      		return http.request.render("academy.index",{
        		'teachers':['Diana Padailla','Jody Caroll','Lester Vaughn'],
      	})

academy/templates.xml

	<openerp>
	<data>
    <template id="index">
      <title>Academy</title>
      <t t-foreach="teachers" t-as="teacher">
          <p><t t-esc="teacher"/></p>
      </t>
    </template>
    </data>
    </openerp>

模板中通过**t-foreach**迭代所有的老师数据(由模板上下文提供)，并在段落中显示老师名称。重启服务并更新模板就能看到要显示的老师数据。

# 在odoo中保存数据

odoo模型对应于数据库中的表。在上一节我们只是简单的显示了来自于字符串的数据，它们并不能保存和编辑，现在让我们将数据移动到数据库中。

## 定义模型

首先定义模型文件并在\_\_init\_\_.py中import。

    from . import controllers
    from . import models
    
academy/models.py

    from openerp import fields
    from openerp import models
    
    class Teachers(models.Model):
        _name = 'academy.teachers'
    
        name = fields.Char()

然后设置访问控制和并放到模块清单中。academy/\_\_openerp\_\_.py

      'depends': ['base'],
    # data files which are always installed
    'data': [
        'ir.model.access.csv',
        'templates.xml',
    ],
  }

academy/ir.model.access.csv

    id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
    access_academy_teachers,access_academy_teachers,model_academy_teachers,,1,0,0,0
这个设置给了所有用户读的权限。

**注意点**:数据文件（xml或csv）文件必须添加到模块清单文件中，python文件没有必要添加但必须在\_\_init\_\_.py中导入。
**警告**  :管理员能够直接访问所有的模型，不管是否被分配到对这个模型的访问权限

## demo数据

第二步，为了能够更方便的测试系统，我们添加一些演示数据。这个可以通过添加demo文件到模块清单中来实现：
academy/\_\_openerp\_\_.py
  
          'ir.model.access.csv',
          'templates.xml',
      ],
      # data files which are only installed in "demonstration mode"
      'demo': [
          'demo.xml',
      ],
    }

 academy/demo.xml
 
    <openerp>
    <data>
      <record id="padilla" model="academy.teachers">
        <field name="name">Diana Padilla</field>
      </record>
      <record id="carroll" model="academy.teachers">
        <field name="name">Jody Carroll</field>
      </record>
      <record id="vaughn" model="academy.teachers">
        <field name="name">Lester Vaughn</field>
      </record>
    </data>
    </openerp>

**注意点**：数据文件在演示模式和非演示模式都会安装。但演示数据只有在启用演示模式时才会被导入系统中，这些数据通常被用来测试或演示。而非演示数据则被用来做系统的初始化。在这个例子中我们使用演示数据，因为一个真实的用户很有可能希望导入他们自己的老师列表来作测试用。

## 访问数据

最后一步是修改模糊和模板来访问我们的演示数据：

1 能过从数据库中获取数据而不使用一个静态的字符列表 
2 因为**search()**方法返回满足条件记录的集合，需要修改模板使它能够打印老师的名称

academy/controllers.py

	class Academy(http.Controller):
    @http.route('/academy/', auth='public')
    def index(self):
        Teachers = http.request.env['academy.teachers']
        return http.request.render('academy.index', {
            'teachers': Teachers.search([]),
        })
academy/templates.xml

	<template id="index">
    	<title>Academy</title>
    	<t t-foreach="teachers" t-as="teacher">
     		<p>
     			<t t-esc="teacher.id"/> 
     			<t t-esc="teacher.name"/>
     		</p>
    	</t>
    </template>

重启服务并更新模块（主要为了更新模块清单、模板文件和加载DEMO数据），然后访问http://localhost:8069/academy。页面会和以前有点不同，在名称前面会有编号（老师在数据库中记录的编号）。

## 网站支持
odoo 提供了一个模块专门用于构建网站。

到目前为止，我们都是直接使用controllers，但在odoo8中可以通过website模块还有其他一些服务（如默认样式、主题）进入更深入的融合。

1. 首先，将website作为academy模块的依赖
2. 然后在controller上加上website=True标记，这样会在请求对象上添加几个新的变量，同时可以让当前模块的模板使用website的布局。
3. 在模板中使用website布局

academy/\_\_openerp\_\_.py	

	'depends':['website'],
	
academy/controllers.py

	from openerp import http

	class Academy(http.Controller):
    	@http.route('/academy/', auth='public', website=True)
    	def index(self):
       		Teachers = http.request.env['academy.teachers']
        	return http.request.render('academy.index', {
        		'teachers':Teachers
        	})

academy/templates.xml

	<openerp><data>
	<template id="index">
    <t t-call="website.layout">
      <t t-set="title">Academy</t>
      <div class="oe_structure">
        <div class="container">
          <t t-foreach="teachers" t-as="teacher">
            <p><t t-esc="teacher.id"/> <t t-esc="teacher.name"/></p>
          </t>
        </div>
      </div>
    </t>
    </template>
    </data></openerp>
    
重启服务并更新模块，访问http://localhost:8069/academy页面，我们会看到一个更漂亮的、有着标题、菜单等元素的页面。

website的布局支持编辑工具：点击右上角的“登陆”，然后使用admin admin登陆。你现在处于odoo的‘proper’：管理员界面。现在点击网站的菜单项(左上角)，我们作为管理员回到了网站，现在可以访问由网站提供的一些高级编辑功能。

* 模板代码编辑工具（Customize->HTML Editor），在这里可以查看并编辑当前页面使用的所有模板
* 点击左上角的编辑按钮将当前页面转入编辑状态，我们可以使用区块和富文本
* 还有很多其他的特性，如移动端预览或SEO


## URLS和ROUTING

controller方法通过**route()**包装器与路由关联，在route()包装器上有表示路径的字符串和一些用来自定义行为和安全的属性。

我们已经看过只能完全匹配的静态路径，但是路由字符串也可以使用转换参数，它可以用来部分匹配URL，将其中部分解析为本地变量。如下例我们创建一个部分匹配的URL：

	@http.route('/academy/<name>/', auth='public', website=True)
    def teacher(self, name):
        return '<h1>{}</h1>'.format(name)
