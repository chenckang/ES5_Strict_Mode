# 深入浅出 ECMAScript5.1 严格模式
这边文章的探讨的话题是ECMAScript中严格模式的特性，以浅入深出的方式来介绍下，包括但不局限于严格模式的意义、严格模式的开启、严格模式到底限制了什么以及其具体原因，文章的最后给出了翻译自ES-262附录部分的内容，语言稍作调整以方便理解与阅读。

从深入浅出的角度来说，本文会从简单到复杂，一步步深入剖析严格模式，希望能够一步步引导读者理解严格模式，并在迫不及待的使用在自己的项目之中。

## 严格模式的总体目标

Javascript语言作为ECMAScript规范的实现者，存在极大的灵活性的同时，也由于松散的语法、弱的校验机制、一撮的特性等等而保守诟病。例如，

    // 原本想写成var a = 1, n = 1; 但错误的将逗号写成了分号，导致在global下创建了一个变量
    var a = 1; n = 1;

我们使用严格模式能够在程序运行时显式的暴露错误，从而帮助我们纠正潜在的威胁（尽管在ESLint等工具的帮助下，我们能够检测到很多的语法错误及写法问题，甚至定制StyleGuide来规范代码的写法以消除潜在问题，但是这是额外的话题）。

为了避免ES5对ES3做出大量的修改，就像ES4的那样，ES5在兼容既有的ES3语法基础上新增了很多特性，这些特性可以靠左ES3的一个依赖。
这其中包括：纠正易出错的语法，将一些__deprecated__的特性给移除掉。

大体而言，严格模式存在的意义如下三点：

+ 首先，严格模式消除了一些Javascript的静默错误（Silent Error），而是显式的抛出异常，从而显式的提示错误，而静默的进行失败处理，则可能导致留下位置的问题，详细内容后面会给出
+ 第二，严格模式消除了一些不利于ECMAScript引擎性能优化的语言特性
+ 第三，严格模式禁止可能在后续的ECMAScript版本中所采用的语法

## 严格模式指令

在任何的其他js代码前加上，`'use strict'`或者`"use strict"`，已开启针对一个__代码单元__（Code Unit
）的严格模式。这串字符就是指令序言（___Directive Prologues___)

ECMAScript 5.1中定义的指令序言如下：

> A Directive Prologues is a sequence of directives represented entirely as string literals which followed by an optional semicolon. A Directive Prologue may be an empty sequence.

然而目前ES5只定义了一个特别序言，也就是严格模式指令序言，尽管可以自己写出一些指令序言，但是它们将不会被当作有效的命令而生效。例如：
    
    'use nostrict';
    'use strict'; // 这行指令才是有效的

ECMAScript对严格指令定义如下___Use Strict Directive___

>A Use Strict Directive is a directive in a Directive Prologue whose string literal is either the exact character sequences "use strict" or 'use strict'. A Use Strict Directive may not contain an escape sequence or line continuation. This directive is used for specifying a strict mode of a code unit.

可以看到`'use strict'`或者`"use strict"`是被明确定义的开启严格模式的方式。

## 严格模式的开启及作用域

如上文所诉，严格模式作用于一个代码单元

总体而言按作用域范围划分存在几种类型的严格模式：

#### `<script></script>`标签内作用域的严格模式
    <script>
        'use strict';
        var eval = 1;
    </script>

#### 函数作用域的严格模式

    var a = function () {/* comment here */'use nostrict'; 'use strict'; var eval = 1;}
    // 虽然上诉函数表达式的使用的指令序言不止一个，但是目前只有'use strict'是生效的。
    // 注释不影响严格模式的生效

#### eval内部的严格模式

    function a() {eval('"use strict";var eval = 1');}

#### 关于作用域的几点特殊说明

一般而言函数从其外部继承严格作用域，也就是如果一个函数处于严格模式作用域之中，那么函数本身也将是严格模式的。

    function a() {'use strict'; eval('var eval = 1');}

    function c() {'use strict'; function e() {var eval = 2; console.log(eval);} e();}

特别要注意的是上述的前提是：函数是定义在严格模式，而不是执行在严格模式的，例如如下：

    function a() {}
    function b() {'use strict'; a();} // 尽管b函数是严格模式，但是a由于不是定义在严格模式之下，所以a并不是严格模式
    function b() {'use strict'; function a() {}} // 此时a是严格模式

额外注意的一点是：
使用Function构造器所创建的函数并不从其定义的外部作用域中继承严格模式，唯一的方式就是在其内部声明使用严格模式。

    'use strict';
    var f = new Function('eval', 'arguments', 'eval = 10; arguments = 1;')
    f(); // f不是严格模式，故而可以执行
    var f = new Function('eval', 'arguments', '"use strict"; eval = 10; arguments = 1;')
    f(); // SyntaxError

最后，即使在严格模式下，___Indirect Eval___ 会在全局环境中创建变量，但是 ___Direct Eval___ 则会在eval作用域中执行

## 严格模式的表现及原理

### 将隐式的错误转换为显示的异常（Throw Errors)

由于JavaScript本身的宽松特性，采取静默处理错误的方式来保持语言的简单性和便捷性，这亦被人看作是一种玩具式语言的理由之一，静默处理的错误可能会为整体程序埋下潜在的定时炸弹，特别是在前端项目越做越大的情况下，这方面的例子诸如: 不经意间创造的全局变量、对象上重复的键名、重复的函数参数声明等。看似简单的错误潜藏在大型项目中，即使是富有经验的工程师也难以排查甚至是自己也不经意间犯错。如果能在解释层面或者编译层面，直接抛出相关异常，会极大程度上杜绝因不经意间的错误而带来的尴尬之境。具体如下：

#### 首先，让不经意间创建全局变量变得不可能

    // 未使用var而创建全局变量，严格模式下会抛出ReferenceError异常
    mistypedVariable = 15 

#### 第二，任何给不可写的属性（[[writable]]为false）赋值、给不可拓展的对象赋新属性（[[extensible]]为false）、以及给NaN赋值，则会抛出异常（TypeError）

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

#### 第七，和继承的属性如果不可写，则尝试赋值会显示抛出TypeError

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

这方面主要在于控制不利于性能优化、及容易出错的变量特性而做出的限制

#### 第一，编译优化

很多编译优化依赖于某个变量在解释及执行的时候就确定相关执行环境，然而部分JS特性使得变量环境直到执行的时候才能确定，最典型的就是 ___with___。严格模式禁用了 ___with___ 语句。

    var x = 17;
    with(obj) { // Syntax Error
        // 在编译期或者解析期，x的值是无法确定的
        // 因而在性能优化方面也无从处理
        x;
    }

#### 第二，eval语句的执行不会对当前LexicalEnvironment带来新的变量

通常情况下，eval('var x;')会为当前环境带来新的变量，也就是eval在运行之前无法得知是否会给环境带来新变量，严格模式中eval不可影响外部变量。

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


## 附录 ES5.1 中的严格模式的说明

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