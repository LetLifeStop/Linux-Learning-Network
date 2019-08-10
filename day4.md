day3补充：

1. TCP状态转换图，三次握手，四次握手（半关闭），主动请求端状态

2. select 和 poll函数的第一个参数上限
3. select 的传入传出参数
4. 注意重复计算的部分
5. **poll函数不能跨平台**



### 多路IO转接服务器

#### epoll函数

epoll函数同样可以修改文件描述符上限

相对于select和poll函数，select是对于所有的遍历一边，poll函数遍历给定的数组，epoll可以提避免这些，从而提高遍历的效率，

在面对大的数据量的时候，select函数和poll函数的效率是差不多的



**基础的API**

1. **创建一个epoll句柄**

```c
int  epoll_create(int size)
```

用来告诉内核需要创建多大的epoll模型（最多能接听多少文件描述符，只是**建议值**）

返回值：ret

int类型的文件描述符

虚拟内存， 3g~4g中pcb进程控制块中，文件描述符表表示的文件描述符，从第三个开始存储。如果调用成功，在内核当中，返回的文件描述符指向一颗红黑树（平衡二叉树）的树根（左右子树高度误差小于 1），查找方式通过log级别的就可以查找到。 



2. **控制某个epoll监控的文件描述符上的事件，注册，修改，删除**

**int epoll_ctl函数**

```c
int epoll_ctl(int epfd ,int op , int fd ,struct epoll_event *event);
```

参数：

1. 把哪棵树的树根加入

2. 操作类型

3. 对哪个文件描述符进行操作

4. ```c
   struct epoll_event{
   uint32_t events;
   epoll_data_t data;
   };
   ```

   event ，三种操作

   EPOLLIN/OUT/ERR

   data;

   ```c
   typedef union epoll_data{  联合体？？
   void *ptr; // 泛型指针
   int fd; // 和参数3中的fd应该相同
   uint32_t u32;
   uint64_t u64;
   }epoll_data_t;
   ```



3. **监听**

```c
epoll_wait(int epfd ,struct epoll_event *events, int maxevents ,int timeout); // timeout为毫秒级别
```

第二个参数和epoll_ctl中的events参数的区别：

* epoll_ctl 中的events表示结构体的地址，为传入参数。
* epoll_wait中的events表示数组，为传出参数，数组中的每一个元素都是上面的结构体类型。

返回值：成功返回有多少文件描述符就绪

自动把有用的文件描述符按照返回值的大小自动从 0开始排序，只需要判断这一段的类型就好了

  

**上述函数的使用流程：**

```c
int epfd = epoll_create(10);
struct epoll_event evt;
evt.events = EPOLLIN; // 是否有客户端向服务器端请求连接
evt.data.fd = lfd; // 向红黑树中加入一个节点
epoll_ctl(epfd , EPOLL_CTL_ADD , lfd ,&evt);
struct epoll_event evts[10 + 1];
int ret = epoll_wait(epfd ,vets, 10 ,-1); // 阻塞的等待
// 如果有两个文件描述符有变动，遍历方法如下：
for (int i = 0 ; i < 2;i++){
    /***/
}
```



**实现方法：**

