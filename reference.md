#ORM
----
##记录集（Recordsets）
----

8.0版本中使用。

模型(model)和记录(records)通过记录(recordsets)集记交互执行，记录集是某个模型排序好的记录集合。

**警告**:与名称暗示的相反，目前记录集(recordsets)很有可能包含重复记录。在将来可能会有所改变。

定义在模型上的方法将会在记录集上执行，他们的**self**参数是一个记录集：

	class AModel(models.Model):
		_name="a.model"
		def a_method(self):
			self.do_operation()
在一个记录集上迭代时会yield一个单条记录(单身singletons)的集合,和在python中对string进行yield时产生单个字符的集合一样：

	def do_operation(self):
		print self	# => a.model(1,3,4,5)
		for record in self:
			print record # => a.model(1),then a.model(2), then a.model(3)

###字段访问
记录集提供了一个“活动记录“接口：模型的字段可以直接在记录上读取和修改，但他仅限于singletons(单条的记录的recordsets)。为一个字段赋值时会触发数据库的更新操作:

	record.name
	record.company_id.name
	record.name="Bob"
在多条记录的recordsets上读取或赋值会引发错误。

访问关系字段时也返回recordset(Many2one,One2many,Many2many)，如果没有值则返回空。

**警告**:为每一个字段赋值时都会触发更新数据库的操作，当需要更改多个字段值或一次修改多条记录(都设成一样的值)时应该使用**write**。
	
	# 3 * len(records)次数据库更新
	for record in records:
		record.a = 1
		record.b = 2
		record.c = 3
	#len(records)次数据库更新
	for record in records:
		record.write({'a':1,'b':2,'c':3})
	#1次数据库更新
	records.write({'a':1,'b':2,'c':3})

### 集合操作
Recordsets是不可变的，但是可以对同一种模型的集合进行操作，返回结果也是一个recordsets。集合操作不保证记录顺序。

* record in set	 	判断**record(必须是只有一条记录的recordset)**是否在**set**中存在
* record not in set 	是上面**in**的反操作
* set1 | set2			返回两个集合中所有的元素
* set1 & set2			返回两个集合中相同的元素
* set1 - set2 			返回**set1**中排除掉 **set2**中记录的元素

### 其他记录集操作
Recordsets are iterable so the usual Python tools are available for transformation (map(), sorted(), ifilter(), ...) however these return either a list or an iterator, removing the ability to call methods on their result, or to use set operations.

因此**recordsets**提供下面的操作以返回recordsets本身(如果可能的话):

* filtered()
	
返回满足谓词函数中条件的记录集。谓词函数也可以是一个布尔类型的字段(作为字符串传入)。
	
	#只保留是自己公司的记录
	records.filtered(lambda r: r.company_id ==user.company_id )
	#只保留是公司的记录
	records.filtered("patner_id.is_company")		
* sorted()

根据函数提供的键值进行排序，返回结果集。如果没有提供键值，则使用模型默认的排序方式。
	
	#根据名称排序
	records.sorted(key=lambda r:r.name)

* mapped()

对recordsets中的每一条记录执行指定的函数，如果结果是recordsets的返回recordsets。

	#返回一条两上字段相同的list
	records.mapped(lambda r: r.field1 + r.field2)

函数可以是提供某个字段值的字符串:
	
	#返回名称的列表
	records.mapped('name')
	#返回一个partners的recordset
	record.mapped('partner_id')
	#returns the union of all partner banks,with duplicates removed
	#返回recordsets中所有记录的相关银行记录集，并且去掉重复的记录.
	record.mapped('partner_id.bank_ids')

##Environment(环境)
---

**Enviroment**保存了ORM需要的各种上下文数据：数据库游标(用来查询数据库)，当前用户（用来检查权限）和当前的上下文(context，保存随意的元数据)。环境同时也保存缓存。

所有的**recordsets**都有一个**Enviroment**，它是不可变的，可以通过env属性访问到当前用户，游标或者是上下文:

	records.env
	records.env.user
	records.env.cr
当在其他的recordset中创建recordset时，环境可以被继承。环境可以在其他recordset中获取一个空的recordset并查询这个模型：

	self.env['res.partner']
	self.env['res.partner'].search([['is_company','=',True],['customer','=',True]])

###改变环境
来自记录录的环境可以定制。使用改变后的环境将返回一个记录集的新版本。

**sudo()**
根据提供的用户集创建一个新的环境，如果没有指定用户则默认使用管理员(绕过安全上下文中的访问权限和规则)，使用新的环境时将回recordset的一个拷贝:

	#作为管理员创建partner对象
	env['res.partner'].sudo().create({'name':'A partner'})
	#列出public用户能够看到的partner
	public = env.ref['base.public_user']
	env['res.partner'].sudo(public).search([])

**with_context()**
* 可以采取一个位置参数,取代当前环境的上下文
* 通过关键字可以任意数量的参数,它被添加到当前环境的上下文或上下文设置在步骤1

	#查找partner,根据指定的时区创建记录
	env['res.partner'].with_context(tz=a_tz).find_or_create(email_address)

**with_env()**
	
完全取代再有环境。


<br/>
<br/>
<br/>

##常见的ORM方法
---

**search()**

使用查询domain，返回符合条件的recordset。也可以返回部分(**offset**,**limit**参数)的recordset，可排序：

	#查询当前模型
	self.search([('is_company','=',True),('customer','=',True)])
	#限制一条记录
	self.search([('is_company','=',True)],limit=1).name

注意点：如果仅仅需要查询记录条数或只想知道是否有符合条件的记录，使用**search_count()**更适合

**create()**

使用包含字段名称和值的字典，成功后返回包含被创建记录的recordset：

	self.created({'name':'New name'})

**write()**

使用包含字段名称和值的字典,将它们写入到当前recordset中的所有记录。不返回任何值：
	
	self.write({'name':'Newer name'})

**browse()**

使用数据库Id或者Id的列表，返回指定Id的recordset，在外部系统根据Id访问Odoo或者在老的API中调用方法时比较有用。
	
	self.browse([7,18,12])

**exists()**

返回一个只包含在数据中存在的记录的recordset。可以用来检查一条记录是否仍然存在。

	if not record.exists():
		raise Exception('The record has been deleted')		
或者当一条记录被移除后调用：

	records.may_remove_some()
	records = records.exists()		

**ref()**

环境的方法，根据提供的额外Id返回记录。

	env.ref('base.group_public')
	#res.group(2)
	
**ensure_one()**

检查当前的recordset是不是单条的，否则的抛出错误

	records.ensure_one()
	#和下面的语句一样，但更简洁
	assert len(records)==1,"Expected singleton"	
## 创建模型
----

模型的字段作为模型的属性进行定义：

	from openerp import models,fields
	class AModel(models.Model):
		_name = "a.model.name"
		
		field1 = fields.Char()
		
**注意点**:这就意味着你不能定义一个拥有相同名称的方法和字段，它们会冲突。

默认情况下，字段的标签是大写版本的字段名，也可以由**string**参数来重写：

	field2 = fields.Integer(string="an other field")

更多的字段类型会在字段引用章节介绍。

字段的默认值通过参数定义，可以是一个值 ：

	a_field = fields.Char(default='a value')
也可以为一个函数:
	
	a_field = fields.Char(default=compute_default_value)
	
	def compute_default_value(self):
		return self.get_value()

###计算字段

字段可以通过**compute**参数计算（而不是直接从数据库读取）。**必须将计算的结果赋值给字段**。如果计算依赖其他字段，则应使用**depends()**指定:

	from openerp import api
	total = fields.Float(compute='_compute_total')
	
	@api.depends('value','tax')
	def _compute_total(self):
		for record in self:
			record.total = record.value + record.value * record.tax
	
* 依赖可以使用.路径的方式使用子字段

	@api.depends('line_ids.value')
	def _compute_total(self):
		for record in self:
			record.total = sum(line.value for line in record.line_ids)
* 计算字段默认情况下不保存，当请求的时候才会计算并返回。设置**store=True**参数会将它们保存到数据库并且自动激活查询功能
* 探索计算字段也可以通过设置**search**参数，它的值是一个能够返回domain的方法的名称:

	upper_name = fields.Char(compute='_compute_upper',search='_search_upper')
	def _search_upper(self,operator,value):
		if operator == 'like':
			operator = 'ilike'
		return [('name',operator,value)]

* 允许为计算字段设置值，使用**inverse**参数。它是一个和计算相对应的函数的名称，设置对应的字段值：

	document = fields.Char(compute='_get_document',inverse='_set_document')
	
	def _get_document(self):
		for record in self:
			with open(record.get_document_path()) as f:
				record.document = f.read()
	
	def _set_document(self):
		for record in self:
			if not record.document : continue
			with open(record.get_document_path()) as f:
				f.write(record.document)
				
* 多个计算字段可以使用一个方法同时计算结果，只需要使用同样的方法并对所有涉及的字段赋值即可：

	discount_value = fields.Float(compute='_apply_discount')
	total = fields.Float(compute='_apply_discount')
	
	@depends('value','discount')
	def _apply_discount(self):
		for record in self:
			discount = self.value * self.discount
			self.discount_value = discount
			self.total = self.value - discount
			
### 依赖字段 

计算字段的一种特殊情况就是依赖字段，它为当前对象提供一个子字段的值。它们通过**related**参数设置，像普通的计算字段一样能够被存储：

	nickname = fields.Char(related='user_id.partner_id.name',store=True)

### onchange:动态更新UI

当用户在表单中选择一个字段值(还没有保存到form中)时，可以用来自动更新基于它的其他字段，如当改变税率或添加一条invoice line的时候能自动更新最终总额。

* 计算字段在后端自动检查和计算，不需要**onchange**
* 对于非计算字段，**onchange()**装饰器被用来提供一个新的字段值

	@api.onchange('field1','field2')
	def check_change(self):
		if self.field1 < self.field2:
			self.field3 = True

改变会在函数中进行然后将值发送到前端程序，用户就可以看见了。

* 计算字段和新的onchanges接口都不需要在页面上设置就可以被前端调用
* 可以通过在视图字段中设置 **on_change="0"** 能阻止指定字段的 onchange,当字段在页面上被编辑的时候不会导致任何页面更新，即使存在功能功能字段或者明确指定onchange依赖于那个字段 。

	<field name='name' on_change='0'/>

**注意点**: **onchange**方法工作在虚记录上，这些记录并没有被写到数据库中，只是用来获取哪个值应该被发送到前端。


###  低层的SQL	

环境的属性**cr**是数据库事务的游标，允许直接招待SQL语句，也可以做一些很难通过ORM实现的查询，或者是考虑到性能问题:

	self.env.cr.execute("some_sql",param1,param2,param3)

因为模型也使用同样的游标，环境中也包含了大量的缓存，如果需要执行更新数据的SQL语句，则必须将缓存设为无效，否则进一步使用模型的时候会导致混乱。

当在SQL中使用**create**,**update**或者**delete**时必须先清理缓存，但**select**操作没有关系。清理缓存可以使用环境的**invalidate_all()**方法。


## 旧API的兼容性




#QWeb
---

QWeb是Odoo中使用的主要模板引擎。它是一个模板引擎，主要用来生成HTML片段或页面。

模板指令使用特殊的XML属性，以**t-**开头，例如 **t-if** 用作条件判断，其他的元素或属性会直接解析。

为避免元素渲染，可以使用占位符**<t>**，它不会输出任何内容包括自己。

	<t t-if="condition">
		<p>Test</p>
	</t>
	
如果条件满足，输出的结果是:
	
	<p>Test</p>
	
如果条件满足，但换成下列内容，

	<div t-if='condition'>
		<p>Test</p>
	</div>

输出的结果如下:
	
	<div>
		<p>Test</p>
	</div>	

## 数据输出

QWeb有一个重要的输出指令 ** esc ** ,它能够自动转换内容中的HTML并能有效阻止XSS攻击，这在显示由用户提供的内容时非常重要。**esc** 使用表达式，会对其进行评估并输出。

	<p><t t-esc="value"/></p>	
如果此处的Value为42，则会被渲染为:
	
	<p>42</p>
还有另外一个输出指定 **raw**,功能与**esc**相同，只是在输出前不会对内容进行html转换。它常被用来显示结构化的标记或者确认无害的用户提供的内容。(译者注：在website的模板中使用较多)。

## 条件
Qwweb有一个条件指令 ** if **，解析作为属性给出的条件表达式。

	<div>
		<t t-if="condition">
			<p>ok</p>
		</t>
	</div>
如果条件为真，输出如下：

	<div>
		<p>ok</p>
	</div>
但如果条件为假，**if** 包含的内容则不会生成:
	<div></div>
对条件语句来讲，<t>不是必须的，上面的表达式也可以这样写,生成的结果与上面完全相同：
	
	<div>
		<p t-if="condition">ok</p>
	</div>
	
## 循环
QWeb有一个迭代指令 **foreach**，它使用一个表达式返回要迭代的集合，使用参数 **t-as**表示此迭代中的当前对象：

	<t t-foreach="[1,2,3]" t-as="i">
		<p><t t-esc="i"/></p>
	</t>
上述代码会返回下面的结果:
	
		<p>1</p>
		<p>2</p>
		<p>3</p>
像**conditions**指令一样，**foreach**指令可以直接在元素上使用，下面的代码和上面的效果完全相同：
	
	<p t-foreach="[1,2,3]" t-as="i">
		<t t-esc="i"/>
	</p>
	
**foreach** 能够迭代Array(当前项就是当前值)、mapping(当前项为当前的KEY)或者一个整数(相当于产生一个从0到指定整数的数组).

除了**t-as**属性，**foreach** 还提供了一些功能。

* 注意点
	原文: $as will be replaced by the name passed to t-as
	$as 将来会被**t-as**替代

* $as_all 该对象被迭代(the object being iterated over)
* $as_value the current iteration value ,identical to $as for lists and integers ,but for mapping it provideds the value(where **$as** provides the key)
* $as_index	当前的迭代的索引（第一个元素索引为0）
* $as_size 	集合的大小（如果它能访问到）
* $as_first 	判断是否是否为第一个元素（相当于**$as_index==0**）
* $as_last 	当前元素是否为最后一个元素(相当于**$as_index + 1 == $as_size **)，当然前提是此迭代对象是可访问的
* **$as_parity** **"even"** 或者 **"odd"**，表示当前元素所在是奇数行还是偶数行
* $as_even 	当前元素是否为偶数行
* $as_odd		当前元素是否为奇数行

##属性
-----
QWeb可以在运行中计算属性，并将计算结果设置到输出的结点上。这种指令存在三种形式：

#### t-att-$name
会创建一个名称为$name的属性，会对属性的值进行计算，其结果作为属性的值。如下：
	
	<div t-att-a="42"/>

产生如下结果:

	<div a="42"></div>
	
#### t-attf-$name
像上面的一样，但参数会使用格式化字符串来替换简单的表达式，经常用来混合静态文字和非静态文字（例如:classes）:

	<t t-foreach="[1,2,3]" t-as="item">
		<li t-attf-class="row {{ item_parity }}">
			<t t-esc="item"/>
		</li>
	</t>
	
生成结果如下：

	<li class="row even">1</li>
	<li class="row odd">2</li>
	<li class="row even">3</li>

#### t-att=mapping
如果参数是一个mapping，每个键值对(key,value)都会生成一个新的属性和值:

	<div t-att="{'a':1,'b':2}"/>
生成结果如下:
	
	<div a="1" b="2"></div>

#### t-att=pair
如果参数是一对(拥有两个元素的元组或数据)，第一项为属性的名称，第二项为属性的值:

	<div t-att="['a','b']"/>
生成结果如下：
	<div a="b"></div>
	
## 设置变量
-----
QWeb允许在模板中创建变量，可以用来多次计算，给一段数据一个清晰的名称......

通过**set**指令，指定需要创建的变量名称。有两种设置值的方式：

* **t-value**属性包含一个表达式，其返回值就是要设置的变量值
	
	<t t-set="foo" t-value="2+1"/>
	<t t-esc="foo"/>
结果为出 ** 3 **

如果没有**t-value**属性，则结点的里面的内容会被渲染并作为变量的值，如下：

	<t t-set="foo">
		<li>ok</li>
	</t>
	<t t-esc="foo"/>
输出的内容如下：
	\&lt;li\&gt;ok\&lt;/li\&gt;(因为我们使用了**t-esc**,内容会被编码)

* 注意点		这种用法在raw指令中用的更多.

## 调用子模板
----
QWeb模板可以用于顶层渲染，也可以通过**t-call**指令在其他模板中使用：

	<t t-call="other-template"/>
这句代码在主模板的执行上下文中调用了名称为**other-template**的模板，如果**other-template**的模板定义如下：
	
	<p><t t-value="var"/></p>
上面的代码执行结果为 <p/>,但如果代码改为下面的形式：

	<t t-set="var" t-value="1"/>
	<t t-call="other-template"/>
	<!-- 变量var在此存在，可以访问 -->
则会生成&lt;p&gt;1&lt;/p&gt;。

然而这种方式会为让变量在作用范围外可见。我们也可以**call**指令的内容中，且在调用子模板之前设定变量，但只会改变局部的上下文。代码如下所示：

	<t t-call="other-template">
		<t t-set="var" t-value="1"/>
	</t>
	<!-- 变量var此处不存在，不可访问-->

**call** 指令的内容中可以设置任意的内容，而不仅仅是调用**t-set**。还可以将要调用的模板作为一个神奇的变量 **0**:
	<div>
		This template was called with content:
		<t t-raw="0"/>
	</div>
调用它的代码如下：

	<t t-call="other-template">
		<em>content</em>
	</t>
输出的结果如下：

	<div>
		This template was called with content:
		<em>content</em>
	</div>


# Python

## 特有的指令

** asset bundles ** (不知道怎么翻译)

** "智能记录"字段格式化 **

** t-field **指令只能在访问智能记录（browse方法的返回值）时使用。 它能够根据字段的类型自动格式化，并且可以和网站中的富文本编辑器结合。

** t-field-options ** 用来自定义字段，widget是最常用的选项，其他选项是 ** field- ** 或者 **widget-dependent**。

## 助手(Helpers)

** 基于请求的 ** (Request-based,这样翻不知道对不对)

Python直接使用QWeb的情况大部分是在controllers中，这里的模板作为视图存储在数据库中，通过openerp.htt.HttpRequest.render()方法调用：

	response = http.request.render("my-template",{
		'context_value':42
	})
这里在controllers中创建并返回了一个Response对象（也有可能进一步操作）。

** 基于视图的 **

比上面的助手方法更深一层的是ir.ui.view的render方法。

** render(cr,uid,id[,values][,engine='ir.qweb'][,context]) **

通过数据库中的Id或额外Id渲染QWeb视图或模板。模板会从ir.ui.view记录中自动加载。在渲染的下下文中设置多个默认值：
	
	* request 		当前的WebRequest
	* debug			当前的请求是否处于调试模式
	* quote_plus 	URL编码通用方法
	* json 			相应的标准库模块
	* time			相应的标准库模块
	* datetime 	相应的标准库模块
	* relativedelta 参考module
	* keep_query	keep_query助手方法

** 参数 **

	* values 上下文中的值被传递到QWeb进行渲染
	* engine 字符类型，被用来做渲染工作的odoo实体(model),可以被扩展也可以自定义本地的(可以通过改造ir.qweb的方式创建一个新的qweb)
	
## API
直接使用ir.qweb是可以的（也可以扩展它或者继承它）

**class** openerp.addons.base.ir.ir_qweb.**QWeb(pool,cr)**

这是基础的QWeb渲染引擎

* 自定义 f-field 渲染，创建ir.qweb.field子类和新的叫ir.qweb.field.widget的模型
* 或者，重写get_converter_for()方法并返回一个任意模型作为 field 转换器

(译者注：英文水平有限，这两条翻译的非常不准确，故附上原文)

	* to customize t-field rendering, subclass ir.qweb.field and create new models called ir.qweb.field.widget
	* alternatively, override get_converter_for() and return an arbitrary model to use as field converter

需要注意的是，如果你需要扩展或改建可能与子系统不相容的功能时，你应该通过继承ir.qweb创建一个本地对象。

** add_template(qwebcontext,name,node) **

添加一个需要被解析的模板到上下文中。用来预处理模板。

** get_converter_for(field_type) **

返回一个用来渲染**t-field**的 Model。默认情况下，会试着获取名为**ir.qweb.field.field_type**的Model, 依附于**ir.qweb.field**。

参数:	field_type(字符型) 字段的类型或widget，用来渲染。

** get_template(name,qwebcontext) **

尝试着根据名称取得模板，无论从上下文的模板缓存或加载一个与上下文相差的加载器加载。

** get_widget_for (widget) **
	
返回一个被用来渲染 t-esc 的Model。

参数：widget(字符型) 小部件的名称或者是None

** load_document(document,res_id,qwebcontext) **

加载 XML文档并安装包含在引擎中的所有模板。

** prefixed_methods(prefix) **

提取所有前缀为指定参数的方法，并返回(t-name,method)的映射，其中t-name名字前缀的方法名称中删除和下划线转换为破折号。

** render(cr,uid,id_or_xml_id,qwebcontext=None,loader=None,context=None) **

渲染指定名称的模板

参数：

* qwebcontext (字典类或者QWebContext的实例) 渲染模板的上下文
* loader		如果qwebcontext是字典类型，loader将设置到上下文实例中来渲染

** render_tag_call_assets(element,template_attribute,generated_attributes,qwebcontext) **

 特殊的t-call标记可以用来合并或缩小javascript和css。
 
** render_tag_field(element,template_attributes,generated_attributes,qwebcontext) **
	
	如: <span t-record="browse_record(res.partner,1)" t-field="phone">+1 555 555 8-69 </span>

**class openerp.addons.base.ir.ir_qweb.FieldConverter(pool,cr)**:
	
用来将一个t-field字段转换为一个html的输出。

**to_html()**是从QWeb转换的入口点。包含以下内容：

* 使用**record_to_html()**将record值转化为html
* 在根结点上生成元数据属性（data-oe-）
* 通过**render_element()**产生根结点本身

**attributes(cr,uid,field_name,record,options,source_element,g_att,t_att,qweb_context,context=None)**

产生元数据属性(在根结点上以 data-oe- 为前缀的属性)。属性值由父一级进行escape。
默认的属性如下：

* model			record模型的名称
* id 			字段所属record的Id
* field 		被转换字段的名称
* type			逻辑字段类型(widget, may not match the field’s type, may not be any Field subclass name,不知道如何翻译)
* translate	一个布尔类型的标志位（0或1），表示此字段是否能够翻译
* expression	原表达式

**record_to_html(cr,uid,field_name,record,options=None,context=None)**

转换指定智能记录的某个字段为HTML

**render_element(cr,uid,source_element,t_att,g_att,qweb_context,content)**
	
最终渲染的钩子，默认情况下直接调用ir.qweb的**render_element**

**to_html(cr,uid,field_name,record,options,source_element,t_att,g_att,qweb_context,context=None)**

把 **t-field**转换为HTML输出。**t-field**可能被**t-field-options**扩展，这个属性的值是一个JSON的字典类型，存放一些配置值。（译者注：{'widget':'image'}）.一个默认的属性就是 **widget**，可以覆原来的字段类型。

**user_lang(cr,uid,context)**

根据context中存储的语言代码获取res.lang对象。如果上下文中没有语言代码或语言代码无效则默认为en_US.
返回值类型为 res.lang browse_record。

**value_to_html(cr,uid,value,field,options=None,context=None)**

将一个单独的值转换为html输出。


## Javascript 
-----
**特定指令**

**定义模板**

**t-name**指令只能放在模板文件的最顶层(直接放在文档的根据下)。

	<templates>
		<t t-name='template-name'>
			<!-- template code -->
		</t>
	</templates>
不需要其他的参数，但可以使用**<t>**元素或其他的。使用**<t>**元素时，**<t>**应该包含一个子元素。
模板名称可以是任意字符串，但多个相关模板(例如sub-templates)一般会按惯例使用圆点分隔名称来显示层次关系。

**模板的继承**

模板的继承通常用来修改已经存在的模板，例如添加一些内容到其他模块中的模板上。

模板的继承使用**t-extends**指令，以被继承的模板名称为参数。
The alteration is then performed with any number of t-jquery sub-directives:（不会翻译）

	<t t-extends="base.template">
		<t t-jquery="ul" t-operation="append">
			<li>new element</li>
		</t>
	</t>
**t-jquery**指令使用**css**选择器。选择器会在被扩展的模板上筛选出相应的元素并招待**t-operation**操作。**t-operation**操作的参数可以下面的值：

* append 		结点中的内容会被放到筛选元素内部的最后
* prepend 		结点中的内容前置到筛选元素中
* before		将内容放到筛选元素的前面
* after 		将内容放到筛选元素的后面
* inner 		将内容替换筛选元素子元素
* replace 		将内容替换筛选元素本身

**没有操作**

如果没有指定t-operation,模板的身体是解释为javascript代码和执行上下文节点。

**警告**

这种操作要比其他操作更强大，也更难调试和维护，建议尽量避免使用。

#### 调试
QWeb的javascript实现提供了几个调试的钩子:

**t-log** 

使用一个表达式参数，在渲染过程中计算表达式并使用console.log记录生成的结果。

**t-debug** 

在渲染模板的时候触发一个调试点

**t-js** 

结点为的内容是javascript代码，当模板渲染时会被执行。Takes a context parameter, which is the name under which the rendering context will be available in the t-js‘s body。（不会翻译）

#### 助手类(Helpers)
**openerp.qweb**

一个QWeb2.Engine()的实例，且已加载了所有模块定义的模板文件，引用了标准助手类**_(underscore)**、**_t(翻译函数)**和**JSON**。

openerp.web.render可以很轻松的被用来渲染基本模块的模板。

#### API	
**class QWeb2.Engine()**

是QWEB的渲染者，包含了绝大部分的QWeb逻辑（加载、解析、编译和渲染模板)。OpenERP为用户实例化一个对象，并将其赋值给instance.web.qweb。它同时也加载了所有模块下的模板文件到QWeb实例。**QWeb2.Engine()**也作为模板的命名空间。

**QWeb2.Engine.render(template[,context]])**

渲染一个之前加载好的模板为字符串，在模板的渲染过程中从**context**查找变量。

**参数**

* template 	字符型，要渲染的模板名称
* context		模板渲染的上下文

**返回值**		字符串


引擎也暴露了一个add_template的方法，在一些场景下会比较有用。（e.g. if you need a separate template namespace with, in OpenERP Web, Kanban views get their own QWeb2.Engine() instance so their templates don’t collide with more general “module” templates)。

**QWeb2.Engine.add_template(templates)**

在QWeb实例中加载一个模板文件（模板的集合，一个模板文件中可以包含多个模板）。参数**templates**可以为下面的几种情况：

* XML字符串，QWeb会尝试将它转化为一个XML文档，然后再加载
* URL地址，QWeb会尝试去下载URL地址的内容，然后再加载 
* 文档或结点,QWeb将遍历文档的第一级(根)提供的子节点和加载任何命名模板或模板覆盖。

QWeb2.Engine也暴露了一些可用于自定义的属性：

* QWeb2.Engine.prefix 指定指令的前缀，默认为**t**
* QWeb2.Engine.debug	 布尔标志位，指定引擎是否处于调试模式。一般情况下，QWeb会拦截渲染过程中发生的所有错误。在调试模式
* Qweb2.Engine.jQuery The jQuery instance used during template inheritance processing. Defaults to window.jQuery.
* QWeb2.Engine.preprocess_node 一个函数，在编译dom节点到模板代码执行。在OpenERP WEB，这个经常被用来自动翻译文本内容和一些模板中的属性。默认为null。


		
