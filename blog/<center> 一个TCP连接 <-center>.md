# <center> TCP窗口与定时器 </center>


## TCP窗口
tcp发送窗口由slide\_window(滑动窗口)、congestion\_window(拥塞窗口)两者决定，代码如下:

```
#已发送未确认的字节数=下一个发送序号-最早的未确认序号
off = tp->snd_nxt - tp->snd_una;
#发送窗口为min(当前发送窗口,拥塞窗口)
win = min(tp->snd_wnd, tp->snd_cwnd);
...
#发送长度=发送窗口-已发送未确认字节数
len = min(so->so_snd.sb_cc, win) - off;
```
### 滑动窗口
上面的snd\_wnd、snd\_una、snd\_nxt三个字段组成了滑动窗口。 如下图所示:
![](/Users/alchemystar/tcpimage/tcp_snd.png)

####发送端窗口
发送端窗口随时间滑动图(不考虑重传)例如下所示:
![](/Users/alchemystar/tcpimage/tcp_window.png)
(1) 我们一共需要发送900字节数据。可发送数据为1-500字节，尚未发送数据。假设首先发送400字节的数据。  
(2) 发送了400字节后，对端返回一个ack表示收到200序号之内的数据且窗口通告为500。于是如图示，窗口向前滑动了200字节。当前已发送未确认字节序号为200-400,可发送字节序号为401-700,假设在此尚未发送数据。  
(3) 对端返回一个ack表示收到400序号内的数据且窗口通告为400。于是如图示，窗口向前滑动了200字节。已确认数据序号为1-400，可发送数据为401-700。
####接收端窗口通告
snd\_wnd此字段主要由接收端的窗口通告决定，接收端窗口通告由当前接收端剩余多少空闲的剩余缓存决定。如下图所示:
![](/Users/alchemystar/tcpimage/tcp_recv.png)
(1)发送端: 写入2KB的数据[seq=0]。       
(2)接收端: 收到数据,初始化接收端缓冲区4K,写入后还剩2K,于是通告ack[seq=2048,win=2048]。  
(3)发送端: 接收到窗口通告为2048,于是最多只能写入2K的数据，将2K数据写入[seq=2048]。  
(4)接收端: 应用层尚未消费缓冲区。接收到2K数据后，缓冲区满。于是通告窗口为0,返回ack[seq=4096,win=0]。  
(5)发送端: 由于发送窗口为0，不能发送任何数据。此时发送端就需要定时的发送0字节的数据去探测接收端窗口。所需的定时器即为持续定时器(TCPT_PERSIST)。  
(6)发送端: 发送0字节的探测数据。  
(7)接收端: 缓冲区满,窗口通告为0,ack[seq=4096,win=0]。  
(8)发送端: 继续发送0字节的探测数据。  
(9)接收端: 缓冲区被应用层消费了2K,缓冲区可用字节为2K,通告窗口为2048,ack[seq=4096,win=2048]。  
(10)发送端: 继续写入1K的数据。  
......

接收端: 
### 拥塞窗口
tcp用拥塞窗口(cwnd)来进行拥塞控制，主要利用了慢启动、拥塞避免、快速恢复这三个算法。
#### 1）慢启动和拥塞避免  
慢启动和拥塞避免共用同一种机制。区别为:  
1>慢启动是在tcp刚建立连接、开始发送数据时候发生。    
2>拥塞避免是检测到拥塞时刻发生。(检测拥塞的方法有多种，例如重传定时器超时、乱序的ack等)。    
拥塞避免假定造成网络拥塞的原因是本地数据发送太快，从而采取降低发送窗口大小的措施。
代码如下所示:

```
{
	u_int win = min(tp->snd_wnd, tp->snd_cwnd) / 2 / tp->t_maxseg;
	if (win < 2)
		win = 2;
	tp->snd_cwnd = tp->t_maxseg;
	tp->snd_ssthresh = win * tp->t_maxseg;
	tp->t_dupacks = 0;
}
```
其将win置为现有窗口的大小,同时慢启动门限tp->snd\_ssthresh设置为现有窗口大小的一半。snd_cwnd(拥塞窗口)被设定为只能容纳一个报文，这样就强迫TCP执行慢启动。之后拥塞窗口会先以指数形式增长，达到慢启动门限snd\_ssthressh之后,再线性增长。此过程如以下代码注释所示:

```
/*
* When new data is acked, open the congestion window.
* If the window gives us less than ssthresh packets
* in flight, open exponentially (maxseg per packet).
* Otherwise open linearly: maxseg per window
* (maxseg * (maxseg / cwnd) per packet).
*/
{
	register u_int cw = tp->snd_cwnd;
	register u_int incr = tp->t_maxseg;

	if (cw > tp->snd_ssthresh)
		incr = incr * incr / cw;
	tp->snd_cwnd = min(cw + incr, TCP_MAXWIN<<tp->snd_scale);
}

```		
慢启动图例:
![](/Users/alchemystar/tcpimage/tcp_slowstart.png)
值得注意的是，TCP连接刚建立时刻也会有慢启动的过程。如果用的是短连接(即发送一个请求之后即抛弃此连接)且发送数据较少的话，大部分时间都耗在了慢启动上面，并没有充分的利用带宽。再加上建立连接所需要三次握手的消耗,导致短连接的效率要远低于长连接。


#### 2)快速恢复
![](/Users/alchemystar/tcpimage/tcp_quick_recv.png)


## TCP定时器
### 连接建立(connection establishment)定时器

### 重传(retransmission)定时器
### 延迟ACK(delayed ack)定时器
### 持续(persist)定时器
### 保活(keepalive)定时器
### FIN\_WAIT\_2定时器
### TIME\_WAIT定时器

