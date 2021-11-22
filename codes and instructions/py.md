#### type and object

Type是class的顶端，object是实例的顶端。

```python
>>> object.__class__
<type 'type'> ##是type的实例
>>> object.__bases__
()  ##无父类

>>> type.__class__
<type 'type'> ##type的类型是自己
>>> type.__bases__
<type 'object'>

```

object是type的一个实例，type是一种object。

![img](C:\Users\admin\Documents\Typora\typora-pics\ca54cfa2cc510d2dcc40e3cc7fb2e051_720w.jpg)

第一列，元类列，type是所有元类的父亲。我们可以通过继承type来创建元类。

第二列，TypeObject列，也称类列，object是所有类的父亲，大部份我们直接使用的数据类型都存在这个列的。

第三列，实例列，实例是对象关系链的末端，不能再被子类化和实例化。



#### super

一般情况（单继承），调用父类方法。

###### MRO列表（method resolution order）

事实上，对于你定义的每一个类，Python 会计算出一个方法解析顺序（Method Resolution Order, MRO）列表，它代表了类继承的顺序，我们可以使用下面的方式获得某个类的 MRO 列表：

```python
>>> print(C.mro())
[<class '__main__.C'>, <class '__main__.A'>, <class '__main__.B'>, <class '__main__.Base'>, <class 'object'>]
```

这个列表真实的列出了类C的继承顺序。C->A->B->Base->object。在方法调用时，是按照这个顺序查找的。
那这个 MRO 列表的顺序是怎么定的呢，它是通过一个 C3 线性化算法来实现的，这里我们就不去深究这个算法了，感兴趣的读者可以自己去了解一下，总的来说，一个类的 MRO 列表就是合并所有父类的 MRO 列表，并遵循以下三条原则：

- 子类永远在父类前面
- 如果有多个父类，会根据它们在列表中的顺序被检查
- 如果对下一个类存在两个合法的选择，选择第一个父类

###### super原理

```python
def super(cls, inst):
	mro = inst.__class__mro()
	return mro[mro.index(cls)+1]
```

对于`super(a_type, obj)`, MRO是指`type(obj)`的MRO，MRO中的类是`a_type`,同时`isinstance(obj, a_type) == True` 。



#### 函数式编程Map/Reduce/filter/sorted

map接收一个函数和 Iterable,map将传入的函数依次作用到序列的每个元素。

reduce() 则是把计算结果和下一个元素做累计计算。

Python内建的`filter()`函数用于过滤序列。

和`map()`类似，`filter()`也接收一个函数和一个序列。和`map()`不同的是，`filter()`把传入的函数依次作用于每个元素，然后根据返回值是`True`还是`False`决定保留还是丢弃该元素。

注意到`filter()`函数返回的是一个`Iterator`，也就是一个惰性序列，所以要强迫`filter()`完成计算结果，需要用`list()`函数获得所有结果并返回list。

`sorted()`函数也是一个高阶函数，它还可以接收一个`key`函数来实现自定义的排序

#### 垃圾回收GC(CPython)

- 引用计数：python中每一个对象就是`pyobject`结构体，包含有一个`ob_refcnt`的值来反映引用量。可以通过`sys`模块的`getrefcount`函数来获得对象的引用计数。引用计数的内存管理方式在遇到循环引用的时候就会出现致命伤，因此需要其他的垃圾回收算法对其进行补充。

- 标记清理（Mark and Sweep）：解决容器类型可能产生的循环引用，该算法在垃圾回收时分为两个阶段：标记阶段，遍历所有的对象，如果对象是可达的（被其他对象引用），那么就标记该对象为可达；清除阶段，再次遍历对象，如果发现某个对象没有标记为可达，则就将其回收。

  > CPython底层维护了两个双端链表，一个链表存放着需要被扫描的容器对象（姑且称之为链表A），另一个链表存放着临时不可达对象（姑且称之为链表B）。为了实现“标记-清理”算法，链表中的每个节点除了有记录当前引用计数的`ref_count`变量外，还有一个`gc_ref`变量，这个`gc_ref`是`ref_count`的一个副本，所以初始值为`ref_count`的大小。执行垃圾回收时，首先遍历链表A中的节点，并且将当前对象所引用的所有对象的`gc_ref`减`1`，这一步主要作用是解除循环引用对引用计数的影响。再次遍历链表A中的节点，如果节点的`gc_ref`值为`0`，那么这个对象就被标记为“暂时不可达” （`GC_TENTATIVELY_UNREACHABLE`）并被移动到链表B中；如果节点的`gc_ref`不为`0`，那么这个对象就会被标记为“可达“（`GC_REACHABLE`），对于”可达“对象，还要递归的将该节点可以到达的节点标记为”可达“；链表B中被标记为”可达“的节点要重新放回到链表A中。在两次遍历之后，链表B中的节点就是需要释放内存的节点。

- 分代回收：在循环引用对象的回收中，整个应用程序会被暂停，为了减少应用程序暂停的时间，Python 通过分代回收（空间换时间）的方法提高垃圾回收效率。

  > 分代回收的基本思想是：**对象存在的时间越长，是垃圾的可能性就越小，应该尽量不对这样的对象进行垃圾回收**。CPython将对象分为三种世代分别记为`0`、`1`、`2`，每一个新生对象都在第`0`代中，如果该对象在一轮垃圾回收扫描中存活下来，那么它将被移到第`1`代中，存在于第`1`代的对象将较少的被垃圾回收扫描到；如果在对第`1`代进行垃圾回收扫描时，这个对象又存活下来，那么它将被移至第2代中，在那里它被垃圾回收扫描的次数将会更少。分代回收扫描的门限值可以通过`gc`模块的`get_threshold`函数来获得，该函数返回一个三元组，分别表示多少次内存分配操作后会执行`0`代垃圾回收，多少次`0`代垃圾回收后会执行`1`代垃圾回收，多少次`1`代垃圾回收后会执行`2`代垃圾回收。需要说明的是，如果执行一次`2`代垃圾回收，那么比它年轻的代都要执行垃圾回收。如果想修改这几个门限值，可以通过`gc`模块的`set_threshold`函数来做到。

#### 迭代器和生成器

```python
class Fib(object):
    def __init__(self, num):
        self.num = num
        self.a, self.b = 0, 1
        self.idx = 0
   
    def __iter__(self):
        return self

    def __next__(self):
        if self.idx < self.num:
            self.a, self.b = self.b, self.a + self.b
            self.idx += 1
            return self.a
        raise StopIteration()
```

迭代器是一个可以记住遍历的位置的对象，有两个基本的方法：`iter()` 和 `next()`。`__iter__()` 方法返回一个特殊的迭代器对象， 这个迭代器对象实现了  `__next__() `方法并通过 `StopIteration` 异常标识迭代的完成。`__next__()` 方法会返回下一个迭代器对象。

```python
def fibonacci(n): # 生成器函数 - 斐波那契
    a, b, counter = 0, 1, 0
    while True:
        if (counter > n): 
            return
        yield a
        a, b = b, a + b
        counter += 1
f = fibonacci(10) # f 是一个迭代器，由生成器返回生成
 
while True:
    try:
        print (next(f), end=" ")
    except StopIteration:
        sys.exit()
```

在 Python 中，使用了 yield 的函数被称为生成器（generator）。

跟普通函数不同的是，生成器是一个返回迭代器的函数，只能用于迭代操作，更简单点理解生成器就是一个迭代器。

在调用生成器运行的过程中，每次遇到 yield 时函数会暂停并保存当前所有的运行信息，返回 yield 的值, 并在下一次执行 next() 方法时从当前位置继续运行。

#### 下划线

· 单引号下划线`_var`： 下划线前缀是向其他程序员的*提示*，即以单个下划线开头的变量或方法供内部使用。前导下划线确实会影响名称从模块导入的方式。

· 单尾划线 `var_`：惯例使用单个尾划线(后缀)来避免与Python关键字的命名冲突。

· 双首下划线 `__var`： 双下划线前缀导致Python解释器重写属性名，以避免子类中的命名冲突。

双下划线开头的命名形式在 Python 的类成员中使用表示名字改编 (Name Mangling)，即如果有一 Test 类里有一成员 __x，那么 dir(Test) 时会看到 _Test__x 而非 __x。这是为了避免该成员的名称与子类中的名称冲突。但要注意这要求该名称末尾没有下划线。

双下划线开头的用法建议为类的私有成员。

· 领先和落后双下划线`__var__`： 双下划线开头双下划线结尾的是一些 Python 的==“魔术”对象==，如类成员的 __init__、__del__、__add__、__getitem__ 等，以及全局的 __file__、__name__ 等。 Python 官方推荐永远不要将这样的命名方式应用于自己的变量或函数，而是按照文档说明来使用

· 单下划线`_`：除了用作临时变量之外，“_”在大多数Python REPLs中是一个特殊变量，它表示解释器计算的最后一个表达式的结果。如果在解释器会话中工作，并且希望访问前面计算的结果，那么这是很方便的。或者，如果你正在动态构建对象，并且想要与它们交互，而不需要先给它们分配一个名称。

#### 闭包

闭包是支持一等函数的编程语言（Python、JavaScript等）中实现词法绑定的一种技术。当捕捉闭包的时候，它的自由变量（在函数外部定义但在函数内部使用的变量）会在捕捉时被确定，这样即便脱离了捕捉时的上下文，它也能照常运行。简单的说，可以将闭包理解为**能够读取其他函数内部变量的函数**。正在情况下，函数的局部变量在函数调用结束之后就结束了生命周期，但是**闭包使得局部变量的生命周期得到了延展**。使用闭包的时候需要注意，闭包会使得函数中创建的对象不会被垃圾回收，可能会导致很大的内存开销，所以**闭包一定不能滥用**。

> 闭包是词法作用域的体现。

闭包概念：在一个内部函数中，对外部作用域的变量进行引用，(并且一般外部函数的返回值为内部函数)，那么内部函数就被认为是闭包。

一言以蔽之：一个持有外部环境变量的函数就是闭包。

```python
def multiply():
    return [lambda x: i * x for i in range(4)]
##在python中loop没有域概念！
print([m(100) for m in multiply()])
>>> [300, 300, 300, 300]

###使用迭代器或生成器
def multiply():
    return (lambda x: i * x for i in range(4))

###OR
def multiply():
    for i in range(4):
        yield lambda x: x * i

print([m(100) for m in multiply()])
>>> [0, 100, 200, 300]
```

#### 装饰器

Python中，函数的参数分为位置参数、可变参数、关键字参数、命名关键字参数。`*args`代表可变参数，可以接收`0`个或任意多个参数，当不确定调用者会传入多少个位置参数时，就可以使用可变参数，它会将传入的参数打包成一个元组。`**kwargs`代表关键字参数，可以接收用`参数名=参数值`的方式传入的参数，传入的参数的会打包成一个字典。定义函数时如果同时使用`*args`和`**kwargs`，那么函数可以接收任意参数。

```python
from functools import wraps

def decorator(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        """doc of wrapper"""
        print('123')
        return func(*args, **kwargs)

    return wrapper
```

##### 内置装饰器

有三种我们经常会用到的装饰器， `property`、 `staticmethod`、 `classmethod`，他们有个共同点，都是作用于类方法之上。

`property` 装饰器用于类中的函数，使得我们可以像访问属性一样来获取一个函数的返回值。

`staticmethod` 装饰器同样是用于类中的方法，这表示这个方法将会是一个静态方法，意味着该方法可以直接被调用无需实例化，但同样意味着它没有 `self` 参数，也无法访问实例化后的对象。

`classmethod` 依旧是用于类中的方法，这表示这个方法将会是一个类方法，意味着该方法可以直接被调用无需实例化，但同样意味着它没有 `self` 参数，也无法访问实例化后的对象。相对于 `staticmethod` 的区别在于它会接收一个指向类本身的 `cls` 参数。

#### 变量作用域

Python中有四种作用域，分别是局部作用域（**L**ocal）、嵌套作用域（**E**mbedded）、全局作用域（**G**lobal）、内置作用域（**B**uilt-in），搜索一个标识符时，会按照**LEGB**的顺序进行搜索，如果所有的作用域中都没有找到这个标识符，就会引发`NameError`异常。









