## [学习笔记] Haskell语言

Haskell的语言特性有：纯函数式，值不可变，支持和类型（sum type），支持模式匹配，支持泛型与高阶泛型，支持特设多态，有较强类型推导能力，不支持子类型。

人们喜欢用各种技巧精简化Haskell程序，但这同时降低了可读性。例如：使用单字母命名、不写类型、使用引起误解的命名、在能重名的地方重名（a既代表类型又代表参数）、能不加括号就不加括号、采用奇怪的柯里化等。这里尽量不做让代码丧失可读性的简化，同时使用对应Java代码进行参考。

函数式编程的基本特点有：允许将函数作为值进行传递、定义一个函数的时候可以从外面捕获值放到函数对象里面（lambda表达式）。对一个函数对象，只能调用它，不能拆开看内部的数据。

### 和类型(sum type)

Haskell中列表是树状结构的（也可以认为是链状结构的）。一个列表可以是空列表，也可以是一个树，左节点是第一个元素，右节点是一个列表代表剩下的元素。

![tree list.png](https://i.loli.net/2021/06/11/hNyFnmx6Ki4BTQq.png)

对列表的定义：

```haskell
data MyList t = EmptyList | ListNode t (MyList t)
```

其中`t`是泛型参数，`|`代表类型的“或”，`EmptyList`是空列表（末节点），`ListNode`代表树节点。`ListNode`和`EmptyList`既是构造函数也可以在模式匹配里用。`ListNode t (MyList t)`代表`ListNode`里面有两个成员，一个类型为`t`表示第一个元素，一个类型为`MyList t`表示剩下的元素构成的子列表。`MyList`有两种可能取值，就是所谓的和类型(sum type)。Rust语言也有该特性。

和类型有一点像C语言里的共用体union：

```C
template<class T>
union MyList {
    void emptyList;
    struct {
        T first;
        MyList<T>* rest;
    } listNode;
}
```

但是这里的union中没有存储具体类型的信息，不知道里面存的是`emptyList`还是`listNode`，所以要加一个标签来表示这个信息，因此和类型也被称为带标签的共用体(tagged union)。和类型对应类型取值集合的并运算。除了和类型之外还有积类型(product type)，结构体、元组就是积类型，对于类型取值集合的笛卡尔积。通过和类型和积类型可以构造代数数据类型(Algebraic data type)。

用面向对象子类型也可以表示“或”关系。用Java语言表示：

```java
interface MyList<T>{};
class EmptyList<T> implements MyList<T> {}
class ListNode<T> implements MyList<T>{
    T first;
    MyList<T> rest;
}
```

注意到和类型与面向对象子类型的差别，这里Java语言允许别人再定义一个类实现`MyList`接口，无法限定`MyList`的取值只有两种，而Haskell中的和类型可以限定取值只有两种。这就是和类型相对于面向对象子类型的优点：可以限定类型取值，方便模式匹配。Rust中的和类型还能允许更紧凑的内存布局，从而提高性能。

> 官方的Haskell列表定义是这样的：`data [] a = [] | a : [a]`

### 函数式思维：递归代替循环

纯函数式编程不允许使用变量，理论上所有的值不可变，这要求一种新的编程思维。传统C语言中对一个数组求和，我们采用一个循环：

```C
int sum = 0; for (int i = 0; i < length; i += 1) { sum += array[i]; }
```

但是纯函数式不允许对变量`i`进行修改。函数式语言采用递归来代替命令式中的循环：

```haskell
sum :: (MyList Integer) -> Integer 
sum EmptyList = 0
sum (ListNode first rest) = first + (sum rest)
```

首先用模式匹配判断是不是末节点（空列表），如果是则和为0。如果不是，用模式匹配分出左边的一个元素和右边的子树，对右边的子树递归求和然后加起来。可以看到Haskell在写函数的同时就可以进行模式匹配，非常方便。

用Java语言表示：

```java
static int sum(MyList<Integer> list) {
    if (list instanceof EmptyList<Integer>) {
        return 0;
    } else if (list instanceof ListNode<Integer>) {
        Integer first = ((ListNode<Integer>)list).first;
        MyList<Integer> rest = ((ListNode<Integer>)list).rest;
        return first + sum(rest);
    }
    throw new RuntimeException();
}
```

可以看到Java语言（现在）没有模式匹配，不得不写一些分支和类型转换。而且由于类型限定不充分，需要加上额外的错误处理代码。

### 柯里化(currying)

Haskell中函数调用与主流语言不同，主流语言是`function(argument1, argument2...)`而Haskell等函数式语言是`function argument1 argument2`，调用函数本身不加括号，这是符合柯里化(currying)的。用柯里化的角度看，每个函数都只接受一个参数输出一个值，一个接受多个参数的函数，在接受到第一个参数后返回一个新的函数，这个新的函数接受剩下的参数。

例如将三个数相加的函数`add3Numbers a b c = a + b + c`，其类型是`Integer -> (Integer -> (Integer -> Integer))`。函数调用就是用具体值代替参数并替换，再对替换后的结果进行求值。对该函数用参数5调用，只对a进行替换，得到`add3Numbers 5 b c = 5 + b + c`，`add3Numbers 5`相当于另一个函数`add2NumbersWith5 b c = a + b + c`，这也相当于把a的值5捕获形成闭包函数。

用Java语言示意：

```java
int add3Numbers(int a, int b, int c)  {return a + b + c; }
Function<Integer, Function<Integer, Integer>> add3NumbersCurried(int a) {
    return b -> (c -> a + b + c)
}
```

在C语言等命令式语言中没有柯里化，调用一个函数需要将所有参数一次性提供然后运行。而对于柯里化，例如说对`add3Numbers`给参数，在给第一个和第二个参数的时候，都是生成新函数，没有进行真正的求值，只有在第三次给参数后才进行真正的求值。对纯函数式语言来说求值的顺序、时机无所谓。但是对C语言来说，如果对函数一个一个给参数，没给够只是生成函数，给够了才执行函数并施加副作用，这一设计就会非常怪异。对纯函数式语言来说，这一设计可以让每个函数只接受一个值，只输出一个值，可以起到简化作用，而且能够让针对第一个参数的闭包更方便。柯里化有时会比较反直觉。

在有了柯里化后，每个函数只接受一个值而且输出一个值，这可以简化设计。

同时，Haskell中的泛型也是不需要加括号的，Java中的`MyList<Integer>`在Haskell中是`MyList Integer`，泛型认为是类型的函数，也可以柯里化。（Haskell中泛型和函数还有所区分，在Idris里泛型和函数是统一的）

Haskell中用`\`来表示lambda表达式，例如`\x -> x + 1`相当于Java中的`x -> x + 1`。

### 函数式思维：变换代替修改

在过程式语言中，我们可以对值进行修改。但是纯函数式语言不允许修改，需要另一种思维，将数据的修改变为数据的变换(transform)。

例如说，我有一个整数列表，现在要在右边加上一个数10。Java中，是直接修改容器`list.add(10)`，但Haskell中我需要生成一个新的容器，这个新的容器是旧的容器右面加上一个数10。

```Haskell
data MyList t = EmptyList | ListNode t (MyList t)

appendRight :: MyList t -> t -> MyList t
appendRight EmptyList element = (ListNode element EmptyList)
appendRight (ListNode first rest) element = ListNode first (appendRight rest element)
```

这段代码用Java语言表示：

```Java
<T> static MyList<T> appendRight(MyList<T> list, T element) {
    if (list instanceof EmptyList) {
        return new ListNode(element, new EmptyList());
    } else if (list instanceof ListNode) {
        T first = ((ListNode)list).first;
        MyList<T> rest = ((ListNode)list).rest;
        return new ListNode(first, appendRight(rest, element));
    }
    throw new RuntimeException();
}
```

我们对列表的右边加了一个新元素，但是旧的列表仍然没有改变，我们只是获得了一个被“改变”了的新列表，这就是用数据变换代替修改的思想。

### 惰性求值

数据的完全不可变允许惰性求值，也就是一个表达式在不需要他的值的时候不计算，需要的时候再计算。通过惰性求值，可以定义一个无限长的列表：

```Haskell
infiniteRepeat element = ListNode element (infiniteRepeat element)
```

对应的Java代码：

```Java
<T> static MyList<T> infiniteRepeat(T element) {
    return new ListNode(element, infiniteRepeat(element));
}
```

这似乎是死递归。在Java语言中这个代码像炫迈一样根本停不下来。但是，Haskell有惰性求值，这个无穷长列表可以以表达式形式存在，没必要全部求值出来，我们可以只求值前面一部分，例如说取出第五个元素：

```haskell
getAt :: MyList t -> Integer -> t
getAt (ListNode first rest) 0 = first
getAt (ListNode first rest) index = getAt rest (index - 1)

getAt (infiniteRepeat 2333) 5
-- 2333
```

Java中`Stream`本身有惰性求值特性，`Stream.generate`可以做到类似功能。

Haskell中一切值都是惰性求值的，这容易带来内存泄漏。

### 特设多态

Haskell中允许定义“接口”，称为typeclass，例如说比较两个对象是否相等的接口：

```haskell
class Eq t where
    equals :: t -> t -> Bool
```

这里t是泛型参数。

例如说我定义了一个装整数的盒子，可以这么实现比较相等的接口。

```Haskell
data MyIntegerBox = MyBox Integer
instance Eq MyIntegerBox where
  equals (MyBox i1) (MyBox i2) = (i1 == i2)
```

与之对应的Java语言：

```Java
class MyIntegerBox {
    public int data;
    @Override
    boolean equals(Object other) {
        if (other == null) {
            return false;
        }
        if (other.getClass() != this.getClass()) {
            return false;
        }
        MyIntegerBox that = (MyIntegerBox) other;
        return this.data == that.data;
    }
}
```

Java语言中在给自己的类实现`equals`的时候，要先检查other是不是null，然后检查other的类型，然后再进行类型转换，花了很多代码在无关的事情上。这一切都是因为类型限定不足，传进来的是`Object`而不是确定的不为空的`MyIntegerBox`。

如果要定义接口让`equals`传进来的是具体类型而不是Object呢？Java中无法定义这样的接口：

```
interface EqualityComparable {
    boolean equals(Self other);
}
```

其中`Self`指自己的类型，Java不支持这个，但Rust支持。`Self`不能替换成`EqualityComparable`，整数和字符串都是`EqualityComparable`，这会允许整数和字符串相比较。

同时Java中的`Comparable`接口有一个泛型参数，但这个泛型参数必须是自己的类型，这个设计非常怪异，但是能增强类型限定，实现Comparable的时候不需要进行类型判断和类型转换了。

主要到子类型多态和特设多态的区别：Java中的interface是子类型多态，只能定义多态成员函数，但Haskell中的特设多态，是对普通函数的多态。Java中一个`EqualityComprable a`变量代表a引用的对象是`EqualityComparable`的子类型，而Haskell中没有子类型，接口是对泛型参数的：

```haskell
allEquals :: (Eq a) => a -> a -> a -> Bool
allEquals a b c = (a == b) && (b == c)
```

这里泛型参数a是可以自动推导的。`(Eq a) =>`代表首先要满足`Eq a`才能推导(`=>`)出后面的函数类型。

Haskell可以进而定义针对泛型的接口，而Java不允许定义针对泛型的接口，后面的Monad就是一个例子。

官方的`Eq`定义：

```haskell
class Eq a where  
  (==) :: a -> a -> Bool
  (/=) :: a -> a -> Bool
  x == y = not (x /= y)
  x /= y = not (x == y)
```

注意到这里==的实现用到了/=（相当于!=），而/=的实现用到了==，只要你实现`==`他就能自动实现`/=`。C++的运算符重载就是特设多态，但是不能做到实现了==就自动实现!=。而且如果你没有实现对应函数则会出现很长的杂乱编译错误，C++20的`concept`就是为了解决这个问题。

注意到，Java中要让一个类实现一个接口，必须在定义类的时候写明，不能在类定义完后再实现新的接口，是“侵入式”接口。而Haskell允许类型定义完后让他实现新的接口，甚至允许对别的库里面的类型实现自己的接口，这不仅增强了灵活性还能降低代码的耦合性。Rust、Go也支持非侵入式接口。

下一篇：[Haskell与Monad](https://github.com/qouteall/qouteall-blog/blob/main/zh_cn/Haskell%20Monad.md)

