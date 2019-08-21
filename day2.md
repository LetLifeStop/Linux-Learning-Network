![1565185370771](C:\Users\acm506\AppData\Roaming\Typora\typora-user-images\1565185370771.png)

**arp数据报作用：**

通过IP地址获取mac地址

```c
netstat -apn | grep 6666
```

查看6666的端口号信息

**如果先关server.c 的话，6666这个端口号就被被写死，无法绑定。**

也可以自己封装函数，包括出错机制，调入参数；可以同时加一个 **.h**文件。

MTU： 最大传输单元，受协议设置

mss：受MTU标示一个数据包携带数据的上限数

win：滑动窗口，当前本段，能接受的数据上限值

##### 滑动窗口（每次数据传送都有）

  控制流量，防止数据包发送过快，数据流失。

**错误处理**

read 返回值

1.  \> 0 ,实际读到的字节数 buf = 1024，只有当读到末尾的时候才会结束
2.  = 0，数据读完（截止条件，读到文件末尾，管道写端关闭，socket代表对端关闭（当对端没有关闭，但是并没有写，会阻塞等待））
3.   -1 ， 异常

        1. errno == EINTR ，被信号中断， 拯救方法：重启/退出
           2. errno == EAGAIN(EWOULDBLOCK ) 非阻塞方式读，并且没有数据
           3. 其他值



read函数的封装

**应该读取的字节数 n**

```c
ssize_t Readn(int fd ,void *vptr ,size_t n)
{
size_t nleft;
ssize_t nread;
char *ptr;

pret = vptr;
nleft = n;
while(nleft > 0){
if((nread = read(fd ,prt, nleft)) < 0){
if(errno == EINTR)
nread = 0;
else return -1;
}
else if(nread == 0)break;

nleft -= nread;
ptr += nread;
}
return n - nleft;
}
```

Writen函数的封装

```c
ssize_t Writen(int fd ,const void *vptr,size_t n)
{
size_t nleft;
ssize_t nwritten;
const char *ptr;
ptr = vptr;
nleft = n;
while(nleft > 0){
if( (nwritten = write(fd ,ptr,nleft)) <= 0){
if(nwritten < 0 && errno ==EINTR){
nwriten = 0;
}
else return -1;
}
nleft -= nwritten;
ptr += nwritten;
}
return n;
}
```



my_read的封装

```c
static ssize_t my_read(int fd, char *ptr)
{
	static int read_cnt;
	static char *read_ptr;
	static char read_buf[100];

	if (read_cnt <= 0) {
again:
		if ( (read_cnt = read(fd, read_buf, sizeof(read_buf))) < 0) {   //"hello\n"
			if (errno == EINTR)
				goto again;
			return -1;
		} else if (read_cnt == 0)
			return 0;

		read_ptr = read_buf;
	}
	read_cnt--;
	*ptr = *read_ptr++;

	return 1;
}
```



readline的封装

```c
ssize_t Readline(int fd, void *vptr, size_t maxlen)
{
	ssize_t n, rc;
	char    c, *ptr;
	ptr = vptr;

	for (n = 1; n < maxlen; n++) {
		if ((rc = my_read(fd, &c)) == 1) {   //ptr[] = hello\n
			*ptr++ = c;
			if (c == '\n')
				break;
		} else if (rc == 0) {
			*ptr = 0;
			return n-1;
		} else
			return -1;
	}
	*ptr = 0;

	return n;
}

```





#### TCP协议

**三次握手和四次握手**

**TCP 协议同样会丢包，而是说包丢了之后可以进行重传**

TCP通讯时序

IP：网络层，不稳定。

传出层处理方法：

1. 完全不弥补 --UDP -无连接不可靠报文传输
2. 完全弥补    --TCP --面向连接的可靠数据包传递

**规定SYN位和FIN位也要占一个序号**

![](C:\暑假学习\linux服务器开发三-网络编程\linux服务器开发三-网络编程资料\第二天资料\2-其他资料\TCP三次四次握手.png)

**三次握手**

模拟打电话的过程，喂？喂！嗯！

确定网络环境是否畅通

目的是：建立连接

SYN 1（0）客户端发送1号包，携带0个字节

SYN（2000），ACK（2）服务器端

 ACK （2001）客户端

**四次握手**

双方互相关闭连接

SYN，ACK，FIN标志位

客户端：FIN 2（0）， ACK2001

服务器：ACK 3 （**完成半关闭**）

服务器端：FIN 2001（0），ACK 3

客户端：ACK 2002

**在判断对端是否关闭的时候，read为0是一个判断依据**

**协议上限分析：**

链路层 网络层 传输层 应用层 数据 校验码

如果超过上限的话，会进行拆包。保证能把所有数据按照完整的包的形式传送出去。



##### **多线程实现并发服务器**

BUFSIZ ->8192

INET_ADDRSTRLEN ->16

练习：

服务器端

```c
/*************************************************************************
	> File Name: multpthread.c
	> Author: 
	> Mail: 
	> Created Time: 2019年08月08日 星期四 10时10分10秒
 ************************************************************************/

#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<ctype.h>
#include<sys/types.h>
#include<sys/socket.h>
#include<string.h>
#include<arpa/inet.h>
#include<pthread.h>

# define MAXLINE 123
# define PORT 8556

struct node{
    struct sockaddr_in msg;
    int clicin;
};

void *cal(void *arg){
    struct node *tmp = (struct node*)arg;
    int len ;
    char buf[BUFSIZ];
    char str[INET_ADDRSTRLEN];
    while(1){

    printf("receive client IP = %s, receive client PORT = %d\n",
    inet_ntop(AF_INET ,&(*tmp).msg.sin_addr ,str,sizeof(str)),
    ntohs((*tmp).msg.sin_port));

    len = read(tmp->clicin , buf , sizeof(buf));
    for (int i =0 ; i < len; i++){
        buf[i] = toupper(buf[i]);
    }
    write(STDOUT_FILENO ,buf ,len);
    write(tmp->clicin , buf, len);
    }
    close(tmp->clicin);
    return NULL;
}
int main()
{
    struct sockaddr_in ser_lid,cli_lid;
    int sfd , cfd;
    socklen_t serlen;
    pthread_t pth;
    struct node tid[100 + 1];
    int i = 0;

    sfd = socket(AF_INET , SOCK_STREAM , 0);
    if(sfd == -1){
        perror("socket error");
        exit(1);
    }

    memset(&ser_lid , 0, sizeof(struct sockaddr_in));
   // bzero(&ser_lid,sizeof(ser_lid));
    ser_lid.sin_family = AF_INET;
    ser_lid.sin_addr.s_addr = htonl(INADDR_ANY);
    ser_lid.sin_port = htons(PORT);

    bind(sfd ,(struct sockaddr *)&ser_lid , sizeof(ser_lid));
   
    listen(sfd , 100);
    printf("Waiting client dadadadadada\n");

    while(1){
    serlen = sizeof(cli_lid);
    cfd = accept(sfd ,(struct sockaddr *)&cli_lid , &serlen);
    tid[i].msg = cli_lid;
    tid[i].clicin = cfd;
    printf("11111\n");
    pthread_create(&pth ,NULL, cal ,(void *)&tid[i]);
    pthread_detach(pth);
    i++;
    }
    return 0;
}
```

客户端：

```c
/*************************************************************************
	> File Name: client.c
	> Author: 
	> Mail: 
	> Created Time: 2019年08月08日 星期四 11时16分54秒
 ************************************************************************/

#include<stdio.h>
#include<string.h>
#include<unistd.h>
#include<arpa/inet.h>
#include<stdlib.h>
#include<ctype.h>
#include<sys/types.h>
#include<sys/socket.h>
#include<arpa/inet.h>

# define MAXLINE 135
# define PORT 8556
int main(){

    struct sockaddr_in serv;
    char buf[MAXLINE];
    int sockfd ,n;
    
    sockfd = socket(AF_INET ,SOCK_STREAM ,0);
    bzero(&serv,sizeof(struct sockaddr));
    serv.sin_family = AF_INET;
    inet_pton(AF_INET ,"127.0.1.2",&serv.sin_addr.s_addr);
    serv.sin_port = htons(PORT);
    //  serv.sin_addr.s_addr = htonl("127.0.0.1");

    connect(sockfd,(struct sockaddr *)&serv,sizeof(serv));

    while(fgets(buf,MAXLINE ,stdin) != NULL){
        write(sockfd , buf , sizeof(buf));
        int len = read(sockfd,buf , MAXLINE);
        write(STDOUT_FILENO ,buf, n);
    }
    close(sockfd);
    return 0;
}
```

**多进程实现并发服务器**

服务器端：

```c
/*************************************************************************
	> File Name: multpthread.c
	> Author: 
	> Mail: 
	> Created Time: 2019年08月08日 星期四 10时10分10秒
 ************************************************************************/

#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<ctype.h>
#include<sys/types.h>
#include<sys/socket.h>
#include<string.h>
#include<arpa/inet.h>
#include<pthread.h>

# define MAXLINE 123
# define PORT 8556

struct node{
    struct sockaddr_in msg;
    int clicin;
};

void *cal(void *arg){
    struct node *tmp = (struct node*)arg;
    int len ;
    char buf[BUFSIZ];
    char str[INET_ADDRSTRLEN];
    while(1){

    printf("receive client IP = %s, receive client PORT = %d\n",
    inet_ntop(AF_INET ,&(*tmp).msg.sin_addr ,str,sizeof(str)),
    ntohs((*tmp).msg.sin_port));

    len = read(tmp->clicin , buf , sizeof(buf));
    for (int i =0 ; i < len; i++){
        buf[i] = toupper(buf[i]);
    }
    write(STDOUT_FILENO ,buf ,len);
    write(tmp->clicin , buf, len);
    }
    close(tmp->clicin);
    return NULL;
}
int main()
{
    struct sockaddr_in ser_lid,cli_lid;
    int sfd , cfd;
    socklen_t serlen;
    pthread_t pth;
    struct node tid[100 + 1];
    int i = 0;
    char str[INET_ADDRSTRLEN];
    char buf[BUFSIZ];
    int len;

    sfd = socket(AF_INET , SOCK_STREAM , 0);
    if(sfd == -1){
        perror("socket error");
       exit(1);
    }
    int opt;
    setsockopt(sfd , SOL_SOCKET,SO_REUSEADDR ,&opt,sizeof(opt));
    memset(&ser_lid , 0, sizeof(struct sockaddr_in));
   // bzero(&ser_lid,sizeof(ser_lid));
    ser_lid.sin_family = AF_INET;
    ser_lid.sin_addr.s_addr = htonl(INADDR_ANY);
    ser_lid.sin_port = htons(PORT);

    bind(sfd ,(struct sockaddr *)&ser_lid , sizeof(ser_lid));
   
    listen(sfd , 100);
    printf("Waiting client dadadadadada\n");
     pid_t ret;
    while(1){
    serlen = sizeof(cli_lid);
    cfd = accept(sfd ,(struct sockaddr *)&cli_lid , &serlen);
    tid[i].msg = cli_lid;
    tid[i].clicin = cfd;
    ret = fork();
         if(ret == 0){
          close(sfd);
         printf("receieve IP = %s , and the PORT = %d\n",
               inet_ntop(AF_INET ,&cli_lid.sin_addr , str ,sizeof(str)),
                ntohs(cli_lid.sin_port)
              );
            while(1){
            //  printf("123123123\n");
            len = read(cfd ,buf ,MAXLINE);
             //   printf("---%d\n",len);
            for (int ii = 0 ; ii < len ; ii++){
                buf[ii] = toupper(buf[ii]);
            }
            write(STDOUT_FILENO , buf , len);
             write(cfd , buf ,len); 
          }
            close(cfd);
            return 0;
        }
        else if(ret > 0){
   ///    close(cfd);   
    }
    }
    return 0;
}
```

客户端：

```c
/*************************************************************************
	> File Name: client.c
	> Author: 
	> Mail: 
	> Created Time: 2019年08月08日 星期四 11时16分54秒
 ************************************************************************/

#include<stdio.h>
#include<string.h>
#include<unistd.h>
#include<arpa/inet.h>
#include<stdlib.h>
#include<ctype.h>
#include<sys/types.h>
#include<sys/socket.h>
#include<arpa/inet.h>

# define MAXLINE 135
# define PORT 8556
int main(){

    struct sockaddr_in serv;
    char buf[MAXLINE];
    int sockfd ,n;
    
    sockfd = socket(AF_INET ,SOCK_STREAM ,0);
    bzero(&serv,sizeof(struct sockaddr));
    serv.sin_family = AF_INET;
    inet_pton(AF_INET ,"127.0.1.2",&serv.sin_addr.s_addr);
    serv.sin_port = htons(PORT);
    //  serv.sin_addr.s_addr = htonl("127.0.0.1");

    connect(sockfd,(struct sockaddr *)&serv,sizeof(serv));

    while(fgets(buf,MAXLINE ,stdin) != NULL){
        write(sockfd , buf , strlen(buf));
        int len = read(sockfd, buf , MAXLINE);
        printf("--------%d---------\n",len);
        write(STDOUT_FILENO ,buf, len);
    }
    close(sockfd);
return 0;

}
```

多进程和多线程并发在linux中，内核的消耗并不是特别大。

