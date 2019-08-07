![1565185370771](C:\Users\acm506\AppData\Roaming\Typora\typora-user-images\1565185370771.png)

**arp数据报作用：**

通过IP地址获取mac地址

```c
netstat -apn | grep 6666
```

查看6666的端口号信息

**如果先关server.c 的话，6666这个端口号就被被写死，无法绑定。**

也可以自己封装函数，包括出错机制，调入参数；可以同时加一个 **.h**文件。



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



**协议上限分析：**

链路层 网络层 传输层 应用层 数据 校验码

如果超过上限的话，会进行拆包。保证能把所有数据按照完整的包的形式传送出去。