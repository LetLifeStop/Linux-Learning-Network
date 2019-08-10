### 高并发服务器

**重要的三个状态：**

ESTABLISHED 

FIN_WAIT_2  主动发起关闭请求的一端已经完成半关闭状态

TIME_WAIT 主动发起关闭请求的一端，等待2MSL（1分钟左右），机制没有收到ACK的话，就直接退出，第四次握手的时候。（**再一次调用的时候注意端口是否调用**）

​	 MSTL（最大段生存时间）不同的版本时间不同

RST 命令  三次握手失败，重新建立关系

为什么主动关闭一端要设计TIME_WAIT状态？

答：确保最后一个发送的ACK（主动关闭连接一端）能顺利到达



**shutdown函数**	

如果有多个区域指向socket所建立的缓冲区，只是close并不能完全处理。

```c
int shutdown(int sockfd , int how);
```

how：

SHUT_RD( 0 );

SHUT_WR( 1 );

SHUT_RDWR( 2 )；

shutdown是把所有的链接都给你整关上了。

ps：

1. 如果有多个进程共享一个套接字，close每被调用一次，计数 -1 ，减为0 时，套接字将被释放
2. 在多个进程中如果一个进程调用了shutdown，其他的进程将无法通信。但是如果一个进程close将不会影响到其他进程



#### 端口复用

```c
int setsockopt(int sockfd , int level ,int optname, const void *optval ,socklen_t optlen);
```

作用：

设置套接字的选项

```
int  opt = 1;
setsockopt(listenfd ,SOL_SOCKET,SO_REUSEADDR ,&opt ,sizeof(opt));
```

在绑定之前设置

作用：

允许重用本地地址

就不会因为TIME_WAIT 而导致无法重新使用端口。



#### 多路I/O转接服务器

**核心思想：**

不再由应用程序来监护客户端，而是使用内核来监护客户端。不需要再阻塞等待链接请求，而是由内核来替代阻塞等待的过程，服务器段只是等待内核发来请求。

**select函数**

```c
int select(int nfds ,fd_set *readfds, fd_set *writefds ,fd_set *exceptfds ,struct timeval *timeout);
```

参数1：

所监听的所有文件描述符中，最大的文件描述符 + 1 .

参数2：

读，监听文件描述符是否可读

参数3：

写，监听文件描述符是否可写

参数4：

异常，监听文件描述符是否异常

**2,3,4参数是一个位图（二进制），并且全部都是传入传出函数**，建立在已经链接好了的基础之上。

参数5：

设定等待多长时间，若没有满足条件的文件描述符，内核就发信号就返回

返回值：

所监听的所有监听集合中，满足条件的总数，即使有重复的，也会计算。假设3号文件描述符三个都满足，那么返回值是3 。 出错返回 -1 

```c
void FD_ZERO(int fd ,fd_set *set); 将set清空
int FD_ISSET(int fd ,fd_set *set);  哦按段fd是否在集合中
int FD_CLR(int fd ,fd_set *set);  将fd从set中请空
void FD_SET(int fd ,fd_set *set); 将fd设置到set集合中
```



select函数弊端：

1. 文件描述符上限为1024。所以监听文件描述符上限为1024
2. 判断文件描述符是否在集合中的时候，是遍历一遍来确认的，可能会浪费时间，可以通过设置监听数组，来进行遍历
3. 监听集合 和 满足监听条件的集合 是一个集合，并且是一个传入传出参数，当下一次要使用的时候，也还需要对监听集合重新赋值，可以通过保存之前的状态，来减少代码量。

当服务器端收到客户端建立链接请求的时候，默认为监听读事件****

**select搭建服务器端：**

```c
/*************************************************************************
	> File Name: select.c
	> Author: 
	> Mail: 
	> Created Time: 2019年08月10日 星期六 08时48分42秒
 ************************************************************************/

#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<ctype.h>
#include<sys/types.h>
#include<netinet/in.h>
#include<string.h>
#include<arpa/inet.h>

# define SERV_PORT 8556

int main(int argc ,char *argv[]){
 
    int i,j,n,maxi;

    int nready, client[FD_SETSIZE];
    int maxfd ,listenfd,connfd ,sockfd;
    char str[INET_ADDRSTRLEN] , buf[BUFSIZ];
     
    struct sockaddr_in clie_addr,serv_addr;
    socklen_t clie_addr_len;
    fd_set rset ,allset;

    listenfd = socket(AF_INET ,SOCK_STREAM ,0);

    int opt = 1;
    setsockopt(listenfd ,SOL_SOCKET ,SO_REUSEADDR ,&opt, sizeof(opt));

    bzero(&serv_addr, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    serv_addr.sin_port = htons(SERV_PORT);

    bind(listenfd ,(struct sockaddr *)&serv_addr ,sizeof(serv_addr));
    
    listen(listenfd ,128);

    maxfd = listenfd;
    maxi = -1;
    for(i = 0; i<FD_SETSIZE ;i++){
        client[i] = -1;
    }
    FD_ZERO(&allset);
    FD_SET(listenfd ,&allset);
    
    while(1){
     rset = allset;
    nready = select(maxfd + 1, &rset ,NULL ,NULL ,NULL);
        if(nready < 0){
        perror("select error");
            exit(1);
        }
        if(FD_ISSET(listenfd , &rset)){
            clie_addr_len = sizeof(clie_addr);
            connfd = accept(listenfd, (struct sockaddr *)&clie_addr, &clie_addr_len);
            printf("received from %s at PORT %d\n" ,inet_ntop(AF_INET ,&clie_addr.sin_addr ,str , sizeof(str)) ,ntohs(clie_addr.sin_port));
            for(i = 0;i < FD_SETSIZE ;i++){
                if(client[i] < 0){
                    client[i] = connfd;
                    break;
                }
            }
            if(i == FD_SETSIZE){
                perror("full");
                exit(1);
            }
            FD_SET(connfd ,&allset);
            if(connfd > maxfd){maxfd = connfd;} 
            if(i > maxi) maxi= i;
            if(--nready == 0)continue; // 非常灵魂的地方，可以用来判断是来发送链接请求的还是别的
        }
        
        for(i = 0 ;i <= maxi ;i++ ){
            if((sockfd = client[i]) < 0){
                continue;
            }
            if(FD_ISSET(sockfd ,&rset)){
                if((n = read(sockfd , buf ,sizeof(buf))) == 0 ){
                    close(sockfd);
                    FD_CLR(sockfd ,&allset);
                    client[i] = -1;
                }
                else if( n > 0 ){
                    for( j = 0;j< n; j++ ){
                        buf[j] = toupper(buf[j]);
                    }
                    write(sockfd ,buf , n);
                    write(STDOUT_FILENO , buf ,n);
                }
                if(--nready == 0){break;}
            }
        }
    }
    close(listenfd);
    return 0;

}
```



**poll函数**

```
#include<poll.h>
int opll(struct pollfd *fds ,nfds_t nfds ,int timeout);
struct pollfd{
int fd;
short events;
short revents;
};
```

参数：

1. fds:

数组的首地址

2. nfds:

   数组中元素的个数

3.  -1 ,阻塞等  0 ：立即返回 。 > 0 等待指定毫秒数

POLLIN 读取

POLLOUT 写入

POLLERR  读取错误

**使用注意：**

```
struct pollfd fds[50000];
fds[0].fd = listenfd;
fds[0].events = POLLIN;
。。。。

poll（fds ,5 ,0）;
```



优点：

1. 使用ulimit -a 查看最多能打开多少文件描述符，可以通过修改文件描述符的上限来增加上限

2. 实现了监听集合与返回集合的分离

3. 搜索范围小



cat /proc/sys/fs/file-max 查看一个进程可以打开的文件描述符上限



修改方法：

sudo vi etc/security/limits.conf

在文件尾部写入以下配置，soft 软限制，hard 硬限制

*soft nofile 65536

*hard nofile 100000



使用示例：

```c
/*************************************************************************
	> File Name: select.c
	> Author: 
	> Mail: 
	> Created Time: 2019年08月10日 星期六 08时48分42秒
 ************************************************************************/

#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<ctype.h>
#include<sys/types.h>
#include<netinet/in.h>
#include<string.h>
#include<arpa/inet.h>
#include<poll.h>
#include<errno.h>

# define SERV_PORT 8556
int main(int argc ,char *argv[]){
 
    int i,j,n,maxi;

    int nready, client[FD_SETSIZE];
    int maxfd ,listenfd,connfd ,sockfd;
    char str[INET_ADDRSTRLEN] , buf[BUFSIZ];
    struct pollfd fds[100];
    struct sockaddr_in clie_addr,serv_addr;
    socklen_t clie_addr_len;
    fd_set rset ,allset;

    listenfd = socket(AF_INET ,SOCK_STREAM ,0);

    int opt = 1;
    setsockopt(listenfd ,SOL_SOCKET ,SO_REUSEADDR ,&opt, sizeof(opt));

    bzero(&serv_addr, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    serv_addr.sin_port = htons(SERV_PORT);

    bind(listenfd ,(struct sockaddr *)&serv_addr ,sizeof(serv_addr));
    
    listen(listenfd ,128);
    for ( i = 0 ;i<100 ;i++ )fds[i].fd = -1;

    maxfd = listenfd;
    maxi = 0 ; 
    fds[0].fd = listenfd;
    fds[0].events = POLLIN;

    while(1){
        nready = poll(fds , maxi + 1 , -1);
        if(fds[0].revents & POLLIN){
            clie_addr_len = sizeof(clie_addr);
            connfd = accept(listenfd, (struct sockaddr *)&clie_addr, &clie_addr_len);
            printf("received from %s at PORT %d\n" ,inet_ntop(AF_INET ,&clie_addr.sin_addr ,str , sizeof(str)) ,ntohs(clie_addr.sin_port));
            for(i = 1;i < FOPEN_MAX ;i++){
                if(fds[i].fd < 0){
                fds[i].fd = connfd;
                    break;
                }
            }
            if(i == FD_SETSIZE){
                perror("full");
                exit(1);
            }
            fds[i].events = POLLIN;
            if(connfd > maxfd){maxfd = connfd;} 
            if(i > maxi) maxi= i;
            if(--nready <= 0)continue;
        }
        
        for(i = 1 ;i <= maxi ;i++ ){
            if((sockfd = fds[i].fd) < 0){
                continue;
            }
            if(fds[i].revents & POLLIN ){  // 通过位运算的方法判断

                n = read(sockfd , buf ,sizeof(buf));
                if(n < 0){
                    if(errno == ECONNRESET){
                        printf("been repeat used"); // 判断是否被覆盖
                        close(sockfd);
                        fds[i].fd = -1;
                        exit(1);
                    }
                }
                else if(n == 0 ){
                    close(sockfd);
                    FD_CLR(sockfd ,&allset);
                    fds[i].fd = -1;
                }
                else if( n > 0 ){
                    for( j = 0;j< n; j++ ){
                        buf[j] = toupper(buf[j]);
                    }
                    write(sockfd ,buf , n);
                    write(STDOUT_FILENO , buf ,n);
                }
                if(--nready <= 0){break;}
            }
        }
    }
    close(listenfd);
    return 0;

}
```



























