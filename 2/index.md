# Python多线程卖票


<!--more-->

```python
#!/usr/bin/python3

import threading
import time
import queue
from colorama import Fore
from rich.progress import Progress

tt = 50
threads = []
q = queue.Queue()
# 控制最大可用窗口数
sem=threading.Semaphore(5)

def sale(sem,progress,task):
    while not q.empty():
        if threading.current_thread().name == "第1窗口":
            print(Fore.GREEN + "{}：卖出第{}张票".format(threading.current_thread().name,q.get()))
        elif threading.current_thread().name == "第2窗口":  
            print(Fore.MAGENTA + "{}：卖出第{}张票".format(threading.current_thread().name,q.get()))
        elif threading.current_thread().name == "第3窗口":  
            print(Fore.BLUE + "{}：卖出第{}张票".format(threading.current_thread().name,q.get()))
        elif threading.current_thread().name == "第4窗口":  
            print(Fore.CYAN + "{}：卖出第{}张票".format(threading.current_thread().name,q.get()))
        else:
            print(Fore.YELLOW + "{}：卖出第{}张票".format(threading.current_thread().name,q.get()))
        if not progress.finished:
            progress.update(task, advance=1)
        time.sleep(5)  
    sem.release()

def main():
    for i in range(tt):
        q.put(i+1)
    with Progress() as progress:
        task = progress.add_task("[cyan]Processing", total=tt)
        # 控制窗口数
        for j in range(20):
            sem.acquire()
            td = threading.Thread(target=sale,args=(sem,progress,task),name="第"+str(j+1)+"窗口")
            # 添加线程到线程列表
            threads.append(td)
            td.start()
        # 等待所有线程完成后退出主线程
        for t in threads:
            t.join()    
    print(Fore.RED +"票已卖完！！！")

if __name__=="__main__":
    main()
```

{{< admonition>}}

存在问题

{{< /admonition>}}

该脚本由于`sale()`函数采用while循环，当票被卖空才会跳出循环，导致当前线程会一直持续到票卖空才结束，如此一来线程只会创建到最大线程数5个（由信号量控制），用这5个线程卖完所有的票。这样38行处的控制窗口数处只需大于最大线程数5便可，无太多意义

在一般的脚本中，多线程执行的函数一般不会存在像该脚本如此的循环（一直循环消耗样本直到结束），正常情况下第38行处应为样本总数，这样才能实现100张票创建100个线程去买，通过信号量来控制最大线程数，线程执行完毕自动销毁

该脚本需做修改：

* 去除`sale()`函数中的while循环

* 第38行处`20`改为样本总数`tt`

