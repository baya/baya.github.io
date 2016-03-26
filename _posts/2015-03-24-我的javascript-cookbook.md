---
layout: post
title: 我的 JavaScript Cookbook
---

## Boolean

### 布尔值为false的类型

~~~javascript
null
undefined
0, NaN
""
false
~~~

### 逻辑运算符

~~~text
运算符	描述	例子
&&	and	(x < 10 && y > 1) 为 true
||	or	(x==5 || y==5) 为 false
!	not	!(x==y) 为 true
~~~

## RegExp

### 构建包含变量的正则表达式

~~~javascript
	var mod_letters = ['T', 'R', 'W', 'A', 'G', 'M', 'Y', 'F', 'P', 'D', 'X', 'B', 'N', 'J', 'Z', 'S', 'Q', 'V', 'H', 'L', 'C', 'K', 'E'];
	var pre_letters = ['X', 'Y', 'Z'];
	var reg1 = new RegExp("^(\\d{8})([" + mod_letters.join(' ') + "])$");
	var reg2 = new RegExp("^([" + pre_letters.join(' ') + "])(\\d{7})([" + mod_letters.join(' ') + "])$");
~~~

## String

### string convert to integer

~~~javascript
var a = parseInt("10");
~~~

## 浏览器相关

### redirect

~~~javascript
var url = window.location.href + "/vote_result";
window.location.href = url;
~~~

### 刷新页面

~~~javascript
$('#something').click(function() {
    location.reload();
});
~~~

~~~javascript
// Javascript刷新页面的几种方法：
history.go(0)
location.reload()
location=location
location.assign(location)
document.execCommand('Refresh')
window.navigate(location)
location.replace(location)
document.URL=location.href
~~~

### 浏览后退 navigation back

~~~javascript
window.history.back()
~~~

## 面向对象

### 函数中this的四种情况

- 参考: http://jjyr.github.io/2014/02/18/javascript-tips-during-my-studying-2/

~~~javascript
//1. 函数绑定到对象上时, this为对象
a = {func: function(){ console.log(this)}}
a.func() // this是a

//2. 没有绑定到对象时，this为全局对象
(function(){ console.log(this) })()
//上面的this在浏览器中是window(全局对象)

//3. 用new调用时this绑定到新构造的对象

//4. 函数对象的apply方法
(function(){console.log(this)}).apply("hello")
//通过apply可以指定function的this值
~~~

### 原型 prototype

- 参考: http://jjyr.github.io/2014/02/18/javascript-tips-during-my-studying-2/

~~~javascript
//javaScript中的原型继承必须通过函数来完成

//function有一个很重要的prototype属性，默认值为{}
(function(){}).prototype
//=> Object {}

//函数的prototype属性是javascript中原型的关键
//用new去调用function时，会以该函数的prototype为原型构建对象(常用此行为来模拟继承)
a = {hello: function(){return "world"}}
//现在来构建一个链接到a的对象，首先需要一个function
A = function(){}
A.prototype = a
//用new调用A时会构建出以对象a为原型的对象
b = new A()
b.hello()
//=> "world"
b.__proto__ == a
//=> true

//所有对象都隐含链接自Object.prototype(相当于new Object())
//没有类型，又能随意的改变原型的键，结果就是可以很方便的动态更改代码
//通过函数和对象的任意组合，可以做到同一个原型对象具有多个“构造器”，可以任意的链接原型等等。比基于面向对象的语言要灵活许多
~~~

### NaN, undefined

~~~javascript
NaN == NaN // false
isNaN(NaN) // true

undefined == undefined // true
~~~

## 弹出框居中显示

~~~javascript
function adjustDialogPosition($dialog){
    $dialog.css({
	  position: 'fixed',
	  top: ((window.innerHeight/2) - ($dialog.height()/2))+'px',
	  left: ((window.innerWidth/2) - ($dialog.width()/2))+'px'
    })
}
~~~
