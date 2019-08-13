---
typora-copy-images-to: ./
---

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

好tm浪费时间。。

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
#include<unistd.h>
#include<arpa/inet.h>
#include<time.h>
#include<sys/epoll.h>
#include<errno.h>
#include<fcntl.h>

# define PORT 8556
# define MAX_EPOLL 100
# define MAX_LEN 1000
int global_epoll;
int lfd ;

struct my_event{
    int fd;                                        // 文件描述符
    int events;                                    // 代表的权限
    void *arg;                                     // 泛型参数
    void (*call_back)(int fd ,int events ,void *arg);// 回调函数
    int status;                                    // 是否在红黑树上 
    char buf[MAX_LEN];                             // 缓冲区字符串
    int len;                                       // 字符串长度
    long last_active;                              // 最近一次活动时间
};

struct my_event mul[MAX_EPOLL + 1];

void read_cal(int fd ,int events , void *arg );
void write_cal(int fd ,int events ,void *arg );

void event_init(struct my_event *tmp ,int fd ,void (*call_back)(int,int, void *),void *arg ){
    tmp->fd = fd;
    tmp->events = 0;
    tmp->call_back = call_back;
    tmp->status = 0;
    tmp->last_active = time(NULL);
    tmp->arg = arg;
    return ;
}

void event_add(int fd ,int event ,struct my_event *ev){ // 向树根中加入新节点

    struct epoll_event tmp = {0 ,{0}};
    int op = 0;

    tmp.events =ev->events =  event;
    tmp.data.ptr = ev;
                                                       
    if(ev->status == 0){
     op = EPOLL_CTL_ADD;
     ev->status = 1;
    }
    //else op = EPOLL_CTL_MOD;

    int ret = epoll_ctl(fd  , op , ev->fd , &tmp);
    if(ret < 0){
        perror("epoll ctl111111111111111111111111 error");
    exit(1);
    }
    printf("event add right");
   return ;
}
void event_del(int fd ,struct my_event *ev){
    
    struct epoll_event epv = {0,{0}};
    if(ev->status == 0){
        return ;
    }

    epv.data.ptr = NULL;
    ev->status = 0;

    epoll_ctl(global_epoll ,EPOLL_CTL_DEL , ev->fd , &epv );

    return ;

}
void accept_con(int lfd ,int events ,void  *arg){
    struct sockaddr_in cin;
    socklen_t len = sizeof(cin);
    int cfd , i;
    
    if((cfd = accept(lfd , (struct sockaddr *)&cin , &len) < 0)){
        if(errno != EAGAIN && errno != EINTR){

        }
        printf("%s:accept ,%s\n",__func__, strerror(errno));
        return ;
    }

    do{
        for( i = 0 ; i < MAX_EPOLL; i++ ){
            if(mul[i].status == 0){
                break;
            }
        }

        if( i==  MAX_EPOLL){
            printf("full hahhaha\n");
            break;
        }

        int flag = 1 ;
        if((flag == fcntl(cfd ,F_SETFL ,O_NONBLOCK)) < 0){
            perror("fcntl error");
            exit(1);
        }

        event_init(&mul[i] , cfd , read_cal , &mul[i]);
        event_add(global_epoll , EPOLLIN , &mul[i]);
     }while(0);

    printf("new conncet %s %d time:%ld pos %d\n", inet_ntoa(cin.sin_addr),
          ntohs(cin.sin_port) , mul[i].last_active ,i);
        return ;
}
        
void read_cal(int fd ,int events ,void *arg){

    struct my_event *tmp = (struct my_event *)arg;
    int len ;
    
    len = read(tmp->fd , tmp->buf , MAX_LEN);
    event_del(global_epoll , tmp);
    
    if(len > 0){
        tmp->len = len ;
        tmp->buf[ len ] ='\0';
        printf("receive %d --%s\n",len ,tmp->buf);

        event_init( tmp , fd , write_cal , tmp );
        event_add( global_epoll ,EPOLLOUT ,tmp );
    }
    else if(len == 0){
        close(tmp->fd);
        printf("closed");
    }
    else {
        close(tmp->fd);
        printf("revdata error");
    }
    return ;
}
void write_cal(int fd ,int events ,void *arg){

    struct my_event *tmp = (struct my_event *)arg;
    int len ;
    len = send(fd , tmp->buf , tmp->len , 0);
    event_del(global_epoll , tmp);

    if(len > 0){
    printf("send fd = %d  %s\n", fd, tmp->buf );
        event_init(tmp , fd , read_cal ,tmp);
        event_add(global_epoll , EPOLLIN , tmp);
    }
    else {
        close(tmp->fd);
        printf(" send error");
    }
    return ;
}

void init_epoll(){

    // 创建套接字
    lfd = socket(AF_INET,SOCK_STREAM , 0);
    int flag;
    flag = fcntl(lfd , F_GETFL);
    flag |= O_NONBLOCK;
    fcntl(lfd , F_SETFL ,flag);  
    if(lfd < 0 ){
        perror("create socker error");
        exit(1);
    }

    struct sockaddr_in tmp;
    int len ;
    memset(&tmp , 0 ,sizeof(tmp));
    tmp.sin_family = AF_INET;
    tmp.sin_addr.s_addr = htonl(INADDR_ANY);
    tmp.sin_port = htons(PORT);
    len = sizeof(tmp);

    int ret = bind(lfd ,(struct sockaddr *)&tmp , len );
    if(ret < 0 ){
        perror("bind error");
        exit(1);
    }

    ret = listen(lfd , MAX_EPOLL);
    if(ret < 0){
        perror("listen error");
        exit(1);
    }
    
    // 初始化结构体
    event_init(&mul[MAX_EPOLL] , lfd , accept_con , &mul[MAX_EPOLL]);
    event_add(global_epoll , EPOLLIN , &mul[MAX_EPOLL]);
    
    return ;
}
int main(int argc ,char *argv[]){
    
    global_epoll = epoll_create(MAX_EPOLL);
    if(global_epoll < 0){
        perror("create epoll error");
        exit(1);
    }

    init_epoll();

    struct epoll_event events[MAX_EPOLL + 1];
    
    int i , checkpos = 0;
    while(1){
        long now = time(NULL);
        for( i = 0 ; i < 100 ;i++ , checkpos++ ){
            if(checkpos == MAX_EPOLL)checkpos = 0;
            if(mul[checkpos].status != 1)continue;

            long del  = now - mul[checkpos].last_active;

            if(del >= 60){

                close(mul[checkpos].fd);
                printf("fd = %d timeout \n" ,mul[checkpos].fd);
                event_del(global_epoll , &mul[checkpos]);
            }
        }
    int nfd = epoll_wait(global_epoll , events , MAX_EPOLL , 1000);
        if(nfd < 0){
            perror("epoll wait error");
            exit(1);
        }

        for ( i = 0 ; i < nfd ;i++ ){
            struct my_event *tmp = (struct my_event *)events[i].data.ptr;
            if((events[i].events & EPOLLIN) &&(tmp->events & EPOLLIN))
            tmp->call_back(tmp->fd , events[i].events , tmp->arg);
            if((events[i].events & EPOLLOUT)&&(tmp->events & EPOLLOUT))
            tmp->call_back(tmp->fd , events[i].events , tmp->arg);
        }
    }
    return 0;
}
```

**心跳包**

作用：

探测客户端和服务器之间是否仍保持链接（类似于应用层的协议）

**乒乓包**

作用：

和心跳包机制差不多，在判别网络是否通顺时，还可以发送少量信息，起到一个信号的作用？

（还是属于应用层的协议）

**TCP自带的属性当中也可以判断链接是否正常**

SO_KEEPALIVE 宏 

如果两小时内无数据传递，TCP发送探测分节给对方

1. 对方正常接收
2. 对方回复一个ACK，对方已崩溃并且重新启动
3. 无回应，再每隔75s发送一个探测分节，一共8个。发送完之后仍无相应就放弃

```c
keepAlive = 1;
setsockopt(lfd ,SOL_SOCKET ,SO_KEEPALIVE ,(void *)keepAlive ,sizeof(keepalive));
```





#### 线程池

![线程池实现思路](C:\Users\acm506\Desktop\Linux\Linux-Learning-Network\线程池实现思路.png)

具体思路：

/*
1. 创建线程池
    1）初始化  （对结构体内容的锁，正在运行的线程个数的锁 ） -> 互斥锁     管家程序访问
                      （判断循环队列是否满的锁 ，通知线程来领任务的锁）  -> 条件变量   可能有多个线程同时询问
                     （线程池中每个线程的id ， 对应的参数）
                     （管理进程id，任务队列）   
    
    ​                （线程池中最大的个数。线程池中最小的个数，当前线程池总的线程个数，正在运行的线程个数，要销毁的线程池个数）
    ​              （任务队列，队首，队尾，实际的任务数，总的能够承受的任务数）
    ​                （整个线程池的开关）
    ​         （初始化互斥量，条件变量）
    
    
    
2. 

    2）申请空间      （子线程信息申请空间）
                       （任务队列申请空间）

    3）启动线程      （子线程最少的申请）
                       （管理线程申请空间）

3. 模拟客户端向服务器中发送信息              
      1）获取队列没有满的锁
      2）向锁中写入数据
      3）发送通知线程来领取任务的信号
      4）把队列没有满的锁的权限放开

4. 管家线程调节线程池去处理任务

      1）首先接受pthread_full信号，这一部分是要被清除掉的

      2）从循环队列中取出一个任务

      3）从线程池中取出一个线程来解决这个任务

5. 管家线程调整线程池中各类型个数

     1）获取线程各部分个数信息的锁
     2）读取当前循环队列中任务的个数
     3）返回对循环队列的信息的锁
     4）获取对当前线程池中正在运行的线程个数的锁
     5）读取当前线程池中正在运行的线程个数
     6）返回对当前线程池中正在运行的线程个数的锁
     7）判断获取的两个数的大小关系，如果不够的话，增加线程个数
     8）线程处理任务

6. 小功能
    1）释放所有的锁
    2）判断某一个线程是否存活
    */

```c
#include<stdlib.h>
#include<stdio.h>
#include<unistd.h>
#include<pthread.h>
#include<string.h>
#include<windows.h>
#include<pthread.h>

# define TOT_PTHREAD 15
# define TOT_QUEUE 11
# define DEFAULT_TIME 10
# define MIN_WAIT_TASK_NUM 10
# define DEFAULT_THREAD_VARY 10
struct thread_task{
void *(*cal)(void *);
void *arg;
};
struct Message{
pthread_mutex_t tot_lock;
pthread_mutex_t num_busy_lock;
pthread_cond_t queue_full;
pthread_cond_t queue_to_pthread;

pthread_t *threads;
pthread_t  con_id;
 thread_task  *task_queue;

int max_pthread;
int min_pthread;
int tot_pthread;
int busy_pthread;
int del_pthread;

int queue_top;
int queue_end;
int queue_act;
int queue_tot;

int shut;
};

void pthread_add(struct Message *pool , void *( *cal)(void *arg) ,void *arg );
void *con_thread(void *pthread);
void *pthread_pool(void *pthread);
bool is_thread_alive(pthread_t pid);
void threadpool_destroy(struct Message *tmp);

struct Message *pthread_pool_init(int max_pthread ,int min_pthread ,int busy_pthread){

struct Message *tmp;
tmp->busy_pthread = busy_pthread;
tmp->max_pthread = max_pthread;
tmp->min_pthread = min_pthread;
tmp->tot_pthread = TOT_PTHREAD;
tmp->del_pthread = 0;
tmp->queue_top = 0;
tmp->queue_end = 0;
tmp->queue_act = 0;
tmp->queue_tot  = TOT_QUEUE;
tmp->shut = 0;

tmp->threads = (pthread_t *)malloc(sizeof(pthread_t) * TOT_PTHREAD);
if(tmp->threads == NULL){
    printf("init thread error");
    exit(1);
}
memset(tmp->threads , 0, sizeof(tmp->threads));

//tmp->task_queue = (struct thread_task )malloc(sizeof(struct thread_task ) * TOT_QUEUE);
tmp->task_queue = new thread_task[TOT_QUEUE + 1];
if(tmp->task_queue == NULL){
    printf("init task queue error");
    exit(1);
}

if(pthread_mutex_init(&(tmp->tot_lock), NULL) != 0 || pthread_mutex_init(&(tmp->num_busy_lock) ,NULL) != 0||
pthread_cond_init(&(tmp->queue_full ), NULL) != 0 || pthread_cond_init(&(tmp->queue_to_pthread) ,NULL) != 0){
    printf("pthread mutex or cond init error");
    exit(1);
}

for(int i = 0 ; i < min_pthread ; i++ )
{
    pthread_create(&(tmp->threads[i]) ,NULL ,pthread_pool , (void *)tmp );
    printf("now !! xiangge start %d hahahah\n", tmp->threads[i]);
}

pthread_create(&(tmp->con_id) , NULL , con_thread, (void *)tmp);

return tmp;
}

void pthread_add(struct Message *pool , void *( *cal)(void *arg) ,void *arg ){

  pthread_mutex_lock( &(pool->tot_lock));

   while((pool->max_pthread == pool->queue_tot && (! pool->shut))){
    pthread_cond_wait( &(pool->queue_full) , &(pool->tot_lock));
   }

   if(pool->shut == 0){
    exit(1);
   }

   if(pool->task_queue[pool->queue_end].arg != NULL  || pool->task_queue[pool->queue_end].cal != NULL){

    pool->task_queue[pool->queue_end ].arg = NULL;
    pool->task_queue[pool->queue_end].arg = NULL ;
   }

   pool->task_queue[pool->queue_end ].cal = cal;
   pool->task_queue[pool->queue_end].arg = arg;
   pool->queue_end = (pool->queue_end + 1) % TOT_QUEUE;
   pool->queue_act ++ ;

   pthread_cond_signal( &(pool->queue_to_pthread));
   pthread_mutex_unlock( &(pool->tot_lock));

  return ;
}

void *con_thread(void *pthread){  ///  管理者控制线程各类的个数
struct Message *tmp = (struct Message *)pthread;
int tot_pthread , busy_pthread , queue_size;
while(tmp->shut == 0 ){

    pthread_mutex_lock(&(tmp->tot_lock));
    tot_pthread = tmp->min_pthread;
    queue_size  = tmp->queue_act;
    pthread_mutex_unlock(&(tmp->tot_lock));

    pthread_mutex_lock(&(tmp->num_busy_lock));
    busy_pthread = tmp->busy_pthread;
    pthread_mutex_lock(&(tmp->num_busy_lock));

    if(queue_size >= MIN_WAIT_TASK_NUM && tot_pthread < tmp->max_pthread ){
        pthread_mutex_lock(&(tmp->tot_lock));
        int add  = 0 ,i ;

        for( i = 0 ; i < tmp->max_pthread &&add < DEFAULT_THREAD_VARY & tmp->tot_pthread
        < tmp->max_pthread ; i++ ){
            if( tmp->threads[i] == 0 || !is_thread_alive( tmp->threads[i])){
                pthread_create(&(tmp->threads[i]) , NULL , pthread_pool , (void *)tmp);
            }
        }
          pthread_mutex_unlock(&(tmp->tot_lock));
    }

    if(busy_pthread * 2 < tot_pthread && tot_pthread >tmp->min_pthread ){

        pthread_mutex_lock(&(tmp->tot_lock));
        tmp->del_pthread = DEFAULT_THREAD_VARY;
        pthread_mutex_unlock(&(tmp->tot_lock));

        for(int i = 0 ; i < tmp->del_pthread ; i++ ){
            pthread_cond_signal(&(tmp->queue_full));
            /// 当这个进程为多余的时候，会安排发送queue_full 信号，使得当前的线程注销掉
        }
    }
}
return NULL;
}
void *pthread_pool(void *pthread){  ///  线程从服务器队列中寻找信息去处理
struct Message *tmp = (struct Message *)pthread;
struct thread_task task;
while(1){
    pthread_mutex_lock(&(tmp->tot_lock));
    while((tmp->queue_tot == 0) &&(!tmp->shut)){
        pthread_cond_wait(&(tmp->queue_full) , &(tmp->tot_lock));

        if(tmp->del_pthread > 0){
            tmp->del_pthread --;
        }

        if(tmp->tot_pthread > tmp->min_pthread ){
            printf("thread 0x%x is exiting\n", (unsigned int)pthread_self());
            tmp->tot_pthread--;
            pthread_mutex_unlock(&(tmp->tot_lock));
            pthread_exit(NULL);
        }
    }

    if( tmp->shut){
        pthread_mutex_unlock(&(tmp->tot_lock));
        printf("thread 0x%x is exiting \n",(unsigned int) pthread_self());
        pthread_exit(NULL);
    }
    task.cal = tmp->task_queue[tmp->queue_top].cal;
    task.arg = tmp->task_queue[tmp->queue_top].arg;

    tmp->queue_top = (tmp->queue_top + 1) %tmp->queue_tot;
    tmp->queue_act --;

    pthread_cond_broadcast(&(tmp->queue_full));

    pthread_mutex_unlock(&(tmp->tot_lock));

    printf("thread 0x %x start working\n",(unsigned int)pthread_self());
    pthread_mutex_lock(&(tmp->num_busy_lock));
    tmp->busy_pthread ++ ;
    pthread_mutex_unlock(&(tmp->num_busy_lock));
    (*(task.cal))(task.arg);

    printf("thread 0x %x finish working\n",(unsigned int)pthread_self());
    pthread_mutex_lock(&(tmp->num_busy_lock));
    tmp->busy_pthread --;
    pthread_mutex_unlock(&(tmp->num_busy_lock));
}
 pthread_exit(NULL);
return NULL;
}

bool is_thread_alive(pthread_t pid){
int kill_rc = pthread_kill(pid , 0);
if(kill_rc == ESRCH){
    return false;
}
return true;
}
void *cal(void *arg){
printf("thread %x working on task %d\n" ,(unsigned int)pthread_self() ,*(int *)arg);
Sleep(10);
printf("task %d is end \n", *(int *)arg);
return NULL;
}

void threadpool_destroy(struct Message *tmp){
if(tmp == NULL)return ;

tmp->shut = 1;
pthread_join(tmp->con_id , NULL);
for(int i = 0; i < tmp->max_pthread ; i++){
    pthread_cond_broadcast(&(tmp->queue_full));
}
for(int i = 0 ; i< tmp->max_pthread ; i++ ){
    pthread_join(tmp->threads[i] , NULL);
}
   return ;
}
int main(){

  struct Message *pool = pthread_pool_init(10 ,5 ,3);

  int times[20 + 1];
  for(int i = 0 ; i < 20 ; i++ ){
    times[i] = i ;
    pthread_add( pool , cal , (void *)&times[i]);
  }
  threadpool_destroy(pool);
return 0;
}

```

