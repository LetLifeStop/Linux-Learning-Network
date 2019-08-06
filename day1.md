# 网络基础

####   协议的概念

   从源到目的地，参与方必须遵循相同的规则。

​      原始协议 -》 标准协议

  **C/S模型：**（client / server ）

优点：

1. 协议选用灵活
2. 可以缓存数据

缺点：

1. 对用户的安全构成威胁
2. 开发工作量大，调试困难

场景：

1. 数据量大
2. 稳定性高

**B/S模型：**

优点：

1. 安全性高一点
2. 开发工作量较小
3. 跨平台

缺点：

1. 支持http协议
2. 不能进行数据缓存

场景：

1. 数据量较小

2. 数据访问工作量小

端口值不能大于65535 ，一般比较小的 < 1000，给系统使用 .

### 分层模型

**OSI七层模型**

https://gss2.bdstatic.com/9fo3dSag_xI4khGkpoWK1HF6hhy/baike/crop%3D0%2C11%2C568%2C375%3Bc0%3Dbaike80%2C5%2C5%2C80%2C26/sign=728f7fc26f380cd7f251f8ad9c748105/b90e7bec54e736d1eddf0c7a91504fc2d46269f0.jpg

网络层 -》IP协议

传输层 -》TCP/UDP协议（TCP协议传输效率低，可靠性高，用于传输可靠性要求高，数据量大的数据；UDP协议传输效率，高数据量小 QQ）

应用层 -》FTP协议

**第一层（从下往上）**

第一层 ，物理层：主要定义物理设备标准，传输比特流

第二层，数据链路层：如何让格式化数据以帧为单位进行传输；提供错误检查和纠正

第三层，网络层：两个主机之间提供链接和路径选择

第四层，传输层：定义了一些传输数据的协议和端口号

第五层，会话层：

第六层，表示层：

第七层，应用层： 方便用户对数据进行访问，为用户的应用程序提供网络服务。

剩余三层分工并不多明确，对传输层数据进行间接的封装和解封装，对数据进行确认，打包处理。

对应分出来就是

![1565094947266](C:\Users\acm506\AppData\Roaming\Typora\typora-user-images\1565094947266.png)

#### 通信过程

**数据要进行传输，必须要进行封装**

应用层（FTP协议） -》传输层（TCP协议） -》网络层（IP协议） -》链路层（以太网驱动程序）

**数据包封装**

![1565095050615](C:\Users\acm506\AppData\Roaming\Typora\typora-user-images\1565095050615.png)

**两个封装好的数据包如何完成网络通信？**

TCP协议，通过路由器寻路，找到之后就按照这个路走

UDP协议，每一次都是寻路



**以太网帧格式**

目的地址（6字节） 源地址（6字节） 类型（2字节）数据（46~1500字节） CRC（4字节）（校验码）

1. 源地址和目的地址是指网卡的硬件地址（MAC地址），长度是48位，是在网卡出厂的时候固定的。在shell中使用ifconfig查看，"HWADDR 00:15:F2:14:9E:3F"（保证唯一） 部分就是硬件地址。协议字段有三种值，分别对应IP , ARP , RARP。

2. 类型，当类型为0800的时候，传输的是IP数据段；当类型为0806，后面为 ARP请求/应答（请求对方的Mac地址）（28字节） PAD（18字节）（PAD是用来填充的）
   
   1. ARP数据报（获取下一跳的mac地址）
   
   ARP 是IP地址请求Mac地址
   
   RARP 是Mac地址请求IP地址
   
   arp攻击

28字节构成

![1565095078301](C:\Users\acm506\AppData\Roaming\Typora\typora-user-images\1565095078301.png)

注意区别 源IP ，目的IP， 发送端IP，接收端IP

 

当ARP数据包进入死循环的时候。可以通过设置TTL（限制次数）来丢弃数据包。



源IP地址 ，目的IP地址也是32位

192.168.1.175 是字符串，我们在传输的时候需要把它转换成 unsigned int



UDP数据包

端口号认为进程 ，防止让其他进程收到数据包

总结：

1. 什么协议
2. 典型的协议
3. cs 模型。bs模型优缺点
4. 七层模型
5. 两台计算机TCP/IP 协议通讯过程
6. 以太网帧格式，各部分大小，类型
7. IP段格式，掌握32位源IP地址，32位目的IP地址



**NAT映射**

192.168.1.35

192这个IP只是在局域网中；发送的时候，通过NAT映射转换成唯一的I公网IP地址，再就是交换机之间互相传送。



**打洞机制**

因为路由器内部保护机制，当接收到陌生的IP第一次发送数据包时，会把数据包屏蔽或者丢弃。

**通过打洞机制，可以实现两台私有服务器之间，可以实时共联（视频），提高传输效率** 

简单的说就是在登录qq的时候，就会访问腾讯服务器，而腾讯服务器也会回一个数据包，这个数据包会携带腾讯公网的IP，相对于来说腾讯的公网IP在A ，B那里都是熟悉的IP。

公对公 ，直接访问

公对私，NAT映射

私对公，NAT映射

私对私，NAT映射，打洞机制  / 局域网



#### Socket编程

**套接字 是linux操作系统当中的一种伪文件**



IP地址可以在网络环境中唯一标示一台主机

端口号可以在主机中唯一标示一个进程

IP + 端口号 就能在网络环境中唯一标示一个进程（Socket）



Socket 类比管道，也有文件描述符；但是对应两个缓冲区

，一个用来读入，一个输出。

管道是半双工，Socket是全双工，所以套接字是成对出现的。



#### 网络字节序

大端法存储：低地址 --高位 （高地址 --低位） 

小端法存储：低 -- 低（，。。）（**计算机使用的方法**）

TCP/IP协议规定，**网络数据流音高采用大端字节序**。 

所以在使用网络通讯的时候，需要做**网络字节序和主机字节序的转换。**

```c
#include<arpa/inet.h>
uint32_t htonl(uint32_t hostlong);
本地字节序转换成网络字节序，IP地址（4字节）
uint16_t htons(uint16_t hostshort);
本地字节转换转成端口号（16位） 
uint32_t ntohl(uint32_t netlong);
无符号长整型数从网络字节顺序转换成主机字节顺序
uint16_t ntohs(uint16_t netshort);
无符号短整型数从网络字节顺序转换成主机字节顺序
```

h 是host ，n是network ，l表示32位长整数，s表示16位短整数



#### IP地址转换函数

```c
#include<aroa/inet.h>
int inet_pton(int af , const char *src ,void *dst);
const char *inet_ntop(int af ,const void *src ,char *dst ,socklen_t size);

```

**inet_pton** 

ip -> net

参数：

af   指定所选用的IP地址的版本 

src  源 (192.168.1.178)

dst 目标 		

**inet_ntop** 

net ->ip

比inet_pton多了一个参数，size指定缓冲区的大小



#### sockaddr 数据结构

​	struct sockaddr 已经被废弃，现在使用struct sockaddr_in ,使用的时候强转。

结构体内部成员：

查看方法： man 7 ip

sin_family ;表示所选用的地址族协议  ipv4 ，ipv6

sin_port ; 端口号

struct in_addr sin_addr ；ip地址（结构体）

struct in_addr{

uint32_t  s_addr;

};



#### 网络套接字函数

```c
int inet_pton(int af ,const char *src ,void *dst);
const char *inet_ntop(int af ,const void *src ,char *dst ,socketlen_t size);
```

将IP地址在“[点分十进制](https://baike.baidu.com/item/点分十进制)”和“二进制整数”之间转换

```c
int socket(int domain ,int type ,int protocol);
```

domain参数：

一般是AF_INET

type:

SOCK_STREAM

传0表示使用默认协议

对于IPv4，domain参数指定为AF_INET。对于TCP协议，type参数指定为SOCK_STREAM，表示面向流的传输协议。如果是UDP协议，则type参数指定为SOCK_DGRAM，表示面向数据报的传输协议。protocol参数的介绍从略，指定为0即可。

返回值：

成功返回文件描述符

**bind函数**

```c
int bind(int sockfd ,const struct sockaddr *addr ,socklen_t addrlen)
```

将文件描述符和IP以及端口进行绑定



将socket文件描述符和add绑在一起，使socket这个用于网络通信的文件描述符监听addr所描述的地址和端口号

**如果没有调用的话，会自动分配一个IP和端口号。**

listen函数

```c
int listen(int sockfd ,int backlog);
```

限制最多允许的客户端连入

指定监听上限数，同时允许多少客户端与我建立连接，排队机制

**accept函数**

```c
int accept(int sockfd ,struct sockaddr *addr,socklen_t *addrlen);
```

等待用户发起链接（阻塞等待）

**addrlen参数：**

传入传出参数

**注意返回值，返回的是一个新的socket文件描述符，用于和客户端通信。**



**connect函数**

```c
int conncet(int sockfd ,const struct sockaddr *addr ,socklen_t addrlen);
```



客户端需要调用这个函数链接服务器

**实现socket套接字通信**

![1565096226194](C:\Users\acm506\AppData\Roaming\Typora\typora-user-images\1565096226194.png)



服务端：

```c

#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <strings.h>
#include <string.h>
#include <ctype.h>
#include <arpa/inet.h>

#define SERV_PORT 9527

int main(void)
{
    int sfd, cfd;
    int len, i;
    char buf[BUFSIZ], clie_IP[BUFSIZ];

    struct sockaddr_in serv_addr, clie_addr;
    socklen_t clie_addr_len;

    /*创建一个socket 指定IPv4协议族 TCP协议*/
    sfd = socket(AF_INET, SOCK_STREAM, 0);

    /*初始化一个地址结构 man 7 ip 查看对应信息*/
    bzero(&serv_addr, sizeof(serv_addr));           //将整个结构体清零
    serv_addr.sin_family = AF_INET;                 //选择协议族为IPv4
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);  //监听本地所有IP地址
    serv_addr.sin_port = htons(SERV_PORT);          //绑定端口号    

    /*绑定服务器地址结构*/
    bind(sfd, (struct sockaddr *)&serv_addr, sizeof(serv_addr));

    /*设定链接上限,注意此处不阻塞*/
    listen(sfd, 64);                                //同一时刻允许向服务器发起链接请求的数量

    printf("wait for client connect ...\n");

    /*获取客户端地址结构大小*/ 
    clie_addr_len = sizeof(clie_addr_len);
    /*参数1是sfd; 参2传出参数, 参3传入传入参数, 全部是client端的参数*/
    cfd = accept(sfd, (struct sockaddr *)&clie_addr, &clie_addr_len);           /*监听客户端链接, 会阻塞*/

    printf("client IP:%s\tport:%d\n", 
            inet_ntop(AF_INET, &clie_addr.sin_addr.s_addr, clie_IP, sizeof(clie_IP)), 
            ntohs(clie_addr.sin_port));

    while (1) {
        /*读取客户端发送数据*/
        len = read(cfd, buf, sizeof(buf));
        write(STDOUT_FILENO, buf, len);

        /*处理客户端数据*/
        for (i = 0; i < len; i++)
            buf[i] = toupper(buf[i]);

        /*处理完数据回写给客户端*/
        write(cfd, buf, len); 
    }

    /*关闭链接*/
    close(sfd);
    close(cfd);

    return 0;
}

```

客户端：

```c
#include <stdio.h>
#include <unistd.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>

#define SERV_IP "127.0.0.1"
#define SERV_PORT 9527

int main(void)
{
    int sfd, len;
    struct sockaddr_in serv_addr;
    char buf[BUFSIZ]; 

    /*创建一个socket 指定IPv4 TCP*/
    sfd = socket(AF_INET, SOCK_STREAM, 0);

    /*初始化一个地址结构:*/
    bzero(&serv_addr, sizeof(serv_addr));                       //清零
    serv_addr.sin_family = AF_INET;                             //IPv4协议族
    inet_pton(AF_INET, SERV_IP, &serv_addr.sin_addr.s_addr);    //指定IP 字符串类型转换为网络字节序 参3:传出参数
    serv_addr.sin_port = htons(SERV_PORT);                      //指定端口 本地转网络字节序

    /*根据地址结构链接指定服务器进程*/
    connect(sfd, (struct sockaddr *)&serv_addr, sizeof(serv_addr));

    while (1) {
        /*从标准输入获取数据*/
        fgets(buf, sizeof(buf), stdin);
        /*将数据写给服务器*/
        write(sfd, buf, strlen(buf));       //写个服务器
        /*从服务器读回转换后数据*/
        len = read(sfd, buf, sizeof(buf));
        /*写至标准输出*/
        write(STDOUT_FILENO, buf, len);
    }

    /*关闭链接*/
    close(sfd);

    return 0;
}

```

