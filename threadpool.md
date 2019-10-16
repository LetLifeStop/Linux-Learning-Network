## 线程池具体思路及理解



### 线程池具体使用思路

大体上可以分为四个模块

1. 生产者（生产任务）
2. 任务队列（循环队列实现）

1. 处理者（从任务队列中解决任务）
2.  管理者（通过查看各种线程之间的比例，开来保证线程资源的合理利用）



### **具体思路**

- 线程池内部进行初始化。互斥锁（结构体的锁，循环队列的锁），条件变量（循环队列不为空，循环队列已满），最小存活的线程数目，最大存活的线程数目，线程数目申请的上限。循环队列的顶部，底部，大小，存储上限。运行线程最小的允许数目，开启管理者线程
- 向线程池中添加任务。首先判断队列是否已满，如果已满就阻塞，等待队列未满的信号发送过来，然后才能继续往任务队列中塞任务，塞进去之后，给线程池发送信号，告诉你我现在向任务队列中添加了任务，你快点去解决。
- 处理者从任务队列中取任务，每次是从队列顶端取任务。如果没有任务，就阻塞等待任务队列发送过来信号告诉你当前有任务了
- 管理者线程就是起着控制线程池中线程的数目的作用，当发现空闲线程数目不合理的话，会对线程池中线程的数目进行一个动态的调整
- 当任务都执行完毕的时候，要对线程池进行一个销毁操作，把锁全部释放掉，结构体释放掉还有任务队列也要释放掉



### 实现

```c
/*************************************************************************
	> File Name: threadpool.c
	> Author: 
	> Mail: 
	> Created Time: 2019年10月16日 星期三 11时12分28秒
 ************************************************************************/

#include<stdio.h>
#include<stdlib.h>
#include<string.h>
#include<pthread.h>
#include<unistd.h>
#include<pthread.h>
#include<signal.h>
#include<errno.h>

# define MIN_WAIT_TASK_NUM 10
# define TOT_PTHREAD 15
# define TOT_QUEUE 11
# define DEFAULT_TIME 10
// 每次创建和销毁线程的数目
# define DEFAULT_THREAD_VARY 10

typedef struct{
    void *(*function)(void *);
    void *arg;
}threadpool_task_t;

struct threadpool_t{
// mutex为互斥锁，只要有一方获取了锁，另一方不能继续获取
pthread_mutex_t lock; // 结构体的锁
pthread_mutex_t thread_counter; // 控制忙碌的线程的数目
// cond为条件变量，自动阻塞一个线程，直到满足特定情况的发生
// 在这里这个信号量值得是任务队列是否满了
pthread_cond_t queue_not_full; 
// 通知客户端此时线程池又新加了线程任务
pthread_cond_t queue_not_empty;

pthread_t *threads;  // 存储线程池中每个线程的tid
pthread_t adjust_tid; // 管理者线程

threadpool_task_t *task_queue; // 通过数组的形式，存储每个线程的回调函数和参数

int max_thr_num; // 线程池最大线程数目
int min_thr_num; // 线程池最小线程数目
int live_thr_num; // 当前存活的线程个数
int busy_thr_num; // 当前忙碌状态线程个数
int wait_exit_thr_num; // 要被销毁的线程数目

int queue_front; // 循环队列的定部
int queue_rear; // 循环队列的底部
int queue_size; // 循环队列实际的任务数
int queue_max_size; // 循环队列可容纳任务数上限

int shut; //标志位，线程池使用状态
};

// 这一步的作用是将 struct threadpool 简化成 threadpool
typedef struct threadpool_t threadpool_t;

threadpool_t *pthread_pool_init(int min_thr_num, int max_thr_num, int queue_max_size);
void threadpool_add(threadpool_t *pool, void *(*function)(void *arg), void *arg);
int threadpool_destroy(threadpool_t *pool);
void *threadpool_thread(void *threadpool);
void *adjust_thread(void *threadpool);
int is_thread_alive(pthread_t tid);
int threadpool_free(threadpool_t *pool);

threadpool_t *pthread_pool_init(int min_thr_num, int max_thr_num, int 
                                  queue_max_size){
   int i;
    threadpool_t *pool = NULL;
   do{
     //申请线程池结构体
     if((pool = (threadpool_t *)malloc(sizeof(threadpool_t))) == NULL){
        printf("malloc threadpool fail");
        break;
     }
     // 初始化结构体信息
     pool->min_thr_num = min_thr_num;
     pool->max_thr_num = max_thr_num;
     pool->live_thr_num = min_thr_num;
     pool->busy_thr_num = 0;
     pool->wait_exit_thr_num = 0;
     
     pool->queue_max_size = queue_max_size;
     pool->queue_size = 0;
     pool->queue_front = 0;
     pool->queue_rear = 0;

     pool->shut = 0;
    
     // 对线程池中的记录线程tid的数组进行空间的申请 
     pool->threads = (pthread_t *)malloc(sizeof(pthread_t) * max_thr_num);
     if(pool->threads == NULL){
        printf("malloc thread fail");
        break;
    }
    memset(pool->threads, 0, sizeof(pthread_t) * max_thr_num);
    
    // 对任务队列进行申请
    pool->task_queue = (threadpool_task_t *)malloc(sizeof(threadpool_task_t) * queue_max_size);
    if(pool->task_queue == NULL){
        printf("malloc task_queue fail");
        break;
    }
   
    // 初始化信号量
    if(pthread_mutex_init(&(pool->lock), NULL) != 0 ||
       pthread_mutex_init(&(pool->thread_counter), NULL) != 0 ||
       pthread_cond_init(&(pool->queue_not_empty), NULL) != 0 ||
       pthread_cond_init(&(pool->queue_not_full), NULL) != 0){
        printf("init the mutex or cond error");
        break;
       }
    
    
    for(i = 0; i < min_thr_num; i++){
    // 创建最小线程
        pthread_create(&(pool->threads[i]), NULL, threadpool_thread, (void *)pool);
        printf("start thread 0x%x\n", (unsigned int)pool->threads[i]);    
    }
    // 创建管理者线程
    pthread_create(&(pool->adjust_tid), NULL, adjust_thread, (void *)pool);
    
    return pool;
   }while(0);
   
   threadpool_free(pool);
   return NULL;
    
};

void *threadpool_thread(void *threadpool){
    
     threadpool_t *pool = (threadpool_t *)threadpool;
     threadpool_task_t task;
    
    while(1){
        // 等待任务队列中加入任务后才执行接受程序
        pthread_mutex_lock(&(pool->lock));
        
        // 当queue_size为0的时候，说明此时任务队列中没有可执行的任务
        while((pool->queue_size == 0) &&(!pool->shut)){
            printf("thread 0x%x is waiting\n", (unsigned int)pthread_self());
        // 释放获得的锁，等待queue_not_empty信号的到来
            pthread_cond_wait(&(pool->queue_not_empty), &(pool->lock));
        
        if(pool->wait_exit_thr_num > 0){
           pool->wait_exit_thr_num--;
        // 将管理者线程认为是空闲的线程可以注销掉   
           if(pool->live_thr_num > pool->min_thr_num){
            printf("thread 0x%x is exiting\n", (unsigned int)pthread_self());
            pool->live_thr_num--;
            pthread_mutex_unlock(&(pool->lock));
            pthread_exit(NULL);
           } 
        }
      }
      
     // printf("1111222222\n");
      if(pool->shut){
        pthread_mutex_unlock(&(pool->lock));
        printf("thread 0x%x is exiting\n", (unsigned int)pthread_self());
        pthread_exit(NULL);
      }
      
      task.function = pool->task_queue[pool->queue_front].function;
      task.arg = pool->task_queue[pool->queue_front].function;
      // 从队列顶端取任务
      pool->queue_front = (pool->queue_front + 1) % pool->queue_max_size;
      pool->queue_size--;
      
      pthread_cond_broadcast(&(pool->queue_not_full));
      pthread_mutex_unlock(&(pool->lock));
      printf("thread 0x%x start working\n", (unsigned int)pthread_self());
      
      pthread_mutex_lock(&(pool->thread_counter));
      pool->busy_thr_num++;
      pthread_mutex_unlock(&(pool->thread_counter));

      (*(task.function))(task.arg);
      printf("thread 0x%x end working\n",(unsigned int)pthread_self());
      pthread_mutex_lock(&(pool->thread_counter));
      pool->busy_thr_num--;
      pthread_mutex_unlock(&(pool->thread_counter));
    }
    pthread_exit(NULL);
    
   // return NULL;
}
void threadpool_add(struct threadpool_t *pool, void *(*function)(void *arg), void *arg){
    // 首先拿到锁
    pthread_mutex_lock(&(pool->lock));
    
    while((pool->queue_size == pool->queue_max_size) &&(!pool->shut)){ 
    // 当任务队列满的时候，是无法再向队列中添加任务的，这个时候要等待任务队列非满的信号    
    pthread_cond_wait(&(pool->queue_not_full), &(pool->lock));
    }
    if(pool->shut){
        pthread_mutex_unlock(&(pool->lock));
    }
    // 清除上次留下的印记
    if(pool->task_queue[pool->queue_rear].arg != NULL){
        free(pool->task_queue[pool->queue_rear].arg);
        pool->task_queue[pool->queue_rear].arg = NULL;
    }
    
    pool->task_queue[pool->queue_rear].function = function;
    pool->task_queue[pool->queue_rear].arg = arg;
    
    pool->queue_rear = (pool->queue_rear + 1) % pool->queue_max_size;
    pool->queue_size++;
    // 发出任务队列中有任务的信号
    pthread_cond_signal(&(pool->queue_not_empty));
    pthread_mutex_unlock(&(pool->lock));
    
    return ;
}

void *adjust_thread(void *threadpool){

    int i;
    struct threadpool_t *pool = (threadpool_t *)threadpool;
    while(!pool->shut){
      // 定时检测  
      sleep(DEFAULT_TIME);
      
      pthread_mutex_lock(&(pool->lock));
       int queue_size = pool->queue_size;
       int live_thr_num = pool->live_thr_num;
    //   int max_thr_num = pool->max_thr_num;
      pthread_mutex_unlock(&(pool->lock));
      
      pthread_mutex_lock(&(pool->thread_counter));
      int busy_thr_num = pool->busy_thr_num;
      pthread_mutex_unlock(&(pool->thread_counter));
      
      if(queue_size >= MIN_WAIT_TASK_NUM && live_thr_num < pool->max_thr_num){
        pthread_mutex_lock(&(pool->lock));
        
        int add = 0;
	//　当线程不够的时候，进行添加操作
        for(i = 0; i < pool->max_thr_num && add < DEFAULT_THREAD_VARY &&
        pool->live_thr_num < pool->max_thr_num; i++){
            if(pool->threads[i] == 0 || !is_thread_alive(pool->threads[i])){
                pthread_create(&(pool->threads[i]), NULL, threadpool_thread, (void *)pool);
                printf("start 0x%x is waiting\n", (unsigned int)pthread_self());
                add++;
                pool->live_thr_num++;
            }
        }
        
        pthread_mutex_unlock(&(pool->lock));
      }
      
      if(busy_thr_num * 2 < live_thr_num && live_thr_num > pool->min_thr_num){
        // 当线程过多的时候，进行销毁操作
        pthread_mutex_lock(&(pool->lock));
        pool->wait_exit_thr_num = DEFAULT_THREAD_VARY;
        pthread_mutex_unlock(&(pool->lock));
        
        for(i = 0; i < DEFAULT_THREAD_VARY; i++){
            // 通知线程池去把这个线程处理掉
            pthread_cond_signal(&(pool->queue_not_empty));
        }
      }
    }
    
    return NULL;
}
// 判断线程是否还存活
int is_thread_alive(pthread_t tid){
    
    int kill_rc = pthread_kill(tid, 0);
    if(kill_rc == ESRCH){
        return 0;
    }
    return 1;
}

int threadpool_destroy(threadpool_t *pool){

    int i;
    if(pool == NULL){
        return -1;
    }
    pool->shut = 1;
    // 以阻塞的方式等待thread指定的线程结束
    pthread_join(pool->adjust_tid, NULL);
    
    for(i = 0; i < pool->live_thr_num; i++){
        pthread_cond_broadcast(&(pool->queue_not_empty));
    }
    for(i = 0; i < pool->live_thr_num; i++){
	// 对存活的线程进行回收
        pthread_join(pool->threads[i], NULL);
    }
    threadpool_free(pool);
    
    return 0;
}

int threadpool_free(threadpool_t *pool){

    if(pool == NULL)
        return -1;
    
    if(pool->task_queue){
        free(pool->task_queue);
    }
    
    if(pool->threads){
        free(pool->threads);
        pthread_mutex_lock(&(pool->lock));
        pthread_mutex_unlock(&(pool->lock));
        pthread_mutex_lock(&(pool->thread_counter));
        pthread_mutex_unlock(&(pool->lock));
        pthread_cond_destroy(&(pool->queue_not_empty));
        pthread_cond_destroy(&(pool->queue_not_full));
    }
    free(pool);
    pool = NULL;
    return 0;
}
void *process(void *arg){
    
   // int tmp = (void *)arg;
    // 模拟添加的任务功能
   // cout << (int)arg <<"----------------" << endl;
    printf("thread 0x%x working on task %d\n",(unsigned int)pthread_self(), *(int *)arg);
    sleep(1);
    printf("task %d is over\n", *(int *)arg);
    
    return NULL;
}

int main(){
    // 初始化，池中最小线程数目为3，最大线程为100，任务队列容量最大为100
     threadpool_t *thp = pthread_pool_init(3, 100, 100);
    // printf("%d\n", thp->max_thr_num);
    printf("pool init ok\n");
    
    int num[40 + 10], i;

    for(i = 0; i < 20; i++){
        num[i] = i;
        printf("add task %d\n", i);
	//　向线程池中添加任务
        threadpool_add(thp, process, (void *)&num[i]); 
    }
    
    sleep(10);
    // 线程池使用完毕，销毁线程池
    threadpool_destroy(thp);
    
    return 0;
}



```



