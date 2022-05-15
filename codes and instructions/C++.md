#### 范围 for

基于范围的FOR循环的遍历是==只读==的遍历，除非将变量变量的类型声明为引用类型。

 在遍历容器的时候，auto自动推导的类型是容器的value_type类型，而不是迭代器，而map中的value_type是std::pair，也就是说val的类型是std::pair类型的，因此需要使用val.first,val.second来访问数据。

> 当以for(auto it:multiTest) 和for(auto& : multiTest)操作STL容器时，it访问STL元素时应该这样: it.first;   
>
> 当使用for(auto it = multiTest.begin(); it != multiTest.end(); it++)这种方式操作时，it其实就是iterator，此时修改multiTest是有效的
>
> 当直接使用for(multimap<int,int>::itereator it = multiTest.begin(); it != multiTest.end(); it++) 时，it是iterator,修改multiTest是有效的。
>
> 当以iterator方式访问STL容器时，应当使用->符号访问元素，例如it->first



#### deque/queue(container/adaptor)

> When to initialize to the copy of a container and when to use an underlying container?

Both of these...

```cpp
std::queue<int> second(mydeck);
std::queue<int,std::list<int> > fourth(mylist);
```

...construct and initialise a queue by copying elements from the container specified as constructor argument (i.e. `mydeck` and `mylist` respectively).

If you mean to ask why the second specifies a second template argument of `std::list<int>`, that's because `std::queue` can store the data in any container that provides the API functions it expects. From [cppreference](http://en.cppreference.com/w/cpp/container/queue), that second template parameter is:

> **Container** - The type of the underlying container to use to store the elements. The container must satisfy the requirements of `SequenceContainer`. Additionally, it must provide the following functions with the usual semantics:

```cpp
    back()
    front()
    push_back()
    pop_front() 
```

> The standard containers std::deque and std::list satisfy these requirements.

Of these, I'd guess that `std::list` would typically be more efficient for one or a very few elements (exactly where the cutoff is depends on object size, memory library performance characteristics, CPU cache sizes, system load etc. - it gets complicated), then `std::deque` will have better average performance and memory usage for larger numbers of elements (with fewer but larger dynamic memory allocations/deallocations). But even an educated guess can go badly wrong for some specific use cases - if you care enough to consider tuning this, you should measure performance with each candidate container, and your actual data and usage, to inform your decision. Having the container be a template parameter allows the programmer to choose what's best for their needs.

The parameter also has a default...

```cpp
template <class T, class Container = std::deque<T>> class queue;
```

...so you only need to explicit specify a container if you're not happy to have it use a `std::deque`.



#### priority_queue

```c++
template<
    class T,
    class Container = std::vector<T>,
    class Compare = std::less<typename Container::value_type>
> class priority_queue;
```

默认为大顶堆

> 因此，在sort函数中，如果comp(a,b)为true，那么sort函数就会将a放在b的前面，否则a就放在b的后面。==sort==默认是升序

>可以看到priority_queue的比较函数的意义与sort函数的比较函数相类似，均是当比较函数返回true时，将第一个参数放在第二个参数前面。段末提到，默认比较函数为less<T>,和小于操作符的作用一样，即让较小的数放在前面。==priority_queue==默认是大顶堆（top是指最后一个，和sort类似)

**==重点==：priority_queue和sort的cmp参数不同，priority_queue第三个参数是类对象（object of this type container)，sort是函数（function）。因此在使用放射函数时传参格式不同**

sort三种cmp：仿函数，重载（less<type>)，函数；

priority_queue: 仿函数，重载；（使用函数指针/ 使用lambda表达式）

> There are two options for you.

1. Make a class, that can be constructed which has this function as a call operator.

   ```cpp
   struct comparator {
       bool operator()(int a, int b) { return comp(a, b); }
   };
   ```

   Then use like you written, pass only the typename:

   ```cpp
   priority_queue< int,vector<int>,comparator > pq;
   ```

2. Pass a reference to your object (which may be a function pointer). So no need for constructing your object again:

   ```cpp
   priority_queue< int,vector<int>,decltype(p) > pq(p)
   ```

   Where p is the function pointer you made before.

   ```cpp
   bool (*p)(int,int);
   p=comp;
   priority_queue< int,vector<int>,decltype(p) > pq(p);
   ```

#### c++ 什么是容器

在数据存储上，有一种对象类型，它可以持有其它对象或指向其它对像的指针，这种对象类型就叫做容器。

#### 仿函数

定义：仿函数(functor)，就是使一个类的使用看上去像一个函数。其实现就是类中实现一个operator这个类就有了类似函数的行为，就是一个仿函数类了。

有时会发现有些功能实现的代码，会不断的在不同的成员函数中用到，但是又不好将这些代码独立出来成为一个类的一个成员函数。但是又很想复用这些代码。

写一个公共的函数，可以，这是一个解决方法，不过函数用到的一些变量，就可能成为公共的全局变量，再说为了复用这么一片代码，就要单立出一个函数，也不是很好维护。

> **这时就可以用仿函数了，写一个简单类，除了那些维护一个[类的成员函数]外，就只是实现一个operator()，==在类实例化时，就将要用的，非参数的元素传入类中。==这样就免去了对一些公共变量的全局化的维护了。又可以使那些代码独立出来，以便下次复用。**

而且这些仿函数，还可以用关联，聚合，依赖的类之间的关系，与用到他们的类组合在一起，这样有利于资源的管理（这点可能是它相对于函数最显著的优点了）。如果在配合上模板技术和policy编程思想，那就更是威力无穷了

#### lambda

```cpp
[capture list] (param list) ->return type {func body}
```

**************

函数对象（function object）又叫仿函数（functor），就是重载了==调用运算符（）==的类，所生成的对象。

**函数指针**：函数指针也是一个函数对象，因为指针在C++中都是对象。

*lambda*也是一个函数对象。

********************

#### 类初始化

> Ÿ 一般变量可以在初始化列表里或者构造函数里初始化，不能直接初始化或者类外初始化
>
> Ÿ 静态成员变量必须在类外初始化
>
> Ÿ 常量必须在初始化列表里初始化
>
> Ÿ 静态常量必须只能在定义的时候初始化(定义时直接初始化)

初始化列表初始化的变量值会覆盖掉声明时初始化的值，而构造函数中初始化的值又会覆盖掉初始化列表的，足以看出初始化的顺序。



#### 输入输出

==getline：==c++输入字符串到string类可以用`getline`函数，第一个参数是`cin`，第二个参数是`string类的变量`，第三个参数是**结束标志**（默认为**回车**）。

该函数**不会读入结束标志**，而是跳过。

`getline`和`cin`的区别，`cin`遇到空白键，tab键，换行符终止，可是这个符号还留在缓冲区中。
而`getline`遇到终止符号停止输入，该符号也不会继续留在缓冲区。

