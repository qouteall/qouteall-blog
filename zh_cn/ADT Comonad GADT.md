### 代数数据类型ADT (Algebraic Data Types)

Haskell允许定义类型的“积”和“和”，例如

```haskell
data IntAndString = IntAndString Integer String
data IntOrString = IsInt Integer | IsString String
```

`IntAndString`包含一个整数和字符串，是类型的积。而`IntOrString`可能包含一个整数，也可能包含一个字符串，是类型的和。

Java语言对应代码（不用Java新特性）：

```java
public class IntAndString {public int a; public String b;}

public interface IntOrString {}
public class IsInt implements IntOrString {public int data;}
public class IsString implements IntOrString {public String data;}
```

用上Java新特性，防止`IntOrString`被错误地扩展：

```java
public sealed interface IntOrString permits IsInt, IsString {}
public final class IsInt implements IntOrString {public int data;}
public final class IsString implements IntOrString {public String data;}
```

注：Go语言使用者可能会混淆积类型与和类型。在Go语言中经常出现这种异常处理方式：

```golang
// 用于打开文件的函数
func Open(name string) (file *File, err error)

// 使用
file, err := os.Open("filename.ext")
```

函数返回了一个元组，包含`file`和`err`，如果成功则`file`不为空而`err`为空，如果失败则`file`为空而`err`不为空。这个元组实际上是积类型，类型层面上可能出现`file`和`err`都为空、`file`和`err`都不为空的情况。用Haskell表示这个类型：

```haskell
data Maybe t = Present t | Missing  -- 和类型
data GoStyleResult = FileOrError (Maybe File) (Maybe Error)  -- 积类型
```



将和类型(sum type)认为是加法，积类型(product type)认为是乘法，可以建立对类型的代数系统
$$
IntAndString = Integer \cdot String
\\
IntOrString = Integer + String
$$
代数系统中两个特殊的东西是0和1，0是加法的幺元和乘法的零元，1是乘法的幺元
$$
0 + A = A\\
0 \cdot A = 0\\
1 \cdot A = A\\
$$
类型可以对应这个类型的取值集合，例如整数类型是所有整数的集合，`Bool`是`true`和`false`组成的集合。从这个角度看，如果1是类型，那么它只有一种可能取值，如果0是类型，它对应空集，没有任何可能取值。将1称为Unit，0称为Void：

```haskell
data Unit = Unit
data Void
```

C、Java、TypeScript等语言中的void类型其实对应这里的Unit，而Void类型对应TypeScript中的never。Void(never)这种类型，你不可能获得它的任何一个值。在TypeScript中，`never`是所有类型的子类型，子类型可以理解为子集。Unit这个类型你可以获得它的值，但是它只有一种取值，本身不包含任何信息。

类型的加法和乘法满足交换律、结合律、分配率：
$$
A + B = B + A \ \ \ \ A \cdot B = B \cdot A\\
A \cdot (B \cdot C) = (A \cdot B) \cdot C \ \ \ \ A + (B + C) = (A + B) + C\\
(A + B) \cdot C = A \cdot C + B \cdot C
$$
再看一个例子，一个对象有两种状态，而这两种状态下都存有一个字符串，在Haskell里可以表示为：

```haskell
data A = FirstState String | SecondState String
```

换另一种表示方法，也可以表示为一个布尔值和一个字符串

```haskell
data Bool = True | False
data A = A Bool String
```

$$
A = String + String\\
\\
Bool = 1 + 1\\
A = Bool \cdot String\\
\\
A = 2 \cdot String
$$

可以看到与代数系统很像，所以称为代数数据类型。同时，每个类型所相对应的整数，也代表了这个类型的取值可能数量。

### 函数类型

函数这种东西，有两种方式去理解：

* 一个函数表示一种计算，给定一个输入，能算出一个输出
* 一个函数表示一个映射表，给定一个输入，可以从映射表中查找到输出

如果从第二种方式去理解，对于函数类型`Bool -> Integer`，可以写一个映射表：

| 输入  | 输出 |
| ----- | ---- |
| True  | 2333 |
| False | 6    |

这个映射表，左边一列固定，右边一列可变，总共包含两个整数的信息，代数数据类型为`Integer * Integer`，两个整数相乘。

如果是函数类型`Integer -> Bool`呢？

| 输入 | 输出  |
| ---- | ----- |
| 0    | True  |
| 1    | False |
| 2    | False |
| ...  | ...   |

假设Integer有2^32种取值，那么这个映射表就是2^32个`Bool`相乘。

总之，输入类型有几种取值，那么输出类型就要乘几下。可以看出，这对应指数
$$
Bool \rightarrow Integer = Integer \cdot Integer = Integer ^ {Bool}
\\
Integer \rightarrow Bool = Bool ^ {Integer}\\
A \rightarrow B = B ^ A
$$

### 递归

对自然数和列表（链表）的定义，是有递归的：

```haskell
data NaturalNumber = Zero | Successor NaturalNumber
data List t = EmptyList | ListNode t (List t)
```

$$
NaturalNumber = 1 + NaturalNumber\\
List \ t = 1 + t \cdot (List \ t)\\
NaturalNumber = List \ 1
$$

### 减法与除法

类型已经有加法、乘法和指数了，那么能否定义加法的逆运算减法、乘法的逆运算除法呢？

假如有减法和除法，那么列表可以写成另外一种形式：
$$
List \ t = 1 + t \cdot (List \ t)\\
List \ t = \dfrac{1}{1-t}\\
NaturalNumber = \dfrac{1}{0}
$$
然而，似乎没有一个比较好的减法和除法的定义，所以先认为减法和除法没有定义。

### 挖洞与求导

代数数据类型是可以求导的。对一个数据结构求导，可以获得带有一个洞的数据结构(one-hole context)。后面提到的Zipper会用到它。

例如这是一个树：

![image.png](https://s2.loli.net/2022/08/20/ftbHe7Sk4zTCIow.png)

扣除一个元素，得到带有一个洞的树：

![image.png](https://s2.loli.net/2022/08/20/qB829JWdRDtHVga.png)

[参考](https://wiki.haskell.org/Zipper)

在数学中，我们可以对函数求导，这些是数学中对函数求导的一部分规则：
$$
\dfrac{dc}{dt} = 0\\
\dfrac{dt}{dt} = 1\\
\dfrac{d(A+B)}{dt} = \dfrac{dA}{dt} + \dfrac{dB}{dt}\\
\dfrac{d(A \cdot B)}{dt} = A \cdot \dfrac{dB}{dt} + B \cdot \dfrac{dA}{dt}\\
\dfrac{d(F(G(t)))}{dt} = \dfrac{d(F (G(t)))}{d(G(t))} \dfrac{d(G(t))}{dt}
$$

很神奇的地方在于，假如我们将求导定义为“挖一个洞”，A对t求导就是将A里面包含的一个t扣成洞，那么这些求导规则刚好能用在挖洞上

* 如果c不含t，那么在c中挖掉t，得到的东西是不存在的：

$$
\dfrac{dc}{dt} = 0
$$

* 对于t本身，挖掉t就只剩下洞，洞本身只有一种取值：

$$
\dfrac{dt}{dt} = 1
$$

* 对于A或B的数据，挖掉t，可能是A挖掉t的结果，也可能是B挖掉t的结果：

$$
\dfrac{d(A+B)}{dt} = \dfrac{dA}{dt} + \dfrac{dB}{dt}
$$

* 对于A与B的数据，挖掉t，洞可能在A里面也可能在B里面。如果洞在A里面，那么挖洞结果是B与有洞的A。如果洞在B里面，那么挖洞结果是A与有洞的B：

$$
\dfrac{d(A \cdot B)}{dt} = A \cdot \dfrac{dB}{dt} + B \cdot \dfrac{dA}{dt}\\
$$

* 假如A中包含B，要在A里面的B里面挖一个洞，那么显然将A中的一个B挖掉，然后将那个B里面挖一个洞：

$$
\dfrac{dA}{dt} = \dfrac{dA}{dB} \cdot \dfrac{dB}{dt}
$$

例如，有两个元素的容器（二元组, pair）和有三个元素的容器（三元组）。二元组中扣一个洞，有两种可能性。三元组中扣一个洞，有三种可能性：


```haskell
data Pair t = Pair t t
data TriTuple t = TriTuple t t t

data PairWithHole t = HoleIsLeft t | HoleIsRight t
data TriTupleWithHole t = HoleIsFirst t t | HoleIsSecond t t | HoleIsThird t t
```

将二元组和三元组的定义进行求导，得到带洞容器：

$$
Pair \ t = t \cdot t\\
PairWithHole \ t = \dfrac{d (Pair \ t)}{dt} = 2t\\
TriTuple \ t = t^3\\
TriTupleWithHole \ t = \dfrac{d (TriTuple \ t)}{dt} = 3t^2
$$

再看带有递归的容器，列表和树（树只有叶子节点存元素）：


```haskell
data List t = EmptyList | ListNode t (List t)
data Tree t = Fork (Tree t) (Tree t) | Leaf t
```

一个列表打洞，可能洞在头部，也可能洞在后面。如果这个树本身是一个叶子节点，对它打洞那么只剩一个洞。如果这个树是一个分支节点，对它打洞，洞可能在左边的子树，也可能在右边的子树：

```Haskell
data ListWithHole t = HoleIsHere (List t) | HoleIsBehind t (ListWithHole t)
data TreeWithHole t = 
	ThisIsHole |
	HoleIsOnLeft (TreeWithHole t) (Tree t) |
	HoleIsOnRight (Tree t) (TreeWithHole t)
```

进行求导：
$$
List \ t = 1 + t \cdot (List \ t)\\
\dfrac{d(List \ t)}{dt} = (List \ t) + t \cdot \dfrac{d(List \ t)}{dt}\\
Tree \ t = (Tree \ t) ^2 + t\\
\dfrac{d(Tree \ t)}{dt} = 2 (Tree \ t) \dfrac{d(Tree \ t)}{dt} + 1
$$

注：带洞列表其实有两种定义方式。除了上面的定义方式之外，带洞其实可以定义为两个列表，两个列表分别代表洞前面的部分和洞后面的部分：

```Haskell
data ListWithHole1 t = HoleIsHere (List t) | HoleIsBehind t (ListWithHole1 t)
data ListWithHole2 t = ListWithHole2 (List t) (List t)
```

通过代数数据类型运算，可以发现两者等价
$$
List \ t = 1 + t \cdot (List \ t) = 1 + t + t^2 + t^3 + ...\\
ListWithHole1 \ t = (List \ t) + t (ListWithHole1 \ t)\\
ListWithHole2 \ t = (List \ t)^2\\\\
ListWithHole1 \ t = (List \ t) + t (List \ t) + t^2 (List \ t) + t^3(List \ t) + ... = \\
(List \ t)(1 + t + t^2 + ...) = (List \ t)^2 = ListWithHole2 \ t
$$

第二种带洞列表定义的好处是，结合惰性求值可以表示两边都无穷延伸的列表。

### Zipper

把一个数据结构挖一个洞，然后再将洞内的原本元素放在旁边，就获得了一个Zipper。

Zipper是对原有数据结构的另一种表示，但是附加了一点额外信息，这个信息就是洞挖在哪。Zipper有点类似于迭代器，指向数据结构内的一个元素，但C++中的迭代器本身只记录位置，不会记录整个数据结构，而Zipper内部记录了整个数据结构。Zipper可以方便纯函数式程序对数据结构的局部的读取与替换。

例如说我定义列表的Zipper，然后就可以修改（实际上是变换）目前所指的元素，还可以让Zipper向右或向左移动一下：

```haskell
-- 自己定义的链表
data List t = EmptyList | ListNode t (List t)

-- 分别表示前面的部分（倒序）和后面的部分
data ListWithHole t = ListWithHole (List t) (List t)

data ListZipper t = ListZipper t (ListWithHole t)

-- 和Maybe类似
data Option t = Some t | None

-- 替换zipper指向元素
replace :: ListZipper t -> t -> ListZipper t
replace (ListZipper old ambient) new = ListZipper new ambient

-- zipper左移
leftOf :: ListZipper t -> Option (ListZipper t)
-- ls left | mid | rs  --> ls | left | mid rs
leftOf (ListZipper mid (ListWithHole (ListNode left ls) rs)) = 
	Some (ListZipper left (ListWithHole (ls) (ListNode mid rs)))
leftOf (ListZipper mid (ListWithHole (EmptyList) rs)) = None

-- zipper右移
rightOf :: ListZipper t -> Option (ListZipper t)
-- ls | mid | right rs --> ls mid | right | rs
rightOf (ListZipper mid (ListWithHole ls (ListNode right rs))) =
	Some (ListZipper right (ListWithHole (ListNode mid ls) rs))
rightOf (ListZipper mid (ListWithHole ls (EmptyList))) = None
```

除此之外还可以有添加、删除等操作。树的Zipper也类似。

### Comonad

Zipper是一种Comonad，其中co表示contra，Comonad表示反过来的Monad。

Monad的两种定义：[前面的学习笔记](https://zhuanlan.zhihu.com/p/387239515)

```haskell
-- 官方定义
class Monad1 m where
  return :: a -> m a
  bind :: m a -> (a -> m b) -> m b
  
-- 一种更易理解的定义
class Monad2 m where
  wrap :: a -> m a
  join :: m (m a) -> m a
```

根据第二种定义，Monad可以将数据套一层，也可以将套了两层的数据合并成一层。现在，将wrap和join的函数箭头反过来，获得Comonad：

```haskell
class Comonad w where
  unwrap :: w a -> a
  duplicate :: w a -> w (w a)
```

unwrap可以将套了一层的数据解开，获得Zipper所指向的元素。而duplicate可以将套了一层的数据再套一层，将一个装有元素的Zipper变成装有Zipper的Zipper，也就是将每个元素替换为指向该元素的Zipper。

为`ListZipper`实现Comonad

```haskell
-- 给定元素beginning和函数f，生成 [f beginning, f (f beginning), f (f (f beginning)) ...]
-- 直到f返回None
keepIterating :: t -> (t -> Option t) -> List t
keepIterating beginning f = case (f beginning) of
  None -> EmptyList
  (Some element) -> ListNode element (keepIterating element f)

instance Comonad ListZipper where
  -- 取出zipper指向元素
  unwrap (ListZipper curr _) = curr
  
  -- 将一个zipper对应列表变成zipper的列表的zipper
  duplicate zipper = (ListZipper
    -- zipper指向的元素，变换为其zipper本身
    zipper
    -- 对zipper不断调用leftOf，产生zipper的列表，右侧同理
    (ListWithHole
      (keepIterating zipper leftOf)
      (keepIterating zipper rightOf)
    ))
```

Comonad的官方定义：(一般来说，Comonad首先是Functor)

```haskell
class Comonad2 w where
  coreturn :: w a -> a
  cobind :: (w a -> b) -> w a -> w b
  
instance (Comonad w, Functor w) => Comonad2 w where
  coreturn = unwrap
  cobind f wa = fmap f (duplicate wa)
```

如果你想迭代数据，而且迭代结果取决于元素周围的信息，例如计算卷积、模拟康威(Conway)生命游戏等，Comonad会比较有用。

### GADT(Generalized Algebraic Data Type)

GADT可以让定义ADT的时候更细化确定类型。

例如列表，可以用普通方式定义，也可以用GADT方式定义：

```haskell
data List t = EmptyList | ListNode t (List t)

-- 采用GADT
data List t where
  EmptyList :: List t
  ListNode :: t -> List t -> List t
```

采用了GADT后，类型的每一项，都表示为一个值或“构造函数”。`EmptyList`本身是一个值，而`ListNode`是一个构造函数，给定一个`t`和`List t`就能构造出一个`List t`。构造函数不仅是函数，也可以在模式匹配中用。

GADT在另一些情况下更加有用，例如，我要定义一个表达式树，可能是整数表达式也可能是布尔表达式：

```haskell
data Expression = 
	IntConstValue Integer |
	BoolConstValue Bool |
	IntAdd Expression Expression |
	IntNegate Expression |
	BoolAnd Expression Expression |
	BoolNegate Expression
```

表达式可能是一个整数类型常量，布尔类型常量，可能是整数加法、整数取相反数、布尔值的与运算、布尔值的取反。

注意到，`IntAdd`理论上只能针对整数类型的表达式，但是这个类型限定不够充分，无法确保`IntAdd`包含的表达式为整数表达式，可以让`IntAdd`包含布尔类型表达式，例如`IntAdd (BoolConstValue True) (IntConstValue 2) `。函数式编程讲究通过类型让非法数据无法表达，进而减少bug，显然我们需要将类型细化避免这种问题。

可以将表达式类型变成泛型，整数类型表达式就是`Expression Integer`，布尔值表达式就是`Expression Bool`，然后采用GADT：

```haskell
data Expression t where
  IntConstValue :: Integer -> Expression Integer
  BoolConstValue :: Bool -> Expression Bool
  IntAdd :: Expression Integer -> Expression Integer -> Expression Integer
  IntNegate :: Expression Integer -> Expression Integer
  BoolAnd :: Expression Bool -> Expression Bool -> Expression Bool
  BoolNegate :: Expression Bool -> Expression Bool
```

对应Java代码

```java
interface Expression<T> {}
class IntConstValue implements Expression<Integer> {int data;}
class BoolConstValue implements Expression<Boolean> {boolean data;}
class IntAdd implements Expression<Integer> {Expression<Integer> a,b;}
class IntNegate implements Expression<Integer> {Expression<Integer> a;}
class BoolAnd implements Expression<Boolean> {Expression<Boolean> a,b;}
class BoolNegate implements Expression<Boolean> {Expression<Boolean> a;}
```

