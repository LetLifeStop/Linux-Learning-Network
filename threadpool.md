线程池的实现

```c
*/
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

