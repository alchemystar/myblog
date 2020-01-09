#解Bug之路-Druid的Bug
笔者很热衷于解决Bug,同时比较擅长(网络/协议)部分，所以经常被唤去解决一些网络IO方面的Bug。现在就挑一个案例出来，写出分析思路，以飨读者，希望读者在以后的工作中能够少踩点坑。   
#前言
此Bug是Druid低版本的Bug,此Bug至少在1.0.12版本就已经修复。 
#Druid的Bug现场
在紧张的新项目开发的日子里,突然收到线上某系统的大量报警,对应系统的人员发现此系统在某一台机器上dump了大量的error日志。日志基本都是:     

```
Druid:  GetConnectionTimeoutException
```  
此系统所有用到数据库的地方都抛出此异常。于是祭出重启大法，重启过后，一切Okay。然后对应的系统人员开始排查这个问题，一直没有结果。      
过了两天，又收到此类型的error日志报警，而且这一次是有两台系统同时爆出此种错误。紧急重启后，将此问题紧急报到我们这边处理。鉴于本人有丰富的IO处理经验，当然落到了本人头上。    
#Bug复盘   
此系统是通过Druid连接后面的数据库分库分表Proxy,再由此Proxy连接后面的数据库。示意图如下所示:    
![codegen](/Users/alchemystar/image/druid_bug/druid_sys_arc.png)     
##缩小Bug范围    
获取连接超时(GetConnectionTimeoutException)此错误的出现，只有两种可能:   

```
1.业务系统本身Druid获取连接失败。    
2.作为中间件的Sharding Proxy获取连接失败。         
```
在这个Bug里面很明显是Druid创建连接失败，原因如下:    

```
1.此系统有10多台机器，仅仅有两台出现此种故障。    
2.这两台重启后一切正常。  
```  
如果说这两台是由于同机房问题出现统一的网络连接异常，那么并不能解释重启后正常这一现象。    
##Druid问题定位   
于是开始分析为何获取连接超时,第一步当然是开始寻找源码中日志抛出异常点。上源码: 
    
```
DruidConnectionHolder holder;
.......
try {
	if (maxWait > 0) {
	   holder = pollLast(nanos);
	}
	else {
	   holder = takeLast();
	}
	......
}finally {
    lock.unlock();
}
if(holder == null){
   ......
	if (this.createError != null) {
        throw new GetConnectionTimeoutException(errorMessage, createError);
    } else {
        throw new GetConnectionTimeoutException(errorMessage);
    }   
}

```    
可见，这边获取到的DruidConnectionHolder为null,则抛出异常。
###Druid获取连接的过程    
在分析这个问题之前，先得看下Druid是如何创建连接的,下面是本人阅读Druid源码后画的示意图:     
![codegen](/Users/alchemystar/image/druid_bug/druid_create_conn.png)     
可见druid创建连接都是通过一个专门的线程来进行的，此图省略了大量的源码细节。     

###为何Holde为null?
继续上源码    

```
private DruidConnectionHolder pollLast(long nanos) throws InterruptedException, SQLException {
    for (;;) {
        if (poolingCount == 0) {
            emptySignal(); // send signal to CreateThread create connection

            if (estimate <= 0) {
                waitNanosLocal.set(nanos - estimate);
                return null;
            }
            ...
            try {
                long startEstimate = estimate;
                estimate = notEmpty.awaitNanos(estimate); 
                ......
            } finally {
              ......
            }
            ......
            if (poolingCount == 0) {
                if (estimate > 0) {
                    continue;
                }
                waitNanosLocal.set(nanos - estimate);
                return null;
            }
        }
        decrementPoolingCount();
        DruidConnectionHolder last = connections[poolingCount];
        connections[poolingCount] = null;
        return last;
    }
}
```
可见，如果触发条件，estimate<=0,则返回null。    
上述源码的过程示意图如下:     
![codegen](/Users/alchemystar/image/druid_bug/druid_code1_show.png)    
###继续追踪    
由此可见，在获取连接的时候一直超时，不停的爆GetConnectionTimeoutException异常，明显是由于创建连接线程出了问题。那到底除了什么问题呢？由于Druid的代码比较多，很难定位问题点，于是还从日志入手。    
##进一步挖掘日志
错误信息量最大的是最初出现错误的时间点，这是笔者多年排查错误的经验总结。由于正好有两台出错，比较其错误的共同点也能对解决问题大有裨益。    
###大量create connection error    
当笔者和同事追查错误的源头的时候，在有大量的create connection error,奇怪的是，到了一个时间点之后，就只剩GetConnectionTimeoutException异常了。   
继续分析，在出现create connection error的时候，还是有部分业务请求能够执行的。即获取到了连接。     
翻了翻源码，create connection error是在创建连接线程打印的，如果这个异常没有继续打印，而同时连接也获取不到，就在很大程度上表明:    


```
创建连接根本就是被销毁了！  
``` 
####错误分界点       
于是开始寻找什么时候create connection error开始销毁。于是通过笔者和同事在无数的错误日志中用肉眼发现了一个不寻常的日志隐蔽在大量的错误日志间:      

```
Druid:create holder error
```    
在这个错误出现之后，就再也没有了create connection error,翻了翻另一台机器的日志，页是同样的现象！    
####源码寻找Bug    
隐隐就感觉这个日志是问题错误的关键，打出这个日志的源码为:    
  
```
创建连接线程
@Override
public void run() {
            runInternal();
}

private void runInternal() {

    for (; ; ) {
        ......
        try{
            holder = new DruidConnectionHolder(DruidDataSource.this,connection);
        }catch(SQLException ex){
            LOG.error("create connection holder error",ex);
            // 这句bread就是罪魁祸首
            // 至少在1.0.12修复
            break;
        }
        ......
    }
}
```
这边竟然在for(;;)这个死循环中break了！！！，那么这个线程也在break后跳出了死循环从而结束了，也就是说创建连接线程被销毁了！！!如下图所示:     
![codegen](/Users/alchemystar/image/druid_bug/Druid_Create_Destory.png) 
###为何create holder error?     
继续搜寻日志,发现create holder error之前，会有获取事务隔离级别的报错。那么结合源码，错误的发生如下图:     
![codegen](/Users/alchemystar/image/druid_bug/Druid_Create_Holder.png)    
即在Druid的创建连接线程创建连接成功之后，还需要拿去数据库的holdability,isolation(隔离级别)等MetaData,在获取MetaData失败的时候，则会抛出异常，导致创建连接线程Break。     
但是如果在handshake的时候就失败，那么由于Druid处理了这种异常，打印create connection error,并继续创建连接。   
于是就有了在create holder error之前大量create connection error，在这之后没有了的现象。     
###完结了么？
Druid的Bug是弄清楚了，但是为何连接如此不稳定，有大量的创建连接异常，甚至与Druid前脚创建连接成功，后脚发送命令就失败呢？    
##Sharding Proxy的Bug
于是此问题又萦绕在笔者心头，在又一番不下于上述过程的努力之后，发现一个月之前上线的新版本的Sharding Proxy的内存泄露Bug导致频繁GC(并定位内存泄露点),导致了上述现象，如下图所示:    
![codegen](/Users/alchemystar/image/druid_bug/Druid_Stop_The_World.png) 
由于每次内存泄露过小，同时Sharding Proxy设置的内存过大。所以上线后过了一个月才有频繁的GC现象。之前上线后，大家观察了一周，发现没有任何异常，就不再关注。与此类似，如果DB负载过高的话，笔者推测也会触发Druid的Bug。   
#最后处理   
笔者去翻Druid最新源码，发现此问题已经修复，紧急联系各业务线升级Druid,同时让Sharding Proxy负责人修改了最新代码并上线。   
终于这次的连环Bug算是填完了。        
#总结   
追查Bug，日志和源码是最重要的两个部分。最源头的日志信息量最大，同时要对任何不同寻常的现象都加以分析并推测，最后结合源码，才能最终找出Bug。
#原文链接













