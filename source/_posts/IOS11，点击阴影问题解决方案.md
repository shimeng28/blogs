---
title: IOS11，点击阴影问题解决方案
date: 2017-12-05 18:06:07
tags: JavaScript
---

**问题描述：** IOS11下，textarea输入后，点击出现阴影，导致无法点击页面中的其他元素。

**解决思路：** iOS下，取消点击阴影可以使用一下css解决
	
	*:not(textarea, input) {
	  -webkit-tap-highlight-color: rgba(0, 0, 0, 0);
	  -webkit-user-select: none;
	  -webkit-touch-callout: none;
	}
上面代码除去textarea和input是因为使用-webkit-user-select:none会导致其不能输入内容。(not选择器在iOS下兼容性比较好)<br />
但是，当需要输入文本框textarea时，输入后点击保存仍然会出现阴影。通过测试个人觉得是textarea没有使用上面的css, 因此更进一步的想法是：<br />
   &nbsp;&nbsp;&nbsp;&nbsp;1.增加一个类， 代码如下
	 
	.remove-webkit-shadow {
	  -webkit-tap-highlight-color: rgba(0, 0, 0, 0);
	  -webkit-user-select: none;
	  -webkit-touch-callout: none;
	}
 &nbsp;&nbsp;&nbsp;&nbsp;2.当textarea为焦点时，不为textarea设置类名，此时textarea在输入内容。<br />
 &nbsp;&nbsp;&nbsp;&nbsp;3.当textarea失去焦点时，为textarea设置.remove-webkit-shadow类名。此时当点击页面其他元素时，就不会出现阴影。<br />
但是通过测试发现，textarea失去焦点并不精准，当点击页面其他元素时，有时textarea并未失去焦点。
<br />因此可以为window增加点击事件，当点击元素不是textarea时，使textarea失去焦点。
综上所述：全部代码为：
	 
	 *:not(textarea, input) {
	  -webkit-tap-highlight-color: rgba(0, 0, 0, 0);
	  -webkit-user-select: none;
	  -webkit-touch-callout: none;
	}
	
	.remove-webkit-shadow {
	  -webkit-tap-highlight-color: rgba(0, 0, 0, 0);
	  -webkit-user-select: none;
	  -webkit-touch-callout: none;
	}
	// 使用的语法是zepto,可以自行转换为其他代码
	textarea.on('focus', function() {
	  textarea.addClass('remove-webkit-shadow');
	});
	textarea.on('blur', function() {
	  textarea.removeClass('remove-webkit-shadow');
	});
	$(window).on('click', function(e) {
	  if (e.target !== textarea) {
	    textarea.blur();
	  }
	});
 