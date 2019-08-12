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

int类型的文件描述符，红黑树的树根

虚拟内存， 3g~4g中pcb进程控制块中，文件描述符表表示的文件描述符，从第三个开始存储。如果调用成功，在内核当中，返回的文件描述符指向一颗红黑树（平衡二叉树）的树根（左右子树高度误差小于 1），查找方式通过log级别的就可以查找到。 



2. **控制某个epoll监控的文件描述符上的事件，注册，修改，删除**

**int epoll_ctl函数**

```c
int epoll_ctl(int epfd ,int op , int fd ,struct epoll_event *event);
```

参数：

1. 把哪棵树的树根加入

2. 操作类型event ，三种操作 EPOLLIN/OUT/ERR

3. 对哪个文件描述符进行操作

4. ```c
   struct epoll_event{
   uint32_t events; // 事件类型
   epoll_data_t data;
   };
   ```

   data结构体

   ```c
   typedef union epoll_data{  
   void *ptr; // 泛型指针
   int fd; // 和参数3中的fd应该相同
   uint32_t u32;
   uint64_t u64;
   }epoll_data_t;
   ```

**void  *ptr 指针：**

可以搞成回调函数

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

注意点：

1. 在建立红黑树根的时候，最后一个参数的文件描述符是 lfd，并且事件为读取
2. 当收到的信息是从lfd传来的时候，这个时候是要创立链接的,也就是要在树上加一个点
3. 在读取的时候，读取是按照最大长度，然后写入的时候是按照读取了多么长的字符串来写入的，否则会出现乱码

```c
/*************************************************************************
	> File Name: poll.c
	> Author: 
	> Mail: 
	> Created Time: 2019年08月11日 星期日 09时58分14秒
 ************************************************************************/

#include<stdio.h>
#include<stdlib.h>
#include<ctype.h>
#include<string.h>
#include<sys/epoll.h>
#include<sys/types.h>
#include<sys/socket.h>
#include<arpa/inet.h>
#include<unistd.h>

# define PORT 8556

int main(){
 
    int lfd , cfd, sockfd ,ret_del ,tot ,i ,j ,n;
    int ret_setsocket , ret_bind ,ret_listen,ret_epoll_create , ret_epoll_ctl;
    struct sockaddr_in ser_addr, cli_addr;
    struct epoll_event rt ,tmp;
    struct epoll_event mul[10 + 1];
    int client[10 + 1] ,maxi = -1;
    socklen_t ser_addr_len , cli_addr_len;
    char buf[100],str[100];


    lfd = socket(AF_INET , SOCK_STREAM , 0);
    if(lfd < 0 ){
        perror("create socket error");
        exit(1);
    } 

    int opt = 1;
    ret_setsocket = setsockopt(lfd , SOL_SOCKET , SO_REUSEADDR ,&opt ,sizeof(opt) );
    if(ret_setsocket <  0){
        perror("setsocket errror");
        exit(1);
    }
    
    bzero(&ser_addr , sizeof(ser_addr));
    ser_addr.sin_port = htons(PORT);
    ser_addr_len = sizeof(ser_addr);
    ser_addr.sin_family = AF_INET;
    ser_addr.sin_addr.s_addr = htonl(INADDR_ANY);

    ret_bind =  bind(lfd ,(struct sockaddr *)&ser_addr, ser_addr_len);
    if(ret_bind < 0 ){
        perror("create bind error");
        exit(1);
    }

    ret_listen = listen(lfd , 100);
    if(ret_listen < 0){
        perror("set listen error");
        exit(1);
    }

    for(int i = 0 ; i < 10 ;i++ ){
        client[i] = -1;
    }
    ret_epoll_create =  epoll_create(10);
    if(ret_epoll_create < 0){
        perror("epoll create error");
        exit(1);
    }

    rt.events = EPOLL_CTL_ADD;
    rt.data.fd =lfd;
    ret_epoll_ctl = epoll_ctl(ret_epoll_create , EPOLL_CTL_ADD , lfd , &rt);
    if(ret_epoll_ctl < 0){
     perror("create root error");
        exit(1);
    } 

    while(1){
         tot = epoll_wait(ret_epoll_create , mul , 10 , -1);
        if(tot < 0){
            perror("epoll wait error") ;
            exit(1);
        }

        for(i = 0 ; i < tot ;i++){
            if(!(mul[i].events & EPOLLIN))continue;
            if(mul[i].data.fd == lfd){
        
             cli_addr_len = sizeof(cli_addr);
             cfd = accept(lfd ,(struct sockaddr *)&cli_addr ,&cli_addr_len );
                if(cfd < 0 ){
                    perror("accept error");
                    exit(1);
                }
            printf("reveieve from %s at PROT %d\n",inet_ntop(AF_INET ,&cli_addr.sin_addr ,str, sizeof(str)), ntohs(cli_addr.sin_port));

                tmp.events = EPOLLIN;
                tmp.data.fd = cfd;
                ret_epoll_ctl = epoll_ctl(ret_epoll_create ,EPOLL_CTL_ADD ,cfd ,&tmp);
                if(ret_epoll_ctl < 0){
                    perror("add cfd error");
                    exit(1);
                }
            }
            else {
              sockfd = mul[i].data.fd;
              n = read(sockfd , buf , sizeof(buf)) ;    
                if(n == 0){
                    printf("client already closed");
                    // 记得从树中去除链接
                    ret_del = epoll_ctl(ret_epoll_create, EPOLL_CTL_DEL, sockfd, NULL);
                    close(sockfd);
                    exit(1);
                }
                else if( n < 0 ){
                    perror("read error");
                    exit(1);
                }
                else if(n > 0){
                    for(j = 0 ;j < n; j++){
                        buf[j] = toupper(buf[j]);
                    }
                    write(STDOUT_FILENO ,buf , n);
                    write(sockfd , buf ,n);
                }
            }
        }
    }
        close(lfd);
        close(cfd);
        return 0;
}
```

（ET）边沿触发： 0 ->1 ， 1 -> 0 电频发生变化（检测发生变化就执行，一瞬间的事）

（LT）水平触发：1 -> 1 （直到发生变化停止）（**默认的触发方式**）

通过这两种方法可以在哪些地方提高程序效率：

当客户端发送过来1000kb的数据，如果客户端只读取了500kb，这个时候epoll是触发还是不触发？

**练习：**

通过epoll监听管道，检验是水平触发还是边沿触发

结论是：边沿触发？ 不是。。。 应该是水平触发

```c
/*************************************************************************
	> File Name: test2.c
	> Author: 
	> Mail: 
	> Created Time: 2019年08月11日 星期日 18时52分16秒
 ************************************************************************/

#include<stdio.h>
#include<stdlib.h>
#include<string.h>
#include<unistd.h>
#include<fcntl.h>
#include<sys/types.h>
#include<sys/epoll.h>

# define MAXLEN 10

int main(){

    pid_t ret;
    int fd[2],ret_pipe ,i ,num, len;
    int ret_epoll;
    char ch = 'a';
    char str[MAXLEN],buf[MAXLEN];

    ret_pipe = pipe(fd);
    if(ret_pipe < 0){
        perror("create pipe error");
        exit(1);
    }

    ret = fork();// 0 -> read ,1 ->write
    if(ret < 0){
        perror("fork error");
        exit(1);
    }

    if(ret == 0){

        close(fd[0]);
        while(1){
        for(i = 0 ; i < MAXLEN/2 ; i++ ){
            str[i] = ch;
        }
        str[MAXLEN/2] = '\n';
        ch++;
        for(; i <MAXLEN; i++ ){
            str[i] = ch;
        }
        write(fd[1] , str , MAXLEN);
        sleep(3);
    }
        close(fd[1]);
    }

    else if(ret > 0){
        close(fd[1]);
     struct epoll_event event;
     struct epoll_event mul[20];

    ret_epoll = epoll_create(10);

    event.events = EPOLLIN;
    event.data.fd = fd[0];
    epoll_ctl(ret_epoll , EPOLL_CTL_ADD ,fd[0], &event);

        while(1){
            num = epoll_wait(ret_epoll ,mul ,10 ,-1 );
            printf("%d---\n",num);
            if(mul[0].data.fd == fd[0]){
                len = read(fd[0] , buf , MAXLEN/2);
                printf("%d!!!!!\n",len);
                write(STDOUT_FILENO ,buf, len);
            }
        }

    close(fd[1]);
    }
   return 0;
}
```

强制改成边沿触发：

一直等到客户端再写数据才会触发，缺点是数据会越攒越多

```c
/*************************************************************************
	> File Name: test2.c
	> Author: 
	> Mail: 
	> Created Time: 2019年08月11日 星期日 18时52分16秒
 ************************************************************************/

#include<stdio.h>
#include<stdlib.h>
#include<string.h>
#include<unistd.h>
#include<fcntl.h>
#include<sys/types.h>
#include<sys/epoll.h>

# define MAXLEN 10

int main(){

    pid_t ret;
    int fd[2],ret_pipe ,i ,num, len;
    int ret_epoll;
    char ch = 'a';
    char str[MAXLEN],buf[MAXLEN];

    ret_pipe = pipe(fd);
    if(ret_pipe < 0){
        perror("create pipe error");
        exit(1);
    }

    ret = fork();// 0 -> read ,1 ->write
    if(ret < 0){
        perror("fork error");
        exit(1);
    }

    if(ret == 0){
        close(fd[0]);
        while(1){
        for(i = 0 ; i < MAXLEN/2 ; i++ ){
            str[i] = ch;
        }
        str[MAXLEN/2] = '\n';
        ch++;
        for(; i <MAXLEN; i++ ){
            str[i] = ch;
        }
        write(fd[1] , str , MAXLEN);
        sleep(3);
    }
        close(fd[1]);
    }

    else if(ret > 0){
        close(fd[1]);
     struct epoll_event event;
     struct epoll_event mul[20];

    ret_epoll = epoll_create(10);

    event.events = EPOLLIN|EPOLLET;
    event.data.fd = fd[0];
    epoll_ctl(ret_epoll , EPOLL_CTL_ADD ,fd[0], &event);

        while(1){
            num = epoll_wait(ret_epoll ,mul ,10 ,-1 );
            printf("%d---\n",num);
            if(mul[0].data.fd == fd[0]){
                len = read(fd[0] , buf , MAXLEN/2);
                printf("%d!!!!!\n",len);
                write(STDOUT_FILENO ,buf, len);
            }
        }

    close(fd[1]);
    }
   return 0;
}
```

这两种方法的辨别方式：

判断时间，当向屏幕写数据的时候，判断写的时间，如果是每隔3s就是边缘触发；

**边沿触发：**

可以通过读取一部分，判断剩余的还有没有必要继续读

设置成非阻塞的方法：

1. fcntl ，修改打开文件的属性
2. 通过函数参数进行调整

```
flag = fcntl(connfd ,F_GETFL);
flag |= O_NONBLOCK;
fcntl(connfd ,F_SETFL ,flag);// 修改connfd为非阻塞得方法
```



最好用的方法：epoll的边沿式触发，减少epoll函数的调用次数，并且实现非阻塞IO

**解释：**

如果是水平触发，会大大增加epoll_wait 函数的调用次数；但是如果是边沿触发，有可能会导致数据堵塞；所以通过对边沿触发进行改造，将客户端的读事件修改成非阻塞，既可以实现对epoll_wait 函数的调用次数，也可以防止数据堵塞。

```c
#include <stdio.h>
#include <string.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <sys/wait.h>
#include <sys/types.h>
#include <sys/epoll.h>
#include <unistd.h>
#include <fcntl.h>

#define MAXLINE 10
#define SERV_PORT 8000

int main(void)
{
    struct sockaddr_in servaddr, cliaddr;
    socklen_t cliaddr_len;
    int listenfd, connfd;
    char buf[MAXLINE];
    char str[INET_ADDRSTRLEN];
    int efd, flag;

    listenfd = socket(AF_INET, SOCK_STREAM, 0);

    bzero(&servaddr, sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
    servaddr.sin_port = htons(SERV_PORT);

    bind(listenfd, (struct sockaddr *)&servaddr, sizeof(servaddr));

    listen(listenfd, 20);

    ///////////////////////////////////////////////////////////////////////
    struct epoll_event event;
    struct epoll_event resevent[10];
    int res, len;

    efd = epoll_create(10);

    event.events = EPOLLIN | EPOLLET;     /* ET 边沿触发，默认是水平触发 */

    //event.events = EPOLLIN;
    printf("Accepting connections ...\n");
    cliaddr_len = sizeof(cliaddr);
    connfd = accept(listenfd, (struct sockaddr *)&cliaddr, &cliaddr_len);
    printf("received from %s at PORT %d\n",
            inet_ntop(AF_INET, &cliaddr.sin_addr, str, sizeof(str)),
            ntohs(cliaddr.sin_port));

    flag = fcntl(connfd, F_GETFL);          /* 修改connfd为非阻塞读 */
    flag |= O_NONBLOCK;
    fcntl(connfd, F_SETFL, flag);

    event.data.fd = connfd;
    epoll_ctl(efd, EPOLL_CTL_ADD, connfd, &event);      //将connfd加入监听红黑树
    while (1) {
        printf("epoll_wait begin\n");
        res = epoll_wait(efd, resevent, 10, -1);        //最多10个, 阻塞监听
        printf("epoll_wait end res %d\n", res);

        if (resevent[0].data.fd == connfd) {
            while ((len = read(connfd, buf, MAXLINE/2)) >0 )    //非阻塞读, 轮询
                write(STDOUT_FILENO, buf, len);
        }
    }
    return 0;
}
```

**epoll 反应堆模型（libevent 核心思想实现）**

优点 ： （反应堆，形容词？）

1. 跨平台
2. 代码量小，实现的功能多 （核心实现：epoll ，反应堆）                                                                                                                                                                        

**epoll反应堆模型：**                                                         （滑动窗口机制）→↓

1. epoll - 服务器 -监听 -cfd -可读 -epoll 返回 - read - **读取完之后，从树上删除cfd，重新设置监听cfd，并且把原来的可读修改成是否可写** - 小写转大写 - 等待epoll_wait 返回  - 回写客户端 - **写完之后，从树上删除cfd ，重新设置监听cfd的读时间事件 - 继续监听

   多了判断 可写 与 可读 ， 考虑到滑动窗口会不会填满，所以可以先判断一下。

2.  ```c
   evt[i].events = EPOLLIN; evt[i].data.fd = cfd;
    ```

   这里的data不再传递cfd，而是结构体中的泛型指针储存一个结构体

   ```c
   void *ptr;
   
   typedef struct{
   
   int fd;
   
   void (*func)(void *arg);
   
   void *arg;
   }
   ```



**代码实现：**

具体思路：

为什么要使用epoll反应堆模型？

1. 对于同一个socket而言，完成手法至少需要占用两个树上的位置，但是交替的话，只需要一个
2. 为什么要可读和可写轮回交替？保证，一问一答

**（sb老师不说清楚）**

![epoll反应堆思路](C:\Users\acm506\Desktop\epoll反应堆思路.png)

https://blog.csdn.net/yyxyong/article/details/62894056 

```c
/*************************************************************************
	> File Name: epoll.c
	> Author: 
	> Mail: 
	> Created Time: 2019年08月12日 星期一 19时58分35秒
 ************************************************************************/

#include<stdio.h>
#include<stdlib.h>
#include<string.h>
#include<sys/types.h>
#include<sys/inet.h>
#include<unistd.h>

# define PORT 8556
# define MAX_EPOLL 100
# define MAX_LEN 1000
int global_epoll;
int lfd ;

struct my_event{
    int fd;                                        // 文件描述符
    int events;                                    // 代表的权限
    void *arg;                                     // 泛型参数
    void *call_back(int fd ,int events ,void *arg);// 回调函数
    int status;                                    // 是否在红黑树上 
    char buf[MAX_LEN];                             // 缓冲区字符串
    int len;                                       // 字符串长度
    long last_active;                              // 最近一次活动时间
};

struct my_events mul[MAX_EPOLL + 1];

void event_init(struct my_events *tmp ,int fd ,void (*call_back)(int,int, void *),void *arg ){
    tmp->fd = fd;
    tmp->events = event;
    tmp->arg = arg;
    tmp->status = 0;
    tmp->last_active = time(NULL);
    return ;
}

void event_add(int fd ,int event ,struct my_events *ev){ // 向树根中加入新节点

    struct epoll_event tmp = {0 ,{0}};
    int op;

    tmp->events = event;
    tmp->arg = ev;

    if(ev->status == 0){
     op = EPOLL_CTL_ADD;
     ev->status = 1;
    }
    else op = EPOLL_CTL_MOD;

    int ret = epoll_ctl(fd  , op , ev->fd , &tmp);
    if(ret < 0){
    perror("epoll ctl error");
    exit(1);
    }
    printf("event add right");
   return ;
}
void event_del(int fd ,struct my_events *ev){
    
    struct epoll_event epv = {0,{0}};
    if(ev->status == 0){
        return ;
    }

    epv.data.ptr = ev;
    ev.status = 0;
    epoll_ctl(global_epoll ,EPOLL_CTL_DEL , ev->fd , &epv );

    return ;

}
void accept_con(){

}
void read_cal(){

}
void write_call(){

}

void init_epoll(){

    // 创建套接字
    lfd = socket(AF_INET,SOCK_STREAM , NULL);
    if(lfd < 0 ){
        perror("create socker error");
        exit(1);
    }

    struct sockaddr_in tmp;
    int len ;
    memset(&tmp , 0 ,sizeof(tmp));
    tmp.sin_family = AF_INET;
    tmp.sin_addr.s_addr = htonl(INADDR_ANY);
    tmp.sin_port = htons(PORt);
    len = sizeof(tmp);

    int ret = bind(lfd ,(struct sockaddr *)*tmp , len );
    if(tmp < 0 ){
        perror("bind error");
        exit(1);
    }

    ret = listen(lfd , MAX_EPOLL);
    if(ret < 0){
        perror("listen error");
        exit(1);
    }

    // 赋值为非堵塞
    int flag;
    flag = fcntl(lfd , F_GETFL);
    flag |= O_NONBLOCK;
    fcntl(lfd , F_SETFL ,flag);  
    
    // 初始化结构体
    event_init(&mul[MAX_EPOLL] , lfd , accept_con , &mul[MAX_EPOLL]);
    event_add(lfd,EPOLL_IN , &mul[MAX_EXPOLL]);
    
    return ;
}
int main(){
    
    global_epoll = epoll_create(MAX_EPOLL);
    if(global_epoll < 0){
        perror("create epoll error");
        exit(1);
    }

    init_epoll();
    
    struct epoll_event events[MAX_EPOLL + 1];
    
    int i;
    while(1){
        

    int nfd = epoll_wait(global_epoll , events , MAX_EPOLL , 1000);
        if(nfd < 0){
            perror("epoll wait error");
            exit(1);
        }

        for ( i = 0 ; i < nfd ;i++ ){
            struct *my_event tmp = (struct my_events *)events[i].ptr;
            if((events[i].events & EPOLLIN) &&(tmp->events & EPOLLIN))
            tmp->call_back(tmp->fd , events[i].event , tmp->arg);
            if((events[i].events & EPOLLOUT)&&(tmp->events & EPOLLOUT))
            tmp->call_back(tmp->fd , events[i].event . tmp->arg);
        }
    }
    return 0;
}
```

