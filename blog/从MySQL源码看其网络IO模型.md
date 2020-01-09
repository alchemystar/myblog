# 从MySQL源码看其网络IO模型
## 前言
MySQL是当今最流行的开源数据库，阅读其源码是一件大有裨益的事情(虽然其代码感觉比较凌乱)。而笔者阅读一个Server源码的习惯就是先从其网络IO模型看起。于是，便有了本篇博客。
## MySQL启动Socket监听
看源码，首先就需要找到其入口点，mysqld的入口点为mysqld\_main,跳过了各种配置文件的加载
之后，我们来到了network\_init初始化网络环节,如下图所示:    
![repeatable_read](/Users/alchemystar/image/mysql_net_io/socket_create.png)   
下面是其调用栈:

```
mysqld_main (MySQL Server Entry Point)
	|-network_init (初始化网络)
		/* 建立tcp套接字 */
		|-create_socket (AF_INET)
		|-mysql_socket_bind (AF_INET)
		|-mysql_socket_listen (AF_INET)
		/* 建立UNIX套接字*/
		|-mysql_socket_socket (AF_UNIX)
		|-mysql_socket_bind (AF_UNIX)
		|-mysql_socket_listen (AF_UNIX)
```
值得注意的是，在tcp socket的初始化过程中，考虑到了ipv4/v6的两种情况:

```
// 首先创建ipv4连接
ip_sock= create_socket(ai, AF_INET, &a);
// 如果无法创建ipv4连接，则尝试创建ipv6连接
if(mysql_socket_getfd(ip_sock) == INVALID_SOCKET)
 	ip_sock= create_socket(ai, AF_INET6, &a);
```
如果我们以很快的速度stop/start mysql,会出现上一个mysql的listen port没有被release导致无法当前mysql的socket无法bind的情况，在此种情况下mysql会循环等待，其每次等待时间为当前重试次数retry * retry/3 +1秒,一直到设置的--port-open-timeout(默认为0)为止,如下图所示:
![repeatable_read](/Users/alchemystar/image/mysql_net_io/bind_retry_out.png)  

## MySQL新建连接处理循环
通过handle\_connections\_sockets处理MySQL的新建连接循环,根据操作系统的配置通过poll/select处理循环(非epoll,这样可移植性较高，且mysql瓶颈不在网络上)。   
MySQL通过线程池的模式处理连接(一个连接对应一个线程，连接关闭后将线程归还到池中)   
如下图所示:
![repeatable_read](/Users/alchemystar/image/mysql_net_io/mysql_handle_accept.png)  
对应的调用栈如下所示: 

```
handle_connections_sockets
	|->poll/select
	|->new_sock=mysql_socket_accept(...sock...) /*从listen socket中获取新连接*/
	|->new THD 连接线程上下文 /* 如果获取不到足够内存，则shutdown new_sock*/
	|->mysql_socket_getfd(sock) 从socket中获取
		/** 设置为NONBLOCK和环境有关 **/
	|->fcntl(mysql_socket_getfd(sock), F_SETFL, flags | O_NONBLOCK);
	|->mysql_socket_vio_new
		|->vio_init (VIO_TYPE_TCPIP)
			|->(vio->write = vio_write)
			/* 默认用的是vio_read */
			|->(vio->read=(flags & VIO_BUFFERED_READ) ?vio_read_buff :vio_read;)
			|->(vio->viokeepalive = vio_keepalive) /*tcp层面的keepalive*/
			|->.....
	|->mysql_net_init
		|->设置超时时间，最大packet等参数
	|->create_new_thread(thd) /* 实际是从线程池拿，不够再新建pthread线程 */
		|->最大连接数限制
		|->create_thread_to_handle_connection
			|->首先看下线程池是否有空闲线程
				|->mysql_cond_signal(&COND_thread_cache) /* 有则发送信号 */
			/** 这边的hanlde_one_connection是mysql连接的主要处理函数 */
			|->mysql_thread_create(...handle_one_connection...)			
```
## MySQL的VIO
如上图代码中，每新建一个连接，都随之新建一个vio(mysql\_socket\_vio\_new->vio\_init),在vio\_init的过程中，初始化了一堆回掉函数,如下图所示:
![repeatable_read](/Users/alchemystar/image/mysql_net_io/vio_init.png)  
我们关注点在vio\_read和vio\_write上,如上面代码所示，在笔者所处机器的环境下将MySQL连接的socket设置成了非阻塞模式(O\_NONBLOCK)模式。所以在vio的代码里面采用了nonblock代码的编写模式,如下面源码所示:     
### vio_read
```
size_t vio_read(Vio *vio, uchar *buf, size_t size)
{
  while ((ret= mysql_socket_recv(vio->mysql_socket, (SOCKBUF_T *)buf, size, flags)) == -1)
  {
    ......
    // 如果上面获取的数据为空，则通过select的方式去获取读取事件，并设置超时timeout时间
    if ((ret= vio_socket_io_wait(vio, VIO_IO_EVENT_READ)))
        break;
  }
}
```
即通过while循环去读取socket中的数据，如果读取为空，则通过vio\_socket\_io\_wait去等待(借助于select的超时机制),其源码如下所示:

```
vio_socket_io_wait
	|->vio_io_wait
		|-> (ret= select(fd + 1, &readfds, &writefds, &exceptfds, 
              (timeout >= 0) ? &tm : NULL))

```
笔者在jdk源码中看到java的connection time out也是通过这,select(...wait_time)的方式去实现连接超时的。    
由上述源码可以看出,这个mysql的read\_timeout是针对每次socket recv(而不是整个packet的)，所以可能出现超过read\_timeout MySQL仍旧不会报错的情况，如下图所示:
![repeatable_read](/Users/alchemystar/image/mysql_net_io/more_read_timeout.png)
### vio_write
vio\_write实现模式和vio\_read一致，也是通过select来实现超时时间的判定,如下面源码所示:

```
size_t vio_write(Vio *vio, const uchar* buf, size_t size)
{
  while ((ret= mysql_socket_send(vio->mysql_socket, (SOCKBUF_T *)buf, size, flags)) == -1)
  {
    int error= socket_errno;

    /* The operation would block? */
    // 处理EAGAIN和EWOULDBLOCK返回，NON_BLOCK模式都必须处理
    if (error != SOCKET_EAGAIN && error != SOCKET_EWOULDBLOCK)
      break;

    /* Wait for the output buffer to become writable.*/
    if ((ret= vio_socket_io_wait(vio, VIO_IO_EVENT_WRITE)))
      break;
  }
}

```
## MySQL的连接处理线程
从上面的代码:

```
mysql_thread_create(...handle_one_connection...)
```
可以发现，MySQL每个线程的处理函数为handle\_one\_connection,其过程如下图所示:

![repeatable_read](/Users/alchemystar/image/mysql_net_io/conn_handle_loop.png)  
代码如下所示:     

```
for(;;){
	// 这边做了连接的handshake和auth的工作
	rc= thd_prepare_connection(thd);
	// 和通常的线程处理一样，一个无限循环获取连接请求
	while(thd_is_connection_alive(thd))
	{
		if(do_command(thd))
			break;
	}
	// 出循环之后，连接已经被clientdu端关闭或者出现异常
	// 这边做了连接的销毁动作
	end_connection(thd);
end_thread:
	...
	// 这边调用end_thread做清理动作，并将当前线程返还给线程池重用
	// end_thread对应为one_thread_per_connection_end
	if (MYSQL_CALLBACK_ELSE(thread_scheduler, end_thread, (thd, 1), 0))
		return;	
	...
	// 这边current_thd是个宏定义，其实是current_thd();
	// 主要是从线程上下文中获取新塞进去的thd
	// my_pthread_getspecific_ptr(THD*,THR_THD);
	thd= current_thd;
	...
}
```
mysql的每个woker线程通过无限循环去处理请求。
### 线程的归还过程
MySQL通过调用one\_thread\_per\_connection\_end(即上面的end\_thread)去归还连接。

```
MYSQL_CALLBACK_ELSE(...end_thread)
	one_thread_per_connection_end
		|->thd->release_resources()
		|->......
		|->block_until_new_connection
```
线程在新连接尚未到来之前，等待在信号量上(下面代码是C/C++ mutex condition的标准使用模式):   

```
static bool block_until_new_connection()
{	
	mysql_mutex_lock(&LOCK_thread_count);
	......
    while (!abort_loop && !wake_pthread && !kill_blocked_pthreads_flag)
      mysql_cond_wait(&x1, &LOCK_thread_count);
   ......
   // 从等待列表中获取需要处理的THD
   thd= waiting_thd_list->front();
   waiting_thd_list->pop_front();
   ......
   // 将thd放入到当前线程上下文中
   // my_pthread_setspecific_ptr(THR_THD,  this)    
   thd->store_globals();
   ......
   mysql_mutex_unlock(&LOCK_thread_count);
   .....
}
```
整个过程如下图所示:
![repeatable_read](/Users/alchemystar/image/mysql_net_io/thread_pool.png
)  
由于MySQL的调用栈比较深，所以将thd放入线程上下文中能够有效的在调用栈中减少传递参数的数量。
# 总结
MySQL的网络IO模型采用了经典的线程池技术，虽然性能上不及reactor模型，但好在其瓶颈并不在网络IO上，采用这种方法无疑可以节省大量的精力去专注于处理sql等其它方面的优化。