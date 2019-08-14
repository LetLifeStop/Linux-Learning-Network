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

4. 不需要维护连接

