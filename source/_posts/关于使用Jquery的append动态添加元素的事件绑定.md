---
title: 关于使用Jquery的append动态添加元素的事件绑定
date: 2016-09-12 21:03:46
tags:
- jquey
categories:
- 前端
- Jquery
---
　　在用jquery对页面元素进行操作的时候，经常会用到 `append()` 函数来对页面元素进行添加，有时候还需要对新添加的元素绑定新的事件，但是如果像普通绑定事件函数一样对新添加的元素进行事件绑定的话，会失败，事件监听不起作用。
``` html
	<div id="button-test">
	    <button id="add-button" class="btn btn-primary">增加按钮</button>
	</div>
	<script>
	    TestData={}
	    TestUtil = {
	        init:function(){
	            $('#add-button').on('click',function () {
	                TestUtil.addButtonFunc();
	            })
	            $('.select').on('click',function () {
	               alert('clicked');
	            });
	        },
	        addButtonFunc:function () {
	            var innerHtml = '<button class="select btn btn-primary">点击按钮</button>';
	            $('#button-test').append(innerHtml);
	        }
	    }
	
	    $(function(){
	        TestUtil.init();
	    })
	</script>
```

这个时候需要使用新的方式来进行事件绑定。

``` html
	<div id="button-test">
	    <button id="add-button" class="btn btn-primary">增加按钮</button>
	</div>
	<script>
	    TestData={}
	    TestUtil = {
	        init:function(){
	            $('#add-button').on('click',function () {
	                TestUtil.addButtonFunc();
	            })
	            $('#button-test').on('click','.select',function () {
	                alert('success clicked');
	            })
	        },
	        addButtonFunc:function () {
	            var innerHtml = '<button class="select btn btn-primary">点击按钮</button>';
	            $('#button-test').append(innerHtml);
	        }
	    }
	
	    $(function(){
	        TestUtil.init();
	    })
	</script>
```

``` html
	$('#button-test').on('click','.select',function () {
	       alert('success clicked');
	})
```
　　其中，$('#button-test')是一直存在的页面元素，click是要绑定的事件类型，'.select'是选择器，在'#button-test'的范围内进行选择，function是绑定的事件
