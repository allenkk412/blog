---
title: Nginx事件管理--定时器事件
date: 2018-01-20 00:53:59
tags: nginx
---
## Nginx的时间管理
- Nginx出于性能的考虑（不需要每次获取时间都调用gettimeofday方法），Nginx使用的时间**缓存在其内存中**;

- Nginx每个进程都会单独地管理当前时间;
- 缓存时间的更新    
1. worker进程nginx启动时更新一次
2. **只能通过ngx_epoll_process_events方法执行，此时flags参数中有NGX_UPDATE_TIME标志位或者ngx_event_timer_alarm标志位位1，就会调用ngx_time_update方法更新缓存**

```c
static ngx_int_t  
ngx_epoll_process_events(ngx_cycle_t *cycle, ngx_msec_t timer, ngx_uint_t flags)  
{  
...  
    events = epoll_wait(ep, event_list, (int) nevents, timer);  
  
    err = (events == -1) ? ngx_errno : 0;  
  
    if (flags & NGX_UPDATE_TIME || ngx_event_timer_alarm) {  
        ngx_time_update();  
    }  
...  
}  

```
 
定义了ngx_time_t结构体是缓存时间变量的类型，以及一系列全局变量用来缓存时间

```c
typedef struct {
    time_t        sec;
    // sec成员精确到秒，msec则是当前时间相对于sec的毫秒偏移量
    ngx_uint_t    msec;
    // 时区
    ngx_int_t     gmtoff;
} ngx_time_t;


// 格林威治时间1970年1月1日凌晨0点0分到当前时间的毫秒数
volatile ngx_msec_t ngx_current_msec;
...
```

## 定时事件的处理

### 超时事件对象的超时检测--两种方案

- **定时检测机制**：通过设定定时器，每过一定时间就对红黑树管理的所有超时时间进行一次超级扫描
- 计算出当前最快发生超时的时间是多久，然后将这个超时差值作为epoll_wait的阻塞时间。然后等待这个时间之后再去进行一次超时检测

```c
if (ngx_timer_resolution) {
        timer = NGX_TIMER_INFINITE;
        flags = 0;

    } else {
    //找到当前红黑树当中的最小的事件，传递给epoll_wait，保证epoll可以该时间内可以超时，可以使得超时的事件得到处理
        timer = ngx_event_find_timer();
        flags = NGX_UPDATE_TIME;
    }
```


#### 方案一

当ngx_timer_resolution不为0时，即方案一timer为无限大。timer在函数ngx_process_events()内被用作事件机制被阻塞的最长时间；

因为正常情况下事件处理机制会监控到某些I/O事件的发生。即便是服务器太闲，没有任何I/O事件发生，工作进程也不会无限等待。因为工作进程一开始就设置好了一个定时器，这实现在初始化函数ngx_event_process_init()内：
```c
static ngx_int_t  
ngx_event_process_init(ngx_cycle_t *cycle)  
{  
        ...  
        sa.sa_handler = ngx_timer_signal_handler;  
        sigemptyset(&sa.sa_mask);  
        itv.it_interval.tv_sec = ngx_timer_resolution / 1000;  
        itv.it_interval.tv_usec = (ngx_timer_resolution % 1000) * 1000;  
        itv.it_value.tv_sec = ngx_timer_resolution / 1000;  
        itv.it_value.tv_usec = (ngx_timer_resolution % 1000 ) * 1000;  
  
        if (setitimer(ITIMER_REAL, &itv, NULL) == -1) {  
            ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,  
                          "setitimer() failed");  
        }  
        ....  
}  
```
**定时时间到了以后，执行回调函数ngx_timer_signal_handler：**
```c
static void  
ngx_timer_signal_handler(int signo)  
{  
    ngx_event_timer_alarm = 1;  
    ...
}  
```
可以看出它仅仅是将标志变量ngx_event_timer_alarm 设置为1。
只有在ngx_event_timer_alarm 为1 的情况下，工作进程才会更新时间。

```c
static ngx_int_t  
ngx_epoll_process_events(ngx_cycle_t *cycle, ngx_msec_t timer, ngx_uint_t flags)  
{  
...  
    events = epoll_wait(ep, event_list, (int) nevents, timer);  
  
    err = (events == -1) ? ngx_errno : 0;  
  
    if (flags & NGX_UPDATE_TIME || ngx_event_timer_alarm) {  
        ngx_time_update();  
    }  
...  
}  
```

在方案一的情况下，||前面的式子为假，那么ngx_event_timer_alarm 不为1 的情况下，更新函数ngx_time_update()不会被执行。那么会导致超时检测函数ngx_event_expire_timers不会被执行。
```c
void  
ngx_process_events_and_timers(ngx_cycle_t *cycle)  
{  
...  
    delta = ngx_current_msec;  
    (void) ngx_process_events(cycle, timer, flags);//事件处理函数  
    delta = ngx_current_msec - delta;  
...  
if (delta) {  
        ngx_event_expire_timers();//超时检测函数  
    }  
...  
}  
```

#### 方案二

1. 在worker进程的每一次循环中都会调用ngx_process_events_and_timers函数，在该函数中就会调用处理定时器的函数ngx_event_expire_timers;
2. ngx_event_expire_timers函数不断的从红黑树中取出时间值最小的，查看他们是否已经超时，然后执行他们的函数，直到取出的节点的时间没有超时为止。



方案二
```c
void
ngx_process_events_and_timers(ngx_cycle_t *cycle)
{
    
    ngx_uint_t  flags;
    ngx_msec_t  timer, delta;

    if (ngx_timer_resolution) {
        timer = NGX_TIMER_INFINITE;
        flags = 0;

    } else {
    //找到当前红黑树当中的最小的事件，传递给epoll_wait，保证epoll可以该时间内可以超时，可以使得超时的事件得到处理
        timer = ngx_event_find_timer();
        flags = NGX_UPDATE_TIME;
    }
    .....
    // 在此处竞争accept锁
    // ngx_trylock_accept_mutex(cycle)
    .....
    
    delta = ngx_current_msec;
    (void) ngx_process_events(cycle, timer, flags);
    delta = ngx_current_msec - delta;
    
    
    if (delta) {
        ngx_event_expire_timers();
    }
}

```
调用ngx_event_find_timer()，若timer > 0，则表示还未超时，返回最快的超时时间与当前时间的差值timer

这里的timer会传给ngx_process_events(cycle, timer, flags)，作为epoll_wait的阻塞时间
```c
ngx_msec_t
ngx_event_find_timer(void)
{
    ngx_msec_int_t      timer;
    ngx_rbtree_node_t  *node, *root, *sentinel;

    if (ngx_event_timer_rbtree.root == &ngx_event_timer_sentinel) {
        return NGX_TIMER_INFINITE;
    }

    ngx_mutex_lock(ngx_event_timer_mutex);

    root = ngx_event_timer_rbtree.root;
    sentinel = ngx_event_timer_rbtree.sentinel;
    
    //找到红黑树中key最小的节点
    node = ngx_rbtree_min(root, sentinel);  

    ngx_mutex_unlock(ngx_event_timer_mutex);

    timer = (ngx_msec_int_t) (node->key - ngx_current_msec);
    
    // 若timer > 0，则表示还未超时，返回最快的超时时间与当前时间的差值
    // 这里的timer会传给ngx_process_events(cycle, timer, flags)，作为epoll_wait的阻塞时间
    return (ngx_msec_t) (timer > 0 ? timer : 0);
}
```
 NGX_UPDATE_TIME 被置位，更新时间
```c
static ngx_int_t  
ngx_epoll_process_events(ngx_cycle_t *cycle, ngx_msec_t timer, ngx_uint_t flags)  
{  
...  
    events = epoll_wait(ep, event_list, (int) nevents, timer);  
  
    err = (events == -1) ? ngx_errno : 0;  
  
    if (flags & NGX_UPDATE_TIME || ngx_event_timer_alarm) {  
        ngx_time_update();  
    }  
...  
}  

```

在ngx_process_events(cycle, timer, flags)处理完epoll_wait网络事件后（或阻塞timer时间后），判断是否到达超时时间，然后执行超时事件扫描处理


```c
void
ngx_process_events_and_timers(ngx_cycle_t *cycle)
{
    .....
    
    delta = ngx_current_msec;
    (void) ngx_process_events(cycle, timer, flags);
    delta = ngx_current_msec - delta;
    
    if (delta) {
        ngx_event_expire_timers();
    }
}

```



for( ; ; ) 循环，处理所有的超时事件，处理完则 break 跳出；

超时事件的处理 ev->handler(ev);

```c
void
ngx_event_expire_timers(void)
{
    ngx_event_t        *ev;
    ngx_rbtree_node_t  *node, *root, *sentinel;

    sentinel = ngx_event_timer_rbtree.sentinel;
    
    // for(;;) 循环，处理所有的超时事件。处理完则 break 跳出
    // 超时事件的处理 ev->handler(ev);
    for ( ;; ) {

        ngx_mutex_lock(ngx_event_timer_mutex);

        root = ngx_event_timer_rbtree.root;

        if (root == sentinel) {
            return;
        }

        node = ngx_rbtree_min(root, sentinel);

        /* node->key <= ngx_current_time */
        // 若node->key <= ngx_current_time，即该节点事件已发生超时

        if ((ngx_msec_int_t) node->key - (ngx_msec_int_t) ngx_current_msec <= 0)
        {
            ev = (ngx_event_t *) ((char *) node - offsetof(ngx_event_t, timer));
            
            ...
            // 在红黑树中删除超时事件
            ngx_rbtree_delete(&ngx_event_timer_rbtree, &ev->timer);

            ngx_mutex_unlock(ngx_event_timer_mutex);

            ev->timer_set = 0;
            
            ...

            ev->timedout = 1;
            ev->handler(ev);
            continue;
        }

        break;
    }

    ngx_mutex_unlock(ngx_event_timer_mutex);
}

```

