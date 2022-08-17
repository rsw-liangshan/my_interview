- 相关链接： https://blog.csdn.net/scgaliguodong123_/article/details/121986409
## 一： 多进程之间为什么要通信？ 
- 数据传输： 一个进程需要将它的数据发送给另一个进程。
- 资源共享： 多个进程之间共享同样的资源。
- 事件通知： 一个进程需要向另一个或一组进程发送消息，通知它们发生了某种事件;
- 进程控制： 有些进程希望完全控制另一个进程的执行(如Debug进程)，该控制进程希望能够拦截另一个进程的所有操作，并能够及时知道它的状态改变。

## 二：进程之间的通信原理
- 每个进程各自有不同的用户地址空间,任何一个进程的全局变量在另一个进程中都看不到，所以进程之间要交换数据必须通过内核,在内核中开辟一块缓冲区,进程1把数据从用户空间拷到内核缓冲区,进程2再从内核缓冲区把数据读走,内核提供的这种机制称为进程间通信机制。

## 三：进程之间的通信方式
- 管道 ， ｜ 
  - 半双工: 数据只能单向流动，而且只能在具有亲缘关系的进程间使用。进程的亲缘关系通常是指父子进程关系。
- 命令管道
  - 有名管道也是半双工的通信方式，但是它允许无亲缘关系进程间的通信。
- 消息队列
  - 消息队列是由消息的链表，存放在内核中并由消息队列标识符标识。消息队列克服了信号传递信息少、管道只能承载无格式字节流以及缓冲区大小受限等缺点。
- 共享内存
  - 共享内存就是映射一段能被其他进程所访问的内存，这段共享内存由一个进程创建，但多个进程都可以访问。共享内存是最快的 IPC 方式，它是针对其他进程间通信方式运行效率低而专门设计的。它往往与其他通信机制，如信号量，配合使用，来实现进程间的同步和通信。
- 信号量
  - 信号量是一个计数器。
- 套接字
  - 它可用于不同机器之间的进程通信。
- 信号
  - 信号是一种比较复杂的通信方式，用于通知接收进程某个事件已经发生。

## 四：Python如何实现进程通信？ 
### 4.1: 管道方式实现进程通信
- 案例： 
  - 主进程给子进程传递消息
  - 主进程是生产者， 子进程是消费者。
```python 
import time
from multiprocessing import Pipe, Process


def consumer(left, right):
    """消费者"""
    left.close()
    while True:
        try:
            get_message = right.recv()
            print("消费者接收到消息： %s" % get_message)
        except EOFError:
            right.close()
            break


def producer(left, right):
    """生产者"""
    right.close()
    # 循环发送10条消息
    for i in range(10):
        left.send(i)
        time.sleep(1)
    left.close()


def pipe_run():
    # 实例化管道
    left, right = Pipe()

    # 开启子进程准备接收管道右端的数据
    consumer_1 = Process(target=consumer, args=(left, right))
    consumer_1.start()

    # 实例化生产者发送消息
    producer(left=left, right=right)

    right.close()
    left.close()
    consumer_1.join()


if __name__ == '__main__':
    pipe_run()
    # 消费者接收到消息： 0
    # 消费者接收到消息： 1
    # 消费者接收到消息： 2
    # 消费者接收到消息： 3
    # 消费者接收到消息： 4
    # 消费者接收到消息： 5
    # 消费者接收到消息： 6
    # 消费者接收到消息： 7
    # 消费者接收到消息： 8
    # 消费者接收到消息： 9 
```

### 4.2: 队列

```python
import time
from multiprocessing import Pipe, Process, Queue, set_start_method
def consumer2(q):
    while True:
        res = q.get()
        if res is None:
            break
        time.sleep(1)
        print('消费者接收到: %s' % res)


def producer2(q):
    for i in range(10):
        res = '第%s个消息' % str(i)
        q.put(res)
        print("生产者发送第%s个消息" % str(i))
        time.sleep(0.5)
    q.put(None)


def queue_test():
    # 设置启动方式
    set_start_method('fork')

    # 创建进程队列
    q = Queue()
    # 生产者子进程, 消费者子进程
    producer_1 = Process(target=producer2, args=(q,))
    consumer_1 = Process(target=consumer2, args=(q,))

    # 开启子进程
    producer_1.start()
    consumer_1.start()


if __name__ == '__main__':
    queue_test()
    # 生产者发送第0个消息
    # 生产者发送第1个消息
    # 生产者发送第2个消息
    # 消费者接收到: 第0个消息
    # 生产者发送第3个消息
    # 消费者接收到: 第1个消息
    # 生产者发送第4个消息
    # 生产者发送第5个消息
    # 消费者接收到: 第2个消息
    # 生产者发送第6个消息
    # 生产者发送第7个消息
    # 消费者接收到: 第3个消息
    # 生产者发送第8个消息
    # 生产者发送第9个消息
    # 消费者接收到: 第4个消息
    # 消费者接收到: 第5个消息
    # 消费者接收到: 第6个消息
    # 消费者接收到: 第7个消息
    # 消费者接收到: 第8个消息
    # 消费者接收到: 第9个消息
```
### 4.3: 共享数据
- 虽然进程间数据独立，但可以通过Manager实现数据共享。
- 案例：我们可以在共享内存中设置一个变量， 然后让每个进程都去修改。
- 共享内存， 并发写问题如何处理： 加互斥锁
```python 
def work(d, lock):
    with lock:  # 不加锁而操作共享的数据,肯定会出现数据错乱
        print(f"计数器减一，当前为：{d['count']}")
        d['count'] -= 1


def manager_test():
    # 进程锁
    lock = Lock()
    with Manager() as m:
        dic = m.dict({'count': 20})
        p_l = []
        for i in range(20):
            p = Process(target=work, args=(dic, lock))
            p_l.append(p)
            p.start()

        for p in p_l:
            p.join()


if __name__ == '__main__':
    manager_test()
    # 计数器减一，当前为：20
    # 计数器减一，当前为：19
    # 计数器减一，当前为：18
    # 计数器减一，当前为：17
    # 计数器减一，当前为：16
    # 计数器减一，当前为：15
    # 计数器减一，当前为：14
    # 计数器减一，当前为：13
    # 计数器减一，当前为：12
    # 计数器减一，当前为：11
    # 计数器减一，当前为：10
    # 计数器减一，当前为：9
    # 计数器减一，当前为：8
    # 计数器减一，当前为：7
    # 计数器减一，当前为：6
    # 计数器减一，当前为：5
    # 计数器减一，当前为：4
    # 计数器减一，当前为：3
    # 计数器减一，当前为：2
    # 计数器减一，当前为：1
```
### 4.4: 信号量
- 互斥锁是同时只允许一个线程更改数据，而Semaphore是同时允许一定数量的线程更改数据。
- 案例：比如厕所有3个坑，那最多只允许3个人上厕所，后面的人只能等里面有人出来了才能再进去，如果指定信号量为3，那么来一个人获得一把锁，计数加1，当计数等于3时，后面的人均需要等待。一旦释放，就有人可以获得一把锁。
- 注意：信号量与进程池的概念很像，但是要区分开，信号量涉及到加锁的概念。
```python 
from multiprocessing import Process,Semaphore
import time,random

def go_wc(sem,user):
    sem.acquire()
    print('%s 占到一个茅坑' %user)
    time.sleep(random.randint(0,3)) # 模拟每个人拉屎速度不一样，0代表有的人蹲下就起来了
    sem.release()


if __name__ == '__main__':
    sem=Semaphore(5)
    p_l=[]
    for i in range(13):
        p=Process(target=go_wc,args=(sem,'user%s' %i,))
        p.start()
        p_l.append(p)

    for i in p_l:
        i.join()
```
### 4.5: 信号/事件
- python进程的事件用于主进程控制其他进程的执行，事件主要提供了三个方法 set、wait、clear。
- 全局定义了一个“Flag”，如果“Flag”值为 False，那么当程序执行 event.wait 方法时就会阻塞，如果“Flag”值为True，那么event.wait 方法时便不再阻塞。
- 其中，clear方法：将“Flag”设置为False，set方法：将“Flag”设置为True。

```python
import multiprocessing
import time

from multiprocessing import Process, Queue, set_start_method

event = multiprocessing.Event()

def xiao_fan(event):
    print('小贩：生产...')
    print('小贩：售卖...')
    # time.sleep(1)
    print('小贩：等待就餐')
    event.set()
    event.clear()
    event.wait()
    print('小贩：谢谢光临')
    event.set()
    event.clear()


def gu_ke(event):
    print('顾客：准备买早餐')
    event.set()
    event.clear()
    event.wait()
    print('顾客：买到早餐')
    print('顾客：享受美食')
    # time.sleep(2)
    print('顾客：付款，真好吃...')
    event.set()
    event.clear()


if __name__ == '__main__':
    set_start_method('fork', True)
    # 创建进程
    xf = multiprocessing.Process(target=xiao_fan, args=(event,))
    gk = multiprocessing.Process(target=gu_ke, args=(event, ))
    # 启动进程

    gk.start()
    xf.start()

    # time.sleep(2)
```

### 4.5: 进程通信方式各自的优缺点
- 管道
  - 单向， 通过缓冲区管理。
- 信号量
  - 文件权限 + 计数器
- 信号
  - 单向：主进程 ---> 子进程
- 共享内存
  - 利用文件权限控制。 
  - 对于共享内存，数据操作最快，因为是直接在内存层面操作，省去中间的拷贝工作。但是共享内存只能在单机上运行，且只能操作基础数据格式，无法直接共享复杂对象。
- 队列
  - 队列可以在多个进程间传递，可以在不同主机上的进程间共享，以实现分布式。
- Socket
  - 基于IP地址， 端口和文件路径寻址。
    - TCP/UDP








