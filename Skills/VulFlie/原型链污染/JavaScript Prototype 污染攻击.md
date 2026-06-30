```table-of-contents
```

# 原型链的污染
JS是由对象组成的，对象与对象之间存在着继承关系
每个对象都有一个指向它的原型的内部链接，而该原型对象也有原型，以此反复直到 null 
实际就是对象的层层继承，而一个实例对象的原型链形成了一条链，这就是JS的原型链

JS中每个函数都有一个**prototype**属性，而每个对象也有一个**proto**属性用来指向实例对象的原型
再次深入，每个原型也都有一个**constructor**属性执行相关联的构造函数，也就是通过 *构造函数* 生成实例化的对象

![](assets/JavaScript%20Prototype%20污染攻击/file-20260617110140611.png)
该图原型链：cat - Cat.protype - Object.prototype - null

由于对象之间存在继承关系，即当使用或输出一个变量就会通过原型链向上搜索，若上层没有则会继续向上上层进行搜索，直到 **null**。若还未找到就会返回 **undefined**

**原型链污染**：（只要修改一个对象的原型，就会影响所有来自于该原型的对象）
**主要污染形式**：对象、数组的键名或属性名 **可控**，同时是**赋值语句**的情况（通常使用Json值）
**常见的危险函数有**：
- `merge()`
- `clone()`：其实内核就是将待操作的对象merge到一个空对象中
以及一些插件

# 区分prototype与__proto__

1. `prototype`是一个类的属性，所有类对象在实例化的时候将会拥有`prototype`中的属性和方法
2. 一个对象的`__proto__`属性，指向这个对象所在的类的`prototype`属性

```JavaScript
function Foo() { 
	this.bar = 1 
} 
Foo.prototype.show = function show() { 
	console.log(this.bar) 
} 
let foo = new Foo() 
foo.show()
```
这里创建了一个 **Foo（）** 函数，当进行 `let foo = new Foo()`（类对象的实例化时） 就默认创建了一个属性名为**prototype** （即已经实现了 `foo.prototype` ）
要创建一个 **show（）** 的函数且也要保留 **Foo（）** 内容，就需要将 **show（）** 的原型指向`foo.prototype`

这样触发 **show（）** 时，无法直接找到，但是由于指定了原型链，它就会跑到 **Foo（）** 去找可以输出的内容

