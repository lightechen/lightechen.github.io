 

前段时间在网易的笔试题中遇到了用原生JS实现一个promise对象，之前只对promise有过概念上的了解，但从来没去考虑过如何实现，所以没有答上来，后来去找了一些promise对象的一些实现思路，又看了下jQuery中对defer对象的实现，感觉收获挺大，在这里做个简单总结。

#### 一.关于Promise规范
在了解jQuery如何实现defer对象之前，我们首先要对Promise规范有所了解。Promise作为CommonJS的规范之一，主要为了帮助前端开发人员更好地控制代码的流程，避免回调函数的多层嵌套。
##### 为什么使用Promise规范？
我们在异步请求中经常会遇到这样的情况，存在两个异步请求，其中第二个请求依赖于第一个请求获取的数据，以下是采用传统回调函数的书写方式：
	
	$.ajax({
		url: url1,
		success: function(res){
			$.ajax({
				url: url2,
				success: function(res){
					...
				}
			})
		}	
	})
	
我们不得不将第二个请求写到第一个请求的成功回调中，这样的嵌套降低了代码结构的可读性，尤其是在请求链很长的情况下，这样的代码很容易造成开发人员的困惑。

而在Promise规范下，这样的请求一般会被书写成如下形式：
	
	var defered = $.ajax({url: url1});
	defered.then(function(){
		...
	})
	
值得一提的是，Promise对象在调用then方法后一般会返回一个新的Promise对象，你可以在之后继续调用相同的方法，使其按序执行，这样的链式写法会更具逻辑性.
##### 如何实现一个简单的Promise?
Promise只是一种规范，虽然各种各样的库对它的实现都不一样，但它们都还是包含一些基本的元素：
1.state: 代表当前执行状态，一般有pending(等待中),resolved(已完成),rejected(已拒绝)
2.done,fail,then: 添加相应状态下的执行函数,done对应resolved,fail对应rejected
3.resolve,reject: 更改状态的函数
在了解这些基本元素后，我们可以考虑用原生JS实现一个简单的Promise对象

	var Promise = function() {
		this.state = "pending"; //promise状态
		this.list = []; //函数列表
	}

	Promise.prototype = {

		constructor: Promise,

		resolve: function(){
			this.state = "resolved";
			for(var i = 0, len = this.list.length; i < len; i++) {
            	this.list[0].call(this);
	            this.list.shift();
    	    }
		},

		then: function(func){
			if(typeof func == "function") {
				this.list.push(func);
			}
		}
	}
	
这是一个极其简化的Promise，其中只实现了resolve和then两种方法，事实上这就是Promise规范的核心流程，其他像done,fail,reject等等只是对应的状态不同而已，原理还是一样的。
那么，我们该如何运用这个Promise对象呢?

	var promise = new Promise();

	setTimeout(function(){

		promise.resolve();

	},2000);

	promise.then(function(){
		console.log("success!");
	});	

在上述代码中，我们通过then方法将一个匿名函数push到该promise对象的函数列表中，并用setTimeout构造了一个简单的异步场景，在页面上等待2秒后，可以看到控制台打印出success，说明then方法中传的函数得到了调用，这就是promise的执行流程，通过then方法给函数队列添加函数，再通过resolve从函数队列中取出函数执行并更改状态。

#### 二.jQuery的$.Callback对象
在jQuery中，对defer对象的实现完全依赖于它的Callback对象，所以在解读defer对象之前，我们必须先了解它的Callback对象。

jQuery构造了一个Callback对象来进行回调函数的统一管理，Callback对象维护一个函数队列，并提供一些外部的方法来操作该队列中的函数，一般来说，我们在外部使用的时候很少会用到Callback对象，然而jQuery内部不少地方都用到了该对象，首先我们先来看看该对象的用法。

	function a(){
		console.log("callback");
	}
	var cb = $.Callbacks();
	cb.add(a);
	setTimeout(function(){
		cb.fire();
	}, 2000)
	
在页面中执行上述代码，可以看到2秒后控制台打印出callback，可以看出，这个流程和我们上述介绍的Promise规范的实现极其相似，只是这次我们通过add添加函数，通过fire来执行。

我们可以看看Callback对象在jQuery的实现部分，以下代码摘自jQuery－2.0.3版本

	jQuery.Callbacks = function( options ) {

		options = typeof options === "string" ?
			( optionsCache[ options ] || createOptions( options ) ) :
			jQuery.extend( {}, options );

		var 
			memory,
			fired,
			firing,
			firingStart,
			firingLength,
			firingIndex,
			list = [],	
			stack = !options.once && [],
			fire = function( data ) {
				...		
			},
			self = {
				add: function() {
					...
				},
				remove: function() {
					...
				},
				has: function( fn ) {
					...
				},
				...
				fireWith: function( context, args ) {
					...
				},
				fire: function() {
					...
				},
				...
			};

		return self;
	};
我们先省略中间函数的逻辑实现部分，单单从该对象的结构来看，可以看出这其实是一个闭包结构，它定义了像memory,fired等等内部变量，返回一个self对象，而该self对象包含各种各样的方法来操作上面定义的内部变量，其中就包括我们在上文使用的add和fire。

那么首先，我们可以看看关于add方法的实现逻辑

	add: function() {
		if ( list ) {
			...	
			(function add( args ) {
				jQuery.each( args, function( _, arg ) {
					var type = jQuery.type( arg );
					if ( type === "function" ) {
						if ( !options.unique || !self.has( arg ) ) {
							list.push( arg );
						}
					} else if ( arg && arg.length && type !== "string" ) {
						add( arg );
					}
				});
			})( arguments );
			...
		}
		return this;
	},
	
主要逻辑是中间的这段匿名函数自执行，它将add方法的arguments对象作为参数传入，然后对其进行遍历，判断其类型是否为函数，如果是就将其push到函数队列中，其中unique作为该对象的一个option主要是为了控制队列中的每个函数是否应该独一无二。
另外值得注意的是参数类型不是函数的情况，从else if的表达式中可以看出，参数拥有length属性并且不是字符串，那么意味着该参数只可能是数组，这时候递归调用add并将该参数传入，也就是说add方法还可以这样使用:

	var cb = $.Callbacks();
	cb.add([a,b]);
	
这么做可以同时将a,b函数加入到函数队列中
再了解了add方法的实现后，我们可以再来看看fire方法是如何实现的

	fireWith: function( context, args ) {
		if ( list && ( !fired || stack ) ) {
			...
			if ( firing ) {
				stack.push( args );
			} else {
				fire( args );
			}
		}
		return this;
	},
	fire: function() {
		self.fireWith( this, arguments );
		return this;
	},
	
可以看出，当我们调用fire方法时，实际上内部调用的是fireWith方法，而fireWith则又进一步调用了Callback内部的fire方法，定义如下：

	fire = function( data ) {
		memory = options.memory && data;
		fired = true;
		firingIndex = firingStart || 0; //目前执行到的顺序
		firingStart = 0; 
		firingLength = list.length; //函数队列的长度
		firing = true;	//相当于锁，fire期间不允许再重复fire
		//遍历队列并进行函数调用
		for ( ; list && firingIndex < firingLength; firingIndex++ ) {
			if ( list[ firingIndex ].apply( data[ 0 ], data[ 1 ] ) === false && options.stopOnFalse ) {
				memory = false;
				break;
			}
		}
		firing = false; //解锁
		...
	},
主要逻辑在中间的for循环，它从firingIndex指定位置遍历了目前的函数队列，并通过apply调用函数，stopOnFalse作为选项为真时，一旦函数返回false就跳出循环。

以上是jQuery对add和fire的实现逻辑，当然Callback对象还有各种各样的选项和方法，在这里由于篇幅有限就不再赘述。

#### jQuery中defer对象的实现
在了解上述部分后，我们就可以正式解读defer对象的实现部分

