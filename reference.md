#QWeb

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

** class openerp.addons.base.ir.ir_qweb.FieldConverter(pool,cr) **:
	
用来将一个t-field字段转换为一个html的输出。
	
 	