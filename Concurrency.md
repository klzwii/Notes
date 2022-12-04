## 死锁产生的必要条件
- 互斥条件：一个资源每次只能被一个进程使用
- 请求与保持条件：一个进程因请求资源而阻塞时，对已获得的资源保持不放
- 不剥夺条件:进程已获得的资源，在末使用完之前，不能强行剥夺
- 循环等待条件:若干进程之间形成一种头尾相接的循环等待资源关系
## 线程进程相关
### 线程共享<cite>[[1]]</cite>
- 进程指令
- 大多数数据
- 打开的文件（描述符）
- 信号和信号处理程序
- 当前工作目录
- 用户和组的ID
### 线程独有<cite>[[1]]</cite>
- 线程ID
- 寄存器集、堆栈指针
- 本地变量的堆栈，返回地址
- 信号掩码
- 优先级
- 返回值： errno
## 内存相关
### 内存屏障
内存屏障涉及多个方面，编译指令重排，cpu乱序执行，invalid/store queue。
其主要目的是对屏障前后的load/store指令进行顺序约束，确保在并发访问过程中修改是对其它线程可见的。
## 高并发IO
### 多路复用

### 传统IO方式

socket绑定某一端口，直到收到连接产生一个新的socket，交给一个工作线程处理。
工作线程会调用read并陷入阻塞直到有可以读取的数据。因此该情况下有多少

### 0拷贝


### 传统使用 I/O 中断方式读取数据：<cited>[[2]]</cited>

1. 用户进程向 CPU 发起 read 系统调用读取数据，由用户态切换为内核态，然后一直阻塞等待数据的返回；
2. CPU 在接收到指令以后对磁盘发起 I/O 请求，将磁盘数据先放入磁盘控制器缓冲区；
3. 数据准备完成以后，磁盘向 CPU 发起 I/O 中断；
4. CPU 收到 I/O 中断以后将磁盘缓冲区中的数据拷贝到内核缓冲区，然后再从内核缓冲区拷贝到用户缓冲区；
5. 用户进程由内核态切换回用户态，解除阻塞状态。

### DMA(Direct Memory Access)

1. 用户进程向 CPU 发起 read 系统调用读取数据，由用户态切换为内核态，然后一直阻塞等待数据的返回；
2. CPU 在接收到指令以后对 DMA 磁盘控制器发起调度指令；
3. DMA 磁盘控制器对磁盘发起 I/O 请求，将磁盘数据先放入磁盘控制器缓冲区，CPU 全程不参与此过程；
4. 数据读取完成后，DMA 磁盘控制器会接受到磁盘的通知，将数据从磁盘控制器缓冲区拷贝到内核缓冲区；
5. DMA 磁盘控制器向 CPU 发出数据读完的信号，由 CPU 负责将数据从内核缓冲区拷贝到用户缓冲区；
6. 用户进程由内核态切换回用户态，解除阻塞状态。

### 存在问题

- 多次内核态到用户态的切换
- 数据需要被从磁盘控制器缓冲区拷贝到内核缓冲区再拷贝到用户缓冲区

### 解决方案

#### mmap
将内核缓冲区部分空间映射到用户缓冲区，减少一次拷贝

#### sendfile(socket_fd, file_fd, len);
对于一些不需要修改而直接进行发送的文件，sendfile具有显著的优势。
其全部操作都在内核态进行，将长度为len的数据从file拷贝到socket

对于支持DMA gather copy的设备，CPU从内核缓冲区到Socket缓冲区的操作也可以被消除。
CPU只需要将fd和长度拷贝到Socket缓冲区，DMA将会直接使用该信息进行数据发送。

#### splice(fd_in, off_in, fd_out, off_out, len, flags)

splice要求fd_in和fd_out之间必须有一个是管道
常见用法是建立一个匿名的管道将需要发送的文件fd与socket 连接fd

#### !!!! after linux 4.18
https://lwn.net/Articles/754681/
https://github.com/torvalds/linux/blob/master/tools/testing/selftests/net/tcp_mmap.c
using getsockopt to read from fd



[1]: https://www.cs.cmu.edu/afs/cs/academic/class/15492-f07/www/pthreads.html
[2]: https://www.cnblogs.com/rickiyang/p/13265043.html