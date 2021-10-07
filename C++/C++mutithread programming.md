# C++mutithread programming

std::lock 可以一次性锁住多个(两个以上)的互斥量，并且没有副作用(思索风险)

*std::mutex main_mutex;*使用更加灵活的锁，这样std::unique_lock实例不会总与互斥量的数据类型相关，使用起来要比std::lock_guard更加灵活。

std::lock() 和 std::unique_lock()的使用

对于std::adopt_lock作为第二参数传入构造函数，对互斥量进行管理；也可以将std::defer_lock作为第二参数传递进去，表明互斥量应保持解锁状态。这样就可以被std::unique_lock对象（不是互斥量）的lock()函数的所获取，或传递std::unique_lock对象到std::lock()中。使用std::unique_lock和std::defer_lock,而非std::lock_guard以及std::adopt_lock。



**std::unique_lock**在这种情况下工作正常，在调用unlock()时，代码不需要在访问共享数据；而后当再次需要对共享数据进行访问时，就可以调用lock()。







## 保护共享数据的替代措施

互斥量是最通用的机制，但其并非保护共享数据的唯一方式。

```c++
void undefined_behaviour_with_double_checked_locking()
{
  if(!resource_ptr)  // 1
  {
    std::lock_guard<std::mutex> lk(resource_mutex);
    if(!resource_ptr)  // 2
    {
      resource_ptr.reset(new some_resource);  // 3
    }
  }
  resource_ptr->do_something();  // 4
}
```



双重检查锁问题

​	这里有一个潜在的条件竞争，未被锁保护的读取操作1 没有与其它线程里被锁保护的写入操作3进行同步。因此就会产生条件竞争，这个竞争不仅覆盖指针本身，还会影响到其指向的对象；即使一个线程知道另一个线程完成指针写入，他可能没有看到新创建的some_resource实例，然后调用do_something()，得到不正确的结果。这个例子就是典型的条件竞争-数据竞争。



C++委员会认为竞争条件的处理很重要，所以只是提供了std::once_flag 和 std::call_once来处理这种情况。比起锁住互斥量，并显式的检查指针，每个线程只需要使用**std::call_once**，在std::call_once的结束时，就能知道指针已经被其它线程初始化。



```c++
std::shared_ptr<some_resource> resource_ptr;
std::once_flag resource_flag;  // 1
void init_resource()
{
  resource_ptr.reset(new some_resource);
}
void foo()
{
  std::call_once(resource_flag,init_resource);  // 可以完整的进行一次初始化
  resource_ptr->do_something();
}
```





## 嵌套锁

当一个线程已经获取一个std::mutex时（已经上锁），并对其再次上锁，这个操作就是错误的。

C++为此提供了std::recursive_mutex类。其功能与std::mutex类似。去功能与std::mutex类似，除了你可以从一个线程获得多次该锁之外，互斥量锁住其它锁之前，你必须释放你拥有的锁，所以当你调用lock()三次时，你也必须调用unlock()三次。正确使用std::lock_guard<std::recursive_mutex>和std::unique_lock<std::recursive_mutex>可以帮助你处理这些问题。



## 条件变量

通过使用条件变量可以唤醒和触发其它正在等待中的线程。

C++一共提供两套实现方式：**std::condition_varaible** 和 **std::condition_varaible_any**

两者都需要与互斥量一起才能工作（互斥量是为了同步）;前者仅限于与std::mutex 一起工作，而后者可以和任何满足最低标准的互斥量一起工作，从而加上_any后缀。

由于std::condition_varaible_any 更加通用，这通常会产生从体积、性能、以及系统资源的使用方面的开销。所以通常情况下，std::condition_varaible是首选类型。



> void condition_variable::wait (unique_lock<mutex>& lock, Predicate pred);
>
> 在功能上等同于
>
> while(!pred()) wait(lock);



使用std::condition_varaible处理数据等待

```c++
std::mutex mut;
std::queue<data_chunk> data_queue;  // 1
std::condition_variable data_cond;
void data_preparation_thread()
{
  while(more_data_to_prepare())
  {
    data_chunk const data=prepare_data();
    std::lock_guard<std::mutex> lk(mut);
    data_queue.push(data);  // 2
    data_cond.notify_one();  // 3
  }
}
void data_processing_thread()
{
  while(true)
  {
    std::unique_lock<std::mutex> lk(mut);  // 4
    data_cond.wait(
         lk,[]{return !data_queue.empty();});  // 5
    data_chunk data=data_queue.front();
    data_queue.pop();
    lk.unlock();  // 6
    process(data);
    if(is_last_chunk(data))
      break;
  }
}
```

这里在4使用unique_lock 要比 lock_guard合适

原因是因为: wait 函数会去检查这些条件，当条件满足时返回。如果条件不满足时，wait()函数将解锁互斥量，并且将这个线程置于阻塞或等待状态。当数据线程调用notify_once()通知条件变量时，处理数据的线程从睡眠中苏醒，重新获取互斥锁，并且对条件再次检查，在满足条件的情况下，从wait()返回并继续持有锁。当条件不满足时，线程将对互斥量解锁，并且重新开始等待。

如果用lock_guard，等待线程无法在休眠期间对互斥量解锁，并在此之后对该变量重新上锁。幸运的是，unique_lock完全符合这一条件。





## 4.12 使用条件变量构建线程安全的队列

当你正在设计一个通用队列，花一些时间想想有哪些操作需要添加到队列中实现。

```c++
template<class T, class Container = std::deque<T>>
class deque
{
    public:
    explicit deque(const Container&);
    explicit deque(Container&& = Container());
    template <class Alloc> explicit deque(const Alloc&);
    
    void swap(deque& q);
    
    bool empty()const;
    size_type size() const;
    
    T& front();
    const T& front() const;
    T& back();
    const T& back() const;
    
    void push(const T& x);
    void push(T&& X);
    void pop();
    template<class... Arg> void emplace(Args&&... args);
}
```

在线程安全队列中，需要对其变量进行防护的也只有push() 和 pop()操作。

```c++
#include<queue>
#include<mutex>
#include<condition_mutex>

template<typename T>
class threadsafe_queue
{
 private:
    std::mutex mt;
    std::queue<T> data_queue;
    std::conditon_varaible data_cond;
    
 public:
    void push(T new_value)
    {
        std::lock_guard<std::mutex> lk(mut);
        data_queue.push(new_value);
        data_cond.notify_one();
    }
    
    void wait_and_pop(T& value)
    {
        std::unique_lock<std::mutex> lk(mt);
        data_cond.wait(lk, [this]{return !data_queue.empty();});
        value = data_queue.front();
        data_queue.pop();
    }
}
```





## 4.3 使用期望等待一次性事件

当一个线程需要等待一次性事件时，从某种程度上来说，它就必须要知道这个事件在未来的表现形式。之后，这个线程周期性的等待或者检查，事件是否触发；在检查期间也会执行其它任务。另外，在等待期间它可以执行另外一种任务，知道对应的任务触发，而后等待期望状态就会变为就绪（ready）。类似于 linux系统中 信号signal 的底层实现。



C++模板中有两种形式可以实现"期望"。即唯一期望(unique futures)(std::futures<>) 和共享期望(shared futures)(std::shared_futures<>)。std::futures的实例只能与一个指定事件相关联，而std::shared_future<>的实例就能关联多个事件。后者的所有实现中，所有实例会同时变为就绪状态。



## 4.2.1 带返回值的后台任务

你可以启动一个新线程来执行这个计算，但是这就意味着你必须关注回传的结果。因为std::thread 并不直接接受返回值机制，这里需要一个std::async函数模板。

当任务的结果你不急着要时，你可以使用std::async启动一个异步任务。与std::thread 对象等待方式不同，std::async会返回一个std::future对象，这个对象持有最终返回出来的结果。当你需要这个值时，你只需要调用这个对象的get()成员函数; 并且会阻塞线程直到”期望“状态到就绪为止。

使用 std::future 从异步任务中获取返回值

```c++
#include<future>
#include<iostream>

int find_the_answer_to_ltuae();
void do_other_stuff();

int main()
{
	std::future<int> the_answer = std::async(find_the_answer_to_ltuae);
	do_other_stuff();
	std::cout<<"The answer is "<<the_answer.get()<<std::endl;
	return 0;
}
```

与std::thread 做的方式一样，std::async允许你通过添加额外的调用参数，向函数传递额外的参数。当第一个参数是指向成员函数的指针时，第二个参数提供有这个函数成员类的具体对象（不是直接的，就是通过指针，还可以包装在std::ref中），剩余参数可以作为成员函数参数传入。

使用std::async 像函数传递参数





## 5.1内存模型基础

从两方面讲内存模型：一方面是基本结构，这与事务在内存中是怎样的布局有关；另一方面是并发。对于并发，基本结构很重要，特别是在低层原子操作。





四个牢记的原则：

1. 每一个变量都是一个对象， 包括作为其成员变量的对象
2. 每个对象至少占有一个内存位置
3. 基本类型都有确定的内存位置（无论类型大小如何，即使它们都是相邻的，或是数组的一部分）
4. 相邻位域是相同内存中的一部分

### 5.1.2 对象、内存位置和并发

> ​	所有东西都在内存中，当两个线程访问不同的内存位置时，不会存在任何问题，一切工作顺利；而另一种情况，当两个线程访问同一个内存位置时，如果没有线程更新内存位置上的数据，一切正常；只读数据不需要保护，当有线程对内存位置上的数据进行修改时，此时有可能产生竞态条件。

为了避免竞态条件，两个线程就需要一定的执行顺序。

第一种方式，使用互斥量来确定访问顺序；当同一互斥量在两个线程同时访问前被锁住，那么同一时间内就只有一个线程能够访问到对应内存位置。

另一种方式，使用原子操作同步机制，决定两个线程的访问顺序。当多于两个线程访问同一个内存地址时，对每个访问都需要定义一个顺序。

如果不去规定两个不同线程对于同一个内存位置访问顺序，那么访问就不是原子的；并且，当两个线程都是"作者"，就会产生数据竞争或者未定义行为。

**数据竞争**：数据竞争绝对是一个严重的错误，并且需要不惜一切代价避免它。

另一个重点是：当程序对同一个内存地址中的数据访问存在竞争，你可以使用原子操作来比避免未定义行为。这并会影响竞争的产生，因为原子操作并没有指定访问顺序，但原子操作把程序拉回了定义行为的区域内。



## 5.2 C++中的原子操作和原子类型

*原子操作*是个不可分割的操作：在系统的所有线程中，你是不可能观察到原子操作完成了一半这种情况的；他要么做了，要么就是没做，只有这两种可能。如果从对象读取值的加载操作是原子的，而且对这个对象所有修改操作也是*原子*的，那么加载操作得到的值要么是对象的初始值，要么是某次修改操作存入的值。

另一方面，非原子操作可能会被另一个线程观察到只完成了一半。如果这个操作是一个存储操作，那么其它线程看到的值，可能既不是存储前的值，也不是存储的值，而是别的什么值。如果这个非原子操作是一个加载操作，他可能先去到对象的一部分，然后值被另一个线程修改，然后它再取剩余部分。所以，它取到的值是两个值的某个组合。



### 5.2.1 标准原子类型

所有相关的原子操作都是定义在<atomic>中

这些类型上的所有操作都是原子操作，不过可以用互斥锁来*模拟*原子操作。





## 5.3同步操作和强制排序

假设你有两个线程，一个向数据结构中填充数据，另一个读取数据结构中的数据。为了避免恶性条件竞争，第一个线程设置一个标志，用来表明数据已经准备就绪，并且第二个线程在这个标志设置前不能读取数据。

不同线程对数据读写

```c++
#include<vector>
#include<atomic>
#include<iostream>

std::vector<int> data;
std::atmoic<bool> data_ready(false);

void reader_thread()
{
    while(!data_ready.load())
    {
        std::this_thread::sleep(std::milliseconds(1));
    }
    cout<<"The answer = " << data[0]<<endl;
   
}

void writer_thread()
{
    data.push_back(42);
    data.ready = true;
}
```



### 5.3.1同步发生

“同步发生”只能在原子类型之间进行操作。例如对一个数据结构进行操作（对互斥量上锁），如果数据结构包含有原子类型，并且操作内部执行了一定的原子操作，那么这些操作就是同步发生关系。

### 5.3.2 先行发生

“先行发生”关系是指，基本构建块的操作顺序；它指定了某个操作去影响另一个操作。对于单线程而言，意味着前者先执行。

但是，如果两者同时发生，因为操作的无序性，他们没有明确的先行关系

如下代码：

```c++
#include<iostream>

void foo(int a, int b)
{
    std::cout<<a<<","<<b<<std::endl;
}

int get_num()
{
    static int i = 0;
    return ++i;
}

int main()
{
    foo(get_num(), get_num());  //无序调用get_num();
}
```





## 6 基于锁的并发数据结构设计

主要考虑两个方面：一是确保访问是安全的，而是能够真正并发访问，

* 确保无线程能够看到，数据结构的“不变量”破坏时的状态
* 小心那些会引起条件竞争的接口，提供完整操作的函数，而非操作步骤
* 注意数据结构的行为是否会产生异常，从而确保“不变量”的状态稳定
* 将死锁的概率降到最低。使用数据结构时，需要限制锁的范围，且避免嵌套锁的存在



### 6.2 基于锁的并发数据结构

基于锁的并发数据结构设计，需要确保访问线程持有锁的时间最短。



线程安全栈——————使用锁

线程安全栈类定义

```c++
#include<exception>

struct empty_stack: exception
{
    const char* what() const throw();
}

template<typename T>
class threadsafe_stack
{
 private:
    std::stack<T> data;
    mutable std::mutex m;
    
 public:
    threadsafe_stack(){}
    threadsafe_stack(const threadsafe_stack& other)
    {
        lock_guard<mutex> lock(other.m);
        data = other.data;
    }
    
    threadsafe_stack& operator = (const threadsafe_stack&) = delete;
    
    void push(T new_value)
    {
        lock_guard<mutex> lock(m);
        data.push(move(new_value));
    }
    
    shared_ptr<T> pop()
    {
        lock_guard<mutex> lock(m);
        if(data.empty()) throw empty_stack();
        shared_ptr<T> const res(
        	std::make_shared<T>(move(data.top())));
        data.pop();
        return res;
    }
    
    void pop(T& value)
    {
        lock_guard<mutex> lock(m);
        if(data.empty()) throw empty_stack();
        value = move(data.pop());
        data.pop();
    }
    
    bool empty() const
    {
        lock_guard<mutex> lock(m);
        return data.empty();
    }
}
```



此段代码中，互斥量M能保证基本的线程安全，那就是对每个成员函数进行加锁保护。这就保证在同一时间段内， 只有一个线程可以访问数据，所以能够保证，数据结构的“不变量”被破坏时，不会被其他线程看到。

其次，在empty()和pop()成员函数之间会存在潜在竞争，不过代码会在pop()函数上锁时，显式的查询栈是否为空，所以这里的竞争是非恶行的。

当调用持有一个锁的用户代码时，这里有两个地方可能会产生死锁。

缺点：虽然该数据结构在多线程情况下，并发的调用成员函数是安全的（因为使用锁），也要保证在单线程情况下，数据结构做出正确反映。序列化线程会隐性限制程序性能，**当一个线程在等待锁时，他就会无所事事。**对于栈来说，等待添加元素也是没有意义的，所以当一个线程需要等待时，其会定期检查empty()和pop()，以及对empty_stack()异常的关注。

这样会限制栈的实现方式，在线程等待时，会浪费宝贵的资源去检查数据，或者要求用户写外部等待和提示代码，这就使锁失去内部意义————意味这资源的浪费。



为了解决上述问题，我们使用了条件变量加上锁的方式实现线程安全队列。

使用条件变量实现线程安全队列

```c++
template<typename T>
class threadsafe_queue
{
 private:
    mutable mutex mut;
    queue<T> data_queue;
    condition_variable data_cond;
 
 public:
    threadsafe_queue(){}
    void push(T new_value)
    {
        lock_guard<mutex> lk(mut);
        data_queue.push(move(new_value));
        data_cond.notify_one(); //1
    }
    
    void wait_and_pop(T& value)
    {
        unique_lock<mutex> lk(mut);
        data_cond.wait(lk, [this]{return !data_queue.empty();});
        value = move(data_queue.front());
        data_queue.pop();
    }
    
    shared_ptr<T> wait_and_pop()
    {
        unique_lock<mutex> lk(mut);
        data_cond.wait(lk, [this]{return !data_queue.empty();});
        shared_ptr<T> res(
        	make_shared<T>(move(data_queue.front()))); //症结点
        data_queue.pop();
        return res;
    }
    
   bool try_pop(T& value)
   {
       lock_guard<mutex> lk(mut);
       if(data_queue.empty())
           return false;
       value = move(data_queue.front());
       data_queue.pop();
       return true;
   }
    
    shared_ptr<T> try_pop()
    {
        lock_guard<mutex> lk(mut);
        if(data_queue.empty())
            return shared_ptr<T>();
        shared_ptr<T> res(
        	make_shared<T>(move(data_queue.front())));
        data_queue.pop();
        return res;
    }
    
    bool empty() const
    {
        lock_guard<mutex> lk(mut);
        return data_queue.empty();
    }
};
```

wait_and_pop()函数是等待队列向栈进行输入的一个解决方案：比起持续调用empty()，等待线程调用wait_and_pop()函数和数据结构处理等待中的条件变量的方式要好很多。对于data_cond.wai()的调用，直到队列中有一个元素的时候，才会返回，所以你就不用担心会出现一个空队列的情况，还有数据会一直被互斥锁保护。

问题：

当不止有一个线程等待队列进行推送操作时，只会有一个线程，因得到data_cond.notify_one()， 而继续工作。但是，如果这个线程在wait_and_pop()中抛出一个异常，例如：构造新的*shared_ptr<>*对象时抛出异常，那么其他线程就会长眠。出现这种情况是不可接受的，有两种方法可以解决这类问题：

1. 这里的调用就需要该成data_cond.notify_all()，这个函数讲唤醒所有的工作线程，不过，当大多数队列发现队列依旧为空，又会耗费很多资源让线程重新进入休眠。
2. 当有异常抛出时，让wait_and_pop()函数调用notify_one()，从而让另一个线程可以去尝试索引存储的值。
3. 第三种方案，将shared_ptr<>初始化过程移动到push()中，并且存储shared_ptr<>实例，而非直接使用数据的值。将shared_ptr<>拷贝到内部queue<>中，就不会抛出异常了，这样wait_and_pop()又是安全的了。



持有std::shared_ptr<> 实例的线程安全队列

```c++
template<typename T>
class threadsafe_queue
{
 private:
    mutable mutex mut;
    queue<shared_ptr<T>> data_queue;
    condition_variable data_cond;
 
 public:
    threadsafe_queue(){}
    
    void wait_and_pop(T& value)
    {
        unique_lock<mutex> lk(mut);
        data_cond.wait(lk, [this]{return !data_queue.empty();}); //1
        value = move(*data_queue.front());
        data_queue.pop();
    }
    
    bool try_pop(T& value)
    {
        lock_guard<mutex> lk(mut);
        if(data_queu.empty())	return false;
        value = move(*data_queue.front()); //2
        data_queue.pop();
        return true;
    }
    
    shared_ptr<T> wait_and_pop()
    {
        unique_lock<mutex> lk(mut);
        data_cond.wait(lk, [this]{return !data_queue.empty();}); //3
        shared_ptr<T> res = data_queue.front();
        data_queue.pop();
        return res;
    }
    
    shared_ptr<T> try_pop()
    {
        lock_guard<mutex> lk(mut);
        if(data_queue.empty()) reutrn shared_ptr<>(); //4
        shared_ptr<T> res = data_queue.front();
        data_queue.pop();
        return res;
    }
    
    void push(T new_value)
    {
        shared<T> data(
        	make_shared<T>(move(new_value))); //5
        lock_guard<mutex> lk(mut);
        data_queue.push(data);
        data_cond.notify_one();
    }
    
    bool empty() const
    {
        lock_guard<mutex> lk(mut);
        return data_queue.empty();
    }
};
```

为了让*shared_ptr<>*持有数据的结果是显而易见的：弹出函数会持有一个变量的引用，为了接受这个新值，必须对存储的指针进行解引用；并且，在返回到函数前，弹出函数都会返回一个shared_ptr<>实例，这里实例可以在队列中检索

**shared_ptr<>**持有数据有一个好处：新的实例分配结束时，不会锁在push()当中。因为内存分配操作的需要在性能上付出很高的代价，所以使用*shared_ptr<>()*的方式对队列的性能有很大的提升，其减少了互斥量持有的时间，允许其他线程在分配内存的同时，对队列进行其他操作。

## 6.2.3 线程安全队列——使用细粒度锁和条件变量

使用互斥量对一个数据队列(data_queue)进行保护。为了使用细粒度锁，需要查看队列内部的组成结构，并且将一个互斥量与每个数据相关联。

对于队列来好，最简单的数据结构就是单链表。

```c++
template <typename T>
class queue
{
 private:
    struct node
    {
        T data;
        std::unique_ptr<node> next;
        
        node(T data_);
        data(move(data_)){}
    };
    
    unique_ptr<node> head;	//1
    node* tail;		//2
 
 public:
    queue() {}
    queue(const queue& other) = delete;
    queue& operator= (cosnt queue& other) = delete;
    shared_ptr<T> try_pop()
    {
        if(!head)
        {
            return shared_ptr<T>();
        }
        shared_ptr<T> const res(
        	make_shared<T>(move(head->data)));
        unique_ptr<node> const old_head = move(head);
        head = move(old_head->next);
        return res;
    }
    
    void push(T new_value)
    {
        unique_ptr<node> p (new node(move(new_value)));
        node* const new_tail = p.get();
        if(tail)	tail->next = move(p);
        else	head = move(p);
        tail = new_tail;
    }
}
```

使用 **unique_ptr<node>**来管理节点，因为其能够保证节点在删除时候，不需要使用delete操作显式删除。

在多线程下，因为有两个数据项 head 以及 tail； 即使，使用两个数据项，来保护头指针和尾指针，也容易出现问题。



### 6.2.3 使用分离数据实现并发

使用一个预分配的虚拟节点分离数据，head和tail

```c++
template<typename T>
class queue
{
private:
  struct node
  {
    std::shared_ptr<T> data;  // 1
    std::unique_ptr<node> next;
  };

  std::unique_ptr<node> head;
  node* tail;

public:
  queue():
    head(new node),tail(head.get())  // 2
  {}
  queue(const queue& other)=delete;
  queue& operator=(const queue& other)=delete;

  std::shared_ptr<T> try_pop()
  {
    if(head.get()==tail)  // 3
    {
      return std::shared_ptr<T>();
    }
    std::shared_ptr<T> const res(head->data);  // 4
    std::unique_ptr<node> old_head=std::move(head);
    head=std::move(old_head->next);  // 5
    return res;  // 6
  }

  void push(T new_value)
  {
    std::shared_ptr<T> new_data(
      std::make_shared<T>(std::move(new_value)));  // 7
    std::unique_ptr<node> p(new node);  //8
    tail->data=new_data;  // 9
    node* const new_tail=p.get();
    tail->next=std::move(p);
    tail=new_tail;
  }
};
```





## 9.1.1 线程池

作为最简单的线程池，其拥有固定数据的工作线程（通常工作线程数量与*std::thread::hardware_concurrency()*相同。当工作需要完成时，可以调用函数将任务挂在任务队列中。每个工作线程都会从任务队列上获取任务，然后执行这个任务，执行完成之后再回来获取新的任务。在最简单的线程池中，线程就不需要等待其他线程完成对应任务了。如果需要等待，就对同步进行管理

最简单的线程池实现。

```c++
class thread_pool
{
	std::atomic_bool done;
    thread_safe_queue<std::function<void()>> work_queue;	//1
    std::vector<std::thread> threads; //2
    
    join_threads joiner; //3
    
    void worker_thread()
    {
        while(!done)	//4
        {
            std::function<void()> task;
            if(work_queue.try_pop(task))	//5
            {
                task(); //6
            }
            else
            {
                std::this_thread::yield(); //7
            }
        }
    }
 
public:
    thread_pool():
    	done(false), joiner(threads)
        {
            usigned const thread_count = std::thread::hardware_concurrency();	//8
            
            try
            {
                for(unsigned i = 0; i < thread_count; ++i)
                {
                    threads.push_back(
                    	thread(&thread_pool::worker_thread, this)); //9
                }
            }
            catch(...)
            {
                done = true;
                throw;
            }
        }
    ~thread_pool()
    {
        done = true;	//11
    }
    
    template<typename FunctionType>
    void submit(FunctionType f)
    {
        work_queue.push(std::function<void>(f));	//12
    }
}
```

实际上有一组工作线程2，并且使用了一个线程安全队列来管理任务队列。这种情况下，所以可以使用*std::function<void()>*对任务进行封装。submit()函数会将函数或可调用对象包装成一个*std::function<void()>*实例，并将其推入队列中。