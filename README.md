# ES5_STRICT_MODE ECMAScript5.1 严格模式

## 严格模式的总体特征
+ 首先，严格模式消除了一些Javascript的哑错误，而是抛出异常
+ 第二，严格模式消除了一些不理由js引擎进行性能优化的语言特性
+ 第三，严格模式禁止可能在后续的js版本中所采用的语法

## 开启严格模式
在任何的其他js代码前加上，'use strict'或者"use strict"，已开启针对一个代码单元（Code Unit
）的严格模式。这串字符就是指令序言（'Directive Prologues')

ECMAScript 5.1中定义的指令序言如下：
> A Directive Prologues is a sequence of directives represented entirely as string literals which followed by an optional semicolon. A Directive Prologue may be an empty sequence.

目前ES5只定义了一个特别序言，也就是严格模式指令

注意，指令序言实际上可以包含一个和多个指令，但浏览器可能会触发一个警告。

## 严格模式的作用域区间
在函数级作用域内生效，将'use strict';放在执行函数的顶部，当时并不要求一定要在第一个位置（但是指令序言整体必须被放在开始的位置），当然在此之前的注释部分并不阻碍严格模式的生效

此外，严格模式的效应会延长到所有的内部函数之中，也就是内部函数是严格模式的，其条件就是要么其本身是严格模式要么其外部函数作用域是严格模式。需要注意的一点是，这种关系是在定义的时候确定的，而不是在执行的时候，也就是从字面量的意义上来确定内部函数是否是严格模式。
例如：

    function a() {}
    function b() {'use strict'; a();} // 尽管b函数是严格模式，但是a由于不是定义在严格模式之下，所以a也不知严格模式
    function b() {'use strict'; function a() {}} // 此时a是严格模式

使用Function构造器所创建的函数并不从其定义的外部作用域中继承严格模式。

    'use strict';
    var f = new Function('eval', 'arguments', 'eval = 10; arguments = 1;')
    f(); // f不是严格模式，故而可以执行
    var f = new Function('eval', 'arguments', '"use strict"; eval = 10; arguments = 1;')
    f(); // SyntaxError

最后，即使在严格模式下，indirect eval会在全局环境中创建变量，但是direct eval则会在eval作用域中执行

## 严格模式对语法及运行时的js都有影响，
### 将mistakes转换为errors
+首先，让不经意间创建全局变量变得不可能
        mistypedVariable = 15 // 由于拼错了变量而创建全局变量，严格模式下会抛出ReferenceError异常
+第二，任何给不能写的属性（[[writable]]为false）赋值，赋值给只读属性，给不可拓展的对象赋新属性（[[extensible]]为false），给NaN赋值，会抛出异常（TypeError）

        function a() {'use strict'; NaN = 1}
        a() // TypeError
        var obj = {};

        Object.defineProperty(obj, 'x', {value: 32, writable: false})
        obj1.x = 9 // TypeError

        var obj2 = {get x() {return 17;}}
        obj2.x = 5 // TypeError

        var obj3 = {};
        Object.preventExtensions(obj3);
        obj3.x = 1; //TypeError

+第三，当尝试去删除不可删除的属性是，抛出TypeError
    
    delete Object.prototype

+第四，要求所有的对象字面量的键值是唯一的不可重复的，而普通模式下会使用最后的那个值
    
    var o = {a: 1, a: 2} // Syntax Error

+第五，严格模式要求函数的参数名唯一，而普通模式系下，最后一个同名参数生效，但仍然能通过arguments[i]来访问

    function foo(a, a, b) { // Syntax Error

    }

+第六，严格模式禁止八进制数字
八进制也不是ECMAScript的规范内容，并且八进制的表达方式极易引起人的误解并且被滥用，特别在parseInt中，会的到令人困惑的以及不一致的结果。
严格模式不会扩展数字字面量的八进制以及转义形式的字符串八进制。即使在ES3中，八进制也是作为一种向后兼容的手段

    var sum = 011; // SyntaxError
    var sum = '\0123' // SyntaxError

parseInt：

    parseInt('07') // 7
    parseInt('08') // 0
    parseInt('09') // 0 ES5中相关的解析算法已被移出

+第七，和继承的属性有关

    "use strict";
    var foo = Object.defineProperty({}, "x", {
      value: 10,
      writable: false
    });
    var bar = Object.create(foo);
    console.log(bar.x); // 10, inherite
    bar.x = 20; // TypeError
    console.log(bar.x); // still 10, if non-strict mode
    Object.defineProperty(bar, "x", { // OK
      value: 20
    });
    console.log(bar.x); // 20

### 简化变量使用
+第一，编译优化
很多编译优化依赖于某个变量就在一个具体的位置，然而，js有时候会使变量所在的对应位置直到运行时才能得到。严格模式让with语句抛出语法异常，所以也就没有对象会在运行期指向一个位置的地方

    var x = 17;
    with(obj) { // Syntax Error
        x;
    }

+第二，eval语句的执行不会对当前LexicalEnvironment带来新的变量
通常情况下，eval('var x;')会为当前环境带来新的变量，也就是eval在运行之前无法得知是否会给环境带来新变量，严格模式中eval不可影响外部变量

+第三，严格模式禁止删除普通变量名，否则会抛出Syntax Error

    var x;
    delete x;

### 使得eval和arguments更简单
+第一：移除了二者的奇怪特性
eval会添加或者删除变量绑定
也阻止了使用eval和arguments作为变量名，eval和arguments将作为关键字来对待

    function c() {'use strict';var eval = 1;} // Syntax Error
    arguments++ // SyntaxError
    var obj = {set x(arguments) {}}
    try {} catch(arguments) {}
    function eval() {}
    new Function('arguments', ...)

+第二：严格模式消除了arguments和函数参数变量的双向关联性
通常，如果函数第一个参数是arg那么，设置arg也就设置了arguments[0]

+第三：arguments.callee不再被支持
该属性很弱下仅仅在于应用函数本身，且潜在的阻止了内联函数的性能优化

### 安全
使得编写安全的js更加容易。
+第一，this可以不为对象，且如果没有指定this，那么this就是undefined而不是全局对象
    原因在于，this的对象转换时一种性能上的开销，同时暴露全局对象也有安全方面的问题

    function b() {'use strict'; console.log('aaa', this)}
    b() // aaa undefined
    这可以用来避免因忘记使用new调用构造函数的问题
    function A(x) {
        this.x = x;
    }
    var a = A(1); // TypeError: undefined.x

+第二，禁止fun.caller, fun.arguments, fun.callee的调用
    
+第三，禁止使用arguments.caller, 打破了本来隐藏起来的保留值

callee以及caller由于存在安全问题而被限制

### 为未来的Javascript版本做铺垫
+首先，保留关键字将不允许被使用
包括，implements, interface, let, private, protected, pubic, package, static, yield，部分的环境中，这些保留的关键字已经作为关键字被使用
+第二，禁止在非函数顶级预内定义函数

    if (true) {
        function f() {} // SyntaxError
    }


