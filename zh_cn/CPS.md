# [学习笔记] Continuation-passing Style

CPS (Continuation-passing Style)，是一种程序的特殊形式，这个形式下程序的后面部分被包成一个函数传给前面的程序，前面的程序计算完后将结果传给参数中的后面的程序，每个函数都输出整个程序最终的结果。

**CPS的核心思想是，将程序运行的某一个状态下的后面要执行的所有内容包装成一个函数，这个函数称为Continuation。**

例如说我有一段简单的Java程序

```java
static String func(Integer a) {
    Float b = a.floatValue();
    Double c = b.doubleValue();
    String d = b.toString() + c.toString();
    return d;
}
```

这个程序有三步计算，第一步将整数a转为单精度浮点数b，第二步将b转为双精度浮点数，第三步将b和c变成字符串连起来。每一步计算都用到了前面的值，同时第三步计算还用到了第一步计算的结果。这三步计算的类型都不一样是为了区分。

现在假如第一步计算已经执行完了，整个程序剩下的内容是第二步和第三步计算：

```java
Float b = a.floatValue();
----------
Double c = b.doubleValue();
String d = b.toString() + c.toString();
return d;
```

将整个程序剩下的内容包装成一个函数。后面的计算用到了第一步计算的结果b，将b通过参数传进去。新包装出来的函数的参数改名为`b_`是为了区分。

```java
Float b = a.floatValue();
return ((Function<Float, String>) (b_) -> {
    Double c = b_.doubleValue();
    String d = b_.toString() + c.toString();
    return d;
}).apply(b);
```

我们拆出来了一个新lambda表达式（新函数），这个新函数代表了程序运行完第一步后剩下的要运行的内容，也就是程序继续(continue)运行的内容，所以被称为continuation。continuation函数接受的输入是上一步计算的结果，输出却是整个程序的最终输出。

我们新包装出来的continuation，也可以一样将第一个运算和后面的运算分开。这次包装出来的函数只需要接受参数c，更前面算出来的b可以通过闭包来传递。

```java
Float b = a.floatValue();
return ((Function<Float, String>) (b_) -> {
    Double c = b_.doubleValue();
    return ((Function<Double, String>) (c_) -> {
        String d = b_.toString() + c_.toString();
        return d;
    }).apply(c);
}).apply(b);
```

这就是CPS变换！

不过这个代码不是很容易复用，对代码进行改造。尝试将前两步运算都抽成单独的函数：

```java
static String calculateB(Integer a, Function<Float, String> continuation) {
    return continuation.apply(a.floatValue());
}
static String calculateC(Float b, Function<Double, String> continuation) {
    return continuation.apply(b.doubleValue());
}
static String func3(Integer a) {
    return calculateB(a, b -> {
        return calculateC(b, c -> {
            String d = b.toString() + c.toString();
            return d;
        });
    });
}
```

`calculateB`中，他的计算结果没有直接输出，而是输给传进来的continuation函数，`calculateB`输出的是整个程序的最终结果。

注意到`calculateB`函数的类型中有String类型，String类型是整个程序的最终输出，但是这个函数里面没有用到String类型的任何操作，只是传递String类值，那么可以把它抽成泛型：

```java
static <R> R calculateB(Integer a, Function<Float, R> continuation) {
    return continuation.apply(a.floatValue());
}
static <R> R calculateC(Float b, Function<Double, R> continuation) {
    return continuation.apply(b.doubleValue());
}
```

`R`代表整个程序的最终输出类型。这就是CPS的标准形式。

接下来把第三步计算和整个程序也变成CPS的标准形式：

```java
static <R> R calculateB(Integer a, Function<Float, R> continuation) {
    return continuation.apply(a.floatValue());
}
static <R> R calculateC(Float b, Function<Double, R> continuation) {
    return continuation.apply(b.doubleValue());
}
static <R> R calculateD(Float b, Double c, Function<String, R> continuation) {
    return continuation.apply(b.toString() + c.toString());
}
static <R> R func(Integer a, Function<String, R> continuation) {
    return calculateB(a, b ->
        calculateC(b, c ->
            calculateD(b, c, continuation)
        )
    );
}
```

这个时候怎样调用`func`呢？我需要传入continuation，continuation代表执行`func`计算后程序要运行的内容，但是如果是单独地调用`func`没有后续要运行的东西了，那么就传入一个什么都不干的恒等变换函数`d -> d`。

```java
func(2333, d -> d)
```

这里的恒等变换函数也被称为CPS程序的停机(halt)。



回顾一下我们是怎样“重新发明”CPS变换的：将程序的第一步计算和剩下的计算分开，剩下的计算包装成continuation函数，continuation函数输入第一步计算的结果，输出整个程序的最终输出。然后把每一步计算抽出来，变成CPS的标准形式，然后把整个程序都变成CPS的标准形式。

---

下面是另一个例子：计算x平方加上y平方：

```java
static Integer squareSum(Integer x, Integer y) {
    Integer xSquare = x * x;
    Integer ySquare = y * y;
    Integer result = xSquare + ySquare;
    return result;
}
```

让这个代码变成CPS的形状：

```java
static <R> R cpsAdd(Integer a, Integer b, Function<Integer, R> continuation) {
    return continuation.apply(a + b);
}
static <R> R cpsMultiply(Integer a, Integer b, Function<Integer, R> continuation) {
    return continuation.apply(a * b);
}
static <R> R cpsSquareSum(Integer x, Integer y, Function<Integer, R> continuation) {
    return cpsMultiply(x, x, xSquare ->
        cpsMultiply(y, y, ySquare ->
            cpsAdd(xSquare, ySquare, continuation)
        )
    );
}
```

注意到最后的加法的时候，不仅用到了上一个计算的结果`ySquare`，还用到了上上个计算的结果`xSquare`，上面所有的计算结果都可以通过闭包传到后面。

同时又注意到，外层的计算是前面的计算，这里是先计算`xSquare`，再计算`ySquare`，但是这两个计算没有依赖关系，也没有副作用，顺序是可以随意的，这里却强行指定了顺序。先计算`ySquare`也是可以的：

```java
static <R> R cpsSquareSum(Integer x, Integer y, Function<Integer, R> continuation) {
    return cpsMultiply(y, y, ySquare ->
        cpsMultiply(x, x, xSquare ->
            cpsAdd(xSquare, ySquare, continuation)
        )
    );
}
```

CPS要求强行指定求值顺序。

---

CPS可以处理递归吗？可以。例如说计算阶乘的代码：

```java
static Integer factorial(Integer n) {
    if (n == 0) {
        return 1;
    } else {
        return factorial(n - 1) * n;
    }
}
```

让这个代码变成CPS的形状：

```java
static <R> R cpsFactorial(Integer n, Function<Integer, R> continuation) {
    if (n == 0) {
        return continuation.apply(1);
    } else {
        return cpsFactorial(
            n - 1, s -> cpsMultiply(s, n, continuation)
        );
    }
}
```

注意到，CPS化的代码中我不得不对中间结果`factorial(n - 1)`取名字，这里取的名字是`s`。



可以看到CPS变换后的程序有这些特点：

* 后面的程序被包成函数传给前面的程序。外层是前面执行的代码，内层是后面执行的代码。
* **整个程序的后面要执行的代码都包含在continuation中，continuation包含了程序运行的状态。**
* 函数输出的是整个程序的最终结果，不是我这一步计算的结果，这一步计算的结果被传进continuation里面了。
* 前面要执行的代码和后面要执行的代码存在先后顺序，顺序无关的代码也要强行指定顺序。
* Continuation中用到了前面计算的结果，上一步计算的结果在参数里，更前面的计算结果通过闭包获取。
* 每个中间表达式都要取名字。

### 栈与尾部调用

我们都知道，程序运行的时候需要一个栈。为什么需要栈？

* 每次函数调用结束后都要返回调用方的代码，因此每次函数调用之前，都将目前的程序地址放入栈，这样函数调用完成之后就可以从栈中取出返回时要跳到的地址。
* 传递函数调用的参数和返回值。
* 局部变量也存在栈中（参数也算局部变量）。局部变量可不可以存在全局变量里面？如果这个函数不会被递归调用，不会同时有两份局部变量数据需要存储，那么就可以。但是一般来说不知道一个函数是否会被递归调用，所以局部变量一般是存在栈上。
* 对于支持异常(Exception)的语言栈中还存跟异常有关的数据。

什么时候函数的调用不需要往栈上存东西？尾部调用(tail call)的时候。尾递归是指函数最后一步是直接返回一个函数调用的结果。

例如说计算阶乘的代码：

```C
int factorial(int n) {
    if (n == 0) {
        return 1;
    } else {
        int f = factorial(n - 1);
        return f * n;
    }
}
```

中间递归调用了factorial，但这不是尾部调用，因为他在获得调用结果后仍然要进行`f * n`的运算。将这个代码改造成尾部调用形式：

```C
int factorialTC(int n, int product) {
    if (n == 0) {
        return product;
    } else {
        return factorialTC(n - 1, product * n);
    }
}
int factorial(int n) {
    return factorialTC(n, 1);
}
```

计算阶乘是将n,n-1,n-2,...,2,1这些数乘起来，`product`参数代表已经乘起来的数，接下来每一步递归都是对它继续乘，直到最后将乘积输出。这里`factorialTC`中调用`factorialTC`的地方是尾部调用，他在调用`factorialTC`之后直接输出结果，而不是做更多运算。

在进行尾部调用的时候，可以进行尾部调用优化(Tail call optimization)，这个时候目前的局部变量是不需要存的，因为尾部调用返回后不需要用到局部变量进行运算，因此可以将局部变量从栈上去掉。同时，尾部调用后直接将值返回给上层，不需要先返回到本层再返回到上层，所以本层的调用处代码地址也可以从栈上去掉。尾部调用将栈上本层的信息去掉，再放入新一层的信息。可以看到，尾部调用不需要消耗额外的栈空间。

CPS变换后，所有的调用都变成尾部调用，因此可以进行尾部调用优化防止爆栈。



### 命令式语言的CPS变换

前面的CPS变换的例子都是函数式编程风格的，没有使用变量，用递归代替循环。那么带有赋值和循环的命令式语言程序也可以做CPS变换吗？答案是可以。



首先看比较简单的情况，只有局部变量赋值，没有分支、循环：

```C
int b = a + 1;
a = b + 1;
b = a + a;
```

这个时候，采用静态单赋值形式(Static single assignment form,SSA)，每次对变量的修改，都改成定义新变量，用到修改后的变量就是用到新变量：

```C
int b = a + 1;
int a1 = b + 1;
int b1 = a1 + a1;
```

然后就可以进行上述的CPS变换了。



下面看更加复杂的有循环的例子，一个对数组求和的C语言程序：

```C
int func(int* array, int length) {
    int sum = 0;
    int i = 0;
    while (i < length) {
        sum += array[i];
        i++;
    }
    return sum;
}
```

将这个代码拆成更原始的形式，用`goto`代替循环：

```C
int func(int* array, int length) {
    int sum = 0;
    int i = 0;
    goto loop;
    
loop:
    if (i < length) {goto loopBody;} else {goto end;}
    
loopBody:
    sum += array[i];
    i++;
    goto loop;
    
end:
    return sum;
}
```

发现代码被分成了4个代码块，第2、3、4个代码块的开始是一个标签(label)，也就是`goto`可能跳到的地方，每个代码块末尾是一个`goto`语句（第一个`goto`语句可以去掉，这里加上是为了更好解释）。

下一步是将每个代码块变成函数，每个`goto`都变成函数调用：（通过参数传递局部变量）

```C
int func(int* array, int length) {
    int sum = 0;
    int i = 0;
    return loop(array, length, sum, i);
}

int loop(int* array, int length, int sum, int i) {
    if (i < length) {
        return loopBody(array, length, sum, i);
    } else {
        return end(array, length, sum, i);
    }
}

int loopBody(int* array, int length, int sum, int i) {
    sum += array[i];
    i++;
    return loop(array, length, sum, i);
}

int end(int* array, int length, int sum, int i) {
    return sum;
}

```

`loopBody`中有赋值，让它变成SSA形式：

```C
int loopBody(int* array, int length, int sum, int i) {
    int sum1 = sum + array[i];
    int i1 = i + 1;
    return loop(array, length, sum1, i1);
}
```

我们把一个带有赋值和循环的命令式程序变成了函数式程序，然后就可以CPS变换了！注意到修改后的调用都是尾部调用，这种情况下可以认为尾部调用和`goto`是等价的。





### 运行状态的保存与恢复

我们有些时候会需要让程序暂停执行，过一会再重新继续运行。这就需要存储程序运行状态。操作系统进行的线程切换就是在干这种事，操作系统所存储的程序运行状态有寄存器、程序运行位置、栈指针等信息。对CPS程序而言，continuation保存了程序运行的状态，存储程序运行状态只需要存储continuation函数。

还有哪些地方需要让程序暂停执行呢？例如：

* 进行耗时的IO操作，例如从网上下载一个文件、从命令行读字符。
* 界面上弹出对话框，程序暂停，需要用户进行一定操作后对话框关掉，程序继续运行。
* 我有一个代码遍历某个数据结构中的元素，现在我需要一个一个地、间歇性地取出元素。

这些地方往往可以使用Coroutine。

例如说，用JavaScript语言，从网上下载用户列表，下载完再下载第一个用户的具体信息，然后获得第一个用户的头像网址，然后下载头像，这些都做完之后在网页上显示用户信息，使用async/await可以这么写：

```javascript
let userList = await downloadUserList(url);
let firstUserInformation = await downloadUserInformation(userList[0]);
let avatar = await downloadImage(firstUserInformation.avatarLink);
...
```

但是在以前没有async/await的时候，采用的方法是传入一个回调函数(callback)，等下载好之后调用你给的回调函数：

```javascript
downloadUserList(url, function(userList) {
    downloadUserInformation(userList[0], function(firstUserInformation) {
        downloadImage(firstUserInformation.avatarLink, function(avatar) {
            // ....
        });
    });
});
```

里面层层嵌套，出现了回调地狱(callback hell)。这也是CPS变换！CPS将普通的直观代码变成了回调地狱！传进去的回调函数是continuation。

这个代码还可以用Promise的方法来写：

```javascript
downloadUserList(url).then(async function(userList) {
    return await downloadUserInformation(userList[0]);
}).then(async function(firstUserInformation) {
    return await downloadImage(firstUserInformation.avaterLink)
});
```

这里的then相当于`flatmap`，用到了Monad的操作。可以看到CPS计算过程也是Monad。



### CPS计算过程也是Monad

例如说对于CPS下的加法运算：

```java
static <R> R cpsAdd(Integer a, Integer b, Function<Integer, R> continuation) {
    return continuation.apply(a + b);
}
```

给他加法运算的两个参数，这个时候我们只要再给他一个continuation就能执行计算。将他的前两个参数a、b确定，得到一个新的函数：

```java
continuation -> cpsAdd(1, 2, continuation)
```

这个函数可以代表这个CPS计算的“结果”。

在Haskell中可以直接柯里化。

```haskell
add :: Integer -> Integer -> Integer
add a b = (a + b)
cpsAdd :: Integer -> Integer -> (Integer -> r) -> r
cpsAdd a b continuation = continuation (a + b)

add1With2 :: (Integer -> r) -> r
add1With2 = cpsAdd 1 2
```

这个表示CPS计算的结果的函数是Monad。在确定了整个程序的最终输出类型`R`后，目前CPS计算的结果类型T是它的泛型参数：

```java
interface CPSComputation<R, T> {
    R run(Function<T, R> continuation);
}
```

cps下的加法可以写成这样：

```java
static <R> CPSComputation<R, Integer> cpsAdd(Integer a, Integer b) {
    return continuation -> continuation.apply(a + b);
}
cpsAdd(1, 2)
```

这个`CPSComputation`可以代表一个CPS运算的结果，但是他里面没有直接存结果，需要输入一个continuation才能进行计算获得结果，这是一个计算过程。

这个东西可以进行`flatmap`操作：

```java
static <R, A, B> CPSComputation<R, B> flatmap(
    CPSComputation<R, A> aComputation,
    Function<A, CPSComputation<R, B>> func
) {
    return (Function<B, R> continuation) -> {
        return aComputation.run(aValue -> {
            CPSComputation<R, B> bComputation = func.apply(aValue);
            return bComputation.run(continuation);
        });
    };
}
```

这个`flatmap`可以串联多个CPS计算过程。`func`中获取前面的计算结果并合成新`CPSComputation`，`flatmap`操作将这两个计算过程合并成一个。

最开始的例子可以写成这样：

```java
static <R> CPSComputation<R, Float> calcB(Integer a) {
    return continuation -> continuation.apply(a.floatValue());
}
static <R> CPSComputation<R, Double> calcC(Float b) {
    return continuation -> continuation.apply(b.doubleValue());
}
static <R> CPSComputation<R, String> calcD(Float b, Double c) {
    return continuation -> continuation.apply(b.toString() + c.toString());
}
public static <R> CPSComputation<R, String> func(Integer a) {
    return flatmap(
        calcB(a),
        b -> flatmap(
            calcC(b),
            c -> calcD(b, c)
        )
    );
}
```



Haskell代码：

```haskell
newtype CPSComputation r t = CPSComputation_ ( (t -> r) -> r )
runCPSComputation (CPSComputation_ cpsFun) continuation = cpsFun continuation
```

（这个东西在Haskell官方叫`Cont`，是`Continuation`的缩写，不过我认为这个名字也不合适，因为它代表的是一个CPS下的计算过程而不是continuation，输给他的函数才叫continuation。）

实现Monad：

```haskell
instance MyMonad (CPSComputation r) where
  wrap value = CPSComputation_ (\f -> f value)
  flatmap (CPSComputation_ aComputation) func =
    CPSComputation_ (\continuation ->
      aComputation (\a -> runCPSComputation (func a) continuation)
    )
```

用do语法糖写最初的例子：

```haskell
calculateB :: Integer -> (Float -> r) -> r
calculateB a continuation = continuation (fromIntegral a)
calculateC :: Float -> (Double -> r) -> r
calculateC b continuation = continuation (realToFrac b)
calculateD :: Float -> Double -> (String -> r) -> r
calculateD b c continuation = continuation ((show b) ++ (show c))

func :: Integer -> (CPSComputation r String)
func a = do
  b <- CPSComputation_ (calculateB a)
  c <- CPSComputation_ (calculateC b)
  d <- CPSComputation_ (calculateD b c)
  return d
```





### Call with Current Continuation (call-cc)

前面提到continuation存储了整个程序的运行状态，可以把continuation存起来，然后某个时候再调用它，这样就可以做到程序的跳转。例如说上面最开始的例子，我想让第一步计算如果遇到输入为2333就出错，出错后直接跳过后面的所有计算，输出"Error!"，模拟异常功能：

```java
static <R> R calculateBWithExit(
    Integer a,
    Function<String, R> exitContinuation,
    Function<Float, R> continuation
) {
    if (a == 2333) {
        return exitContinuation.apply("Error!");
    }
    return continuation.apply(a.floatValue());
}
static <R> R calculateC(Float b, Function<Double, R> continuation) {
    return continuation.apply(b.doubleValue());
}
static <R> R calculateD(Float b, Double c, Function<String, R> continuation) {
    return continuation.apply(b.toString() + c.toString());
}
static <R> R func(Integer a, Function<String, R> continuation) {
    Function<String, R> currentContinuation = continuation;
    
    return calculateBWithExit(
        a,
        currentContinuation,
        b -> calculateC(b, c -> calculateD(b, c, continuation))
    );
}
```

我在`func`中将目前的continuation存了下来，称为current continuation，然后计算B的代码中用到了这个外面的current continuation，如果出错直接忽略原本的continuation，调用外面的continuation，输出"Error!"，跳过后面的计算c和d的代码。

Scheme语言中有一个函数`call-with-current-continuation`，简称`call/cc`，他传入一个函数f，然后把current continuation传给函数f。例如`((call/cc f) e2)`中，将`(call/cc f)`替换成一个“洞”(hole)，也就是continuation的参数，获得current continuation：`(lambda (c) (c e2)`，然后把这个传给f。最后整个表达式`((call/cc f) e2)`的值，就是f里面调用continuation的值。这个跟上面的保存current continuation的原理是类似的，不过不需要显式将程序CPS变换。

在Java语言里写一个类似的东西：

```java
static <R, T, InnerT>
CPSComputation<R, T> callWithCurrentContinuation(
    Function<
        Function<T, R>,
        CPSComputation<R, T>
    > computationGenerator
) {
    return (Function<T, R> continuation) -> {
        CPSComputation<R, T> computation = computationGenerator.apply(continuation);
        return computation.run(continuation);
    };
}
```

传进去一个`computationGenerator`，它接受current continuation输出一个`CPSComputation`。

可以这么用：

```java
static <R> CPSComputation<R, Float> calcBWithExit(
    Integer a, Function<String, R> exitFunc
) {
    if (a == 2333) {
        return continuation -> exitFunc.apply("Error!");
    }
    return continuation -> continuation.apply(a.floatValue());
}
public static <R> CPSComputation<R, String> func(Integer a) {
    return callWithCurrentContinuation(exitContinuation ->
        flatmap(
            calcBWithExit(a, exitContinuation),
            b -> flatmap(
                calcC(b),
                c -> calcD(b, c)
            )
        )
    );
}
```

不过Haskell的`callCC`不是这个形式的。Haskell中的`callCC`是

```haskell
-- 官方
callCC :: ((a -> Cont r b) -> Cont r a) -> Cont r a
callCC f = cont $ \h -> runCont (f (\a -> cont $ \_ -> h a)) h
-- 或者：
callWithCurrentContinuation :: ((a -> CPSComputation r b) -> CPSComputation r a) -> CPSComputation r a
callWithCurrentContinuation func = CPSComputation_ 
  (\continuation ->
    runCPSComputation (
      func (\aValue -> CPSComputation_ (\innerContinuation -> continuation aValue))
    ) continuation
  )
```

转换成Java代码:

```java
static <R, T, InnerT>
CPSComputation<R, T> haskellCallWithCurrentContinuation(
    Function<
        Function<T, CPSComputation<R, InnerT>>,
        CPSComputation<R, T>
    > computationGenerator
) {
    return (Function<T, R> continuation) -> {
        Function<T, CPSComputation<R, InnerT>> exitInstructionGenerator =
            (T returnValue) ->
                (Function<InnerT, R> innerContinuation) -> {
                    return continuation.apply(returnValue);
                };
        
        CPSComputation<R, T> computation = computationGenerator.apply(exitInstructionGenerator);
        return computation.run(continuation);
    };
}
```

跟我上面的`callWithCurrentContinuation`差别在于`Function<T, R>`变成了`Function<T, CPSComputation<R, InnerT>>`，把current continuation包装成了一个输入值获得`CPSComputation`的函数。使用方法稍微变一下：

```java
static <R> CPSComputation<R, Float> hsCalculateBWithExit(
    Integer a, Function<String, CPSComputation<R, Float>> exitFunc
) {
    if (a == 2333) {
        return exitFunc.apply("Error!");
    }
    return continuation -> continuation.apply(a.floatValue());
}
public static <R> CPSComputation<R, String> hsFuncWithExit(Integer a) {
    return haskellCallWithCurrentContinuation(
        (Function<String, CPSComputation<R, Float>> exitFunc) ->
            flatmap(
                hsCalcBWithExit(a, exitFunc),
                b -> flatmap(
                    calcC(b),
                    c -> calcD(b, c)
                )
            )
    );
}
```

















