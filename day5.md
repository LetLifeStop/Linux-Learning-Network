**查看结构体定义方法：**

grep "struct ip_mreqn" -r /usr/include -n

然后 vi  (出现的内容)  行号

### UDP服务器

> UDP协议 与 TCP 协议优缺点以及适用场景 

TCP：面向连接的可靠数据包传递

1. 优点： 
   - 稳定 ，**丢包回执机制**使得丢包概率变得很小； 
   - 传输速率稳定，铺好路之后就很稳定
   - 流量稳定，自带的滑动窗口机制不会造成数据包的丢失



2. 缺点：
   - 效率低，握手过程
   - 速度慢
3. 使用场景

​        大文件的传输 ，重要文件的传输

UDP：无连接的不可靠报文传递 

1. 缺点：
   - 丢包概率大，不稳定
   - 传输速率不稳定
   - 流量不稳定
2. 优点：
   - 效率高
   - 速度快
3. 使用场景 

​      对实时性要求较高的场合，视频会议，，，，

腾讯：TCP + UDP ， 应用层 UDP + 应用层自定义协议弥补UDP的丢包

当UDP的缓冲区被填满后，再接受数据会出现丢包的现象 。可以通过以下两种方法：

1. 服务器应用层设计流量控制，控制发送数据速度

2. 借助setsockpot函数改变缓冲区大小

   ```c
   int setsockopt(int sockfd ,int level ,int optname ,const void *optval , socklen_t optlen);
   int n = 220x1024;
   setsockopt(sockfd , SOL_SOCKET , SO_RCVBUF ,&n ,sizeof(n));
   ```



练习：

#### 借助UDP实现C/S 模型

服务器端：

```c
/*************************************************************************
	> File Name: udp.c
	> Author: 
	> Mail: 
	> Created Time: 2019年08月14日 星期三 16时34分07秒
 ************************************************************************/

#include<stdio.h>
#include<stdlib.h>
#include<string.h>
#include<ctype.h>
#include<sys/socket.h>
#include<sys/types.h>
#include<arpa/inet.h>
#include<unistd.h>

# define PORT 8556

int main(){
   
    int serv_fd , ret ;
    struct sockaddr_in sock, clie;
    int  len , len_send;
    char buf[BUFSIZ], str[100];
    socklen_t len_clie;

    serv_fd = socket(AF_INET, SOCK_DGRAM , 0);
    if(serv_fd < 0){
        perror("create socket error");
        exit(1);
    }

    bzero(&sock , sizeof(sock));
    sock.sin_port = htons(PORT);
    sock.sin_family = AF_INET;
    sock.sin_addr.s_addr = htonl(INADDR_ANY); 
  // 只要是往这个端口发送的IP都能连上

    ret = bind(serv_fd, (struct sockaddr *)&sock , sizeof(sock));
     
    while(1){
        
        len_clie = sizeof(clie);
       len =  recvfrom(serv_fd , buf, BUFSIZ ,0 , (struct sockaddr *)&clie , &len_clie);
        printf("len = %d\n", len);
        if(len < 0){
            perror("recv from error");
            exit(1);
        }
        printf("receive IP %s ,and receive PORT = %d\n ",inet_ntop(AF_INET ,& clie.sin_addr, str , sizeof(str)) , ntohs(clie.sin_port));

        for(int i = 0 ; i < len ; i++){
            buf[i] = toupper(buf[i]);
        }
       
        len_send = sendto(serv_fd,buf , len , 0 ,(struct sockaddr *)&clie , sizeof(clie) );
        if(len_send == -1){
            perror("sento error");
            exit(1);
        }
    }
    close(serv_fd);

    return 0;

}
```



客户端：

```c
/*************************************************************************
	> File Name: udp2.c
	> Author: 
	> Mail: 
	> Created Time: 2019年08月14日 星期三 17时09分35秒
 ************************************************************************/

#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<string.h>
#include<arpa/inet.h>
#include<sys/types.h>
#include<ctype.h>
#include<sys/types.h>

# define SERV_PORT 8556

int main(){

    int cli_fd ,n ;
    char buf[100];
    ssize_t len ;
    struct sockaddr_in servaddr;

    cli_fd = socket(AF_INET , SOCK_DGRAM , 0);
    if(cli_fd < 0 ){
        perror("create socket error");
        exit(1);
    }
    
    bzero(&servaddr , sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    servaddr.sin_port = htons(SERV_PORT);
    inet_pton(AF_INET , "127.0.0.1" , &servaddr.sin_addr);
    
    while(fgets(buf , 100 , stdin) != NULL){
    
        len = sendto(cli_fd , buf , strlen(buf), 0 ,(struct sockaddr *)&servaddr , sizeof(servaddr));
        if( n < 0 ){
            perror("send to error");
            exit(1);
        }

        len = recvfrom(cli_fd , buf , 100 , 0, NULL , 0);
        if( len  < 0){
            perror("recvfrom error");
            exit(1);
        }

        write(STDOUT_FILENO , buf ,len );
    }
    close(cli_fd);
    return 0;
}
```



#### 练习总结：

1. 在TCP中，我们绑定的是SOCK_STREAM , 这个流 可以保证接受的数据字节与发送的是瞬息完全一致的，但是 ！！**（通信之间必须建立一个连接）**

​    2.   为了实现UDP之间的通信，我们少了accept和bind的过程，也就是说可以在不建立链接的情况下发送远程进程的，这个时候就需要用到 **SOCK_DGRAM**；也就是说socket里面的参数需要改变。

3. 

```c
sockaddr_in {
short in sin_family;   协议族， socket编程中按照AF_INET写
unsigned short int sin_port; 端口号
struct in_addr sin_addr;  
};
struct in_addr{
unsigned long s_addr;   
} 
```

**sin_addr 储存的是本机的网络IP地址，然后 sin_addr.s_addr 储存的是 允许哪些 网络IP地址  访问服务器端**

![1565838603546](C:\Users\acm506\AppData\Roaming\Typora\typora-user-images\1565838603546.png)

4. 不需要维护连接



#### UDP实现广播

IP：192.168.42.255（255为广播地址）

IP：192.168.42.1（1 为网关）

（0,0,0,0）？

https://www.cnblogs.com/xiluhua/p/10657917.html

服务器端：

注意点：

CLI_IP 的最后一个为255

```c
/*************************************************************************
	> File Name: boradcast_ser.c
	> Author: 
	> Mail: 
	> Created Time: 2019年08月15日 星期四 09时22分59秒
 ************************************************************************/

#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<ctype.h>
#include<sys/types.h>
#include<sys/socket.h>
#include<string.h>
#include<arpa/inet.h>

/*
 * 1. socket创建套接字
 * 2. 服务器端初始化自己的信息 
 * 3. setsocket 调整套接字的属性
 * 4. 初始化服务器端的属性
 * 5.  发送数据
*/

# define SERV_PORT 8556
# define CLI_PORT 9000
# define CLI_IP "192.168.18.255"

int main(){

    int ret_sock , ret_bind ,set_sock;
    struct sockaddr_in ser; 
    struct sockaddr_in cli;
    char buf[100];

    ret_sock = socket(AF_INET , SOCK_DGRAM , 0); 
    if(ret_sock < 0){
        perror("create socket error");
        exit(1);
    }

    bzero(&ser , sizeof(ser));
    ser.sin_family = AF_INET;
    ser.sin_port = htons(SERV_PORT);
    ser.sin_addr.s_addr = htonl(INADDR_ANY);
  // 放开广播权限
    ret_bind = bind(ret_sock , (struct sockaddr *)&ser , sizeof(ser));
    if(ret_bind < 0){
        perror("bind error");
        exit(1);
    }
 
    int opt = 1;
    set_sock = setsockopt(ret_sock , SOL_SOCKET , SO_BROADCAST ,&opt, sizeof(opt));
    if(set_sock < 0){
        perror("set sock opt error");
        exit(1);
    }

    bzero(&cli , sizeof(cli));
    cli.sin_family = AF_INET;
    cli.sin_port = htons(CLI_PORT);
    inet_pton(AF_INET , CLI_IP , &cli.sin_addr.s_addr);
     // 客户端能接受的IP地址只有 CLI_IP
    int i , n ;
    while(1){
      sprintf(buf ,"wakaka  %d\n" , i++);
      n = sendto(ret_sock ,buf , strlen(buf) , 0 ,(struct sockaddr *)&cli , sizeof(cli) );
        sleep(1);
    }
    close(ret_sock);

    return 0;
}
```



客户端：

```c
/*************************************************************************
	> File Name: broadcast_cli.c
	> Author: 
	> Mail: 
	> Created Time: 2019年08月15日 星期四 09时56分12秒
 ************************************************************************/

#include<stdio.h>
#include<stdlib.h>
#include<sys/types.h>
#include<arpa/inet.h>
#include<ctype.h>
#include<string.h>
#include<unistd.h>

/*
 * 1. 创建套接字
 * 2. 初始化客户端信息
 * 3. 绑定
 * 4. 接受数据
*/
# define CLI_PORT 9000

int main(){
  
    int ret_sock,ret_bind;
    struct sockaddr_in cli;
    char buf[BUFSIZ];

    ret_sock = socket(AF_INET , SOCK_DGRAM , 0);
    if(ret_sock < 0 ){
        perror("create socket error");
        exit(1);
    }

    bzero(&cli , sizeof(cli));
    cli.sin_family = AF_INET;
    cli.sin_port = htons(CLI_PORT);
    
    inet_pton(AF_INET ,"0.0.0.0", &cli.sin_addr.s_addr);
    // cli.sin_addr.s_addr = htonl(INADDR_ANY);
    
    ret_bind = bind(ret_sock , (struct sockaddr *)&cli , sizeof(cli));
    if(ret_bind < 0){
        perror("bind error");
        exit(1);
    }
    printf("lalalalla\n");
    int n ,i;
    while(1){

    n = recvfrom(ret_sock ,buf , BUFSIZ ,0 ,NULL , 0);

    write(STDOUT_FILENO , buf , n);

    }
    close(ret_sock);
    return 0;

}
```



### 组播

> 组播地址

224.0.0.0 ~ 224.0.0.255 预留的组播地址（永久组地址，）保留不做分配

224.0.1.0 ~ 224.0.1.255 公用组播地址，可用于intenet， 欲使用请申请

224.0.2.0 ~ 238.255.255.255 用户 可用的组播地址（临时），全网范围有效

239.0.0.0 ~239.255.255.255 本地管理组播地址，仅在特定的本地范围内有效



可以使用ip ad 查看网卡编号

网卡编号：

```c
 if_nametoindex(const char *ifname);
```

把网卡名称传进去，返回网卡编号

```c
struct ip_mreqn{
struct in_addr imr_multiaddr;  组IP
struct in_addr imr_address ;   自己的IP
int imr_ifindex;    网卡编号
}
```

组播练习：

服务器端：

```c
/*************************************************************************
	> File Name: mul.c
	> Author: 
	> Mail: 
	> Created Time: 2019年08月15日 星期四 14时31分15秒
 ************************************************************************/

#include<stdio.h>
#include<stdlib.h>
#include<string.h>
#include<unistd.h>
#include<sys/socket.h>
#include<sys/types.h>
#include<ctype.h>
#include<arpa/inet.h>
#include<net/if.h>

# define GROUP "239.0.0.2"
# define SERVPORT 6666
# define CLIEPORT 7777
/* 实现组播
 * 服务器端
 *1. 创建套接字
 *2. 对组播信息进行初始化
 *3. bind绑定
 *4. 对客户端进行初始化，将客户端放入组中
 *5. 组播发送信息
*/
 
int main(){

    int ret_sock ,ret_bind,ret_setsock;
    struct sockaddr_in serv, clie;
    struct ip_mreqn group;
    char buf[100];

    ret_sock = socket(AF_INET ,SOCK_DGRAM , 0);
    if(ret_sock < 0){
        perror("create sock error");
        exit(1);
    }

    bzero(&serv , sizeof(serv));
    serv.sin_family = AF_INET;
    serv.sin_port = htons(SERVPORT);
    serv.sin_addr.s_addr = htonl(INADDR_ANY);

    ret_bind = bind(ret_sock , (struct sockaddr *)&serv , sizeof(serv));
    if(ret_bind < 0 ){
        perror("bind error");
        exit(1);
    }

    inet_pton(AF_INET, GROUP ,&group.imr_multiaddr);
    inet_pton(AF_INET ,"0.0.0.0", &group.imr_address); // 表示往哪个组中传输信息
    group.imr_ifindex = if_nametoindex("ens33"); // 返回网卡的编号

    ret_setsock = setsockopt(ret_sock , IPPROTO_IP , IP_MULTICAST_IF ,&group,sizeof(group)); // 设置组
    if(ret_setsock < 0){
        perror("setsock error");
        exit(1);
    }

    bzero(&clie, sizeof(clie));
    clie.sin_family = AF_INET;
    clie.sin_port = htons(CLIEPORT);
    inet_pton(AF_INET ,GROUP ,&clie.sin_addr.s_addr);

    int i;
    while(1){
        sprintf(buf , "serv lalala %d\n",i++);
        sendto(ret_sock , buf , strlen(buf) , 0,(struct sockaddr *)&clie , sizeof(clie));
        sleep(1);
    }
    close(ret_sock);

    return 0;
}
```



客户端：

```c
/*************************************************************************
	> File Name: mul2.c
	> Author: 
	> Mail: 
	> Created Time: 2019年08月15日 星期四 15时13分45秒
 ************************************************************************/

#include<stdio.h>
#include<stdlib.h>
#include<string.h>
#include<sys/types.h>
#include<sys/socket.h>
#include<arpa/inet.h>
#include<unistd.h>
#include<net/if.h>

# define CLIEPORT 7777
# define GROUP "239.0.0.2"

int main(){

    int ret_sock,ret_bind,ret_set;
    struct sockaddr_in clie;
    char buf[100];
    struct ip_mreqn group;
    
    ret_sock = socket(AF_INET ,SOCK_DGRAM , 0 );
    if(ret_sock < 0){
        perror("create socket error");
        exit(1);
    }
   
    bzero(&clie , sizeof(clie));
    clie.sin_family = AF_INET;
    clie.sin_port = htons(CLIEPORT);
    clie.sin_addr.s_addr = htonl(INADDR_ANY);

    ret_bind = bind(ret_sock ,(struct sockaddr *)&clie , sizeof(clie));
    if(ret_bind < 0){
        perror("bind error");
        exit(1);
    }
    
    bzero(&group , sizeof(group));
    inet_pton(AF_INET, GROUP ,&group.imr_multiaddr);
    inet_pton(AF_INET,"0.0.0.0",&group.imr_address); // 表示加入哪个组播组中
    group.imr_ifindex = if_nametoindex("ens33");

    ret_set = setsockopt(ret_sock ,IPPROTO_IP ,IP_ADD_MEMBERSHIP ,&group ,sizeof(group));
    if(ret_set < 0 ){ // 表示加入哪个组中
        perror("set sock error");
        exit(1);
    }

    while(1){
        int n = recvfrom(ret_sock , buf , 100 ,0 ,NULL,0); 
        write(STDOUT_FILENO , buf, n);
    }

    close(ret_sock);

    return 0;
}

```



总结：

到目前为止setsockopt作用：

1. 端口复用
2. 设置缓冲区大小
3. 开放广播权限
4. 开放组播权限
5. 加入组播组



#### 共享屏幕软件实现基本思路：

1. 屏幕截图模块  
2. 截取帧数
3. 对图片进行压缩  MB -》KB
4. 数据包的压缩
5. 传递
6. 解压缩
7. 成像

（分屏软件源码）



#### domain本地套

1. pipe ，fifo   实现最简单
2. mmap         非血缘关系进程
3. 信号             开销小
4. domain

socket都位于内核当中，不仅仅可以通过 IP + 端口号链接；

```c
#include<sys/un.h>
struct sockaddr_un{
__kernel_sa_famly_t  sum_family;  AF_UNIX
char sum_path[UNIX_PATH_MAX];  路径
}
```

1. UNIX Domain socket 是**全双工**
2. 没有使用TCP/ UDP 协议



流式协议，报式协议 

**offset宏,求出结构体成员在结构体中的偏移位置**

```c
define offsetof(type ,member)((int)&((type *)0)->MEMBER)
```

**注意**

客户端和服务器端在bind之前，应该首先unlink掉自己对应的文件名称，然后再bind。因为**必须确保之前的path文件不存在，bind会自动创建该文件，如果这个文件不unlink掉，bind就会失败**

代码实现：

服务器端：

accept 返回的len和客户端那边的传入是一样的

```c
/*************************************************************************
	> File Name: domain_serv.c
	> Author: 
	> Mail: 
	> Created Time: 2019年08月15日 星期四 16时49分30秒
 ************************************************************************/

#include<stdio.h>
#include<stdlib.h>
#include<string.h>
#include<ctype.h>
#include<sys/types.h>
#include<sys/socket.h>
#include<unistd.h>
#include<sys/un.h>
#include<stddef.h>

/* 本地套接字的实现
 * 1. 创建socket
 * 2. unllink
 * 3. bind
 * 4. listen
 * 5. accept 阻塞等待
*/

# define SERV_ADDR "serv.socket"

int main(){
   
    int ret_sock ,ret_bind , cfd, len , size;
    struct sockaddr_un serv ,cli;
    char buf[4096];

    ret_sock = socket(AF_UNIX ,SOCK_STREAM ,0);
    // 流式协议
    if(ret_sock < 0 ){
        perror("create socket error");
        exit(1);
    }
        
    bzero(&serv ,sizeof(serv));
    serv.sun_family = AF_UNIX;
    strcpy(serv.sun_path , SERV_ADDR);
       
    len = offsetof(struct sockaddr_un ,sun_path) + strlen(serv.sun_path);
    
    unlink(SERV_ADDR);
    ret_bind = bind(ret_sock , (struct sockaddr *)&serv , len);
    if(ret_bind < 0){
        perror("bind error");
        exit(1);
    }

    listen(ret_sock ,20);

    printf("accept  \n");

   while(1){
        
        len = sizeof(cli);
        cfd = accept(ret_sock ,(struct sockaddr *)&cli , (socklen_t *)&len);
     //   printf("11  len = %d\n", len); 
        len -= offsetof(struct sockaddr_un, sun_path);
     //   printf("len = %d\n",len);
         cli.sun_path[len] = '\0';
     //   write(STDOUT_FILENO , cli.sun_path , len);
        printf("client bind filename = %s\n",cli.sun_path);

        while((size = read(cfd , buf, sizeof(buf))) > 0){
            printf("size = %d\n",size);
            for(int i = 0; i < size ; i++){
                buf[i] = toupper(buf[i]);
            }
            write(cfd , buf , size);
            write(STDOUT_FILENO , buf, size);
        }
       close(cfd);
    }
    close(ret_sock);

    return 0;
}
```

客户端：

```c
/*************************************************************************
	> File Name: domain_clie.c
	> Author: 
	> Mail: 
	> Created Time: 2019年08月15日 星期四 17时12分51秒
 ************************************************************************/

#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<ctype.h>
#include<sys/types.h>
#include<sys/socket.h>
#include<string.h>
#include<sys/un.h>
#include<stddef.h>

# define SERV_ADDR "serv.socket"
# define CLIE_ADDR "clie.socket"

/*domain 客户端
 * 1. socket
 * 2. unlink
 * 3. bind
 * 4. connect
*/
int main(){
    
    int cfd ,len;
    struct sockaddr_un cli,ser;
    char buf[4096];

    cfd = socket(AF_UNIX ,SOCK_STREAM , 0);
    if(cfd < 0){
        perror("create socket error");
        exit(1);
    }
 
    bzero(&cli,sizeof(cli));
    cli.sun_family = AF_UNIX;
    strcpy(cli.sun_path , CLIE_ADDR);

    len = offsetof(struct sockaddr_un , sun_path) + strlen(cli.sun_path);

    unlink(CLIE_ADDR);
    bind(cfd , (struct sockaddr *)&cli ,len);

    bzero(&ser , sizeof(ser));
    ser.sun_family = AF_UNIX;
    strcpy(ser.sun_path , SERV_ADDR);

    len = offsetof(struct sockaddr_un , sun_path) + strlen(ser.sun_path);

    connect(cfd , (struct sockaddr *)&ser,len);

    while(fgets(buf, sizeof(buf) , stdin) != NULL){
       len =  write(cfd , buf, strlen(buf));
        printf("len = %d\n",len);
        len = read(cfd , buf, sizeof(buf));
        write(STDOUT_FILENO , buf ,len);
    }
    close(cfd);
   return 0;

}
```



#### 对len的进一步说明

```c
struct sockaddr_un{
sun_family;  2B
sun_path;  108B
};
```

如果是直接sizeof(struct sockaddr_un)  == 110B

正确做法：offsetof()求出sun_path 在整个结构体的偏移量



