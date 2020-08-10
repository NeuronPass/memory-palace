# 使用concurrent.futures实现并发

## 简介

concurrent.futures模块提供了一套管理异步任务的接口，异步任务可以用进程实现，也可以用线程实现。由于Python中GIL锁的存在，CPU密集型任务应尽可能用多进程完成并行，而IO密集型任务则应尽量使用多线程达到并发的效果。

## 进程池和线程池

下面以一个并行计算的任务为例介绍进程池的使用方法。假定现在要计算从1到2000000之间所有素数的个数，统计代码如下：

```
from concurrent import futures
from math import sqrt

def is_prime(x):
    if x <= 1:
        return False
    for i in range(2, int(sqrt(x)+1)):
        if x % i == 0:
            return False
    return True

def count_prime(left, right):
    result = 0
    for num in range(left, right + 1):
        if is_prime(num):
            result = result + 1
    return result

def parallel_count():
    workers = 4
    unit = 2000000 // workers
    with futures.ProcessPoolExecutor(max_workers=workers) as executor:
        to_do = []
        for i in range(workers):
            to_do.append(executor.submit(count_prime, unit*i + 1, unit*(i+1) + 1))
        count = 0
        for future in futures.as_completed(to_do):
            count += future.result()
    return count
```

代码中首先创建了一个大小为4的进程池，然后将统计区间均分成4份，并依次调用`executor.submit`方法产生4个`Future`对象，接着将这些`Future`对象组成的列表传递给`futures.as_completed`方法等待进程运行结束，此时代码阻塞，随后按照进程执行的快慢依次调用`future.result`获取执行结果。`Futures`对象又称为期程，表示在将来要执行的时间，我们可以调用期程相关的方法进行状态查询、获取结果、添加回调以及取消一个期程等，详见<https://docs.python.org/3.10/library/concurrent.futures.html>。

注意`futures.as_completed`方法返回的结果顺序并不与`Future`对象加入到队列中的顺序一致，如果顺序很重要，可以使用`executor.map`方法。此外，如果没有指定进程池大小，默认大小与CPU核心数相等。

线程池的使用方法与进程池基本一致，只需要把`ProcessPoolExecutor`换成`ThreadPoolExecutor`即可。下面以一个并行下载的例子进行说明。

```
from concurrent import futures
import requests

def repeat_download(url, n):
    for i in range(n):
        resp = requests.get(url)

def concurrent_download():
    workers = 4
    unit = 200 // workers
    url = "http://baidu.com"
    with futures.ThreadPoolExecutor(max_workers=4) as executor:
        to_do = []
        for i in range(workers):
            to_do.append(executor.submit(repeat_download, url, unit))
        for future in futures.as_completed(to_do):
            future.result()
```

代码中开启4个线程并发下载200个网页的内容，每个线程下载50个。对比可见，`ThreadPoolExecutor`和`ProcessPoolExecutor`拥有一致的API。

## 多进程和多线程对比

下面对比上述两个场景下的多线程代码和多进程代码的性能差异，其中进程池和线程池的大小均为4。

| 场景  | 串行  | 多进程 | 多线程 |
|---|---|---|---|
| 计算2000000以内的素数个数  | 11.26s | 4.83s | 11.68s |
| 从网络上下载200个网页内容 | 12.27s | 2.56s | 2.08s |

从实验结果可以看出，Python中的多进程适合CPU密集型应用，能够充分发挥多核特性。而多线程更适合IO密集型应用，对CPU密集型应用无法带来性能上的提升，这是由于Python解释器的GIL使得Python中的多线程变成了“假”线程，任意时刻只能有一个线程得到执行，无法利用多核特性，然而在执行IO操作时，Python解释器会释放GIL锁，从而使别的线程也有机会得到调度。