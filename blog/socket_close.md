# 从linux源码看socket的close
笔者一直觉得如果能知道从应用到框架再到操作系统的每一处代码，是一件Exciting的事情。上篇博客讲了socket的阻塞和非阻塞，这篇就开始谈一谈socket的close(以tcp为例,且基于linux-2.6.24内核版本)   
## TCP关闭状态转移图:
众所周知，TCP的close过程是四次挥手，状态机的变迁也逃不出TCP状态转移图，如下图所示:
![codegen](/Users/alchemystar/image/socket_close/tcp_state_close.png)    
tcp的关闭主要分主动关闭、被动关闭以及同时关闭(特殊情况,不做描述)
## 主动关闭
### close(fd)的过程
以C语言为例，在我们关闭socket的时候，会使用close\(fd\)函数:

```
int    socket_fd;
socket_fd = socket(AF_INET, SOCK_STREAM, 0);
...
// 此处通过文件描述符关闭对应的socket
close(socket_fd)
```
而close\(int fd\)又是通过系统调用sys\_close来执行的:   

```
asmlinkage long sys_close(unsigned int fd)
{
	// 清除(close_on_exec即退出进程时）的位图标记
	FD_CLR(fd, fdt->close_on_exec);
	// 释放文件描述符
	// 将fdt->open_fds即打开的fd位图中对应的位清除
	// 再将fd挂入下一个可使用的fd以便复用
	__put_unused_fd(files, fd);
	// 调用file_pointer的close方法真正清除
	retval = filp_close(filp, files);
}
	
```
我们看到最终是调用的filp_close方法:

```
int filp_close(struct file *filp, fl_owner_t id)
{
	// 如果存在flush方法则flush
	if (filp->f_op && filp->f_op->flush)
		filp->f_op->flush(filp, id);
	// 调用fput	
	fput(filp);
	......
}
```
紧接着我们进入fput:

```
void fastcall fput(struct file *file)
{
	// 对应file->count--,同时检查是否还有关于此file的引用
	// 如果没有，则调用_fput进行释放
	if (atomic_dec_and_test(&file->f_count))
		__fput(file);
}
```
同一个file(socket)有多个引用的情况很常见，例如下面的例子:    
![codegen](/Users/alchemystar/image/socket_close/file_ref.png)     
所以在多进程的socket服务器编写过程中，父进程也需要close(fd)一次，以免socket无法最终关闭


然后就是_fput函数了:

```
void fastcall __fput(struct file *file)
{
	// 从eventpoll中释放file
	eventpoll_release(file);
	// 如果是release方法，则调用release
	if (file->f_op && file->f_op->release)
		file->f_op->release(inode, file);
}
```
由于我们讨论的是socket的close,所以，我们现在探查下file->f_op->release在socket情况下的实现:     
### f_op->release的赋值
我们跟踪创建socket的代码，即

```
socket(AF_INET, SOCK_STREAM, 0);
	|-sock_create  // 创建sock
	|-sock_map_fd  // 将sock和fd关联
			|-sock_attach_fd
					|-init_file(file,...,&socket_file_ops);
							|-file->f_op = fop; //fop赋值为socket_file_ops
```
socket\_file\_ops的实现为:

```
static const struct file_operations socket_file_ops = {
	.owner =	THIS_MODULE,
	......
	// 我们在这里只考虑sock_close
	.release =	sock_close,
	......
};
```
继续跟踪:

```
sock_close
	|-sock_release
		|-sock->ops->release(sock);
```
在上一篇博客中，我们知道sock->ops为下图所示:
![codegen](/Users/alchemystar/image/nonblock/socket_ops_prot.png) 
即(在这里我们仅考虑tcp,即sk\_prot=tcp\_prot):

```
inet_stream_ops->release
	|-inet_release
			|-sk->sk_prot->close(sk, timeout);
				|-tcp_prot->close(sk, timeout);
					|->tcp_prot.tcp_close
```

关于fd与socket的关系如下图所示:   
![codegen](/Users/alchemystar/image/socket_close/fd_socket_close.png) 
上图中红色线标注的是close(fd)的调用链
### tcp_close

```
void tcp_close(struct sock *sk, long timeout)
{
	if (sk->sk_state == TCP_LISTEN) {
		// 如果是listen状态，则直接设为close状态
		tcp_set_state(sk, TCP_CLOSE);
	}
	// 清空掉recv.buffer
	......
	// SOCK_LINGER选项的处理
	......
	else if (tcp_close_state(sk)){
		// tcp_close_state会将sk从established状态变为fin_wait1
		// 发送fin包
		tcp_send_fin(sk);
	}
	......
}

```
### 四次挥手
现在就是我们的四次挥手环节了，其中上半段的两次挥手下图所示:    
![codegen](/Users/alchemystar/image/socket_close/active_close.png) 
首先，在tcp\_close\_state(sk)中已经将状态设置为fin_wait1,并调用tcp\_send\_fin

```
void tcp_send_fin(struct sock *sk)
{
	......
	// 这边设置flags为ack和fin
	TCP_SKB_CB(skb)->flags = (TCPCB_FLAG_ACK | TCPCB_FLAG_FIN);
	......
	// 发送fin包，同时关闭nagle
	__tcp_push_pending_frames(sk, mss_now, TCP_NAGLE_OFF);
}
```
如上图Step1所示。
接着，主动关闭的这一端等待对端的ACK，如果ACK回来了，就设置TCP状态为FIN\_WAIT2,如上图Step2所示,具体代码如下:

```
tcp_v4_do_rcv
	|-tcp_rcv_state_process
int tcp_rcv_state_process(struct sock *sk, struct sk_buff *skb, struct tcphdr *th, unsigned len)	
{
	......
	/* step 5: check the ACK field */
	if (th->ack) {
		...
		case TCP_FIN_WAIT1:
			// 这处判断是确认此ack是发送Fin包对应的那个ack
			if (tp->snd_una == tp->write_seq) {
				// 设置为FIN_WAIT2状态
				tcp_set_state(sk, TCP_FIN_WAIT2);
				......
				// 设定TCP_FIN_WAIT2定时器，将在tmo时间到期后将状态变迁为TIME_WAIT
				// 不过是这时候改的已经是inet_timewait_sock了
				tcp_time_wait(sk, TCP_FIN_WAIT2, tmo);
				......
			}
	}
	/* step 7: process the segment text */
	switch(sk->sk_state) {
	case TCP_FIN_WAIT1:
	case TCP_FIN_WAIT2:
		......
	case TCP_ESTABLISHED:
		tcp_data_queue(sk, skb);
		queued = 1;
		break;
	}
	.....
}
```
值的注意的是，从TCP\_FIN\_WAIT1变迁到TCP\_FIN\_WAIT2之后，还调用tcp\_time\_wait设置一个TCP\_FIN\_WAIT2定时器，在tmo+(2MSL或者基于RTO计算超时)超时后会直接变迁到closed状态(不过此时已经是inet\_timewait\_sock了）。这个超时时间可以配置,如果是ipv4的话,则可以按照下列配置:

```
net.ipv4.tcp_fin_timeout   
/sbin/sysctl -w net.ipv4.tcp_fin_timeout=30
```
如下图所示:
![codegen](/Users/alchemystar/image/socket_close/fin_wait2_expire.png)
有这样一步的原因是防止对端由于种种原因始终没有发送fin,防止一直处于FIN_WAIT2状态。
 
接着在FIN\_WAIT2状态等待对端的FIN，完成后面两次挥手:
![codegen](/Users/alchemystar/image/socket_close/active_close_2.png) 
由Step1和Step2将状态置为了FIN\_WAIT\_2，然后接收到对端发送的FIN之后,将会将状态设置为time_wait,如下代码所示:   

```
tcp_v4_do_rcv
	|-tcp_rcv_state_process
		|-tcp_data_queue
				|-tcp_fin
static void tcp_fin(struct sk_buff *skb, struct sock *sk, struct tcphdr *th)
{
	switch (sk->sk_state) {
		......	
		case TCP_FIN_WAIT1:
			// 这边是处理同时关闭的情况
			tcp_send_ack(sk);
			tcp_set_state(sk, TCP_CLOSING);
			break;
		case TCP_FIN_WAIT2:
			/* Received a FIN -- send ACK and enter TIME_WAIT. */
			// 收到FIN之后，发送ACK同时将状态进入TIME_WAIT
			tcp_send_ack(sk);
			tcp_time_wait(sk, TCP_TIME_WAIT, 0);
	}
}		   
```
time\_wait状态时，原socket会被destroy,然后新创建一个inet\_timewait\_sock,这样就能及时的将原socket使用的资源回收。而inet\_timewait\_sock被挂入一个bucket中，由
inet\_twdr\_twcal\_tick定时从bucket中将超过(2MSL或者基于RTO计算的时间)的time\_wait的实例删除。
我们来看下tcp\_time\_wait函数

```
void tcp_time_wait(struct sock *sk, int state, int timeo)
{
	// 建立inet_timewait_sock
	tw = inet_twsk_alloc(sk, state);
	// 放到bucket的具体位置等待定时器删除
	inet_twsk_schedule(tw, &tcp_death_row, time,TCP_TIMEWAIT_LEN);
	// 设置sk状态为TCP_CLOSE,然后回收sk资源
	tcp_done(sk);
}
```
具体的定时器操作函数为inet\_twdr\_twcal\_tick,这边就不做描述了
## 被动关闭
### close_wait
在tcp的socket时候，如果是established状态，接收到了对端的FIN,则是被动关闭状态,会进入close\_wait状态,如下图Step1所示:   
![codegen](/Users/alchemystar/image/socket_close/passive_close.png) 
具体代码如下所示:    

```
tcp_rcv_state_process
	|-tcp_data_queue
static void tcp_data_queue(struct sock *sk, struct sk_buff *skb)
{
	...
	if (th->fin)
		tcp_fin(skb, sk, th);
	...
}
```
我们再看下tcp_fin

```
static void tcp_fin(struct sk_buff *skb, struct sock *sk, struct tcphdr *th)
{
	......
	// 这一句表明当前socket有ack需要发送
	inet_csk_schedule_ack(sk);
	......
	switch (sk->sk_state) {
			case TCP_SYN_RECV:
			case TCP_ESTABLISHED:
				/* Move to CLOSE_WAIT */
				// 状态设置程close_wait状态
				tcp_set_state(sk, TCP_CLOSE_WAIT);
				// 这一句表明，当前fin可以延迟发送
				// 即和后面的数据一起发送或者定时器到时后发送
				inet_csk(sk)->icsk_ack.pingpong = 1;
				break;
	}
	......
}
``` 
这边有意思的点是，收到对端的fin之后并不会立即发送ack告知对端收到了，而是等有数据携带一块发送,或者等携带重传定时器到期后发送ack。      

如果对端关闭了，应用端在read的时候得到的返回值是0,此时就应该手动调用close去关闭连接

```
if(recv(sockfd, buf, MAXLINE,0) == 0){
	close(sockfd)
}
```
我们看下recv是怎么处理fin包，从而返回0的,上一篇博客可知，recv最后调用tcp\_rcvmsg,由于比较复杂，我们分两段来看:    
tcp\_recvmsg第一段

```
		......
		// 从接收队列里面获取一个sk_buffer
		skb = skb_peek(&sk->sk_receive_queue);
		do {
			// 如果已经没有数据，直接跳出读取循环，返回0
			if (!skb)
				break;
			......
			// *seq表示已经读到多少seq
			// TCP_SKB_CB(skb)->seq表示当前sk_buffer的起始seq
			// offset即是在当前sk_buffer中已经读取的长度
			offset = *seq - TCP_SKB_CB(skb)->seq;
			// syn处理 
			if (tcp_hdr(skb)->syn)
				offset--;
			// 此处判断表示，当前skb还有数据可读，跳转found_ok_skb
			if (offset < skb->len)
				goto found_ok_skb;
			// 处理fin包的情况	
			// offset == skb->len,跳转到found_fin_ok然后跳出外面的大循环
			// 并返回0
			if (tcp_hdr(skb)->fin)
				goto found_fin_ok;
			BUG_TRAP(flags & MSG_PEEK);
			skb = skb->next;
		} while (skb != (struct sk_buff *)&sk->sk_receive_queue);
		......
```
上面代码的处理过程如下图所示:
![codegen](/Users/alchemystar/image/socket_close/sk_buffer_fin_process.png) 
我们看下tcp_recmsg的第二段:

```
found_ok_skb:
		// tcp已读seq更新
		*seq += used;
		// 这次读取的数量更新
		copied += used;
		// 如果还没有读到当前sk_buffer的尽头，则不检测fin标识
		if (used + offset < skb->len)
			continue;
		// 如果发现当前skb有fin标识，去found_fin_ok
		if (tcp_hdr(skb)->fin)
			goto found_fin_ok;
		......
found_fin_ok:
		/* Process the FIN. */
		// tcp已读seq++
		++*seq;
		...
		break;
} while(len > 0);					
```
由上面代码可知，一旦当前skb读完了而且携带有fin标识，则不管有没有读到用户期望的字节数量都会返回已读到的字节数。下一次再读取的时候则在刚才描述的tcp\_rcvmsg上半段直接不读取任何数据再跳转到found\_fin\_ok并返回0。这样应用就能感知到对端已经关闭了。
如下图所示:    
![codegen](/Users/alchemystar/image/socket_close/tcp_recvmsg_buttom.png) 

### last_ack
应用层在发现对端关闭之后已经是close\_wait状态，这时候再调用close的话，会将状态改为last\_ack状态，并发送本端的fin,如下代码所示:   

```
void tcp_close(struct sock *sk, long timeout)
{
	......
	else if (tcp_close_state(sk)){
		// tcp_close_state会将sk从close_wait状态变为last_ack
		// 发送fin包
		tcp_send_fin(sk);
	}
}
```
在接收到主动关闭端的last\_ack之后，则调用tcp\_done(sk)设置sk为tcp_closed状态，并回收sk的资源,如下代码所示:   

```
tcp_v4_do_rcv
	|-tcp_rcv_state_process
int tcp_rcv_state_process(struct sock *sk, struct sk_buff *skb, struct tcphdr *th, unsigned len)	
{
	......
	/* step 5: check the ACK field */
	if (th->ack) {
		...
		case TCP_LAST_ACK:
			// 这处判断是确认此ack是发送Fin包对应的那个ack
			if (tp->snd_una == tp->write_seq) {
					tcp_update_metrics(sk);
					// 设置socket为closed，并回收socket的资源
					tcp_done(sk);
					goto discard;
			}
		...
	}
}	
```
上述代码就是被动关闭端的后两次挥手了,如下图所示:   

![codegen](/Users/alchemystar/image/socket_close/passive_last_ack.png)

### 出现大量close_wait的情况
linux中出现大量close\_wait的情况一般是应用在检测到对端fin时没有及时close当前连接。有一种可能如下图所示: 
![codegen](/Users/alchemystar/image/socket_close/too_many_close_wait.png)
当出现这种情况，通常是minIdle之类参数的配置不对(如果连接池有定时收缩连接功能的话)。给连接池加上心跳也可以解决这种问题。   
如果应用close的时间过晚，对端已经将连接给销毁。则应用发送给fin给对端，对端会由于找不到对应的连接而发送一个RST(Reset)报文。
### 操作系统何时回收close_wait
如果应用迟迟没有调用close\_wait,那么操作系统有没有一个回收机制呢，答案是有的。
tcp本身有一个包活(keep alive)定时器，在(keep alive)定时器超时之后，会强行将此连接关闭。可以设置tcp keep alive的时间
  
```
/etc/sysctl.conf
net.ipv4.tcp_keepalive_intvl = 75
net.ipv4.tcp_keepalive_probes = 9
net.ipv4.tcp_keepalive_time = 7200  
```
默认值如上面所示，设置的很大，7200s后超时，如果想快速回收close\_wait可以设置小一点。但最终解决方案还是得从应用程序着手。    
关于tcp keepalive包活定时器可见笔者另一篇博客:    
https://my.oschina.net/alchemystar/blog/833981
## 进程关闭时清理socket资源
进程在退出时候(无论kill,kill -9 或是正常退出)都会关闭当前进程中所有的fd(文件描述符)

```
do_exit
	|-exit_files
		|-__exit_files
			|-close_files
					|-filp_close
```
这样我们又回到了博客伊始的filp_close函数，对每一个是socket的fd发送send\_fin

## Java GC时清理socket资源
Java的socket最终关联到AbstractPlainSocketImpl,且其重写了object的finalize方法

```
abstract class AbstractPlainSocketImpl extends SocketImpl 
{
	......
    /**
     * Cleans up if the user forgets to close it.
     */
    protected void finalize() throws IOException {
        close()
    }
    ......
}
```
所以Java会在GC时刻会关闭没有被引用的socket,但是切记不要寄希望于Java的GC,因为GC时刻并不是以未引用的socket数量来判断的，所以有可能泄露了一堆socket,但仍旧没有触发GC。
## 总结
linux内核源代码博大精深，阅读其代码很费周折。之前读\<\<TCP/IP详解卷二\>\>的时候由于有先辈引导和梳理，所以看书中所使用的BSD源码并不觉得十分费劲。直到现在自己带着问题独立看linux源码的时候，尽管有之前的基础，仍旧被其中的各种细节所迷惑。希望笔者这篇文章能帮助到阅读linux网络协议栈代码的人。