---
title: 自己写一个mini模块加载器
date: 2018-01-13 20:14:00
tags: JavaScript模块化
---
requireJS作为一个模块化加载器，前段时间学习了官网，这两天写了个mini版的模块化加载器。[mini-require](https://github.com/shimeng28/Tools/blob/master/mini-requireJS/mini-require.js)

模块化加载器主要由两个功能：定义和加载，也就是define和require。而我们若想要实现这两个方法，我们必须决定好模块module这个数据结构。首先我给它设定的属性有name(名称), dep(依赖), cb(回调), src(模块路径)。其中src路径地址是通过name转换的。所以一下是代码：

	function Module(name, dep, cb) {
		this.name = name;
		this.dep = dep;
		this.cb = cb;
		this.src = moduleNameToModulePath(name);
		// 将模块引用到module对象里
		modules[name] = this;
		
		this.init();
	}

可以看到，我这里面多了一个modules对象，这个对象作为存储所有的模块。init作为其原型上的方法，做一些初始化的工作。

我把一个模块定义了四个状态
	
	// 模块的状态 相当于生命周期
	const moduleStatus = {
		fetching: 1,
		fetchFinish: 2,
		fetchError: 3,
		executeCallBack: 4,
	};

init方法所做的工作一是为模块赋予其随着状态改变而改变的能力，二是将其状态设置为fetching


	const fn = Module.prototype;
	fn.init = function() {
		this.defineStatus();
		this.status = 'fetching';
	};
	fn.defineStatus = function(status) {
		Object.defineProperty(this, 'status', {
			get() {
				return status;
			},
			set(newStatus) {
				status = newStatus;
				switch(moduleStatus[status]) {
					// 1.fetching
					case 1:
						this.fetch();
					break;
					// 2.fetchFinish
					case 2:
						this.fetchFinish();
					break;
					// 3.fetchFinish
					case 3:
						this.fetchError();
					break;
					case 4:
						this.executeCallBack();
					break;
				}
			}
		});
	};


当模块状态为fetching时，它会加载模块，并执行模块内部的代码

	fn.fetch = function() {
		const script = document.createElement('script');
		const that = this;
		script.type = 'text/javascript';
		script.src = this.src;
		document.head.appendChild(script);
		script.onload = function() {
			that.status = 'fetchFinish';
		};
		script.onerror = function() {
			that.status = 'fetchError';
		};
	};

模块内部代码执行成功之后其状态改变为fetchFinish,失败则为fetchError。失败的时候就不用说了，也就是打个log以作提醒。当fetchFinish时，我们应该让引用了它的依赖的模块的依赖数目减一。OK，那这个时候我们就来看看当我们require了一个模块时发生了什么。

首先，我把require()执行相当于产生一个Task对象，其基本和Module对象区别不大，主要是当我们require一个模块时，我们需要分析它的依赖，而一个Module对象是没有必要关注依赖的问题，我们把依赖的问题交给Task对象。

先看Task对象

	function Task(dep, cb) {
		this.dep = dep;
		this.cb = cb;

		this.init();
	}
	Task.prototype = Object.create(Module.prototype);
	Task.prototype.init = function() {
		this.defineStatus();
		this.parseDep();
	};

从上面的代码可以看出，Task对象只需要两个参数，依赖(dep)和回调(cb),其原型继承于Module.prototype。主要的区别就在于init方法中调用了parseDep().

那么parseDep()方法又是何方神圣呢？
它的功能主要是解析依赖，我们给调用它的实体一个depNum属性，代表着其包含的依赖的数目，并且当depNum为0时，调用回调函数(cb)。另一点是，如果有依赖模块的话，先将依赖生成为Module对象，然后我们将遍历所有依赖模块，并将其添加到depToModule对象中，其key为依赖模块name，Value为包含依赖模块的模块的数组。

当然看代码会发现其不止这两个功能

	fn.parseDep = function() {
		const dep = this.dep || [];

		// 依赖中包含有require时
		if (dep.indexOf('require') !== -1) {
			this.requireDepPos = dep.indexOf('require');
			dep.splice(this.requireDepPos, 1);
		}
		let depNum = dep.length;
		// 是否循环引用 循环引用则会报错
		let circularLength = this.circularRefer().length;
		if (circularLength) {
			depNum -= circularLength;
		}
		if (depNum === 0) this.status = 'executeCallBack';

		Object.defineProperty(this, 'depNum', {
			get() {
				return depNum;
			},
			set(newDepNum) {
				depNum = newDepNum;
				// 依赖模块执行完毕
				if (depNum === 0) {
					this.status = 'executeCallBack'
				}
			}
		});

		// 该模块没有依赖 返回
		if (!depNum) return;

		dep.forEach((depModuleName) => {
			// 如果依赖模块不存在modules中
			if (!modules[depModuleName]) {
				let module = new Module(depModuleName);
				modules[module.name] = module;
			}
			// 如果依赖模块对象depToModule中没有该依赖模块
			if (!depToModule[depModuleName]) {
				depToModule[depModuleName] = [];
			}
			// 将包含依赖模块的模块压入依赖模块所在的索引中
			depToModule[depModuleName].push(this);
		});

我将依赖中如果存在require，以及出现模块间循环引用都放在了这个地方处理。

最后一个状态也就是fexecuteCallBack这个时候是调用回调函数。

	fn.executeCallBack = function() {
		// 传入依赖参数
		const arg = (this.dep || []).map((dep) => {
			return modules[dep].exports;
		});
		// 如果含有require依赖的话，将require插入函数参数中
		if (typeof this.requireDepPos === 'number' && this.requireDepPos !== -1){
			arg.splice(this.requireDepPos, 0, require);
		}

		this.exports = this.cb && this.cb.apply(this, arg);
	};

上面这个函数，我们会发现，如果是一个没有依赖的模块执行到了这里，那么arg为undefined，当我们执行回调的时候传入的也就是undefined. 但是会将回调返回的数据赋值给模块的exports属性，当下一个模块引入了这个模块作为依赖时，就会将这个模块的exports属性值作为其回调的参数。

以上就是我写的这个模块加载器的大体思路，更加详细的可以看源码[源码](https://github.com/shimeng28/Tools/blob/master/mini-requireJS/mini-require.js)