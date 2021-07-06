## [学习笔记] Haskell与Monad



### Functor, Monad

Functor和Monad是针对泛型的接口。这里牵扯到高阶泛型。普通的泛型，其泛型参数只能取值为具体的类型，而高阶泛型允许将泛型本身作为泛型参数。例如`std::vector<int>`是具体类型，`std::vector`是泛型，将`std::vector`本身作为泛型参数传入是高阶泛型。

在谈论Functor和Monad之前，先看熟知的map和flatmap操作。一般来说，一个函数式语言爱好者会怎样向一个命令式语言程序员安利函数式语言呢？他一般会介绍`map` `filter` `reduce` `flatmap`等操作如何简化复杂的命令式循环。Java语言中现在已经支持这些操作：

```java
List<Integer> list = List.of(1, 2, 3, 4);
list.stream().map(n -> n * 2).collect(Collectors.toList())
// 2, 4, 6, 8

list.stream().flatmap(n -> Stream.of(Integer.toString(n), "hello")).collect(Collectors.toList())
// "1", "hello", "2", "hello", "3", "hello", "4", "hello"
```

Java中`map`和`flatmap`都对`Stream`做操作，`Stream`可以认为是惰性求值的容器，采用惰性求值是为了提高性能。`map`将容器的每个元素经过一个函数替换成另一个元素，而`flatmap`将每个元素替换成一个容器，再将这些容器连接起来。

**可以认为，Functor就是可以进行map的容器或计算过程，Monad就是可以进行flatmap的容器或计算过程。**

列表是容器。Optional也是容器，包含0个或1个元素。两种容器的定义：

```haskell
data Optional t = Present t | None
data MyList t = EmptyList | ListNode t (MyList t)
```

> 注：`Optional`也被称为`Maybe`。这里我认为`Optional`是更合适的名字。

Functor与Monad的定义

```haskell
class MyFunctor m where
  map :: (m a) -> (a -> b) -> (m b)

class MyMonad m where
  wrap :: a -> m a
  flatmap :: m a -> (a -> m b) -> m b
```

> 注：官方的`flatmap`记作`>>=`，也被称作`bind`（bind是范畴论中的名字），`wrap`记作`return`，`Functor`的`map`记作`fmap`。我认为`flatmap`比`>>=`更直观，`return`名称有误导性。同时，这里map的两个参数的顺序和官方是反的。

这里的`m`泛型参数接受一个泛型，可以是`Optional`或`MyList`，不是具体类型例如`MyList Integer`。`map`输入一个a的容器，和一个a到b的函数，输出b的容器。`flatmap`输入a的容器，和一个a到装b的容器的函数，输出b的容器。`wrap`将一个普通值包装在容器中。（`Optional`在官方叫`Maybe`）

对两种容器实现`Functor`和`Monad`

```haskell
instance MyFunctor Optional where
  map (None) f = None
  map (Present value) f = Present (f value)

instance MyMonad Optional where
  wrap value = Present value
  flatmap (Present value) func = func value
  flatmap None func = None

instance MyFunctor MyList where
  map (EmptyList) f = EmptyList
  map (ListNode first rest) f = ListNode (f first) (map_ rest f)

instance MyMonad MyList where
  wrap value = elementToList value
  flatmap EmptyList func = EmptyList
  flatmap (ListNode first rest) func = listConcat (func first) (flatmap rest func)
```

Functor也可以叫做Mappable，Monad也可以叫做FlatMappable。

### 计算过程也是Monad

除了容器之外，“计算过程”也可以是Monad。

为了处理异步计算，有一种东西叫Future，Future代表一个计算过程以及这个计算过程的结果，结果必须在计算完成后获得。

例如Java中的CompletionStage，可以认为是Future。

```Java
interface CompletionStage<T>{
    <U> CompletionStage<U> thenApply(Function<T, U> fn);
    <U> CompletionStage<U> thenCompose(Function<T, CompletionStage<U>> fn);
}
```

`thenApply`就是map，`thenCompose`就是flatmap。`thenApply`就是在运行完毕之后，用将结果转换成另一个值。`thenCompose`就是运行完毕之后，根据结果产生另一个计算过程，将两个先后的计算过程合成一个计算过程。

在JavaScript中，Future被称为Promise，`then`函数既可以map也可以flatmap。

计算过程也可以认为是一种“暂时打不开的容器”。map是将容器内的值置换成另一个值，flatmap是将容器内的值置换成另一个容器，然后将两层容器融合成一层容器。对于列表，合并两层容器是将各个列表合并，对于计算过程，合并两层容器是将先后的两个计算过程看成一个计算过程。

Haskell中函数本身就是一种计算过程，函数的输出值就是计算过程的结果。最普通的计算过程，函数，也是Monad。将函数包装成`Function`，将输入类型柯里化，然后实现Monad：

```haskell
newtype Function fromType toType = Function_ (fromType -> toType)
getFunction :: Function fromType toType -> (fromType -> toType)
getFunction (Function_ func) = func

instance MyMonad (Function fromType) where
  wrap value = Function_ (\whatever -> value)
  flatmap function mapping =
    Function_ (\input -> let
      firstResult = (getFunction function) input
      newFunction = mapping firstResult
      finalResult = (getFunction newFunction) input
      in finalResult
    )
```

（这里的`Function`在官方叫`Reader`，因为这个东西可以允许一堆操作读一个共同的参数）

flatmap的时候，生成一个新的计算过程，这个新的计算过程中，先运行原来的函数，获得结果，结果传入mapping得到新的函数，再运行新的函数输出结果。

对应的Java代码：

```java
<Input, A, B> static Function<Input, B> flatmap(
    Function<Input, A> function, Function<A, Function<Input, B>> mapping
) {
    return input -> {
        A firstResult = function.apply(input);
        Function<Input, B> newFunction = mapping.apply(firstResult);
        B finalResult = newFunction.apply(input);
        return finalResult;
    };
}
```



下面要说的状态转移函数也是一种计算过程：

### 状态转移函数也是Monad

前面说到，纯函数式编程要求用数据变换代替修改。例如说用列表表示一个栈，栈可以从左边放入元素、取出元素。push操作，是输入一个要压栈的值和栈，输出一个新的栈。pop操作，是输入一个栈，输出取出的值和新的栈。

```haskell
pushLeft value stack = ListNode value stack
popLeft (ListNode value rest) = (value, rest)
```

> `(value, rest)`指生成一个有两个元素的元组(tuple)

现在要写一个函数，通过栈操作，将栈顶部两个元素互换

```haskell
exchangeTop2 stack0 = let
  (top1, stack1) = popLeft stack0
  (top2, stack2) = popLeft stack1
  stack3 = pushLeft top1 stack2
  stack4 = pushLeft top2 stack3
  in stack4
```

可以看到，我们需要把“变化”的栈的值不断传来传去，写起来比较麻烦，于是尝试对这个状态转移进行抽象。

状态转移函数，是输入旧状态，对状态进行操作，输出新状态和这一步的输出值。例如push操作，将要压栈的值闭包在状态转移函数里面，输入旧的栈，输出新的栈，输出值为空。pop操作，输入旧的栈，输出新的栈和出栈的值。

```haskell
stateTransferFunction :: State -> (State, Output)
```

如果用状态机来看，状态机每一步都要根据其状态和外部输入，决定新状态和输出。这里，状态转移函数本身就是状态机的输入，输入给状态机的值包含在状态转移函数里面。状态转移函数的输出中包含状态机的输出和新状态。

定义一个包装状态转移函数的类型。

```Haskell
newtype StateTransfer stateType outputType = StateTransfer_ (stateType -> (stateType, outputType))
```

（这里`StateTransfer_`是构造函数和模式，没有写作`StateTransfer`是为了区分。官方的这个东西叫`State`，不过我认为`State`这个名字不合适，因为它不是状态，是状态转移函数。）

这个状态转移函数可以被看做是一个计算过程，也是Monad。这个计算过程的结果是状态转移的输出，不是状态，例如pop操作计算过程的结果是从栈中取出的值，push操作计算过程的结果是空。将这个状态转移函数看做容器，容器需要你给他一个旧状态才能打开，打开后获得计算结果以及新状态。

上面的push和pop操作可以写成这样。push操作需要结合一个值才能变成状态转移函数，状态转移输出为空元组`()`。pop操作本身就是状态转移函数，状态转移输出为出栈的值。

```haskell
push value = StateTransfer_ (\list -> (ListNode value list, ()))
pop = StateTransfer_ (\(ListNode value rest) -> (rest, value))
```

对状态转移函数这种Monad，flatmap首先执行第一个状态转移函数，然后把输出送到给定的函数里，获得第二个状态转移函数，然后执行第二个状态转移函数。这里我们对泛型进行了柯里化，本来`StateTransfer`接受两个类型，现在给定状态类型后他只接受一个类型了，相当于对状态转移输出值的容器。

状态转移函数是一个计算过程，这个计算过程的结果是状态转移的输出值（不是新状态），对计算过程进行flatmap，就是将第一个运算的结果置换成第二个计算过程，再将两个先后的计算过程合成一个计算过程。

```haskell
instance MyMonad (StateTransfer stateType) where
  wrap output = StateTransfer_ (\whatever -> (whatever, output))
  flatmap (StateTransfer_ stateTransferFunc1) outputToNewStateTransfer =
    StateTransfer_ (\state0 -> let
      (state1, output1) = stateTransferFunc1 state0
      (StateTransfer_ stateTransferFunc2) = outputToNewStateTransfer output1
      (state2, output2) = stateTransferFunc2 state1
      in (state2, output2)
    )
```

等价的Java语言：

```java
interface StateTransfer<StateType, OutputType> {
    Pair<StateType, OutputType> run(StateType oldState);
}

<StateType, OutputType1, OutputType2>
static StateTransfer<StateType, ResultType2> flatmap(
    StateTransfer<StateType, OutputType1> stateTransferFunc1,
    Function<OutputType1, StateTransfer<StateType, OutputType2>> outputToNewStateTransfer
) {
    return (StateType state0) -> {
        Pair<StateType, OutputType1> pair1  = stateTransferFunc1.run(state0);
        StateType state1 = pair1.first;
        OutputType1 output1 = pair1.second;
        
        StateTransfer<StateType, OutputType2> stateTransferFunc2 = outputToNewStateTransfer.apply(output1);
        
        Pair<StateType, OutputType1> pair2 = stateTransferFunc2.run(state1);
        StateType state2 = pair2.first;
        OutputType2 output2 = pair2.second;
        
        return new Pair(state2, output2);//可以不用把pair2拆开再合成，这里是为了示意
    };
}
```

采用了Monad对状态转移函数进行抽象，上面的栈操作程序可以变成这样

```haskell
push value = StateTransfer_ (\list -> (ListNode value list, ()))
pop = StateTransfer_ (\(ListNode value rest) -> (rest, value))

exchangeTop2 :: StateTransfer (MyList t) ()
exchangeTop2 = flatmap
  pop (\top1 -> flatmap
  pop (\top2 -> flatmap 
  (push top1) (\whatever ->
  push top2)))
```

等价Java语言：

```java
<T> static StateTransfer<MyList<T>, Void> push(T value) {
    return (MyList<T> oldStack) -> new Pair<>(new ListNode<>(value, oldStack), null);
}
<T> static StateTransfer<MyList<T>, T> pop() {
    return (MyList<T> oldStack) -> {
        ListNode<T> node = (ListNode<T>)oldStack;
        return new Pair<>(node.rest, node.first);
    }
}
<T> static StateTransfer<MyList<T>, T> exchangeTop2() {
    return flatmap(
        pop(), (T top1) -> flatmap(
            pop(), (T top2) -> flatmap(
                push(top1), whatever -> push(top2)
            )
        )
    );
}
```

注意到，后面的状态转移操作在更深的套娃里面，可以通过闭包获得前面状态转移操作的结果。

Haskell提供do语法糖可以让状态转移函数合成更简洁

```haskell
exchangeTop2 = do
  top1 <- pop
  top2 <- pop
  push top1
  push top2
```

看起来和命令式语言差不多了，但实际上内部的抽象比命令式语言复杂得多。

### Haskell对IO的处理

IO(input/output)，是程序与外界的数据交流。C语言中用putchar函数向命令行输出一个字符，getchar函数读取一个字符。但Haskell中无法实现这样的putchar、getchar。Haskell中一个函数的输出完全由输入的参数决定，但getchar函数的输出是程序外部与程序进行交互决定的。同时，Haskell中函数没有副作用，一个表达式不求值、求值一次和求值多次是无所谓的，这对putchar函数显然不行。

Haskell中函数的输出完全由输入决定，而getchar的输出跟外部世界有关。前面说过，命令式编程中的数据修改在函数式编程中是数据变换。所以我们尝试让getchar对“外部世界”这个值进行变换，输入旧的“外部世界”，输出新的“外部世界”和读出来的字符。

```haskell
getchar :: (RealWorld -> (RealWorld, result))
```

这刚好就是上文所说的状态转移函数。我们定义针对IO的状态转移函数：

````haskell
newtype IO resultType = IO (RealWorld -> (RealWorld, resultType))
````

我们的Haskell程序居然可以对“外部世界”进行变换！怎么理解这一点？下面是一个玄学的理解方式：

> 将整个宇宙认为是四维时空，这个四维时空同时包含过去、现在和未来，时间的流逝只是第四个维度上的移动。从这个角度看宇宙，没有“变化”的概念。所谓的“变化”和“修改”，只不过是宇宙在时间维度上的不同。Haskell语言中，RealWorld就是某一个时间下的宇宙，IO是整个宇宙的状态转移函数，输入过去的宇宙，输出IO的结果和未来的宇宙。
>
> 注意到，对同样的一个过去宇宙，进行不同IO操作会得到不同的未来宇宙。例如说，`putchar 'a'`和`putchar 'b'`是两个不同的IO操作，对同一个宇宙使用这两个不同的IO操作，会得到两个不同的宇宙，一个宇宙中计算机输出了`a`，另一个宇宙中计算机输出了`b`。这两个宇宙可以认为是平行宇宙。
>
> 一个Haskell程序是一个整个宇宙的状态转移函数，Haskell程序的运行就是将现在的宇宙输入状态转移函数，对未来的可能的平行宇宙中的一个时间线进行求值的过程，在求值可能的未来平行宇宙的过程中操纵计算机进入这个平行宇宙。

这是一个玄学的理解方式。实际上这个`RealWorld`在解释器中被优化掉了，根本无法代表整个宇宙。

对`IO`求值不会导致副作用的执行，因为`IO`只是状态转移函数，状态转移函数需要输入一个状态才能开始“运行”。一个Haskell程序`main`，它的类型是`IO()`，对main的求值过程中只是合成状态转移函数，没有真正执行状态转移，真正的状态转移在合成完状态转移函数后输入`RealWorld`进行。

### 进一步理解Monad

Monad是一种容器或计算过程，这种容器或计算过程可以进行`flatmap`操作。前面提到的Monad有：一个可以空的单元素容器`Optional`(`Maybe`)、列表`MyList`(`[]`)，计算过程`Future`，函数`Function`(`Reader`)，状态转移函数`StateTransfer`(`State`,`IO`)。除此之外这些东西也是Monad：

* `Writer`，一种单元素容器，这个容器除了内容之外还附加一个字符串记录信息，每次flatmap都添加字符串记录信息。
* `Either`，一种单元素容器，可以有两种类型的取值。在将第一个泛型参数柯里化后，相当于Optional，也是一种Monad。
* `Cont`，CPS计算过程。牵扯到CPS变换。

**Monad也可以认为是在某个环境(context)下的值**。容器内元素可以用模式匹配取出，容器内的值是在模式匹配下的环境下的值。对于计算过程，我们定义一个函数，函数的参数有计算过程的输入值，那么在这个函数里面的环境下我们可以获得计算过程的输出值。map操作是将这个环境中的值替换成另一个值，flatmap将这个环境中的值替换成另一个同类环境下的值，然后将两个同类环境合并成一个环境。wrap操作将一个值包含在默认环境中。

map操作干了什么事？对于容器而言，首先查看这个容器的内部结构，（对Optional而言就是看有没有元素），然后将容器内部的值通过一个函数进行替换。对于计算过程而言，是合成一个新的计算过程，这个新的计算过程中，先运行原有计算过程，在获得结果后喂给函数，并将替换后的值作为大计算过程的结果。

flatmap操作干了什么事？对容器而言，首先还是查看容器的内部结构，然后将容器内部的值替换成另一个同类容器，得到容器套容器，然后再将两层容器融合成一层，容器本身的附加信息不能丢失。对Optional而言，融合两层容器，就是两层中有一层没有就全都没有。对列表而言，融合两层容器就是将一堆列表合起来。那么计算过程的flatmap干了什么？合成一个新的计算过程，这个新的计算过程先运行原有的计算过程，获得结果后，通过结果得到另一个计算过程，然后大的计算过程包括了这两个计算过程。

总而言之，**map进行替换。flatmap进行替换和两层容器合并，也就是在map的基础上进行一个“合并(flatten)”操作**：

```haskell
flatten :: (MyMonad m) => (m (m t)) -> (m t)

flatmap monad func = flatten (map_ monad func)
```

`flatten`也被称为为`join`

同时，用flatmap可以实现map: (`liftM`)

````haskell
mapMonad monad func = flatmap monad (\value -> wrap (func value))
````

因此Monad是Functor。

由此可以想到，可以换一种方式定义Monad，Monad可以只有flatten操作，再借助Functor的map操作就可以实现flatmap：

```haskell
class (MyFunctor m) => AnotherMonad m where
  flatten :: (m (m t)) -> m t
  
  flatmap :: (m a) -> (a -> m b) -> (m b)
  flatmap m func = flatten (map m func)
```

对`Optional`实现`AnotherMonad`

```haskell
instance AnotherMonad Optional where
  flatten (None) = None
  flatten (Present (None)) = None
  flatten (Present (Present value)) = Present value
```

对状态转移函数实现`AnotherMonad`

```haskell
instance AnotherMonad (StateTransfer stateType) where
  flatten (StateTransfer_ transfer0) = StateTransfer_
    (\state0 -> let
      (state1, (StateTransfer_ transfer1)) = transfer0 state0
      (state2, result) = transfer1 state1
      in result
    )
```

实现了flatmap就可以实现flatten，实现了flatten就可以实现flatmap，因此这两种Monad定义是等价的。

map和flatmap都可以在不拆开容器的前提下对容器进行操作，保持容器本身的结构和附加信息。对于计算过程，计算过程这个“容器”本身就拆不开，用map和flatmap来基于已有计算过程构造新计算过程。在实现map和flatmap的时候，不能丢掉容器内部结构信息，不能丢掉容器附加信息，也不能丢掉计算过程。

容器和计算过程这两种Monad，他们的有不同的特性：

* 容器的内部值可以立即取得，但是这些值有内部结构。（Optional中可能有可能没有，列表中有先后顺序的结构，树中也有结构，Either和Writer本身就有容器附加信息。）可以用模式匹配解开容器结构这个环境，获取内部值，可能需要递归。进行flatmap的时候，我传进去的函数被立即调用。
* 计算过程的内部值不能立即取得，但是计算过程肯定有且只有一个结果，容器的内部结构体现在计算过程的函数上。需要在新定义的一层函数里面，通过自己定义的参数，获取计算过程结果。进行flatmap的时候，我传进去的函数没有被立即调用，而是在新的计算过程中被调用。

### Monad的数学性质

Functor必须满足的数学性质有：

* `map m id`等价于 `m`。用恒等变换函数变换内部值，相当于什么都没干。map除了变换内部值之外不能干其他的事情。
* `map m (f . g)`等价于 `map (map m g) f`。`.`指函数组合。先进行f再进行g的变换，等价于一下子进行f和g的变换。

Monad必须满足的数学性质有：

* 融合两层环境的时候，如果外层的是默认环境，那么融合起来的结果和只有内层是一样的，有左幺元。`flatten (wrap m)`等价于`m`。`flatmap (wrap x) func`等价于`func x`。
* 融合两层环境的时候，如果内层的是默认环境，那么融合起来的结果和只有外层是一样的，有右幺元。`flatten (map m wrap)`等价于`m`。`flatmap m wrap`等价于`m`。
* 如果有三层环境，先融合外面的两层和先融合里面的两层是等价的，这是结合律。 `flatten (flatten m)`等价于`flatten (map m flatten)`。`flatmap (flatmap m f) g`等价于`flatmap m (\x-> flatmap (f x) g)`。

对于函数类型`t -> m t`，我们可以定义一种针对这种函数的运算

```haskell
operation :: (t -> m t) -> (t -> m t) -> (t -> m t)
operation f1 f2 = \value -> flatmap (f1 value) f2
```

根据上面的Monad性质1和2，可以看出这个运算有幺元`wrap`

```
operation wrap f <=> f
operation f wrap <=> f
```

根据Monad性质3，这个运算满足结合律

```
operation a (operation b c) <=> operation (operation a b) c
```

所以`t -> m t`上的这个运算构成幺半群。不过这个知识对编程没有什么帮助。

### Applicative

Functor允许替换环境中的值，Monad允许将一个环境中的值替换成另一个环境，然后将**嵌套**的两层环境合并成一层。Applicative也是一种环境（容器），它可以将两个环境合并，但是不要求这两个环境是嵌套关系。

Applicative里面，将两个环境合并，指的是将一个装有函数的环境和另一个装有值的环境“合并”：

```haskell
class Applicative m where
  <*> :: m (a -> b) -> m a -> m b
```

前面给了Monad两种不同定义，同理Applicative也可以有两种不同定义。换成直观的“合并”的定义，给定两个环境，再给一个函数进行合并：

```haskell
class Combinable where
  combine :: m a -> m b -> (a -> b -> c) -> m c
```

Applicative首先是Functor。这两种不同的定义也可以相互转化。Monad是Applicative，但Applicative不是Monad。

Monad和Applicative都有“合并两个环境”的功能，但是Monad可以合并有嵌套关系的环境，将`m (m t)`合并成`m t`，但是Applicative不能合并这种嵌套的两层环境，Applicative只能将两个并列关系的环境融合。

假如这里的环境指的是计算过程，Monad可以融合两个计算过程，后面的计算过程依赖于前面的计算过程的结果，而Applicative也可以融合两个计算过程，但是第二个计算过程本身不能依赖于第一个计算过程的结果。后面提到的用状态转移函数和Optional实现字符串解析，类似于parsec，如果有上下文有关文法（后面解析的文法依赖于前面解析出来的数据），必须用Monad功能来组合，但如果是上下文无关文法可以用Applicative来组合。

### Monad Transformer

有时候需要一个Monad同时有两种Monad的性质，例如说Optional代表可能出错的计算结果，它的flatmap可以串联可能出错的计算，我又想把它跟StateTransfer状态转移函数结合。也就是说，我需要一种套娃Monad，在一个Monad里套另一个Monad，把两层Monad看成一种新容器。Monad transformer是一种容器，他通过泛型参数输入一个Monad容器类型`m`，然后将这个Monad容器与自己的Monad容器套娃，形成新的Monad容器。

Monad transformer还要实现另一个函数 (`lift`)，将一个monad包到自己里面，这里叫`wrapTransformer`

```haskell
class MonadTransformer transformer where
  wrapTransformer :: (MyMonad m) => (m a) -> (transformer m a)
```

每种Monad都有对应的Monad transformer：

| Monad类型                                                | 对应的Monad Transformer的类型              |
| -------------------------------------------------------- | ------------------------------------------ |
| `Optional t`                                             | `m (Optional t)`                           |
| `MyList t`                                               | `m (MyList t)`                             |
| `Computation` : `inputType -> outputType`                | `inputType -> (m outputType)`              |
| `StateTransfer` : `stateType -> (stateType, resultType)` | `stateType -> (m (stateType, resultType))` |

Monad有两种：容器和计算过程。对于容器，Monad transformer是用新monad将整个容器包住，而对于计算过程，只是将函数的输出包住。

对于Optional的Monad transformer `m (Optional t)`可以这么定义并实现：

```haskell
newtype OptionalT m t = OptionalT_ (m (Optional t))
takeOptionalT (OptionalT_ content) = content

flatmapOptionalWrapped :: (MyMonad m) => (Optional t) -> (t -> m (Optional u)) -> (m (Optional u))
flatmapOptionalWrapped (None) func = wrap None
flatmapOptionalWrapped (Present value) func = func value

instance (MyMonad m) => MyMonad (OptionalT m) where
  wrap value = OptionalT_ (wrap (Present value))
  flatmap (OptionalT_ optinalInMonad) func =
    OptionalT_ (flatmap optinalInMonad (
        \optional -> flatmapOptionalWrapped optional (\value -> takeOptionalT (func value))
    ))

instance MonadTransformer OptionalT where
  wrapTransformer valueInMonad = OptionalT_ (mapMonad valueInMonad (\value -> (Present value)))
```



状态转移函数的Monad transformer：

```haskell
newtype StateTransferT stateType m resultType = StateTransferT_ (stateType -> m (stateType, resultType))
getTransferT (StateTransferT_ func) = func

instance (MyMonad m) => MyMonad (StateTransferT stateType m) where
  wrap value = StateTransferT_ (\state -> wrap (state, value))
  flatmap (StateTransferT_ stateTransfer1) func = StateTransferT_
    (\state0 -> flatmap (stateTransfer1 state0) (\(state1, result1) -> let
        (StateTransferT_ stateTransfer2) = func result1
        in stateTransfer2 state1
    ))

instance MonadTransformer (StateTransferT stateType) where
  wrapTransformer monad = StateTransferT_ (\state -> mapMonad monad (\val -> (state, val)))
```

---

可能会想到，Monad transformer为什么是这个定义？例如说，`Optional t`的Monad transformer可不可以是`Optional (m t)`？答案是不行。对于`Optional (m t)`这种容器，如果尝试一下，就会发现无法正常实现flatmap操作或者flatten操作。他的flatten操作的类型是：

```haskell
flatten :: Optional (m (Optional (m a))) -> Optional (m a)
```

flatten输出的Optional的容器状态需要是输入的两层Optional容器状态的合并，但是如果m是一个计算过程，那么输入的内层的Optional容器状态不确定，因此这个flatten输出的容器的状态也不能确定。对于列表等容器的Monad Transformer也是如此。

对于计算过程类Monad，例如函数`Function`(`Reader`)的Monad transformer，假如定义为`m (input -> output)`，也无法实现flatmap或flatten，他的flatten操作的类型是

```haskell
flatten :: m (input -> m (input -> output)) -> m (input -> output)
```

可以看到最后输出值的m需要由输入值的两层m共同组合确定，但是内层的m的状态是不确定的，需要输入input值才能确定，但是这里无法获取input值，所以这个flatten无法实现。

状态转移函数的Monad transformer的类型可不可以是`state -> (state, m result)`？他的flatten的类型是

```haskell
flatten :: (state -> (state, m (state -> (state, m result)))) -> (state -> (state, m result))
```

如果m是计算过程，那么内层的状态转移函数无法取出，就算给定输入状态也不能确定输出状态。因此这个也不能实现。

---

可以看到，在有了Monad transformer之后，我们不需要实现普通Monad了，转而用Monad transformer，使用各种Monad transformer套娃得到我想要的Monad，套娃的起始是一个最简单的容器`SimpleBox`：

```haskell
newtype SimpleBox t = SimpleBox_ t
instance (MyMonad SimpleBox) where
  wrap value = SimpleBox_ value
  flatmap (SimpleBox_ value) func = func value
```

这个最简单的容器也被称为Identity Monad (`Identity`)，负责将一个值包装成Monad的样子。

例如说，我想要将字符串解析成数据，他是一个字符一个字符读，读到的位置是状态，这是一个状态转移过程。同时，一个解析过程可能会失败，失败就输出空。这个东西就可以用Monad Transformer得到，可以将Optional和状态转移函数相组合。

还有其他例子。单纯的状态转移函数只能做一个状态的变换。前面用栈操作对状态转移函数进行了示例，但是那个栈操作只是操作一个栈。可不可以用状态转移函数操作多个状态？一般的做法是将两个状态放在一个更大的状态对象里。用状态转移函数的套娃也可以做到。例如说我要定义一个套娃状态转移函数`moveTop2Elements`，同时操作两个栈，将第一个栈的两个元素出栈并放入第二个栈：

```haskell
toStateTransferT :: (MyMonad m) => (StateTransfer stateType resultType) -> (StateTransferT stateType m resultType)
toStateTransferT (StateTransfer_ func) = StateTransferT_ (\oldState -> wrap (func oldState))

moveTop2Elements :: StateTransferT (MyList Integer) (StateTransferT (MyList Integer) SimpleBox) ()
moveTop2Elements = flatmap (toStateTransferT pop) (\top1 ->
    flatmap (toStateTransferT pop) (\top2 ->
      flatmap (wrapTransformer (toStateTransferT (push top1))) (\whatever1 ->
        (wrapTransformer (toStateTransferT (push top2)))
      )
    )
  )
-- 用do语法糖：
moveTop2Elements = do
  top1 <- (toStateTransferT pop)
  top2 <- (toStateTransferT pop)
  wrapTransformer (toStateTransferT (push top1))
  wrapTransformer (toStateTransferT (push top2))
```

(这里第一个栈的状态转移是外层Monad，第二个栈的状态转移在内层Monad)

使用这个套娃状态转移函数：

```haskell
let stack1 = (ListNode 1 (ListNode 2 EmptyList))
let stack2 = EmptyList
let (SimpleBox_ (newStack2, (newStack1, _))) = getTransferT (getTransferT moveTop2Elements stack1) stack2
-- newStack1 = EmptyList
-- newStack2 = ListNode 2 (ListNode 1 EmptyList)
```

