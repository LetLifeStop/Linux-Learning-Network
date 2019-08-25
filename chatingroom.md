client.h

```c
#include<stdlib.h>
#include<stdio.h>
#include<string.h>
#include<fcntl.h>
#include<limits.h>
#include<sys/types.h>
#include<sys/stat.h>
#include<unistd.h>
#include<ctype.h>

#define SERVER_FIFO_NAME "serve_fifo"  // client -> server
#define CLIENT_FIFO_NAME "client_%d_fifo" // server ->client

# define BUFFER_SIZE PIPE_BUF
# define MESSAGE_SIZE 20
# define NAME_SIZE 256

 typedef  struct message{
    pid_t client_pid;   // 客户的文件pid
    char data[MESSAGE_SIZE+10]; 
}message;
```



客户端

```c
#include"client.h"

int main(){
    int server_fifo_fd;
    int client_fifo_fd;
    int res;
    message msg;
    char client_fifo_name[1024];
    
    msg.client_pid = getpid();
     // 给client_fifo_name 赋值
    sprintf(client_fifo_name , CLIENT_FIFO_NAME , msg.client_pid);
   
    // 创建管道
  if(mkfifo(client_fifo_name , 0777) == -1){
      perror("creat client_fifo_name error");
      exit(1);
  }
   // 服务器向私有管道中写数据
   server_fifo_fd = open(SERVER_FIFO_NAME ,O_WRONLY);
  if(server_fifo_fd == -1){
      perror("creat server_fifo error");
      exit(1);
  }

  sprintf(msg.data , "hello from %d",msg.client_pid);
  printf("%d sent %s",msg.client_pid , msg.data);
   // 客户端向公共管道中写数据
  write(server_fifo_fd , &msg , sizeof(msg));
    
   // 客户端从私有管道中读取数据
   client_fifo_fd = open(client_fifo_name ,O_RDONLY);  
   if(client_fifo_fd == -1){
      perror("creat client fifo fd error");
      exit(1);
  }
  
  res = read(client_fifo_fd , &msg , sizeof(msg));
  if(res > 0){
      printf("received %s\n" , msg.data);
  }

  close(client_fifo_fd);
  close(server_fifo_fd);
  unlink(client_fifo_name);
  return 0;
}
```



服务器端：

```c
#include"client.h"

int main(){
    int client_fifo_fd;
    int server_fifo_fd;
    char client_fifo_name[1024];
    // 创建公共管道
	if(mkfifo(SERVER_FIFO_NAME,0777) == -1 ){
        perror("creat SERVER_FIFO error");
        exit(1);
    } 
    //服务器从管道中读取数据 
    server_fifo_fd = open(SERVER_FIFO_NAME , O_RDONLY);
    if( client_fifo_fd < 0 ){
        perror("creat client fifo fd error");
        exit(1);
    }
    sleep(5);
   
	message msg;
    char *p;
    int res ;
    
	while(res = read(server_fifo_fd , &msg , sizeof(msg)) > 0 ) {
        p = msg.data;
	
		while(*p){
			*p = toupper(*p);
			++p;
		}
		sprintf(client_fifo_name, CLIENT_FIFO_NAME, msg.client_pid);
        client_fifo_fd = open(client_fifo_name, O_WRONLY);
        if (client_fifo_fd == -1) {
            perror("creat client_fifo_fd error");
            exit(1);
        }
        write(client_fifo_fd, &msg, sizeof(msg));
        close(client_fifo_fd);
    }
     close(server_fifo_fd);
     unlink(SERVER_FIFO_NAME);
  return 0;
}
```

