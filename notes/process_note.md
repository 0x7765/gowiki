# 多进程编程

## 原生`fork`

`windows`平台不支持

`fork`是一个系统调用。每次调用`fork`时，操作系统将当前的进程复制一份，然后分别执行返回。

`fork`出来的子进程永远返回0，父进程返回子进程的`pid` .子进程 只需要调用`getppid` 就可以返回父进程的`pid`

```python
# -*- coding: utf-8 -*-
# 原生 fork 支持(unix/linux/mac) window 不支持
import os
import logging

logging.basicConfig(level=logging.INFO)

if __name__ == '__main__':
    logging.info('current pid is {pid} '.format(pid=os.getpid()))
    fid = os.fork()
    if fid == 0:
        logging.info('this is child, parent pid is {parent_pid} \t self pid is {self_pid}'
                     .format(parent_pid=os.getppid(), self_pid=os.getpid()))
    else:
        logging.info(
            'this is parent, pid is {self_pid} \t created a child process. child pid is {child_pid}'.format(
                self_pid=os.getpid(), child_pid=fid))
```

## `multiprocessing` 

基本使用 demo

```python
# -*- coding: utf-8 -*-
# multiprocessing 是一个跨平台的多进程支持模块

from multiprocessing import Process
import logging
import requests
import os

logging.basicConfig(level=logging.INFO)


def get_from_url(url):
    logging.info('Run child process pid is {pid}...'.format(pid=os.getpid()))
    resp = requests.get(url=url)
    logging.info('get {url}, resp status is {code}'.format(url=url, code=resp.status_code))
    logging.info('cookies is \t{cookies}'.format(cookies=resp.cookies))
    logging.info('headers is \t{headers}'.format(headers=resp.headers))


if __name__ == '__main__':
    logging.info('main process pid is {pid}...'.format(pid=os.getpid()))
    '''
        target 指定一个函数 代表新进程要执行的内容 
        args 是函数的参数 是个tuple
        name 指定新进程的名字
    '''
    new_process = Process(target=get_from_url, args=('https://www.baidu.com',), name='http')
    # 执行进程
    new_process.start()
    # join 等待新进程完成
    new_process.join()
    logging.info('new process done')
```

## 进程池

```python
# -*- coding: utf-8 -*-
# multiprocessing 是一个跨平台的多进程支持模块

from multiprocessing import Pool
from single_process import get_from_url
import logging

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s  %(filename)s : %(levelname)s  %(message)s',
    datefmt='%Y-%m-%d %A %H:%M:%S',
)

if __name__ == '__main__':
    # 创建一个进程池 大小是4
    ps = Pool(4)

    urls = [
        'https://www.baidu.com',
        'https://www.google.com',
        'https://gocn.vip',
        'https://studygolang.com',
        'https://github.com/']

    for url in urls:
        ps.apply_async(get_from_url, args=(url,))
    logging.info('Waiting for all subprocesses done...')

    # 调用join()
    # 之前必须先调用close()，调用close()
    # 之后就不能继续添加新的Process
    ps.close()
    ps.join()
    logging.info('All processes done.')
```

