# idris语言(1):初探依赖类型

idris语言是一个类型系统极为强大的语言，可以将类型当成值进行运算，并且有依赖类型(dependent type)，可以让类型依赖于运行时的值，并进行定理证明等复杂操作。

> 注：idris中类型和函数都是“原子”的，对给定类型或函数只能去用它组合，不能去拆开它进行模式匹配。

### 如何在Windows中的WSL安装idris

直接在Windows中安装idris比较麻烦（可能因为idris社区中很少有人用Windows），不过Windows可以很容易安装Linux子系统（WSL）。只要在管理员权限下运行`wsl --install`就可以轻松安装（注：需要梯子）。在WSL中用`/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"`安装brew，然后运行brew显示推荐的命令，再然后`brew install idris2`就可以轻松安装idris。具体可查看[idris官方文档](https://idris2.readthedocs.io/en/latest/tutorial/starting.html)。

### 依赖类型(Dependent Type)

依赖类型，就是指一些依赖于值的类型。C++中普通的类型`int`本身不依赖任何值，`std::array<int, 5>`依赖于值`5`，属于依赖类型。

不过C++的依赖类型只允许依赖编译期确定的常量，不能依赖运行时才确定的变量：

```C++
int n;
std::cin >> n; // n要运行时才确定
std::array<int, n> myArray; // C++不允许将运行时的值用于依赖类型
```

而idris语言中的类型可以依赖运行时才确定的值。通过依赖类型可以实现非常灵活的类型，利用这些灵活的类型可以更好地用类型保证程序的正确性（进行定理证明），同时也能更好地帮助IDE进行补全。

### 声明代数数据类型(ADT)

Idris与Haskell很像，都是纯函数式语言，语法也很像。（[前面对Haskell的学习笔记](https://zhuanlan.zhihu.com/p/387237772)，[前面的ADT的学习笔记](https://zhuanlan.zhihu.com/p/517941138?)）（[idris官方教程](https://idris2.readthedocs.io/en/latest/tutorial/index.html)） （idris与haskell的语法区别包括：表明类型用`:`，lambda表达式中箭头为`=>`）

例如要定义普通的列表（链表），跟Haskell很像

```idris
data MyList a = EmptyList | ListNode a (MyList a)
```

将上述代码写入一个代码文件，例如`hello.idr`，然后命令`idris hello.idr`将会进入idris环境，可以查看`MyList`的类型：

```
Main> :t MyList
Main.MyList : Type -> Type
```

`MyList`本身是一个泛型类型，而这个泛型类型的类型是`Type -> Type`，就是一个函数，这个函数输入一个类型输出一个类型。在idris中，类型是一等公民，类型也可以当做值输入输出函数。

用GADT的方式去定义这个列表：（后面的依赖类型大都是用GADT的方式去定义的）

```idris
data MyList : Type -> Type where
    EmptyList : MyList a
    ListNode : a -> (MyList a) -> (MyList a)
```

这个GADT的定义语法和Haskell中不一样。其中`Type -> Type`代表我们要定义的是一个类型函数，输入一个类型，输出一个类型。`EmptyList`是一个实例，`ListNode`是一个构造函数，其中的`a`会自动被推导为泛型参数（不用主动声明泛型参数）。

对其中的参数也可以起名字：

```
data MyList : (t : Type) -> Type where
    EmptyList : MyList a
    ListNode : (first : a) -> (rest : MyList a) -> (MyList a)
```

看一下EmptyList和ListNode的类型都是什么：

```
Main> :t EmptyList
Main.EmptyList : MyList a
Main> :t ListNode
Main.ListNode : a -> MyList a -> MyList a
```

`EmptyList`的类型是`MyList a`，泛型参数`a`将会在使用的时候自动推导。`ListNode`是一个构造函数，输入一个`a`类型的值，再输入一个`MyList a`类型的值，得到一个`MyList a`类型的值。这个构造函数不仅是一个函数，还能在模式匹配中用。

**idris标准库中`List`是这样的，其中中缀写法的`::`相当于`ListNode`，`Nil`相当于`EmptyList`**

```
data List a = Nil | (::) a (List a)
```

idris中自然数不是内置的，是手动定义的。自然数有两种：0，以及某个自然数的后继(successor)

```idris
data NaturalNumber : Type where
    Zero: NaturalNumber
    Successor: NaturalNumber -> NaturalNumber
```

0就是`Zero`，1就是`Successor Zero`，2就是`Successor (Successor Zero)`，以此类推。（这是皮亚诺公理定义的自然数）

**在idris标准库中是将`Zero`简称为`Z`，`Successor`简称为`S`，`NaturalNumber`简称为`Nat`。**

根据自然数的基本定义，就可以基于模式匹配来实现自然数加法和乘法

```idris
plus : Nat -> Nat -> Nat
plus Z     y = y
plus (S k) y = S (plus k y)

mult : Nat -> Nat -> Nat
mult Z     y = Z
mult (S k) y = plus y (mult k y)
```

第二行表示0 + y = y，第三行表示k的后继加y等于(k + y)的后继。通过对第一个参数递归进行模式匹配，可以实现对所有自然数的加法。乘法也是类似，0 * x = 0，k的后继乘y等于k乘y加y。

idris中写出自然数可以自动转为它里面的自然数表示，`2`会自动转为`S (S Z)`。实际上idris在运行时会将将类似自然数结构的类型用整数表示，以提高性能。

### 声明一个依赖类型：有上限的自然数

如果我想要在类型层面就保证数组访问不会越界，首先要定义一种特殊的整数，这种整数不能超过一个值，例如说对长度为5的数组，索引必须是最大为4的自然数。

这时候需要定义一个有上限的自然数`Fin n`表示小于n的自然数(Fin指finite)。我们希望`Fin 5`只能包括0,1,2,3,4。

在没有接触idris之前，一个简单的想法是从集合的角度出发，用一个谓词去筛选集合元素，一个小于5的整数可以写成集合`{k ∈ N | k < 5}`，相当于是整数的集合中小于5的元素构成的集合，这种东西叫做Refinement Type(F*语言中有这种东西)。但是idris中不是这样的，依赖类型是构造出来的而不是用集合操作或者谓词限制得来的。而这里的“构造”就是指定义特殊的代数数据类型。

那么，要怎样构造出有上限的自然数？有上限的自然数和自然数的构造是一样的。自然数中，0、1、2分别是`Z`、 `S Z`、 `S (S Z)`，那么有上限自然数的结构也是一样的，将Z改名为FZ，S改名为FS，则0、1、2分别是`FZ`、`FS FZ`、`FS (FS FZ)`。

那么先初步写出大概的结构，`?`代表还不确定的地方（这个不是idris代码）：

```
data Fin : Nat -> Type where
    FZ: ?
    FS: ? -> ?
```

对于零`FZ`，它属于`Fin 1`,`Fin 2`,`Fin 3`等类型，但不属于`Fin 0`。`S k`这个构造刚好可以表示1、2、3等自然数但不包括0（k是自然数）。于是填入FZ的类型：

```
data Fin : Nat -> Type where
    FZ: Fin (S k)
    FS: ? -> ?
```

注意到这里`FZ`的类型有一个自由变量`k`，也就是说`FZ`的类型是不确定的，但在使用`FZ`的时候idris可以将`k`推导出来，跟Haskell中泛型参数自动推导一样。

如果`a`属于`Fin k`，也就是`a < k`，那么`a + 1 < k + 1`，`FS a`属于`Fin (S k)`：

```
data Fin : Nat -> Type where
    FZ: Fin (S k)
    FS: (Fin k) -> (Fin (S k))
```

从这个类型定义来看，`FZ`可以是`Fin 1`, `Fin 2`, `Fin 3`等类型，而`FS FZ`的类型，根据前面`FZ`类型的不同，分别是是`Fin 2`, `Fin 3`, `Fin 4`，而`FS (FS FZ)`的类型，分别是`Fin 3`, `Fin 4`, `Fin 5`。用一个表格来表示：

| `FZ`的类型 | `FS FZ`的类型 | `FS (FS FZ)`的类型 |
| ---------- | ------------- | ------------------ |
| `Fin 1`    | `Fin 2`       | `Fin 3`            |
| `Fin 2`    | `Fin 3`       | `Fin 4`            |
| `Fin 3`    | `Fin 4`       | `Fin 5`            |

`FS (FS FZ)`表示2，它的类型可以是`Fin 3`, `Fin 4`, `Fin 5`等。如果指定`FS (FS FZ)`的类型为`Fin 5`，那么最内层的`FZ`会被推导为`Fin 3`。如果指定`FS (FS FZ)`的类型为`Fin 6`，`FZ`会被推导为`Fin 4`。

可以看到，光定义一个有上限的自然数，就已经有一点复杂了。

### 声明一个数组，但是在类型里确定长度

C++中可以在类型里面指定数组长度，例如`std::array<int, 3>`，但是长度必须是编译期就能知道的常量，而idris中这个长度可以不是常量，可以是一个在运行时才确定的值（这个长度值不一定在运行的时候存在内存中，长度可能只是作为编译期类型检查使用的表达式）。

普通的类型中不包括长度的列表可以这么定义：

```
data MyList : Type -> Type where
    EmptyList : MyList a
    ListNode : a -> (MyList a) -> (MyList a)
```

其中`a`为泛型参数，表示元素的类型。

在类型参数中包括长度的列表`MyVector`，是这么定义的：

```
data MyVector : Nat -> Type -> Type where
	EmptyVector : MyVector Z a
	VectorNode : a -> MyVector k a -> MyVector (S k) a
```

`EmptyVector`是长度为0的列表`MyVector Z a`，其中`a`为存储的元素的类型。`VectorNode`包括一个`a`元素，一个长度为k的列表，合成一个长度为k+1的列表。`VectorNode 1 (VectorNode 2 EmptyVector)`表示一个包含1,2两个元素的列表，类型为`MyVector 2 Integer`。

在标准库中这个东西叫`Vect`：

```
data Vect : Nat -> Type -> Type where
   Nil  : Vect Z a
   (::) : a -> Vect k a -> Vect (S k) a
```

`(::)`表示一个中缀表达式，`VectorNode 1 (VectorNode 2 EmptyVector)`和`1 :: 2 :: Nil`是一样的，写起来更方便。

### 如何保证数组访问不越界

定义一个函数`index`来获取列表中的一个元素。如果我们用没有上限的自然数`Nat`去定义：

```idris
index : Nat -> Vect n a -> a
index Z (first :: rest) = first
index (S i) (first :: rest) = index i rest
```

会导致编译错误，函数定义没有涵盖所有情况：

```
Missing cases:
    index 0 []
    index (S _) []
```

对于上面的index函数，如果输入的索引数值越界，那么递归到最后会遇到第二个参数是空列表的情况，而空列表中取不到任何元素，在其他语言中这会导致运行时出异常，而idris语言中我们可以通过更灵活的类型限定，使用`Fin`保证输入的索引值不越界：

```idris
index : Fin n -> Vect n a -> a
index FZ (first :: rest) = first
index (FS i) (first :: rest) = index i rest
```

此时没有编译错误了。可以正常使用这个函数，`index (FS FZ) (1::2::3::Nil)`得到2，而`index (FS FZ) (1::Nil)`导致编译错误。

为什么第一个用`Nat`的`index`函数无法通过编译，而第二个用`Fin`的`index`函数能通过编译？为什么第一个`index`定义有缺失情况而第二个`index`定义没有缺失的情况？

首先，上面的index函数省略了一些东西，函数类型中的变量`n`一般是idris自动推导的，而我们可以在前面的`{}`中手动写明隐含变量：

```
index : Fin n -> Vect n a -> a
index {n=S k} FZ (first :: rest) = first
index {n=S k} (FS i) (first :: rest) = index i rest
```

重新看`Fin`的定义，可以看到对`Fin Z`是没有定义的(显然没有自然数小于0)：

```idris
data Fin : Nat -> Type where
    FZ: Fin (S k)
    FS: (Fin k) -> (Fin (S k))
```

因此`index`函数中的`n`只能是`S k`的形式。

先看`index`第一个情况的定义：

```
index {n=S k} FZ (first :: rest) = first
```

`n = S k`，所以第二个参数的类型为`Vect (S k) a`，然后根据`Vect`的定义：

```
data Vect : Nat -> Type -> Type where
   Nil  : Vect Z a
   (::) : a -> Vect k a -> Vect (S k) a
```

`Vect (S k) a`形式的定义只能符合第二种定义，不符合第一种，所以直接将第二个参数解构成`(first :: rest)`是没有遗漏情况的。

再看`index`第二个情况的定义：

```
index {n=S k} (FS i) (first :: rest) = index i rest
```

跟上一个同理，将第二个参数解构成`(first :: rest)`是没有遗漏情况的。这个时候`rest`的类型是`Vect k a`，而`i`的类型是`Fin k`，递归调用也通过了类型检查。

这就是idris通过编译时证明保证数组访问不会越界的原理。在C语言中数组访问越界可能直接崩掉，也可能导致黑客通过缓冲区溢出植入恶意代码，而像Java、C#等内存安全语言中可能会抛出异常。idris中通过类型层面的证明就保证数组访问不会越界，但代价就是你需要提供证明，提供界内整数`Fin n`。

### 更多列表操作

可以实现对列表元素的`map`操作：

```
map: (a -> b) -> Vect n a -> Vect n b
map {n=Z}   f Nil             = Nil
map {n=S Z} f (first :: rest) = (f first) :: (map f rest)
```

将两个列表连接`concat`：

```idris
add : Nat -> Nat -> Nat
add Z     n = n
add (S n) m = S (add n m)

concat : Vect n a -> Vect m a -> Vect (add n m) a
concat {n=Z}   Nil       ys = ys
concat {n=S k} (x :: xs) ys = x :: (concat xs ys)
```

可以看到`{n=Z}`刚好对应`add`的第一个情况，`{n=S k}`刚好对应`add`的第二个情况。

但如果我修改`add`的定义，改为对第二个参数进行模式匹配：

```
add : Nat -> Nat -> Nat
add m     Z = m
add m (S n) = S (add m n)
```

那么`concat`就无法通过编译：

```
Error: While processing right hand side of concat. Can't solve constraint between: m and add 0 m.
```

这就需要进行更多证明才能通过编译。

### 尝试实现`filter`

下面，我想实现列表的`filter`操作。由于filter后的长度是在运行时才确定的，这里使用另一个变量表示长度：

```
myfilter: (a -> Bool) -> Vect n a -> (Vect m a)
myfilter f Nil = Nil
myfilter f (first :: rest) = if (f first) then (first :: (myfilter f rest)) else (myfilter f rest)
```

然后出现编译错误：

```
Error: While processing right hand side of myfilter. When unifying:
    Vect 0 a
and:
    Vect m a
Mismatch between: 0 and m.
```

为什么idris不允许这样写？因为函数输出类型中的隐含变量`m`默认都应该是能推导出来的，而这个时候`m`推导不出来，`m`是一个运行时才知道的值，跟列表里的内容、提供的函数相关。

前面写的`Vect n a`中，`n`默认是以编译时表达式的形式存在的，在运行时不会去存储`n`，编译时完成类型检查后`n`就被擦除了（跟Java中的泛型擦除类似）。

既然`m`不能在编译时推导出来，可以在运行时得出`m`并存起来，让`m`依赖于运行时存起来的一个值：

### 运行时存储列表的长度

定义`VectWithLen`，里面存储列表长度`len`，以及一个该长度的列表`Vect len a`

```
data VectWithLen: Type -> Type where
    VectWithLen_: (len: Nat) -> (Vect len a) -> VectWithLen a
```

与`Vect`相比，它在里面存储了列表的长度，于是在类型里不需要提供列表长度值了。

让`filter`返回`VectWithLen`，那么就可以写出通过编译的`filter`了：

```
myfilter: (a -> Bool) -> Vect n a -> VectWithLen a
myfilter f Nil = VectWithLen_ Z Nil
myfilter f (first :: rest) = let
    (VectWithLen_ restFilteredLen restFiltered) = (myfilter f rest)
    in if (f first) 
        then (VectWithLen_ (S restFilteredLen) (first :: restFiltered)) 
        else (VectWithLen_ restFilteredLen restFiltered)
```

对于空列表，filter后也为空，其长度为Z：

```
myfilter f Nil = VectWithLen_ Z Nil
```

对于一个非空列表`first :: rest`，首先

```
let (VectWithLen_ restFilteredLen restFiltered) = (myfilter f rest) in ...
```

将rest部分调用myfilter，获得rest部分过滤后的内容`restFiltered`和长度`restFilteredLen`

对`first`元素测试是否通过过滤，如果通过过滤则返回`first :: restFiltered`，其长度为`S restFilteredLen`，如果不通过过滤就直接返回rest部分过滤的内容：

```
if (f first) 
        then (VectWithLen_ (S restFilteredLen) (first :: restFiltered)) 
        else (VectWithLen_ restFilteredLen restFiltered)
```

### 依赖对(Dependent Pair)

我为了将`Vect n a`中类型参数n存下来，单独定义了`VectWithLen`，如果对其他类型也这么做就要定义很多类似`VectWithLen`的存储类型参数的新类型。idris提供依赖对(Dependent Pair，"对"指pair)，让我们不需要定义类似`VectWithLen`这样的类型。

```
data DPair : (a : Type) -> (p : a -> Type) -> Type where
   MkDPair : {p : a -> Type} -> (x : a) -> p x -> DPair a p
```

依赖对类型参数中包含类型为`a -> Type`的函数`p`，内部包含两个成员，第一个成员是`a`类型的值`x`，第二个是`p x`类型的一个值。第二个成员的类型`p x`可以依赖第一个成员`x`。

这时候就可以用DPair弄一个存储了长度的列表，例如说一个长度为2的列表可以这么写：

```
vec : DPair Nat (\n => Vect n Int)
vec = MkDPair 2 [3, 4]
```

其中`\n => Vect n Int`是一个lambda表达式。

采用语法糖，写得更简单：

```
vec : (n : Nat ** Vect n Int)
vec = (2 ** [3, 4])
```

进一步简写：

```
vec : (n ** Vect n Int)
vec = (_ ** [3, 4])
```

采用依赖对，重写`filter`函数：

```
myfilter: (a -> Bool) -> Vect n a -> (k ** Vect k a)
myfilter f Nil = (Z ** Nil) -- 这里一定要加括号
myfilter f (first :: rest) = let
    (restFilteredLen ** restFiltered) = (myfilter f rest)
    in if (f first) 
        then ((S restFilteredLen) ** (first :: restFiltered)) 
        else (restFilteredLen ** restFiltered)
```

### 实现类型安全printf

C语言的`printf`不是类型安全的。例如说，可以在要传字符串的地方传一个整数，然后在运行时崩掉：

```
printf("%s", 2333)
```

由于C语言的类型系统很简单，类型系统无法去解析字符串并检查参数类型。但是Idris拥有非常强大的类型系统，可以解析字符串并确保传入的参数类型正确。

我希望的在idris中的printf是这样调用的：

```
printf "hello %d %s" 3 "worlds"
```

输出 "hello 3 worlds"。

然后如果输入错误参数类型：

```
printf "hello %d %s" "worlds" 3
```

将会有编译错误。

在`printf "hello %d %s" 3 "worlds"`中，`printf "hello %d %s"`将会得到一个函数，后面的参数柯里化喂给这个函数。

如果printf没有参数，则返回一个字符串，例如`printf "hello world"`的返回值就是`"hello world"`。如果printf有参数，则返回返回一个函数，这个函数依次接受参数，最终返回`String`。

（注：这里`printf`只返回结果字符串而不会向命令行输出。输出用`putStrLn`）

```
printf "hello %d %s" : Integer -> String -> String
printf "hello %s %d" : String -> Integer -> String
printf "hello world" : String
```

目前支持的格式有：`%s`字符串，`%d`整数。除此之外`%%`转意为`%`。那么先定义格式数据：

```idris
data PrintfFormatUnit: Type where
    LiteralChar: Char -> PrintfFormatUnit
    IntegerArgument: PrintfFormatUnit
    StringArgument: PrintfFormatUnit

parsePrintfFormat: (List Char) -> (List PrintfFormatUnit)
parsePrintfFormat Nil = Nil
parsePrintfFormat ('%' :: 's' :: rest) = ((StringArgument) :: (parsePrintfFormat rest))
parsePrintfFormat ('%' :: 'd' :: rest) = ((IntegerArgument) :: (parsePrintfFormat rest))
parsePrintfFormat ('%' :: '%' :: rest) = ((LiteralChar '%') :: (parsePrintfFormat rest))
parsePrintfFormat (firstChar :: rest) = ((LiteralChar firstChar) :: (parsePrintfFormat rest))
```

printf格式包括`PrintfFormatUnit`列表，`PrintfFormatUnit`可以是一个固定字符、一个整数参数或一个字符串参数。`parsePrintfFormat`将字符串解析为`PrintfFormatUnit`列表。

>  对字符串用`List Char`类型，因为idris似乎不允许模式匹配字符串。使用的时候要用`unpack`将字符串变成字符列表。

简单测试一下：

```
Main> parsePrintfFormat (unpack "a%d%d%s%%")
[LiteralChar 'a', IntegerArgument, IntegerArgument, StringArgument, LiteralChar '%']
```

然后定义`printf`返回值的类型

```
printfFormatToFunctionType: (List PrintfFormatUnit) -> Type
printfFormatToFunctionType Nil = String
printfFormatToFunctionType ((LiteralChar c) :: rest) = printfFormatToFunctionType rest
printfFormatToFunctionType ((StringArgument) :: rest) = String -> (printfFormatToFunctionType rest)
printfFormatToFunctionType ((IntegerArgument) :: rest) = Integer -> (printfFormatToFunctionType rest)
```

简单测试一下：

```
Main> printfFormatToFunctionType (parsePrintfFormat (unpack "a%d%d%s%%"))
Integer -> Integer -> String -> String
```

然后就是将解析好的printf格式变成能用的printf函数。

首先，`printf`的过程类似于一个循环，需要不断将一个字符串进行修改，[前面文章](https://zhuanlan.zhihu.com/p/387239708)提到，**将命令式程序变成纯函数式程序，需要将循环变成递归，循环中修改的变量变成参数不断传递。**printf过程中不断对一个字符串后面添加内容，相当于循环中要修改的变量，这里变成`accumulated`参数：

```
printfFormatToFunction: 
    (accumualted: List Char) -> (format: List PrintfFormatUnit) -> (printfFormatToFunctionType format)
printfFormatToFunction accumualted Nil = 
    (pack accumualted)
printfFormatToFunction accumualted ((LiteralChar c) :: rest) = 
    printfFormatToFunction (accumualted ++ [c]) rest
printfFormatToFunction accumualted ((StringArgument) :: rest) = 
    \argument => printfFormatToFunction (accumualted ++ (unpack argument)) rest
printfFormatToFunction accumualted ((IntegerArgument) :: rest) = 
    \argument => printfFormatToFunction (accumualted ++ (unpack (show argument))) rest

printf: (template: String) -> (printfFormatToFunctionType (parsePrintfFormat (unpack template)))
printf template = printfFormatToFunction [] (parsePrintfFormat (unpack template))
```

最终效果：

```
Main> printf "hello %d %s" 3 "worlds"
"hello 3 worlds"
Main> printf "hello %d %s" "worlds" 3
Error: Can't find an implementation for FromString Integer.
```

类型安全printf的所有代码：

```
data PrintfFormatUnit: Type where
    LiteralChar: Char -> PrintfFormatUnit
    IntegerArgument: PrintfFormatUnit
    StringArgument: PrintfFormatUnit

parsePrintfFormat: (List Char) -> (List PrintfFormatUnit)
parsePrintfFormat Nil = Nil
parsePrintfFormat ('%' :: 's' :: rest) = ((StringArgument) :: (parsePrintfFormat rest))
parsePrintfFormat ('%' :: 'd' :: rest) = ((IntegerArgument) :: (parsePrintfFormat rest))
parsePrintfFormat ('%' :: '%' :: rest) = ((LiteralChar '%') :: (parsePrintfFormat rest))
parsePrintfFormat (firstChar :: rest) = ((LiteralChar firstChar) :: (parsePrintfFormat rest))

printfFormatToFunctionType: (List PrintfFormatUnit) -> Type
printfFormatToFunctionType Nil = String
printfFormatToFunctionType ((LiteralChar c) :: rest) = printfFormatToFunctionType rest
printfFormatToFunctionType ((StringArgument) :: rest) = String -> (printfFormatToFunctionType rest)
printfFormatToFunctionType ((IntegerArgument) :: rest) = Integer -> (printfFormatToFunctionType rest)

printfFormatToFunction: 
    (accumualted: List Char) -> (format: List PrintfFormatUnit) -> (printfFormatToFunctionType format)
printfFormatToFunction accumualted Nil = 
    (pack accumualted)
printfFormatToFunction accumualted ((LiteralChar c) :: rest) = 
    printfFormatToFunction (accumualted ++ [c]) rest
printfFormatToFunction accumualted ((StringArgument) :: rest) = 
    \argument => printfFormatToFunction (accumualted ++ (unpack argument)) rest
printfFormatToFunction accumualted ((IntegerArgument) :: rest) = 
    \argument => printfFormatToFunction (accumualted ++ (unpack (show argument))) rest

printf: (template: String) -> (printfFormatToFunctionType (parsePrintfFormat (unpack template)))
printf template = printfFormatToFunction [] (parsePrintfFormat (unpack template))
```

