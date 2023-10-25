## 为什么组合优于继承

面向对象编程中，继承可以用来实现多态并复用代码，而经过大量实践后，人们发现**组合优于继承**的理念，新语言Rust和Go都推崇组合优于继承的理念，抛弃了继承并加入了非侵入式接口。

组合与继承都可以用来复用代码，代码复用有很多种，这里主要指让不同的类型共用一部分相同的逻辑与数据结构。如果我已经有一个了类`A`，我要在`B`里面复用A，这是继承（用Java示例）：

```java
class B extends A { ... }
```

这是组合：

```java
class B {
	private A a;
	...
}
```

继承与组合的区别

|                | 继承                                      | 组合                               |
| -------------- | ----------------------------------------- | ---------------------------------- |
| 关系           | is-a  (分类关系)                          | has-a                              |
| 关系怎样确定   | 必须在代码中写死继承哪个类                | 可以在运行时灵活确定               |
| 耦合程度       | 强，可以重写内部方法                      | 弱，只能重写外部方法               |
| 封装能力       | 无法隐藏父类public内容                    | 可以隐藏内部对象的public内容       |
| 代码重用方式   | 白盒(必须知道父类实现才能正确继承)        | 黑盒(组合不需要知道组件的具体实现) |
| 生命周期       | 父类与子类部分必须一起创建(C++中一起销毁) | 生命周期不耦合                     |
| 单元测试方便性 | 不方便，不能离开父类测试子类              | 方便(可以进行mock)                 |
| 手动写代理代码 | 不需要                                    | 需要(Go语言嵌入不需要)             |
| 是否允许抽象类 | 可以继承没有实现所有方法的抽象类          | 组合所用的类必须是完整的           |

> 注：C++中的private继承允许隐藏父类的public内容，但private继承实际上相当于组合。

[参考](https://octo.vmware.com/golang-embedding/)

## 继承的强耦合

继承的耦合程度比组合要强，正如上面的表格所说，子类的扩展部分不能离开父类部分存在，生命周期耦合。单元测试也不能离开父类测试子类的扩展部分。继承哪个类也必须在代码里写死，不能运行时灵活决定。

例如我设计了`Print`类用于在命令行打印一个字符串

```java
public class Print {
    public void print(String str){
        System.out.print(str);
    }
}
```

后面对其进行继承，设计了`PrintWithTime`，在打印内容的前面加上时间：

```java
public class PrintWithTime extends Print {
    @Override
    public void print(String str) {
        DateTimeFormatter dtf = DateTimeFormatter.ofPattern("yyyy/MM/dd HH:mm:ss");
        LocalDateTime now = LocalDateTime.now();
        System.out.print(dtf.format(now) + " ");
        super.print(str);
    }
}
```

又进一步扩展，在打印时间和内容的同时打印当前线程的名字

```java
public class PrintWithTimeAndThread extends PrintWithTime {
    @Override
    public void print(String str) {
        Thread t = Thread.currentThread();
        System.out.print(t.getName() + " ");
        super.print(str);
    }
}
```

我可以这样进行使用：

```java
Print p = new PrintWithTimeAndThread();
p.print("Message");
```

然后就能输出

```
main 2022/07/17 21:23:08 Message
```

问题来了：如果我希望只打印线程名字，不打印时间，怎么办？由于继承要求写死父类，已有的`PrintWithTimeAndThread`竟然无法复用，不得不写重复代码：

```java
public class PrintWithThread extends Print {
    @Override
    public void print(String str) {
        Thread t = Thread.currentThread();
        System.out.print(t.getName() + " ");
        super.print(str);
    }
}
```

除此之外，如果我想要先打印时间再打印线程，还是要继续另写，非常麻烦。

## 用组合代替继承-1

用组合可以轻松解决这个问题：

```java
public interface Printable {
    public void print(String str);
}

public class RawPrinter implements Printable {
    public void print(String str) {
        System.out.print(str);
    }
}

public class PrintWithTime implements Printable {
    private final Printable delegate;
    
    public PrintWithTime(Printable delegate) {this.delegate = delegate;}
    
    @Override
    public void print(String str) {
        DateTimeFormatter dtf = DateTimeFormatter.ofPattern("yyyy/MM/dd HH:mm:ss");
        LocalDateTime now = LocalDateTime.now();
        System.out.print(dtf.format(now) + " ");
        delegate.print(str);
    }
}

public class PrintWithThread implements Printable {
    private final Printable delegate;
    
    public PrintWithThread(Printable delegate) {this.delegate = delegate;}
    
    @Override
    public void print(String str) {
        Thread t = Thread.currentThread();
        System.out.print(t.getName() + " ");
        delegate.print(str);
    }
}
```

这个刚好是设计模式中的代理(Proxy)模式。

可以这么使用

```java
PrintWithThread printer = new PrintWithThread(new PrintWithTime(new RawPrinter()));
printer.print("Message");
```

如果不想要打印时间，可以`new PrintWithThread(new RawPrinter())`，如果要先打印时间再打印线程，可以`new PrintWithTime(new PrintWithThread(new RawPrinter()))`

换成lambda表达式能更简洁，顺便具有函数式风格：

```java
public static interface Printable {
    public void print(String str);
    public static Printable rawPrinter(){
        return str -> System.out.print(str);
    }
    public static Printable printWithTime(Printable delegate){
        return str -> {
            DateTimeFormatter dtf = DateTimeFormatter.ofPattern("yyyy/MM/dd HH:mm:ss");
            LocalDateTime now = LocalDateTime.now();
            System.out.print(dtf.format(now) + " ");
            delegate.print(str);
        };
    }
    public static Printable printWithThread(Printable delegate){
        return str -> {
            Thread t = Thread.currentThread();
            System.out.print(t.getName() + " ");
            delegate.print(str);
        };
    }
}
```

## 脆弱基类问题(Fragile base class problem)

继承带来强耦合，意味着你必须了解父类的实现方式的情况下才能正确继承，正确重写方法，必须将父类看做白盒，而不能将父类看做黑盒去使用。

> 注：这里并不是说黑盒比白盒更好。一旦要性能优化，就不得不打开黑盒了解内部实现。例如说，内存本来可以看做黑盒，但一旦要性能优化就必须注意缓存与内存布局。数据库的查询方式也可以当做黑盒，但要优化查询性能必须了解数据库的查询方式。

如果父类的接口不变但是实现变了，那么子类可能就无法正常工作。

有一个例子 [(参考)](https://cscalfani.medium.com/goodbye-object-oriented-programming-a59cda4c0e53)。例如说在Java里有一个普通的链表容器：

```java
public class MyLinkedList<T> {
    private static class Node<T1> {
        public T1 element; public Node<T1> next; public Node<T1> prev;
        public Node(T1 element) {this.element = element;}
    }
    
    Node<T> head = null;
    Node<T> tail = null;
    
    public MyLinkedList() {}
    
    public void add(T element) {
        Node<T> node = new Node<>(element);
        addNode(node);
    }
    
    private void addNode(Node<T> node) {
        if (head == null) {head = node; tail = node;}
        else {node.prev = tail; tail.next = node; tail = node;}
    }
    
    public boolean remove(T element) {
        Node<T> curr = head;
        while (curr != null) {
            if (curr.element.equals(element)) {
                removeNode(curr); return true;
            }
            curr = curr.next;
        }
        return false;
    }
    
    private void removeNode(Node<T> node) {
        if (node.prev != null) node.prev.next = node.next;
        else head = node.next;
        if (node.next != null) node.next.prev = node.prev;
        else tail = node.prev;
        node.prev = null; node.next = null;
    }
    
    public void addAll(Collection<T> collection) {
        for (T t : collection) {
            addNode(new Node<T>(t)); // 注意，这里即将修改
        }
    }
}
```

> 注：这里只是举一个例子，实际上不应该这么写链表

这个链表不能记录元素的个数，我想要给这个链表加入记录元素个数的功能，但是那个链表是别人写的，于是我用继承来扩展：

```java
public class MyCountedLinkedList<T> extends MyLinkedList<T> {
    private int size = 0;
    
    public int getSize() {
        return size;
    }
    
    @Override
    public void add(T element) {
        super.add(element);
        size++;
    }
    
    @Override
    public boolean remove(T element) {
        boolean removed = super.remove(element);
        if (removed) { size--; }
        return removed;
    }
    
    @Override
    public void addAll(Collection<T> collection) {
        super.addAll(collection);
        size += collection.size();
    }
}
```

然后一切正常工作。不过后面那个别人写的链表中`addAll`方法被进行了修改：

```java
public class MyLinkedList<T> {
	......
	
    public void allAll(Collection<T> collection) {
        for (T t : collection) {
            add(t); // 已被修改，原来是 addNode(new Node<T>(t));
        }
    }
}
```

然后这个时候使用`MyCountedLinkedList`的`addAll`方法就会导致size被增加了两遍(被重写的`add`里面增加了一遍,被重写的`addAll`里面又增加了一遍)，导致程序出bug。

用继承进行扩展，往往依赖于父类的具体实现，一旦父类发生改变，子类就可能坏掉。

## 用组合代替继承-2

这里采用组合（代理模式）的方式来实现记录元素个数的链表：

```java
public class BetterCountedLinkedList<T> {
    private final MyLinkedList<T> list = new MyLinkedList<>();
    private int size = 0;
    
    public int getSize() {
        return size;
    }
    
    public void add(T element) {
        list.add(element);
        size++;
    }
    
    public boolean remove(T element) {
        boolean removed = list.remove(element);
        if (removed) {
            size--;
        }
        return removed;
    }
    
    public void addAll(Collection<T> collection) {
        list.addAll(collection);
        size += collection.size();
    }
}
```

这样，`MyLinkedList`实现的变化就不会导致`BetterCountedLinkedList`出现问题。这个例子体现了组合优于继承。

## 继承难以表达的分类关系

继承表示is-a关系，B继承A表示B是一种A。同时还有里氏替换原则，任何用A的地方传入B对象都没有问题。继承与分类关系是绑定在一起的。这个时候遇到了问题：

**继承不能同时表达两种不同的分类方式。**

一般用继承可以表达分类关系，例如说形状，圆形、正方形、长方形都是形状，正方形是长方形，所以正方形类继承长方形。

```java
abstract class Shape{}
class Circle extends Shape{}
class Rectangle extends Shape{}
class Square extends Rectangle{}
```

这种情况下，继承关系与分类关系完美契合，但事情没这么简单。

有一天，我想对形状加入着色功能，加入带颜色的圆形、带颜色的长方形、带颜色的正方形、当然还有一个抽象的带颜色图形类：

```java
abstract class ColoredShape extends Shape{}
class ColoredCircle extends ColoredShape{}
class ColoredRectangle extends ColoredShape{}
class ColoredSquare extends ColoredRectangle{}
```

`ColoredCircle`是`ColoredShape`的子类，但是显然`ColoredCircle`应该也是`Circle`的子类。如果使用C++的多重继承，让`ColoredCircle`同时继承`Circle`和`ColoredShape`，就会遇到上面说的菱形继承问题。不过，将`ColoredShape`变成一个interface就没问题了，从这个例子可以看出实现interface比直接继承更灵活。

这些混乱都源于出现了两种分类方式，第一种分类方式是图形的形状，第二种分类方式是有没有颜色。

![u_3941628830,1087044509_fm_253_app_138_size_w931_n_0_f_JPEG_fmt_auto.webp](https://s2.loli.net/2022/08/11/hDm3JlOgHUsKqAi.webp)

如图，如果让继承同时按A和B分类，交叉的地方就会很尴尬。这时遇到了经典的**菱形继承问题**：

![image.png](https://s2.loli.net/2022/07/24/eb9d6DYKN2IWlLh.png)

图中，`Scanner`内部存放了一份`PoweredDevice`的数据，`Printer`中也存放了一份`PoweredDevice`的数据，`Copier`中要存放`Scanner`的数据和`Printer`的数据，总共包含两份`PoweredDevice`的数据。

C++为了解决这个问题，加入了虚继承的概念，让`Scanner`和`Printer`对`PoweredDevice`虚继承，这样`Copier`里面只有一份`PoweredDevice`的数据。

而Java的解决方案是引入接口(interface)的概念，不允许多继承，但允许实现多个接口，实现接口只进行1. 产生子类型关系 2.继承父类API 两件事，不进行对父类数据的继承，接口类只包含一些API，不包含数据。（但由于Java中的接口允许对某些方法提供默认实现，因此仍有可能遇到菱形问题，不过在实践中很少遇到）

那么，采用接口，就能不尴尬地表达这种分类关系了吗？

![u_3941628830,1087044509_fm_253_app_138_size_w931_n_0_f_JPEG_fmt_auto.webp](https://s2.loli.net/2022/08/11/hDm3JlOgHUsKqAi.webp)

答案仍然是否定的。如果让A或者B变成interface，就可以表达，但是总共要4个类，如果加入第三种分类方式C，可能就要8个类，再加一种分类关系就16个类，这就更加尴尬了。

**类(class)根本不适合表达分类(classification)关系，除非只有一种分类关系**。

C++的标准库的继承关系也出现了这种混乱：

![](https://upload.cppreference.com/mwiki/images/0/06/std-io-complete-inheritance.svg)

`basic_fstream`从分类上说有两种继承方式：第一种：继承`basic_iostream`，第二种：继承`basic_ofstream`和`basic_ifstream`，实际上采用了第一种，然后读文件的代码在`basic_fstream`和`basic_ifstream`中重复两次。由于同时读写文件的与只读文件在缓冲区处理上不一样，所以采用多重继承不能自动将读写组合起来。

对于继承难以表达的多重分类关系，解决方案是将对象的每一种"特性"都单独做成一个对象，再将"特性对象"组合起来。

## 使用接口，将API与类解耦

> 这里的API指一个对象可以对其调用的方法。

B继承A就是继承了A的所有API（A能调用的方法对B也能调用），同时也继承了A的所有数据。接口(interface)只包含API，不包含数据，将API与类本身解耦。

采用组合代替继承，遇到了怎样实现多态的问题。如果B不是继承A，而只是组合了一个A，B不会自动获得A的所有API。解决方案是将A的方法单独抽成一个接口，再让A和B去实现那个接口。

这时，有时候会需要对一个已有的类取实现我自己定义的接口，需要非侵入式接口。

## 非侵入式接口

Java中，你可以为自己写的一个类实现一个接口，但不能为别人写的一个类实现接口。例如说我定义了一个接口`StringSerializable`，我想让`String`，`ArrayList`等不是我自己写的类去实现`StringSerializable`是做不到的（当然可以自己定义一个类进行包装，但是会更麻烦）。Java在定义一个类的同时确定实现哪些接口，这被称为侵入式接口。

Rust与Go都支持非侵入式接口，我可以给别人写的类去实现自己定义的接口。

例如说Java中有`toString`方法，但是`toString`会带来大量临时字符串对象，拼接大量临时字符串对象消耗性能，我想直接使用`StringBuffer`去拼接字符串，避免产生临时对象优化性能。所以我定义了一个`StringSerializable`:

```java
interface StringSerializable {
	void appendToStringBuffer(StringBuffer buf);
}
```

我可以为我自己定义的类实现`StringSerializable`

```java
class Point implements StringSerializable {
    int x; int y;
    
    @Override
    void appendToStringBuffer(StringBuffer buf) {
    	buf.append(x);
    	buf.append(",");
    	buf.append(y);
    }
}
```

但是这个时候我想对`Integer`和`ArrayList`实现我的`StringSerializable`

```java
class Integer implements StringSerializable {
	...
	
	@Override
    void appendToStringBuffer(StringBuffer buf) {
    	buf.append(this);
    }
}

class ArrayList<T> implements StringSerializable {
    ...
    
	@Override
	void appendToStringBuffer(StringBuffer buf) {
		for (T obj: this) {
            ((StringSerializable) obj).appendToStringBuffer(buf);
        }
	}
}
```

但是Java不允许这种操作，已经定义好的`Integer`类和`ArrayList`类不能实现我的新接口。

而Rust和Go都支持这种操作：

Rust:

```rust
trait StringSerializable{
    fn append_to_string(&self, s: &mut String);
}

impl StringSerializable for i32{
    fn append_to_string(&self, s: &mut String){
        s.push_str(&self.to_string());
    }
}

struct Point{
    x: i32,
    y: i32
}

impl StringSerializable for Point{
    fn append_to_string(&self, s: &mut String){
        self.x.append_to_string(s);
        s.push_str(",");
        self.y.append_to_string(s);
    }
}
```

Go:

```go
type StringSerializable interface {
	AppendToStringBuilder(sb *strings.Builder)
}

type Point struct {
	x int
	y int
}

func (p *Point) AppendToStringBuilder(sb *strings.Builder) {
	sb.WriteString(strconv.Itoa(p.x))
	sb.WriteString(",")
	sb.WriteString(strconv.Itoa(p.y))
}

type Int int

func (i Int) AppendToStringBuilder(sb *strings.Builder) {
	sb.WriteString(strconv.Itoa(int(i)))
}
```

> 注：Go语言不允许直接对内置类型`int`实现方法，`Int`是另一个包装`int`的类型。

虽然Java不允许非侵入式接口，但可以自己写包装类，不过要手动写代理代码（将接口的方法调用代理到内部对象），还要手动将原本的类转换成包装类：

```java
class MyInteger implements StringSerializable {
	int data;
    
	@Override
    void appendToStringBuffer(StringBuffer buf) {
    	buf.append(this);
    }
}

class MyArrayList<T> implements List<T>, StringSerializable {
    ArrayList<T> data;
    
    @Override
    void appendToStringBuffer(StringBuffer buf) {
    	...
    }
    
    ...
    
    // 代理代码，List的每个方法都要写，重复且易出错
    @Override
    boolean add(T obj) { return data.add(obj); }
    @Override
    boolean remove(T obj) { return data.remove(obj); }
    ...
}
```

Go语言的嵌入功能可以免除写这些重复、易出错的代理代码。

## Go语言的嵌入(Embedding)

Go语言允许三种嵌入：

* 将一个struct嵌入到另一个struct里面（类似继承但不一样）
* 将一个interface嵌入到一个struct里面（实现代理模式）
* 将一个interface嵌入到另一个interface里面（A接口里面嵌入B接口，对应于Java中的A继承B）

前两种嵌入，相当于进行组合，并自动生成代理代码。



### Go语言中struct嵌入struct

Java中继承的作用总共有三个：

1. 产生子类型关系。子类是父类的子类型，子类引用可以自动转为父类引用，一个父类引用可能实际指向一个子类对象。

2. 父类的所有数据被继承到子类，子类对象中包含父类对象的所有数据。

3. 父类的方法被继承到子类，父类对接口的实现也被继承到子类。父类的一个方法对子类对象也可以调用，子类可以重写其中一些方法，或者对没有实现的方法提供实现。

而Go语言中，将struct嵌入struct的嵌入是类似继承的，能继承数据和方法。但相比Java，少了第一个作用，也就是产生子类型关系。

嵌入与继承的第二个区别，就是只能重写外部调用的方法，不能重写内部调用的方法。

嵌入与继承的区别：

| 继承                               | Go语言嵌入(组合)                                     |
| ---------------------------------- | ---------------------------------------------------- |
| 产生子类型关系                     | 不产生子类型关系，但自动实现被嵌入类型实现的所有接口 |
| 父类部分不是单独的对象             | 被嵌入的对象可以单独引用(获取内部指针)               |
| 父类的代码内部可以调用重写了的方法 | 被嵌入的对象不会调用重写了的方法                     |





例如，在Java中这样的代码：

```java
static class Base {
    protected int id;
    
    public void setId(int i) {setIdInternal(i);}
    
    public void setIdInternal(int i) {
        System.out.println("Base setIdInternal");
        this.id = id;
    }
    
    public int getId() {return id;}
}
static class Sub extends Main.Base {
    public String name;
    
    @Override // 重写内部调用的方法
    public void setIdInternal(int i) {
        System.out.println("Sub setIdInternal");
        this.id = i;
    }
    
    @Override // 重写外部调用的方法
    public int getId() {return id + 1;}
}
public static void main(String[] args) {
    Sub sub = new Sub();
    sub.setId(1);
    System.out.println(sub.getId());
}
```

子类Sub继承父类Base，覆盖了`setIdInternal`以及`getId`。运行后输出

```
Sub setIdInternal
2
```

可以看到两个方法都被成功覆盖。但是对应的Go语言代码（用嵌入代替继承）：

```go
type IdAccess interface {
	getId() int
}

type Base struct {
	id int
}

func (b *Base) setIdInternal(i int) {
	fmt.Print("Base setIdInternal")
	b.id = i
}

func (b *Base) setId(i int) {
	b.setIdInternal(i)
}

type Sub struct {
	Base // 将Base嵌入Sub
	name string
}

func (b *Sub) setIdInternal(i int) { // 实际上没有覆盖方法
	fmt.Print("Sub setIdInternal")
	b.id = i
}

func (b *Sub) getId() int { // 重写外部调用的方法
	return b.id + 1
}

func main() {
	var obj *Sub = &Sub{}
	obj.setId(1)
	fmt.Print(obj.getId())
}
```

却输出：

```
Base setIdInternal
2
```

`getId`被覆盖但`setIdInternal`没有，调用`setIdInternal`的时候实际上是在调用`Base`的方法。

Go语言允许将从**外部直接调用**的方法`getId`覆盖，但是在方法里面**内部调用**的`setIdInternal`却不能覆盖。因此Go语言可以防止脆弱基类问题，你无法通过重写去“侵入”父类的内部实现，前面的`MyCountedLinkedList`在Go语言中写不出来，不得不用组合代替继承，写出前面的`BetterCountedLinkedList`。Go语言从设计上就是鼓励组合替代继承。

Go中嵌入不会产生子类型关系，也就是说获得一个Base类型指针`*Base`的时候，它一定指向一个`Base`类型对象，不可能是`Sub`类型对象，如果需要多态的话只要定义另外的interface即可。反之，在Java中，获得`Base`引用的时候，它可能实际上指向`Sub`对象，如果你既想要允许`Base`的继承（不标记为`final`），但又想要从类型上限定只接受`Base`类型对象，Java是不允许的（当然可以在运行时进行检查，就是写起来稍微麻烦一点）。

关于Rust： Rust既没有继承也没有类似Go语言中嵌入的功能，有[跟继承相关的RFC](https://github.com/rust-lang/rfcs/issues/349)。

## 客观看待继承

上面说了许多继承的缺点，但实际上继承并不是一无是处，在GUI方面还是比较合适的，GUI中组件的分类关系很适合用继承关系表达，同时浏览器中DOM的不同类型也适合用继承来表达。

这两种比较合适的继承的使用情况都恰好跟图形界面相关。实际上图形界面有三种编程范式：声明式（包括前端React和Vue），命令式（包括Qt与JavaFX），实时模式界面（包括[ImGUI](https://github.com/ocornut/imgui)，类似游戏一直60帧实时渲染界面），这就是另一个话题了。

除此之外，能用单一分类方式分类的数据适合用继承表示，例如游戏中不同物件、实体的分类关系。

尽管组合、非侵入式接口比面向对象更有优势，但仍然不能阻止写出糟糕代码，用传统面向对象也能写出优秀代码，总而言之，没有银弹。







