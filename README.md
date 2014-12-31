#作用力和小球
![最终效果](https://raw.githubusercontent.com/cyclegtx/force_ball/master/images/1.gif)  
<a href="http://cyclegtx.github.io/force_ball/" target="_blank">测试地址</a>  
上面的这种神奇的效果是使用D3.js实现的，d3的是代码库和教程请参见[这里](https://github.com/mbostock/d3)。d3是一个极其强大的数据图表库，尤其擅长操作svg，虽然被设计用来展示数据，但是其丰富的svg操作方法还有物理引擎可以被我们用来制作页面的展示。例子中的展示页面就是目前我们公司主要的产品，采用此种展示方式其效果自然是不言而喻。  

分析下效果，主要是一些节点与连线，再加上节点间相互的引力与斥力还有重力效果。节点与连接线好说，使用d3绘制出svg即可；相互作用力可以使用d3提供的force中相应的方法实现。关于d3详细教程请参见其官网，这里只介绍使用到的部分。  
首先将节点数据储存到数据文件中data.json  
```json
{
  "nodes":[
   	{"name":"买房邦","group":1,"class":"node","tourl":"http://www.iloushi.cn/m/maifangbang/","img":"...","imgx":"-54","imgy":"-54","w":"88","h":"88"},
    {"name":"楼盘码","group":2,"class":"node","tourl":"http://www.iloushi.cn/m/loupanma/","img":"...","imgx":"-28","imgy":"-28","w":"56","h":"56"},
    {"name":"i楼市","group":3,"class":"node","tourl":"http://www.iloushi.cn/m/iloushi/","img":"...","imgx":"-34","imgy":"-34","w":"68","h":"68"},
	{"name":"乐生活","group":4,"class":"node","tourl":"http://www.iloushi.cn/m/leshenghuo/","img":"...","imgx":"-48","imgy":"-48","w":"83","h":"83"},
    {"name":"卖房邦","group":5,"class":"node","tourl":"http://www.iloushi.cn/m/maifangbangv/","img":"...","imgx":"-35","imgy":"-37","w":"97","h":"97"},
    {"name":"多媒体楼书","group":6,"class":"node","tourl":"http://www.iloushi.cn/m/ebook/","img":"...","imgx":"-34","imgy":"-34","w":"63","h":"63"},
    {"name":"房产智能TV","group":7,"class":"node","tourl":"http://www.iloushi.cn/m/tv/","img":"...","imgx":"-36","imgy":"-36","w":"72","h":"72"},
    {"name":"微客智慧WIFI","group":7,"class":"node","tourl":"http://www.iloushi.cn/m/wifi/","img":"...","imgx":"-35","imgy":"-35","w":"71","h":"70"},
    {"group":1,"class":"fs drops","img":"...","imgx":"-7","imgy":"-7","w":"14","h":"14"},
    {"group":1,"class":"fs drops","img":"...","imgx":"-4","imgy":"-4","w":"9","h":"9"},
    {"group":1,"class":"fs drops","img":"...","imgx":"-5","imgy":"-5","w":"10","h":"10"},
    {"group":1,"class":"fs drops","img":"...","imgx":"-7","imgy":"-7","w":"14","h":"14"},
    {"group":1,"class":"fs drops","img":"...","imgy":"-3","w":"6","h":"6"}
  ],
  "links":[
    {"source":0,"target":2},
    {"source":1,"target":2},
    {"source":4,"target":2},
    {"source":6,"target":7},
    {"source":5,"target":7},
    {"source":3,"target":7},
    {"source":7,"target":1},
    {"source":8,"target":6},
    {"source":9,"target":3},
    {"source":4,"target":10},
    {"source":2,"target":11},
    {"source":3,"target":1}
  ]
}
```  
这个json的结构是d3的规范，其中分为nodes和links。nodes即效果中的圆球，包括大的彩色球和小的蓝色球，links即圆球间的连线。  
links数组中元素的属性比较简单，```source```和```target```，source即连线开始与哪个节点（小球），target是结束与哪个个节点。其值表示nodes数组中节点的索引，即第几个元素。  
nodes数组中元素属性比较复杂，下面解释下。  
```json
{
  "nodes":[
   	{
   		"name":"楼盘码",//自定义属性，节点名称
   		"group":2,//分组，暂时没明白做什么用的
   		"class":"node",//节点的class属性
   		"tourl":"http://www.iloushi.cn/m/loupanma/",//自定义属性，节点点击后跳转的url
   		"img":"...",//自定义属性，节点的图片地址,这里可以把图片base64加密后直接复制在这里
   		"imgx":"-28",//自定义属性，节点的坐标x,这里取节点宽度一半的负值是为了让图片居中
   		"imgy":"-28",//自定义属性，节点的坐标y,这里取节点高度一半的负值是为了让图片居中
   		"w":"56",//节点的宽度
   		"h":"56"//节点的高度
   	},
  ]
}
```     	
数据文件中定义了有几个节点几条线，哪两个节点相连，节点的图片。至于连接线的长短是根据节点之间的作用力强弱自动算出来的，所以如果连线过于集中导致最终小球都挤在一起的话，可以调整小球之间的连线。下面就是用过d3读出数据文件，然后根据数据绘制页面了。  
再读数据之前先初始化作用力，利用d3.layout.force中的方法设置作用力。  
```javascript
var wWidth = window.innerWidth,
wHeight = window.innerHeight;
//在body下新建svg元素并设置宽高为屏幕宽高
var	svg = d3.select("body").append("svg").attr("width", wWidth).attr("height", wHeight).attr("style", "margin: 0 auto;display: block;");
//初始化作用力并设置参数
var force = d3.layout.force()
			  .gravity(.05)//引力强度
			  .distance(100)
			  .charge( - 250)//节点的电荷数.(电荷数决定结点是互相排斥还是吸引)
			  .theta(.01)//电荷间互相作用的强度
			  .size([wWidth, wHeight]);//作用力的布局宽高

```  
设置好引力、引力和斥力后就要读取数据了。  
```javascript
//从data.json中读取节点数据
d3.json("data.json",function(error,data) {
	//绘制节点间连线
	var links = svg.selectAll(".link")
				   .data(data.links)//将data.json中的连线数据传入svg中
				   .enter()//进入每条连接线
				   .append("line")//将连接线绘制成line元素
				   .attr("class", "link");
	//绘制节点
	var	nodes = svg.selectAll(".node")
				   .data(data.nodes)//将data.json中的节点数据传入svg中
				   .enter()//进入每个节点
				   .append("g")//将节点绘制成g(分组)元素
				   .attr("class", "node")
				   .call(force.drag)//让节点相应作用力中的拖拽
				   .append("image")//在每个g元素下新建image元素
				   .attr("xlink:href",function(e) {return e.img})//将image元素的href属性设置为data.json中的img
				   .attr("tourl",function(e) {return e.tourl})//将image元素的tourl属性设置为data.json中的tourl
				   .attr("x",function(e) {return e.imgx})//将image元素的x属性设置为data.json中的imgx
				   .attr("y",function(e) {	return e.imgy})//将image元素的y属性设置为data.json中的imgy
				   .attr("width",function(e) {return e.w})//将image元素的width属性设置为data.json中的w
				   .attr("height",	function(e) {return e.h	});//将image元素的height属性设置为data.json中的h
});

```  
运行代码：  
![](https://raw.githubusercontent.com/cyclegtx/force_ball/master/images/1.jpg)    
由于还没有将数据填入force(作用力)，所有的球都挤在了一起。我们用```d3.json```从文件中读取数据,回调中的data即为文件中的数据。svg为我们新建的svg画布，d3的链式调用很神奇，比如在还没有将node添加进去之前就可以使用```svg.selectAll(".node")```选择节点了，看起来是先选择节点再建立节点,根据官方的api应该是调用```data()```后再调用```enter()```可以选择按照data新建元素的占位符，也就是说只是占位符还没有真的建立元素。可能运行时不一定是按顺序执行，其内部采用回调的方式执行，如果熟悉d3的朋友可以给我留言这是什么原理，十分感谢。还有一个神奇的地方是```attr()```方法,第一个参数是属性名称,第二个参数是回调函数，在回调中调用函数的参数e就可以获得这个节点的信息，即nodes数组中单个元素，这样就省去了遍历数组。所以整个代码没有循环也把数据中的数组一一添加到了svg画布中，整段代码显得十分整洁。在链式调用中可以使用```append()```添加svg标签，然后使用```attr()```设置属性，这样就可以在节点中任意添加元素了。  
上面只是把节点添加到svg中绘制出来，为了让节点可以移动我们还要把节点添加到force里面去，这样物理引擎与渲染引擎一一对应就可以联动起来了。   
```javascript  
//将节点传入作用力
force.nodes(data.nodes)
	 .links(data.links)
	 .start();
//让节点在每一帧都根据作用力的变化重新绘制
force.on("tick",function() {
	links.attr("x1",function(e) {return e.source.x})//将x1设置为前节点的x
		 .attr("y1",	function(e) {return e.source.y})//将y1设置为前节点的y
		 .attr("x2",	function(e) {return e.target.x})//将x2设置为后节点的x
		 .attr("y2",	function(e) {return e.target.y});//将y2设置为后节点的y
	nodes.attr("transform",function(e) {return "translate(" + e.x + "," + e.y + ")"});//更新节点的x,y
});
```  
其中的```tick```事件表示每一帧类似于```requestAnimationFrame```。这样整个页面就根据作用力的约束动起来了。 

时间仓促未作详细测试，如有任何bug请在Issues中提出。  

项目地址[github](https://github.com/cyclegtx/force_ball)  
如有问题或者建议请微博<a href="http://weibo.com/uedtianji" target="_blank">@UED天机</a>。我会及时回复  
更多教程请关注<a href="http://ued.sexy" target="_blank">ued.sexy</a>

