## Python3 thread pool

```python
from concurrent.futures import ThreadPoolExecutor
import threading
import time


# 定义一个准备作为线程任务的函数
def action(max):
    my_sum = 0
    for i in range(1, max + 1):
        # print(threading.current_thread().name + '  ' + str(i))
        my_sum += i
        time.sleep(0.01)
    return my_sum


# 创建一个包含2条线程的线程池
with ThreadPoolExecutor(max_workers=2) as pool:
    # 向线程池提交一个task, 50会作为action()函数的参数
    future1 = pool.submit(action, 50)
    # 向线程池再提交一个task, 100会作为action()函数的参数
    future2 = pool.submit(action, 100)
    # 为future1添加线程完成的回调函数
    future1.add_done_callback(lambda r: print(r.result()))
    # 为future2添加线程完成的回调函数
    future2.add_done_callback(lambda r: print(r.result()))
    print('--------------')
```

## threading local()

```python
import threading

local = threading.local()


def process():
    # 3.获取当前线程关联的resource:
    res = local.resource
    print (res + "http://c.biancheng.net/python/")
    

def process_thread(res):
    # 1.为当前线程绑定ThreadLocal的resource:
    local.resource = res
    # 2.线程的调用函数不需要传递res了，可以通过全局变量local访问
    process()
t1 = threading.Thread(target=process_thread, args=('t1线程',))
t2 = threading.Thread(target=process_thread, args=('t2线程',))
t1.start()
t2.start()
```

