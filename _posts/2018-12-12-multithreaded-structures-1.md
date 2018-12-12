---
layout: post
title: "用于并行计算的多线程数据结构 - 设计并发数据结构"
date: 2018-12-11 19:00:00
---

## 简介

现在，您的计算机有四个 CPU 核心；*并行计算*是最时髦的主题，您急于掌握这种技术。但是，并行编程不只是在随便什么函数和方法中使用互斥锁和条件变量。C++ 开发人员必须掌握的关键技能之一是设计并发数据结构。本文是两篇系列文章的第一篇，讨论如何在多线程环境中设计并发数据结构。对于本文，我们使用 POSIX Threads 库（也称为 Pthreads），但是也可以使用 Boost Threads 等实现。

本文假设您基本了解基本数据结构，比较熟悉 POSIX Threads 库。您还应该基本了解线程的创建、互斥锁和条件变量。在本文的示例中，会相当频繁地使用 pthread_mutex_lock、pthread_mutex_unlock、pthread_cond_wait、pthread_cond_signal 和 pthread_cond_broadcast。

## 设计并发队列

我们首先扩展最基本的数据结构之一：队列。我们的队列基于链表。底层列表的接口基于 Standard Template Library（STL）。多个控制线程可以同时在队列中添加数据或删除数据，所以需要用互斥锁对象管理同步。队列类的构造函数和析构函数负责创建和销毁互斥锁，见清单 1。

清单 1. 基于链表和互斥锁的并发队列

```cpp
#include <pthread.h>
#include <list.h> // you could use std::list or your implementation

template <typename T>
class Queue {
public:
   Queue( ) {
       pthread_mutex_init(&_lock, NULL);
    }
    ~Queue( ) {
       pthread_mutex_destroy(&_lock);
    }
    void push(const T& data);
    T pop( );
private: 
    list<T> _list; 
    pthread_mutex_t _lock;
}
```

### 在并发队列中插入和删除数据

显然，把数据放到队列中就像是把数据添加到列表中，必须使用互斥锁保护这个操作。但是，如果多个线程都试图把数据添加到队列中，会发生什么？第一个线程锁住互斥并把数据添加到队列中，而其他线程等待轮到它们操作。第一个线程解锁/释放互斥锁之后，操作系统决定接下来让哪个线程在队列中添加数据。通常，在没有实时优先级线程的 Linux 系统中，接下来唤醒等待时间最长的线程，它获得锁并把数据添加到队列中。清单 2 给出代码的第一个有效版本。

清单 2. 在队列中添加数据

```cpp
void Queue<T>::push(const T& value ) {
    pthread_mutex_lock(&_lock);
    _list.push_back(value);
    pthread_mutex_unlock(&_lock);
}
```

取出数据的代码与此类似，见清单 3。

清单 3. 从队列中取出数据

```cpp
T Queue<T>::pop( ) {
    if (_list.empty( )) {
        throw "element not found";
    }
    pthread_mutex_lock(&_lock); 
    T _temp = _list.front( );
    _list.pop_front( );
    pthread_mutex_unlock(&_lock);
    return _temp;
}
```

清单 2 和 清单 3 中的代码是有效的。但是，请考虑这样的情况：您有一个很长的队列（可能包含超过 100,000 个元素），而且在代码执行期间的某个时候，从队列中读取数据的线程远远多于添加数据的线程。因为添加和取出数据操作使用相同的互斥锁，所以读取数据的速度会影响写数据的线程访问锁。那么，使用两个锁怎么样？一个锁用于读取操作，另一个用于写操作。清单 4 给出修改后的 Queue 类。

清单 4. 对于读和写操作使用单独的互斥锁的并发队列

```cpp
template <typename T>
class Queue {
public:
   Queue( ) {
       pthread_mutex_init(&_rlock, NULL); 
       pthread_mutex_init(&_wlock, NULL);
    }
    ~Queue( ) {
       pthread_mutex_destroy(&_rlock);
       pthread_mutex_destroy(&_wlock);
    }
    void push(const T& data);
    T pop( );
private:
    list<T> _list;
    pthread_mutex_t _rlock, _wlock;
}
```

清单 5 给出 push/pop 方法的定义。

清单 5. 使用单独互斥锁的并发队列 Push/Pop 操作

```cpp
void Queue<T>::push(const T& value ) {
    pthread_mutex_lock(&_wlock);
    _list.push_back(value);
    pthread_mutex_unlock(&_wlock);
}
 
T Queue<T>::pop( ) {
    if (_list.empty( )) {
        throw ”element not found”;
    }
    pthread_mutex_lock(&_rlock);
    T _temp = _list.front( );
    _list.pop_front( );
    pthread_mutex_unlock(&_rlock);
    return _temp;
}
```

## 设计并发阻塞队列

目前，如果读线程试图从没有数据的队列读取数据，仅仅会抛出异常并继续执行。但是，这种做法不总是我们想要的，读线程很可能希望等待（即阻塞自身），直到有数据可用时为止。这种队列称为阻塞的队列。如何让读线程在发现队列是空的之后等待？一种做法是定期轮询队列。但是，因为这种做法不保证队列中有数据可用，它可能会导致浪费大量 CPU 周期。推荐的方法是使用条件变量 — 即 pthread_cond_t 类型的变量。在深入讨论语义之前，先来看一下修改后的队列定义，见 清单 6。

清单 6. 使用条件变量的并发阻塞队列

```cpp
template <typename T>
class BlockingQueue {
public:
    BlockingQueue ( ) {
        pthread_mutex_init(&_lock, NULL);
        pthread_cond_init(&_cond, NULL);
    } 
    ~BlockingQueue ( ) {
       pthread_mutex_destroy(&_lock);
       pthread_cond_destroy(&_cond);
    }
    void push(const T& data);
    T pop( );
private:
    list<T> _list;
    pthread_mutex_t _lock;
    pthread_cond_t _cond;
}
```

清单 7 给出阻塞队列的 pop 操作定义。

清单 7. 从队列中取出数据

```cpp
T BlockingQueue<T>::pop( ) {
    pthread_mutex_lock(&_lock);
    if (_list.empty( )) {
        pthread_cond_wait(&_cond, &_lock);
    }
    T _temp = _list.front( );
    _list.pop_front( );
    pthread_mutex_unlock(&_lock);
    return _temp;
}
```

当队列是空的时候，读线程现在并不抛出异常，而是在条件变量上阻塞自身。pthread_cond_wait 还隐式地释放 mutex_lock。现在，考虑这个场景：有两个读线程和一个空的队列。第一个读线程锁住互斥锁，发现队列是空的，然后在 _cond 上阻塞自身，这会隐式地释放互斥锁。第二个读线程经历同样的过程。因此，最后两个读线程都等待条件变量，互斥锁没有被锁住。

现在，看一下 push() 方法的定义，见 清单 8。

清单 8. 在阻塞队列中添加数据

```cpp
void BlockingQueue<T>::push(const T& value ) {
    pthread_mutex_lock(&_lock);
    const bool was_empty = _list.empty( );
    _list.push_back(value);
    pthread_mutex_unlock(&_lock);
    if (was_empty) 
        pthread_cond_broadcast(&_cond);
}
```

如果列表原来是空的，就调用 pthread_cond_broadcast 以宣告列表中已经添加了数据。这么做会唤醒所有等待条件变量 _cond 的读线程；读线程现在隐式地争夺互斥锁。操作系统调度程序决定哪个线程获得对互斥锁的控制权 — 通常，等待时间最长的读线程先读取数据。

并发阻塞队列设计有两个要注意的方面：

- 可以不使用 pthread_cond_broadcast，而是使用 pthread_cond_signal。但是，pthread_cond_signal 会释放至少一个等待条件变量的线程，这个线程不一定是等待时间最长的读线程。尽管使用 pthread_cond_signal 不会损害阻塞队列的功能，但是这可能会导致某些读线程的等待时间过长。
- 可能会出现虚假的线程唤醒。因此，在唤醒读线程之后，要确认列表非空，然后再继续处理。清单 9 给出稍加修改的 pop() 方法，强烈建议使用基于 while 循环的 pop() 版本。

清单 9. 能够应付虚假唤醒的 pop() 方法

```cpp
T BlockingQueue<T>::pop( ) {
    pthread_cond_wait(&_cond, &_lock);
    while(_list.empty( )) { 
        pthread_cond_wait(&_cond);
    }
    T _temp = _list.front( );
    _list.pop_front( );
    pthread_mutex_unlock(&_lock);
    return _temp;
}
```

## 设计有超时限制的并发阻塞队列

在许多系统中，如果无法在特定的时间段内处理新数据，就根本不处理数据了。例如，新闻频道的自动收报机显示来自金融交易所的实时股票行情，它每 n 秒收到一次新数据。如果在 n 秒内无法处理以前的一些数据，就应该丢弃这些数据并显示最新的信息。根据这个概念，我们来看看如何给并发队列的添加和取出操作增加超时限制。这意味着，如果系统无法在指定的时间限制内执行添加和取出操作，就应该根本不执行操作。清单 10 给出接口。

清单 10. 添加和取出操作有时间限制的并发队列

```cpp
template <typename T>
class TimedBlockingQueue {
public: 
    TimedBlockingQueue ( );
    ~TimedBlockingQueue ( );
    bool push(const T& data, const int seconds);
    T pop(const int seconds);
private: 
    list<T> _list;
    pthread_mutex_t _lock;
    pthread_cond_t _cond;
}
```

首先看看有时间限制的 push() 方法。push() 方法不依赖于任何条件变量，所以没有额外的等待。造成延迟的惟一原因是写线程太多，要等待很长时间才能获得锁。那么，为什么不提高写线程的优先级？原因是，如果所有写线程的优先级都提高了，这并不能解决问题。相反，应该考虑创建少数几个调度优先级高的写线程，把应该确保添加到队列中的数据交给这些线程。清单 11 给出代码。

清单 11. 把数据添加到阻塞队列中，具有超时限制

```cpp
bool TimedBlockingQueue<T>::push(const T& data, const int seconds) {
    struct timespec ts1, ts2;
    const bool was_empty = _list.empty( );
    clock_gettime(CLOCK_REALTIME, &ts1);
    pthread_mutex_lock(&_lock);
    clock_gettime(CLOCK_REALTIME, &ts2);
    if ((ts2.tv_sec – ts1.tv_sec) < seconds) {
        was_empty = _list.empty( );
        _list.push_back(value);
    }
    pthread_mutex_unlock(&_lock);
    if (was_empty) 
        pthread_cond_broadcast(&_cond);
}
```

clock_gettime 例程返回一个 timespec 结构，它是系统纪元以来经过的时间。在获取互斥锁之前和之后各调用这个例程一次，从而根据经过的时间决定是否需要进一步处理。

具有超时限制的取出数据操作比添加数据复杂；注意，读线程会等待条件变量。第一个检查与 push() 相似。如果在读线程能够获得互斥锁之前发生了超时，那么不需要进行处理。接下来，读线程需要确保（这是第二个检查）它等待条件变量的时间不超过指定的超时时间。如果到超时时间段结束时还没有被唤醒，读线程需要唤醒自身并释放互斥锁。

有了这些背景知识，我们来看看 pthread_cond_timedwait 函数，这个函数用于进行第二个检查。这个函数与 pthread_cond_wait 相似，但是第三个参数是绝对时间值，到达这个时间时读线程自愿放弃等待。如果在超时之前读线程被唤醒，pthread_cond_timedwait 的返回值是 0。清单 12 给出代码。

清单 12. 从阻塞队列中取出数据，具有超时限制

```cpp
T TimedBlockingQueue<T>::pop(const int seconds) {
    struct timespec ts1, ts2; 
    clock_gettime(CLOCK_REALTIME, &ts1); 
    pthread_mutex_lock(&_lock);
    clock_gettime(CLOCK_REALTIME, &ts2);
 
    // First Check 
    if ((ts1.tv_sec – ts2.tv_sec) < seconds) { 
        ts2.tv_sec += seconds; // specify wake up time
        while(_list.empty( ) && (result == 0)) { 
            result = pthread_cond_timedwait(&_cond, &_lock, &ts2);
        }
        if (result == 0) { // Second Check 
            T _temp = _list.front( );
            _list.pop_front( );
            pthread_mutex_unlock(&_lock);
            return _temp;
        }
    }
    pthread_mutex_unlock(&lock);
    throw "timeout happened";
}
```

清单 12 中的 while 循环确保正确地处理虚假的唤醒。最后，在某些 Linux 系统上，clock_gettime 可能是 librt.so 的组成部分，可能需要在编译器命令行中添加 –lrt 开关。

### 使用 pthread_mutex_timedlock API

清单 11 和 清单 12 的缺点之一是，当线程最终获得锁时，可能已经超时了。因此，它只能释放锁。如果系统支持的话，可以使用 pthread_mutex_timedlock API 进一步优化这个场景。这个例程有两个参数，第二个参数是绝对时间值。如果在到达这个时间时还无法获得锁，例程会返回且状态码非零。因此，使用这个例程可以减少系统中等待的线程数量。下面是这个例程的声明：

```cpp
int pthread_mutex_timedlock(pthread_mutex_t *mutex,
        const struct timespec *abs_timeout);
```

## 设计有大小限制的并发阻塞队列

最后，讨论有大小限制的并发阻塞队列。这种队列与并发阻塞队列相似，但是对队列的大小有限制。在许多内存有限的嵌入式系统中，确实需要有大小限制的队列。

对于阻塞队列，只有读线程需要在队列中没有数据时等待。对于有大小限制的阻塞队列，如果队列满了，写线程也需要等待。这种队列的外部接口与阻塞队列相似，见 清单 13。（注意，这里使用向量而不是列表。如果愿意，可以使用基本的 C/C++ 数组并把它初始化为指定的大小。）

清单 13. 有大小限制的并发阻塞队列

```cpp
template <typename T>
class BoundedBlockingQueue {
public:
    BoundedBlockingQueue (int size) : maxSize(size) {
       pthread_mutex_init(&_lock, NULL);
       pthread_cond_init(&_rcond, NULL);
       pthread_cond_init(&_wcond, NULL);
       _array.reserve(maxSize);
    } 
    ~BoundedBlockingQueue ( ) { 
       pthread_mutex_destroy(&_lock);
       pthread_cond_destroy(&_rcond);
       pthread_cond_destroy(&_wcond);
    } 
    void push(const T& data);
    T pop( ); 
private:
    vector<T> _array; // or T* _array if you so prefer
    int maxSize;
    pthread_mutex_t _lock;
    pthread_cond_t _rcond, _wcond;
}
```

在解释添加数据操作之前，看一下 清单 14 中的代码。

```cpp
void BoundedBlockingQueue<T>::push(const T& value ) {
    pthread_mutex_lock(&_lock);
    const bool was_empty = _array.empty( );
    while (_array.size( ) == maxSize) { 
        pthread_cond_wait(&_wcond, &_lock);
    } 
    _ array.push_back(value);
    pthread_mutex_unlock(&_lock);
    if (was_empty) 
        pthread_cond_broadcast(&_rcond);
}
```

对于 清单 13 和 清单 14，要注意的第一点是，这个阻塞队列有两个条件变量而不是一个。如果队列满了，写线程等待 _wcond 条件变量；读线程在从队列中取出数据之后需要通知所有线程。同样，如果队列是空的，读线程等待 _rcond 变量，写线程在把数据插入队列中之后向所有线程发送广播消息。如果在发送广播通知时没有线程在等待 _wcond 或 _rcond，会发生什么？什么也不会发生；系统会忽略这些消息。还要注意，两个条件变量使用相同的互斥锁。清单 15 给出有大小限制的阻塞队列的 pop() 方法。

清单 15. 从有大小限制的阻塞队列中取出数据

```cpp
T BoundedBlockingQueue<T>::pop( ) {
    pthread_mutex_lock(&_lock);
    const bool was_full = (_array.size( ) == maxSize);
    while(_array.empty( )) {
        pthread_cond_wait(&_rcond, &_lock) ;
    }
    T _temp = _array.front( );
    _array.erase( _array.begin( ));
    pthread_mutex_unlock(&_lock);
    if (was_full)
        pthread_cond_broadcast(&_wcond);
    return _temp;
}
```

注意，在释放互斥锁之后调用 pthread_cond_broadcast。这是一种好做法，因为这会减少唤醒之后读线程的等待时间。

## 结束语

本文讨论了几种并发队列及其实现。实际上，还可能实现其他变体。例如这样一个队列，它只允许读线程在数据插入队列经过指定的延时之后才能读取数据。
