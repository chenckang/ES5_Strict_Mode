# ECMAScript5.1 严格模式浅入深出

## 严格模式的总体目标

大体而言，严格模式存在的意义如下三点：

+ 首先，严格模式消除了一些Javascript的静默错误（Silent Error），而是显式的抛出异常，从而显式的提示错误，而静默的进行失败处理，则可能导致留下位置的问题，详细内容后面会给出
+ 第二，严格模式消除了一些不利于ECMAScript引擎性能优化的语言特性
+ 第三，严格模式禁止可能在后续的ECMAScript版本中所采用的语法

## 开启严格模式

在任何的其他js代码前加上，`'use strict'`或者`"use strict"`，已开启针对一个代码单元（Code Unit
）的严格模式。这串字符就是指令序言（___Directive Prologues___)

ECMAScript 5.1中定义的指令序言如下：
> A Directive Prologues is a sequence of directives represented entirely as string literals which followed by an optional semicolon. A Directive Prologue may be an empty sequence.

目前ES5只定义了一个特别序言，也就是严格模式指令序言。

注意，当指令序言包含一个和多个指令时，浏览器可能会触发一个警告。

总体而言按作用域范围划分存在几种类型的严格模式：

#### `<script></script>`标签内作用域的严格模式
    <script>
        'use strict';
        var eval = 1;
    </script>

#### 函数作用域的严格模式

    var a = function(){/* comment here */'use nostrict'; 'use strict'; var eval = 1;}
    // 虽然上诉函数表达式的使用的指令序言不止一个，但是目前只有'use strict'是生效的。
    // 注释不影响严格模式的生效

#### eval内部的严格模式

    function a(){eval('"use strict";var eval = 1');}

最后，即使在严格模式下，___Indirect Eval___会在全局环境中创建变量，但是___Direct Eval___则会在eval作用域中执行

#### 严格作用域继承的关系
一般而言函数从其外部继承严格作用域，也就是如果一个函数处于严格模式作用域之中，那么函数本身也将是严格模式的。

    function a(){'use strict'; eval('var eval = 1');}

    function c() {'use strict'; function e() {var eval = 2; console.log(eval);} e();}

特别要注意的是上述的前提是：函数是定义在严格模式，而不是执行在严格模式的，例如如下：

    function a() {}
    function b() {'use strict'; a();} // 尽管b函数是严格模式，但是a由于不是定义在严格模式之下，所以a也不知严格模式
    function b() {'use strict'; function a() {}} // 此时a是严格模式

额外注意的一点是：
使用Function构造器所创建的函数并不从其定义的外部作用域中继承严格模式，唯一的方式就是在其内部声明使用严格模式。

    'use strict';
    var f = new Function('eval', 'arguments', 'eval = 10; arguments = 1;')
    f(); // f不是严格模式，故而可以执行
    var f = new Function('eval', 'arguments', '"use strict"; eval = 10; arguments = 1;')
    f(); // SyntaxError




## 严格模式对语法及运行时的js都有影响

### 将mistakes转换为errors

#### 首先，让不经意间创建全局变量变得不可能

    mistypedVariable = 15 // 由于拼错了变量而创建全局变量，严格模式下会抛出ReferenceError异常

#### 第二，任何给不能写的属性（[[writable]]为false）赋值，赋值给只读属性，给不可拓展的对象赋新属性（[[extensible]]为false），给NaN赋值，会抛出异常（TypeError）

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

#### 第三，当尝试去删除不可删除的属性是，抛出TypeError

    delete Object.prototype

#### 第四，要求所有的对象字面量的键值是唯一的不可重复的，而普通模式下会使用最后的那个值

    var o = {a: 1, a: 2} // Syntax Error

#### 第五，严格模式要求函数的参数名唯一，而普通模式系下，最后一个同名参数生效，但仍然能通过arguments[i]来访问

    function foo(a, a, b) { // Syntax Error
    }

#### 第六，严格模式禁止八进制数字

八进制也不是ECMAScript的规范内容，并且八进制的表达方式极易引起人的误解并且被滥用，特别在parseInt中，会的到令人困惑的以及不一致的结果。
严格模式不会扩展数字字面量的八进制以及转义形式的字符串八进制。即使在ES3中，八进制也仅仅作为一种向后兼容的手段

    var sum = 011; // SyntaxError
    var sum = '\0123' // SyntaxError

parseInt：

    parseInt('07') // 7
    parseInt('08') // 0
    parseInt('09') // 0 ES5中相关的解析算法已被移出

#### 第七，和继承的属性有关

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
#### 第一，编译优化

很多编译优化依赖于某个变量就在一个具体的位置，然而，js有时候会使变量所在的对应位置直到运行时才能得到。严格模式让with语句抛出语法异常，所以也就没有对象会在运行期指向一个位置的地方

    var x = 17;
    with(obj) { // Syntax Error
        x;
    }

#### 第二，eval语句的执行不会对当前LexicalEnvironment带来新的变量

通常情况下，eval('var x;')会为当前环境带来新的变量，也就是eval在运行之前无法得知是否会给环境带来新变量，严格模式中eval不可影响外部变量

#### 第三，严格模式禁止删除普通变量名，否则会抛出Syntax Error

    var x;
    delete x;

### 使得eval和arguments更简单
#### 第一：移除了二者的奇怪特性

eval会添加或者删除变量绑定
也阻止了使用eval和arguments作为变量名，eval和arguments将作为关键字来对待

    function c() {'use strict';var eval = 1;} // Syntax Error
    arguments++ // SyntaxError
    var obj = {set x(arguments) {}}
    try {} catch(arguments) {}
    function eval() {}
    new Function('arguments', ...)

#### 第二：严格模式消除了arguments和函数参数变量的双向关联性

通常，如果函数第一个参数是arg那么，设置arg也就设置了arguments[0]

#### 第三：arguments.callee不再被支持

该属性很弱下仅仅在于应用函数本身，且潜在的阻止了内联函数的性能优化

### 安全

使得编写安全的js更加容易。

#### 第一，this可以不为对象，且如果没有指定this，那么this就是undefined而不是全局对象
    原因在于，this的对象转换时一种性能上的开销，同时暴露全局对象也有安全方面的问题

    function b() {'use strict'; console.log('aaa', this)}
    b() // aaa undefined
    这可以用来避免因忘记使用new调用构造函数的问题
    function A(x) {
        this.x = x;
    }
    var a = A(1); // TypeError: undefined.x

#### 第二，禁止fun.caller, fun.arguments, fun.callee的调用
    
#### 第三，禁止使用arguments.caller, 打破了本来隐藏起来的保留值

callee以及caller由于存在安全问题而被限制

### 为未来的Javascript版本做铺垫

#### 首先，保留关键字将不允许被使用
包括，implements, interface, let, private, protected, pubic, package, static, yield，部分的环境中，这些保留的关键字已经作为关键字被使用

#### 第二，禁止在非函数顶级预内定义函数

    if (true) {
        function f() {} // SyntaxError
    }


## ES5.1 中的严格模式的说明

在ES5.1规范中，严格模式在文档最后的附录部分被具体列出了，尽管在其ES5.1的正文中提到了很多细节，以及具体实现的算法描述，下面给出附录中的总体描述：

+ 标识符`implement`, `interface`, `let`, `package`, `private`, `protected`, `public`, `static`以及`yield`被作为未来保留关键字指令
+ 禁止对数字字面量___NumericLiteral___的拓展以支持八进制数字字面量___OctalIntegerLiteral___
+ 禁止对转移字符串字面量___EscapeSequence___的拓展以支持八进制转移字符串___OctalEscapeSequence___
+ 赋值语句中向未声明的标示符或者无指向的引用并不会在全局作用域中创建变量，同时___LeftHandSide___的左边也不能指向一个`{[[Writable]: false}`的对象的成员属性，或者`{[[set]]: undefined}`的对象的方法属性，以及不扩扩展对象(`[[Extensible]]`属性为false)的任意成员。
+ eval以及arguments不能出现在赋值操作符或者后缀表达式___PostfixExpress___或者在前缀自增符号及后缀自增符号中的一元表达式___UnaryExpress___的___LeftHandSideExpression___
+ Arguments对象定义了不可配置的属性（Non-configurable）的`caller`及`callee`属性，调用这些属性将抛出`TypeError`异常
+ Arguments对象与函数实际传入参数之间没有动态绑定关系
+ `arguments`变量标识符是作为对arguments对象的不可变引用，因而也就不能被作为赋值表达式的赋值目标
+ 如果对象字面量中包含重复的成员属性名则抛出`SyntaxError`异常
+ 如果`eval`及`arguments`作为对象的属性名称则抛出`SyntaxError`（原文如下：在属性赋值语句中，`eval`及`arguments`出现在属性赋值表达式___PropertyAssignment___中的属性赋值参数列表___PropertySetParameterList___中, 作为标示符使用，也就是说`eval`及`argument`不能作为属性名称使用）
+ 严格模式不能够将`eval`外部的变量及函数赋给`eval`内部，而是`eval`函数内部会创建一个新的变量环境，用于`eval`函数内部声明绑定使用。
+ 如果`this`变量在严格模式下使用，则`this`不会被强制转换为对象类型，`null`及`undefined`的`this`值不会被转化为全局对象，同时基本类型不会被转换为包裹对象
+ 如果`delete`作为一元操作符，直接作用于一个变量、函数参数或者函数名，则抛出`SyntaxError`异常
+ 如果`delete`删除的对象的对象的属性描述符是`{[[configurable: false}`，则抛出`TypeError`异常
+ 当变量声明存在`eval`及`arguments`变量，则抛出`SyntaxError`异常
+ 严格模式中不能存在___WithStatement___，__with语句__将抛出`SyntaxError`异常
+ 如果___TryStatement___中的`catch`的变量中存在`eval`及`arguments`，则抛出`SyntaxError`异常
+ 如果函数声明或者函数表达式中`eval`及`arguments`作为函数参数存在则抛出`SyntaxError`异常
+ 在函数表达式或者函数声明，或者使用`Function`构造函数创建的函数中，存在两个以上的同名参数，则抛出`SyntaxError`异常
+ 不能创建及更改名为`caller`及`arguments`且挂在在函数本身身上的属性
+ 尝试使用`eval`或者`arguments`作为变量标识符、函数声明、函数表达式或者作为函数参数将抛出`SyntaxError`异常

[ECMAScript5.1 Annex C, The Strict Mode of ECMAScript 原文部分链接](http://www.ecma-international.org/ecma-262/5.1/#sec-C)