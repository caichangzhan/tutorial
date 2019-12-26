如果你的程序运行一次就退出了，你可能体会不到内存管理的重要性。如果你写的程序需要 7x24 小时持续不断地运行，那么内存管理就非常重要，尤其对于重要的服务，不能出现内存泄漏。

这里的内存泄漏不是出内存出现的数据丢失，或者说内存空间在物理上消失了，而是指程序本身没有设计好，导致占用的内存应该释放出来而实际上没有释放，导致系统可用的内存严重不足，出现系统或服务因此而崩溃。

Python 是如何进行垃圾回收的呢？ 换句话说 Python 是怎么回收不再使用的内存空间的呢？



## 1、如何找到可以回收的内存？

众所周知，Python 中的万物皆对象，对象占用一定的内存，我们通过变量来访问一个对象，变量的本质，就是对象的一个指针（地址）。

如何让我们自己决定回收哪一个对象的空间，很容易想到这样的方法：没有变量指向该对象时，说明它已经没用了，它占用的空间就可以回收。事实上，Python 就是这么做的，我们来看一段代码和结果：


```python
import os
import psutil


# 显示当前 python 程序占用的内存大小
def show_memory_info(hint):
    pid = os.getpid()
    p = psutil.Process(pid)
    info = p.memory_info()
    memory = info.rss / 1024.0 / 1024
    print("{} 内存占用: {} MB".format(hint, memory))


def func():
    show_memory_info("func 调用前")
    a = [i for i in range(10000000)]
    show_memory_info("func 调用结束前")


if __name__ == "__main__":
    func()
    show_memory_info("func 调用结束后")

```
运行结果如下：

```
func 调用前 内存占用: 29.63671875 MB
func 调用结束前 内存占用: 414.1640625 MB
func 调用结束后 内存占用: 27.2265625 MB

```

通过这个例子，可以看出在函数 func 中创建一个大列表 a 后，内存占用迅速增加到 400 MB，func 调用结束后内存又恢复到 27 MB，说明在 func 调用结束后，Python 知道变量 a 不再被使用，于是便进行垃圾回收。

从另外一个角度理解：函数内部声明的列表 a 是局部变量，在函数返回后，局部变量的引用会注销掉；此时，列表 a 所指代对象的引用数为 0，Python 便会执行垃圾回收，因此之前占用的大量内存就又回来了。

如果我们修改 func 函数中的变量 a 为全局变量，那么函数调用结束后，a 仍然会被使用，此时内存将不会被回收：

```python
def func():
    show_memory_info("func 调用前")
    global a
    a = [i for i in range(10000000)]
    show_memory_info("func 调用结束前")
```

执行结果为：

```
func 调用前 内存占用: 29.625 MB
func 调用结束前 内存占用: 416.796875 MB
func 调用结束后 内存占用: 416.80078125 MB
```

也就是说当一个变量的引用计数为 0 时，Python 解释器可以对其进行增加回收。 那么问题来了：如何判断某个对象被引用的个数？

好在 Python 标准库提供了一个可以直接查看变量引用计数的函数 `sys.getrefcount(var)` 。 使用方法如下：

```python

import sys

a = []

# 两次引用，一次来自 a，一次来自 getrefcount
print(sys.getrefcount(a))

def func(a):
    # 四次引用，a，python 的函数调用栈，函数参数，和 getrefcount
    print(sys.getrefcount(a))

func(a)

# 两次引用，一次来自 a，一次来自 getrefcount，函数 func 调用已经不存在
print(sys.getrefcount(a))

```
输出 

```
2
4
2
```
简单介绍一下，sys.getrefcount() 这个函数，可以查看一个变量的引用次数。这段代码本身应该很好理解，不过别忘了，getrefcount 本身也会引入一个计数。

另一个要注意的是，在函数调用发生的时候，会产生额外的两次引用，一次来自函数栈，另一个是函数参数。

```python

import sys

a = []

print(sys.getrefcount(a)) # 两次

b = a

print(sys.getrefcount(a)) # 三次

c = b
d = b
e = c
f = e
g = d

print(sys.getrefcount(a)) # 八次

```
看到这段代码，需要你稍微注意一下，a、b、c、d、e、f、g 这些变量全部指代的是同一个对象，而 sys.getrefcount() 函数并不是统计一个指针，而是要统计一个对象被引用的次数，所以最后一共会有八次引用。

现在你已经明白，Python 是会自动回收垃圾的。

## 2、可以手动回收内存吗？

虽然 Python 可以自动回收内存，可我偏偏想手动回收内存，可以吗？ 非常简单，假如有一个变量 a，后面不想再用它了，那么执行两条代码搞定：

```python

del a
gc.collect()
```

## 3、引用计数为 0 是否是垃圾回收的充要条件？

我想，肯定有人觉得自己都懂了，那么，如果此时有面试官问：引用次数为 0 是垃圾回收启动的充要条件吗？还有没有其他可能性呢？

如果你也被困住了，别急。我们不妨小步设问，先来思考这么一个问题：如果有两个对象，它们互相引用，并且不再被别的对象所引用，那么它们应该被垃圾回收吗？

```python
import os,sys
import psutil


# 显示当前 python 程序占用的内存大小
def show_memory_info(hint):
    pid = os.getpid()
    p = psutil.Process(pid)
    info = p.memory_info()
    memory = info.rss / 1024.0 / 1024
    print("{} 内存占用: {} MB".format(hint, memory))


def func2():
    show_memory_info("func2 调用前")
    a = [i for i in range(1000000)]
    b = [i for i in range(1000000)]
    a.append(b)
    b.append(a)
    show_memory_info("func2 调用结束前")



if __name__ == "__main__":
    func2()
    show_memory_info("func2 调用结束后")
```
输出结果如下：

```
func2 调用前 内存占用: 29.65625 MB
func2 调用结束前 内存占用: 795.01953125 MB
func2 调用结束后 内存占用: 795.09375 MB
```

这里，a 和 b 互相引用，并且，作为局部变量，在函数 func 调用结束后，a 和 b 这两个指针从程序意义上已经不存在了。但是，很明显，依然有内存占用！为什么呢？因为互相引用，导致它们的引用数都不为 0。

试想一下，如果这段代码出现在生产环境中，哪怕 a 和 b 一开始占用的空间不是很大，但经过长时间运行后，Python 所占用的内存一定会变得越来越大，最终撑爆服务器，后果不堪设想。

当然，有人可能会说，互相引用还是很容易被发现的呀，问题不大。可是，更隐蔽的情况是出现一个引用环，在工程代码比较复杂的情况下，引用环还真不一定能被轻易发现。

如果真的怕有引用环的出现而没有检查出来的话，可以调用 `gc.collect()` 回收垃圾，在上述代码 func2 调用结束的位置调用 `gc.collect()` 后

```python

if __name__ == "__main__":
    func2()
    gc.collect()
    show_memory_info("func2 调用结束后")
```
执行结果

```
func2 调用前 内存占用: 29.625 MB
func2 调用结束前 内存占用: 804.62109375 MB
func2 调用结束后 内存占用: 30.54296875 MB
```

上述是我们手工回收的演示，事实上 Python 可以自动处理，Python 使用标记清除（mark-sweep）算法和分代收集（generational），来启用针对循环引用的自动垃圾回收。

先来看标记清除算法：我们先用图论来理解不可达的概念。对于一个有向图，如果从一个节点出发进行遍历，并标记其经过的所有节点；那么，在遍历结束后，所有没有被标记的节点，我们就称之为不可达节点。显而易见，这些节点的存在是没有任何意义的，自然的，我们就需要对它们进行垃圾回收。 当然，每次都遍历全图，对于 Python 而言是一种巨大的性能浪费。所以，在 Python 的垃圾回收实现中，mark-sweep 使用双向链表维护了一个数据结构，并且只考虑容器类的对象（只有容器类对象才有可能产生循环引用）。具体算法这里我就不再多讲了，毕竟我们的重点是关注应用。

再来看分代收集：Python 将所有对象分为三代。刚刚创立的对象是第 0 代；经过一次垃圾回收后，依然存在的对象，便会依次从上一代挪到下一代。而每一代启动自动垃圾回收的阈值，则是可以单独指定的。当垃圾回收器中新增对象减去删除对象达到相应的阈值时，就会对这一代对象启动垃圾回收。事实上，分代收集基于的思想是，新生的对象更有可能被垃圾回收，而存活更久的对象也有更高的概率继续存活。因此，通过这种做法，可以节约不少计算量，从而提高 Python 的性能。

刚面的问题，你应该能回答得上来了吧！没错，引用计数是其中最简单的实现，不过切记，引用计数并非充要条件，它只能算作充分非必要条件；至于其他的可能性，我们所讲的循环引用正是其中一种。


## 4、如何调试内存泄漏？

像前文提到的手环引用，有没有办法将变量的引用关系使用一个树状的图来表示呢？ 这样就可以调试内存泄漏了。事实上，真有，它叫 objgraph，一个非常好用的可视化引用关系的包。在这个包中，我主要推荐两个函数，第一个是 `show_refs()`，它可以生成清晰的引用关系图。

```python
import objgraph

a = [1, 2, 3]
b = [4, 5, 6]

a.append(b)
b.append(a)

objgraph.show_refs([a])

objgraph.show_backrefs([a])
```
它们以图形的方式显示引用关系，运行下就知道了，非常便于调试。 [官方文档](https://mg.pov.lt/objgraph/) 。


## 5、总结

1、Python 会自动进行垃圾回收。
2、引用计数为 0 时回收是最简单的一种情况，还会有循环引用。
3、Python 有两种自动回收的算法。
4、调试内存泄漏方面， objgraph 是很好的可视化分析工具。

关注微信公众号：Python七号，像玩游戏一样学习 Python。