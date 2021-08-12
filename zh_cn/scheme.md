## [学习笔记] Lisp语言中的Scheme语言及其宏

Lisp语言有这些特性：

* 是函数式编程语言，很早就支持lambda表达式和闭包
* 允许可变性（Haskell不允许可变性）
* 动态类型（Haskell是静态类型）
* 很早就支持垃圾回收
* 有独特的宏特性

Lisp语言有很多方言，其中包括Scheme，这里主要讨论Scheme语言。有些Scheme代码会与等价JavaScript代码进行对比。

[一个Scheme教程](https://ds26gte.github.io/tyscheme/index-Z-H-1.html#TAG:__tex2page_toc)

### 前缀表达式

Lisp中函数调用需要把整个表达式用括号包住，参数之间不加逗号，Lisp中`(func a b)`相当于主流语言中的`func(a,b)`。

Lisp中没有中缀表达式，其他语言中的`1 + 2`在Lisp中是`(+ 1 2)`，这就是前缀表达式(S表达式)。

中缀表达式如果不定义优先级会有歧义: `1 + 2 * 3`可能是`(1 + 2) * 3`也可能是`1 + (2 * 3)`，而lisp中强行要求加括号：`(+ 1 (* 2 3))`，不需要定义优先级。

中缀表达式如果不定义是左结合还是右结合也会有歧义，例如`a * b * c`可以是`(a * b) * c`也可以是`a * (b * c)`，如果这里的乘法不满足结合律那么就会有歧义。在lisp中强行要求加括号也没有这个问题。

S表达式看似不符合习惯，但是可以消除歧义，简化词法分析。另一个缺点是让代码容易堆积括号。

主流语言中对普通值可以随便套括号，`1 + 2`等同于`(1) + (2)`，但在lisp中括号有函数调用的作用，不能随便加括号，`(+ (1) (2))`会导致解释器将`1`认为是函数要执行。

### 各种基础操作

define可以定义变量或函数

`(define x 1)`相当于JavaScript里的`var x = 1;`

`(define (x) 1)`相当于JavaScript里的`function x() {return 1;}`

`(define (f x) (+ x 1))`相当于JavaScript里的`function f(x) {return x + 1;}`

`(lambda (x) (+ x 1))`相当于JavaScript里的`x => x + 1`

`(define (f x) (+ x 1))`等同于`( define f (lambda (x) (+ x 1)) )`

`( let ((a 1) (b 2)) (+ a b) )`相当于JavaScript里的`((a, b) => a + b)(1, 2)` ，let可以在子表达式里声明中间变量。

`(set! a 1)`相当于`a = 1;`

`(if cond a b)`相当于`cond ? a : b`

### Lambda表达式

Scheme中支持lambda表达式（闭包）：

```scheme
(define (f a)
  (lambda (b) (+ a b))
)
```

相当于JavaScript中的

```typescript
function f(a) {
    return b => a + b
}
```

这里返回的函数捕获了外面的a。Scheme和JavaScript捕获的方式是在函数里面存储**外面的环境的引用**。Java语言和C++语言也有lambda表达式，但是他们捕获的方式是**将捕获的值存储在函数对象里面**，而不是存储对外面环境的引用。

这两种捕获方式的区别在于，Lisp和JavaScript支持修改捕获的变量：

```scheme
(define (getFunc) 
    (define variable 1)
    (define (func) variable)
    (set! variable 100)
    func
)

(display ((getFunc)))
```

对于JavaScript代码

```javascript
function getFunc() {
    let variable = 1
    let func = () => variable;
    variable = 100
    return func
}
console.log(getFunc()())
```

都会输出100而不是1。在func捕获variable后，对variable修改，func所捕获的值也跟着修改。

Java不支持这个功能：

```java
Supplier<Integer> getFunc() {
    Integer variable = 1;
    Supplier<Integer> func = () -> variable;
    variable = 100;
    return func;
}
```

这段代码会有编译错误。Java不支持这一功能是因为Java将捕获的变量存储在函数对象里面（这个设计是为了提高性能），而JavaScript和Scheme中函数对象里面不直接存储捕获的变量，而是存储对外面环境的引用，捕获的变量在环境里面。

### Lisp语言中的环境

前面说到Lisp和JavaScript中捕获外部变量的方法是在函数里面存储对环境(environment)的引用。那么"环境"是什么？在一个环境里可以定义变量，环境里面记录了这些变量的名称与目前的值。与此同时，环境可以嵌套，内层环境存有对外层环境的引用。

如下代码中

```scheme
(define (create-accumulator) 
    (define value 0)
    (define (accumulator delta) 
        (set! value (+ value delta))
        value
    )
    accumulator
)

(define a (create-accumulator))
(define b (create-accumulator))

(a 1) ; 值为1
(a 3) ; 值为4
(b 2) ; 值为2
```

产生了这些环境：

![lisp env _2_.png](https://i.loli.net/2021/08/12/WkPThGRn2rgY1ML.png)

可以看到a和b各自引用了不同的环境。每次调用函数都会创建一个环境，第一次调用`(create-accumulator)`创建了E1，第二次创建了E2。调用`(a 1)`创建了E3，在这个环境里面寻找变量`value`，无法直接找到，于是向外层环境E1查找，找到变量位置。

调用函数可以产生新环境，`let`语句也可以产生新环境。



JavaScript中的机理也是类似的。在JavaScript中可能会犯这种错误：

```javascript
let functionList = []
for (var i = 0 ; i < 5; i++) {
    functionList.push(() => console.log(i))
}

for (let func of functionList) {
    func()
}
```

这里希望在数组中存五个函数，这五个函数可以分别输出0到4。但实际上这五个函数都捕获的是同一个外部环境的变量`i`，这个代码的结果是输出五个5。在C++或Java中，捕获是将变量存到函数里面，因此不会出现这种问题。



### 数据组合以及列表

`(cons 1 2)`得到由1和2组成的有序对（只有两个元素的元组），相当于JavaScript中的`{first:1, second:2}`

`(car a)`从元组a中取第一个元素，相当于JavaScript中`a.first`。`(cdr a)`取第二个元素。

三个元素可以用嵌套元组组合`(cons (cons 1 2) 3)`，通过这种二元素元组可以将任意的数据组合在一起。

[跟Haskell一样](https://zhuanlan.zhihu.com/p/387237772)，Lisp用树状结构（链状结构）表示列表。`(list 1 2 3)`得到由1,2,3组成的列表，相当于`(cons 1 (cons 2 (cons 3 nil)))`，`nil`代表空。

![lispo _1_.png](https://i.loli.net/2021/08/12/bZ9IT1Aofw6xqhi.png)

### 引用

`'a`表示对a的引用(quote)：

* 引用常量，得到的还是原来的常量。`'1`的值为`1`
* 引用一个名称，得到符号(symbol)。`'a`可以得到符号a，代表对变量a的引用，在不同环境下可能代表不同的变量。符号不是字符串，但跟字符串一样可以进行相等比较
* 引用一个列表，相当于对列表里面的每个值进行引用，获得一个新列表。`'(a b 1)`得到`(list 'a 'b 1)`，`'()`得到`()`

在代码中写`(+ 1 2)`，这个表达式会被求值得到3，但是如果写`'(+ 1 2)`则不会求值这个表达式，不会得到3，而是会得到得到这个表达式本身，这个表达式本身是一个列表，相当于`(list '+ 1 2)`得到的值。

Lisp中表达式都是列表，语法树由嵌套的列表组成，列表可以作为代码执行。在将列表作为代码执行的时候，先看第一个元素，如果第一个元素是函数则将剩下的元素求值然后调用函数。如果第一个元素是宏，则进行宏展开。这个设定刚好契合前缀表达式。

### 宏

Lisp语言中宏(macro)的能力较强。C语言的宏只能进行简单的字符串拼接，宏本身不能进行计算。而Lisp语言中，宏可以用一个转换函数(transformer procedure)将语法树进行转换，干很多复杂的事情。

Lisp中的宏有这些独特的特性：

* 宏相当于一个符号和一个转换函数，这个转换函数函数输入的是语法树，输出的是语法树
* Lisp中语法树用列表表示，对列表进行操作就是对程序进行操作
* 假如一段代码里用了一处宏，这段代码可能会执行多次，但宏的转换函数只会执行一次
* 转换函数里面不能对输入的表达式求值，因为表达式中变量的具体值取决于不同的环境



一个简单的宏例子：例如说我不习惯前缀(prefix)表达式的写法`+ 1 2`，想要使用中缀(infix)表达式写法`1 + 2`，可以定义这个宏

```scheme
(define (transformer-procedure a b c) (list b a c))

(define-macro infix transformer-procedure)

(infix 1 + 2) ;相当于(+ 1 2)
```

这里定义了一个转换函数`transformer-procedure`，和宏`infix`。在代码中写出`(infix 1 + 2)`后，其参数的语法树会被输入宏的转换函数，调用`(transformer-procedure '1 '+ '2)`，得到`(list '+ 1 2)`，将这个列表作为代码，就是`(+ 1 2)`，于是`(infix 1 + 2)`被替换成`(+ 1 2)`。

宏定义也可以写成：

```scheme
(define-macro infix
  (lambda (a b c) (list b a c))
)
```

### 循环

Lisp中可以用递归实现循环的效果，例如说，输出0到5：

```scheme
(define i 0)
(define (loop-func) (
    if (< i 6)
    (begin
        (display i)
        (set! i (+ i 1))
        (loop-func)
    )
    ()
))

(loop-func)
```

> `begin`的作用是执行一系列操作，并取最后一个表达式的值。可以将多个操作包装成一个表达式。

相当于JavaScript中的

```javascript
var i = 0
function loop_func() {
    if(i < 6) { console.log(i); i += 1; loop_func(); }
}
loop_func()
```

由于Scheme有尾部调用优化，不需要担心尾递归爆栈的问题（[前面文章解释了尾部调用优化](https://zhuanlan.zhihu.com/p/387239708)）。

### 用宏定义循环结构

Lisp中语言核心不包含while,for等循环结构，但是可以用宏定义它们。

可以把上面的循环模式定义为宏`my-while`

```scheme
(define-macro my-while
    (lambda (condition body)
        `(begin
            (define (loop-func) (
                if ,condition
                (begin
                    ,body
                    (loop-func)
                )
                ()
            ))
            (loop-func)
        )
    )
)
```

这里用到了半引用(quasiquote)，用 \` 表示。跟引用类似，但是里面遇到由`,`标记的表达式则不被引用而是被求值。`,@`将列表展开到外面的列表里。

```
`a  ==>  'a
`1  ==>  1
`,a  ==>  a
`(a ,b 1)  ==> (list 'a b 1)
`(a b ,@(c d) e)  ==> (list 'a 'b c d 'e)
```

使用`my-while`宏：

```scheme
(define i 0)
(my-while (< i 5) 
    (begin
        (display i)
        (set! i (+ i 1))
    )
)
```

###  可变参数的宏

可以改进一下这个宏，让循环体不必用`begin`包成一个表达式。Scheme中可以定义参数数量可变的函数(variadic function)，例如`(lambda (first-arg . rest-args) ...)`，第一个参数为first-arg，剩下的参数组成的列表是rest-args。利用参数可变函数，可以将这个宏改进为：

```scheme
(define-macro my-while
    (lambda (condition . statements)
        `(begin
            (define (loop-func) (
                if ,condition
                (begin
                    ,(cons 'begin statements)
                    (loop-func)
                )
                ()
            ))
            (loop-func)
        )
    )
)

(define i 0)
(my-while (< i 5)
    (display i)
    (set! i (+ i 1)) 
)
```

### 重名问题

这个宏存在不卫生(hygiene)的情况，这个宏生成的代码里面定义了`loop-func`，可能会有重名问题。解决办法是在宏内调用`gensym`(generate symbol)来生成一个不会重名的符号作为`loop-func`的名字：

```scheme
(define-macro my-while
    (lambda (condition . statements)
        (define loop-func-id (gensym))
        `(begin
            (define (,loop-func-id) (
                if ,condition
                (begin
                    ,(cons 'begin statements)
                    (,loop-func-id)
                )
                ()
            ))
            (,loop-func-id)
        )
    )
)
```



### 用宏实现模式匹配

前面看到Lisp中的宏能力非常强大，可以定义自己的语法。现在再来定义模式匹配(pattern match)。

Lisp中允许用`cons`将数据组合成元组，而且可以嵌套，甚至组成列表，但是想取出来就要用`car` `cdr`，有时还要判断里面嵌套的值是不是元组，写起来就比较蛋疼。

希望的模式匹配是这样的：

```scheme
(pattern-match x
    ( ((a b) (c d)) (+ a b c d) )
    ( (a b) (+ a b) )
    ( a a )
)
```

如果x为`(cons (cons 1 2) (cons 3 4))`则得到10，如果为`(cons 1 2)`则得到3，如果为`1`则得到1。

实现模式匹配宏，需要从两方面进行递归。识别模式`((a b) (c d))`的时候，用`(pair? x)`测试x是不是元组，然后递归地对`(car x)`匹配模式`(a b)`，对`(cdr x)`匹配模式`(c d)`。如果第一条匹配失败，则递归地匹配剩下的。

一个具体的代码实现：

```scheme
;生成一条模式匹配代码
(define (get-pattern-match-code
    value-symbol pattern mapping-expression on-failure-expression-getter
)
    (if (not (pair? pattern)) ;(x ...)这样的模式，直接识别成功
        `(let ((,pattern ,value-symbol)) ,mapping-expression)
        (let (
            (left-pattern (car pattern)) ;((a b) ...)这样的模式，左右分别递归识别
            (right-pattern (car (cdr pattern)))
        )
            `(if (pair? ,value-symbol)
                ,(get-pattern-match-code `(car ,value-symbol) left-pattern
                    (get-pattern-match-code
                        `(cdr ,value-symbol) right-pattern mapping-expression on-failure-expression-getter
                    )
                    on-failure-expression-getter
                )
                ,(on-failure-expression-getter) ;这一条识别失败
            )
        )
    )
)

;生成多条模式匹配代码
(define (pattern-match-trans value-symbol pattern-mappings)
    (if (equal? pattern-mappings ())
        `(error "no pattern satisfies") ;识别失败
        (let* (
            (first-mapping (car pattern-mappings)) ;取第一条
            (pattern (car first-mapping))
            (mapping-expressions (cdr first-mapping))
            (mapping-expression (cons 'begin mapping-expressions)) ;多条语句合成一个表达式
            (on-failure-expression-getter (lambda() (pattern-match-trans value-symbol (cdr pattern-mappings)))) ;如果匹配失败，则匹配剩下的模式
        )
            (get-pattern-match-code value-symbol pattern mapping-expression on-failure-expression-getter)
        )
    )
)

(define-macro pattern-match (lambda (value-expression . pattern-mappings)
    (define value-symbol (gensym))
    `(begin
        (let ((,value-symbol ,value-expression))
            ,(pattern-match-trans value-symbol pattern-mappings)
        )
    )
))
```



### 中缀表达式宏的改进

前面提到了中缀表达式宏，`(infix 1 + 2)`变成`(+ 1 2)`，现在对这个宏进行史诗级加强，让它支持更复杂的表达式，例如`(infix 2 * 3 + 5 * (6 + 7 * 8))`，而且要处理运算优先级。

这个代码实现中用到了前面的模式匹配，如果没有模式匹配代码会更复杂：

```scheme
;中缀变前缀
(define (infix-to-prefix expression)
    (pattern-match expression
        (;(1 + 2)变为(+ 1 2)
            (expression1 (operator expression2))
            (list operator (infix-to-prefix expression1) (infix-to-prefix expression2))
        )
        (;处理括号((1 + 2))
            (arg1 list-end)
            (list (infix-to-prefix arg1))
        )
        (;数值不变
            value value
        )
    )
)

;获得优先级,只处理加法和乘法
(define (get-precedence symbol)
    (if (equal? symbol '+) 1 2)
)

;修正优先级
(define (correct-precedence expression)
    (pattern-match expression
        (;(1 * 2 + 3)在上面被识别为(* 1 (+ 2 3))，修正为(+ (* 1 2) 3)
            (op1 (a ((op2 (b (c list-end))) list-end)))
            (
				if (> (get-precedence op1) (get-precedence op2))
				(list op2 (list op1 (correct-precedence a) (correct-precedence b)) (correct-precedence c))
				(list op1 (correct-precedence a) (list op2 (correct-precedence b) (correct-precedence c)))
			)
        )
        (;(op1 a e2)递归
            (op1 (a (e2 list-end)))
            (list op1 (correct-precedence a) (correct-precedence e2))
        )
        (;(e)递归
            (e list-end)
            (list (correct-precedence e))
        )
        (;数值不变
            value value
        )
    )
)

;移除多余括号
(define (remove-unnecessary-parentheses expression)
    (pattern-match expression
        (;递归移除括号
            (a (b c))
            (list a (remove-unnecessary-parentheses b) (remove-unnecessary-parentheses c))
        )
        (;(1)变为1
            (a list-end)
            (remove-unnecessary-parentheses a)
        )
        (value value)
    )
)

(define-macro infix
    ;定义lambda时参数不加括号，则多个参数组成一个列表输入
    (lambda expression 
        ;第一步中缀变为前缀，第二步修正优先级，第三步移除多余括号
        (remove-unnecessary-parentheses (correct-precedence (infix-to-prefix expression)))
    )
)
```



### 总结

上面可以看到Lisp中宏的功能非常强大，可以定义各种语法，非常灵活，但是这个灵活性是有代价的。如果宏生成的代码里面报错，可能很难确定是哪里出了问题，与之相反，Java语言十分啰嗦，但是报错更容易知道哪里出了问题。如果瞎定义宏，可以轻易让代码复杂化。

同时，Lisp语言中很多地方强制加括号，例如说JavaScript中的`x => x + 1`必须写作`(lambda (x) (+ x 1))`，多一层括号、少一层括号都不行，各种地方括号积累起来导致括号非常多，容易产生压迫感。







