---
title: Javascript面向对象写法总结
date: 2017-07-12 14:07:52
tags: [JS, 学习笔记]
---

JS面向对象总是记不住怎么写，写篇总结就当是笔记吧。长篇大论写不来，直接用代码说话。

<!-- more -->

## 创建对象

### 工厂模式

``` javascript 
function Rect1(a, b) {
    var r = new Object();
    r.a = a;
    r.b = b;
    r.prop = ["a", "b", "S"];
    r.C = function() {
        return 2 * (this.a + this.b);
    }
    r.S = function() {
        return this.a * this.b;
    }
    return r;
}
```

测试：

``` javascript 
var rect1 = Rect1(10, 5);
console.log(rect1.constructor.name); // Object
console.log(rect1 instanceof Object); // true
console.log(rect1 instanceof Rect1); // false
console.log(rect1.C()); // 30
console.log(rect1.S()); // 50
```

缺点：判断不出对象的类型。

### 构造函数模式

``` javascript
function Rect2(a, b) {
    this.a = a;
    this.b = b;
    this.prop = ["a", "b", "S"];
    this.C = function() {
        return 2 * (this.a + this.b);
    }
    this.S = function() {
        return this.a * this.b;
    }
}
```

把成员函数拿出外面解决函数重复创建的问题。

``` javascript 
function Rect2(a, b) {
    this.a = a;
    this.b = b;
    this.prop = ["a", "b", "S"];
    this.C = C;
    this.S = S;
}
function C() {
    return 2 * (this.a + this.b);
}
function S() {
    return this.a * this.b;
}
```

测试：

``` javascript
var rect2 = new Rect2(10, 5);
console.log(rect2.constructor.name); //Rect2
console.log(rect2 instanceof Object); // true
console.log(rect2 instanceof Rect2); // true
console.log(rect2.C()); // 30
console.log(rect2.S()); // 50
```

缺点：看上去没问题了，可是如果有很多成员函数，就必须在全局定义很多个函数，这些「全局」函数只能被某个对象调用，而面向对象的类也没有把函数封装起来，好像有点怪怪的。

### 原型模式

``` javascript
function Rect3() {
}

Rect3.prototype.a = 10;
Rect3.prototype.b = 5;
Rect3.prototype.prop = ["a", "b", "S"];
Rect3.prototype.C = function() {
    return 2 * (this.a + this.b);
}
Rect3.prototype.S = function() {
    return this.a * this.b;
}
```

测试：

``` javascript
var rect3 = new Rect3();
console.log(rect3.constructor.name); // Rect3
console.log(rect3 instanceof Object); // true
console.log(rect3 instanceof Rect3); // true
console.log(rect3.C()); // 30
console.log(rect3.S()); // 50

console.log(rect3.prop); // a,b,S
var r1 = new Rect3();
r1.prop.push("C");
console.log(rect3.prop); // a,b,S,C
console.log(r1.prop); // a,b,S,C
```

缺点：不能通过传递参数来给对象的成员属性赋值，且所有对象实例会共享引用类型的成员属性。

### 混合模式

``` javascript
function Rect4(a, b) {
    this.a = a;
    this.b = b;
    this.prop = ["a", "b", "S"];
}
Rect4.prototype.C = function() {
    return 2 * (this.a + this.b);
}
Rect4.prototype.S = function() {
    return this.a * this.b;
}
```

测试：

``` javascript
var rect4 = new Rect4(10, 5);
console.log(rect4.constructor.name); // Rect4
console.log(rect4 instanceof Object); // true
console.log(rect4 instanceof Rect4); // true
console.log(rect4.C()); // 30
console.log(rect4.S()); // 50

console.log(rect4.prop); // a,b,S
var r2 = new Rect4(8, 3);
r2.prop.push("C");
console.log(rect4.prop); // a,b,S
console.log(r2.prop); // a,b,S,C
```

### 动态原型模式

混合模式已经基本上没问题了，动态原型模式把成员函数放进构造函数里，看起来会爽一些。

``` javascript
function Rect5(a, b) {
    this.a = a;
    this.b = b;
    this.prop = ["a", "b", "S"];
    if(typeof(this.S) !== "function") {
        Rect5.prototype.C = function() {
            return 2 * (this.a + this.b);
        }
        Rect5.prototype.S = function() {
            return this.a * this.b;
        }
    }
}
```

## 继承

### 原型链继承

``` javascript
function Rect4(a, b) {
    this.a = a;
    this.b = b;
    this.prop = ["a", "b", "S"];
}
Rect4.prototype.C = function() {
    return 2 * (this.a + this.b);
}
Rect4.prototype.S = function() {
    return this.a * this.b;
}

function Cuboid1(c) {
    this.c = c;
}
Cuboid1.prototype = new Rect4(10, 5);
// Cuboid1.prototype.constructor = Cuboid1;
Cuboid1.prototype.C = function() {
    return 4 * (this.a + this.b + this.c);
}
Cuboid1.prototype.S = function() {
    return 2 * (this.a * this.b + this.a * this.c + this.b * this.c);
}
Cuboid1.prototype.V = function() {
    return this.a * this.b * this.c;
}
```

测试：

``` javascript
var cuboid1 = new Cuboid1(3);
console.log(cuboid1.constructor.name); // Rect4
console.log(cuboid1 instanceof Object); // true
console.log(cuboid1 instanceof Rect4); // true
console.log(cuboid1 instanceof Cuboid1); // true
console.log(cuboid1.C()); // 72
console.log(cuboid1.S()); // 190
console.log(cuboid1.V()); // 150

console.log(cuboid1.prop); // a,b,S
var c1 = new Cuboid1(4);
c1.prop.push("C");
console.log(cuboid1.prop); // a,b,S,C
console.log(c1.prop); // a,b,S,C
```

缺点：同原型模式创建对象，不能传参给基类属性赋值，且所有对象会共享一个引用类型的成员属性，在不重新指定构造器的情况下，构造器是「基类构造函数」。

### 借用构造函数继承

``` javascript
function Rect4(a, b) {
    this.a = a;
    this.b = b;
    this.prop = ["a", "b", "S"];
}
Rect4.prototype.C = function() {
    return 2 * (this.a + this.b);
}
Rect4.prototype.S = function() {
    return this.a * this.b;
}

function Cuboid2(a, b, c) {
    Rect4.call(this, a, b);
    this.c = c;
}
Cuboid2.prototype.C = function() {
    return 4 * (this.a + this.b + this.c);
}
Cuboid2.prototype.S = function() {
    return 2 * (this.a * this.b + this.a * this.c + this.b * this.c);
}
Cuboid2.prototype.V = function() {
    return this.a * this.b * this.c;
}
```

测试：

``` javascript
var cuboid2 = new Cuboid2(10, 5, 3);
console.log(cuboid2.constructor.name); // Cuboid2
console.log(cuboid2 instanceof Object); // true
console.log(cuboid2 instanceof Rect4); // false
console.log(cuboid2 instanceof Cuboid2); // true
console.log(cuboid2.C()); // 72
console.log(cuboid2.S()); // 190
console.log(cuboid2.V()); // 150

console.log(cuboid2.prop); // a,b,S
var c2 = new Cuboid2(8, 3, 4);
c2.prop.push("C");
console.log(cuboid2.prop); // a,b,S
console.log(c2.prop); // a,b,S,C
```

缺点：用instanceof运算符不能判断基类。

### 组合继承

``` javascript
function Rect4(a, b) {
    this.a = a;
    this.b = b;
    this.prop = ["a", "b", "S"];
}
Rect4.prototype.C = function() {
    return 2 * (this.a + this.b);
}
Rect4.prototype.S = function() {
    return this.a * this.b;
}

function Cuboid3(a, b, c) {
    Rect4.call(this, a, b);
    this.c = c;
}
Cuboid3.prototype = new Rect4();
Cuboid3.prototype.constructor = Cuboid3;
Cuboid3.prototype.C = function() {
    return 4 * (this.a + this.b + this.c);
}
Cuboid3.prototype.S = function() {
    return 2 * (this.a * this.b + this.a * this.c + this.b * this.c);
}
Cuboid3.prototype.V = function() {
    return this.a * this.b * this.c;
}
```

测试：

``` javascript
var cuboid3 = new Cuboid3(10, 5, 3);
console.log(cuboid3.constructor.name); // Cuboid3
console.log(cuboid3 instanceof Object); // true
console.log(cuboid3 instanceof Rect4); // true
console.log(cuboid3 instanceof Cuboid3); // true
console.log(cuboid3.C()); // 72
console.log(cuboid3.S()); // 190
console.log(cuboid3.V()); // 150

console.log(cuboid3.prop); // a,b,S
var c3 = new Cuboid3(8, 3, 4);
c3.prop.push("C");
console.log(cuboid3.prop); // a,b,S
console.log(c3.prop); // a,b,S,C
```

## 结论

创建对象常用**混合模式**，继承常用**组合继承**。

## 参考文献

* 《JavaScript高级程序设计（第3版）》
