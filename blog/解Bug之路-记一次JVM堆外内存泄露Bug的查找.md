# 解Bug之路-记一次JVM堆外内存泄露Bug的查找
##前言
JVM的堆外内存泄露的定位一直是个比较棘手的问题。此次的Bug查找从堆内内存的泄露反推出堆外内存，同时对物理内存的使用做了定量的分析，从而实锤了Bug的源头。笔者将此Bug分析的过程写成博客，以飨读者。   
由于物理内存定量分析部分用到了linux kernel虚拟内存管理的知识，读者如果有兴趣了解请看ulk3(《深入理解linux内核第三版》)
## 内存泄露Bug现场
一个线上稳定运行了三年的系统，从物理机迁移到docker环境后，运行了一段时间，突然被监控系统发出了某些实例不可用的报警。所幸有负载均衡，可以自动下掉节点，如下图所示:
![gc_local](https://static.oschina.net/uploads/img/201801/08104437_3KUb.png "gc_local")      
登录到对应机器上后，发现由于内存占用太大，触发OOM，然后被linux系统本身给kill了。
## 应急措施
紧急在出问题的实例上再次启动应用，启动后，内存占用正常，一切Okay。
## 奇怪现象
当前设置的最大堆内存是1792M，如下所示:
```
-Xmx1792m -Xms1792m -Xmn900m -XX:PermSi
ze=256m -XX:MaxPermSize=256m -server -Xss512k 
```   
查看操作系统层面的监控，发现内存占用情况如下图所示:
![gc_upper](https://static.oschina.net/uploads/img/201801/08104501_x86B.png "gc_upper")      
上图蓝色的线表示总的内存使用量，发现一直涨到了4G后，超出了系统限制。    
很明显，有堆外内存泄露了。
## 查找线索
### gc日志
一般出现内存泄露，笔者立马想到的就是查看当时的gc日志。   
本身应用所采用框架会定时打印出对应的gc日志，遂查看，发现gc日志一切正常。对应日志如下:
![gc_log](https://static.oschina.net/uploads/img/201801/08104524_biUf.png "gc_log")       
查看了当天的所有gc日志，发现内存始终会回落到170M左右，并无明显的增加。要知道JVM本身占用的内存可是4G，进一步确认是堆外内存导致。
### 排查代码
打开线上服务对应对应代码，查了一圈，发现没有任何地方显式利用堆外内存，其没有依赖任何额外的native方法。关于网络IO的代码也是托管给Tomcat，很明显，作为一个全世界广泛流行的Web服务器，Tomcat不大可能有堆外内存泄露。
## 进一步查找
由于在代码层面没有发现堆外内存的痕迹，那就继续找些其它的信息，希望能发现蛛丝马迹。
### Dump出JVM的Heap堆
由于线上出问题的Server已经被kill，还好有其它几台，登上去发现它们也 占用了很大的堆外内存，只是还没有到触发OOM的临界点而已。于是就赶紧用jmap dump了两台机器中应用JVM的堆情况，这两台留做现场保留不动，然后将其它机器迅速重启，以防同时被OOM导致服务不可用。   
使用如下命令dump:

```
jmap -dump:format=b，file=heap.bin [pid]
```
### 使用MAT分析Heap文件
挑了一个heap文件进行分析，堆的使用情况如下图所示:
![gc_heap_dump](https://static.oschina.net/uploads/img/201801/08104550_ukiC.png "gc_heap_dump")       
一共用了200多M，和之前gc文件打印出来的170M相差不大，远远没有到4G的程度。    
不得不说MAT是个非常好用的工具，它可以提示你可能内存泄露的点:
![gc_cached_bns_client](https://static.oschina.net/uploads/img/201801/08104610_EzHN.png "gc_cached_bns_client")        
这个cachedBnsClient类有12452个实例，占用了整个堆的61.92%。    
查看了另一个heap文件，发现也是同样的情况。这个地方肯定有内存泄露，但是也占用了130多M，和4G相差甚远。   
### 查看对应的代码   
系统中大部分对于CachedBnsClient的调用，都是通过注解Autowired的，这部分实例数很少。   
唯一频繁产生此类实例的代码如下所示: 
    
```
@Override
    public void fun() {
            BnsClient bnsClient = new CachedBnsClient();
          // do something
    		return  ;
	}
```      
此CachedBnsClient仅仅在方法体内使用，并没有逃逸到外面，再看此类本身

```
public class CachedBnsClient   {
    private ConcurrentHashMap<String， List<String>> authCache = new ConcurrentHashMap<String， List<String>>();
    private ConcurrentHashMap<String， List<URI>> validUriCache = new ConcurrentHashMap<String， List<URI>>();
    private ConcurrentHashMap<String， List<URI>> uriCache = new ConcurrentHashMap<String， List<URI>>();
	......
}
```
没有任何static变量，同时也没有往任何全局变量注册自身。换言之，在类的成员(Member)中，是不可能出现内存泄露的。    
当时只粗略的过了一过成员变量，回过头来细想，还是漏了不少地方的。
## 更多信息
由于代码排查下来，感觉这块不应该出现内存泄露(但是事实确是如此的打脸)。这个类也没有显式用到堆外内存，而且只占了130M，和4G比起来微不足道，还是先去追查主要矛盾再说。
### 使用jstack dump线程信息
现场信息越多，越能找出蛛丝马迹。先用jstack把线程信息dump下来看下。
这一看，立马发现了不同，除了正常的IO线程以及框架本身的一些守护线程外，竟然还多出来了12563多个线程。    

```
"Thread-5" daemon prio=10 tid=0x00007fb79426e000 nid=0x7346 waiting on condition [0x00007fb7b5678000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
	at java.lang.Thread.sleep(Native Method)
	at com.xxxxx.CachedBnsClient$1.run(CachedBnsClient.java:62)
```
而且这些正好是运行再CachedBnsClient的run方法上面！这些特定线程的数量正好是12452个，和cachedBnsClient数量一致!
### 再次check对应代码
原来刚才看CachedBnsClient代码的时候遗漏掉了一个关键的点!

```
    public CachedBnsClient(BnsClient client) {
        super();
        this.backendClient = client;
        new Thread() {
            @Override
            public void run() {
                for (; ; ) {
                    refreshCache();
                    try {
                        Thread.sleep(60 * 1000);
                    } catch (InterruptedException e) {
                        logger.error("出错"， e);
                    }
                }
            }
    }
```
这段代码是CachedBnsClient的构造函数，其在里面创建了一个无限循环的线程，每隔60s启动一次刷新一下里面的缓存!   
## 找到关键点
在看到12452个等待在CachedBnsClient.run的业务的一瞬间笔者就意识到，肯定是这边的线程导致对外内存泄露了。下面就是根据线程大小计算其泄露内存量是不是确实能够引起OOM了。
### 发现内存计算对不上
由于我们这边设置的Xss是512K，即一个线程栈大小是512K，而由于线程共享其它MM单元(线程本地内存是是现在线程栈上的)，所以实际线程堆外内存占用数量也是512K。进行如下计算:

```
12563 * 512K = 6331M = 6.3G
```
整个环境一共4G，加上JVM的1.8G(1792M)，已经明显的超过了4G。

```
(6.3G + 1.8G)=8.1G > 4G
```
如果按照此计算，应用应用早就被OOM了。
### 怎么回事呢？
为了解决这个问题，笔者又思考了好久。如下所示:
#### Java线程底层实现
JVM的线程在linux上底层是调用NPTL(Native Posix Thread Library)来创建的，一个JVM线程就对应linux的lwp(轻量级进程，也是进程，只不过共享了mm\_struct，用来实现线程)，一个thread.start就相当于do\_fork了一把。    
其中，我们在JVM启动时候设置了-Xss=512K(即线程栈大小)，这512K中然后有8K是必须使用的，这8K是由进程的内核栈和thread\_info公用的，放在两块连续的物理页框上。如下图所示:      
![gc_thread_task](https://static.oschina.net/uploads/img/201801/08104646_NYvq.png "gc_thread_task")      

众所周知，一个进程(包括lwp)包括内核栈和用户栈，内核栈+thread_info用了8K，那么用户态的栈可用内存就是:

```
512K-8K=504K
```
如下图所示:  
![gc_kernel_user](https://static.oschina.net/uploads/img/201801/08104804_Bk9f.png "gc_kernel_user")            
   
### Linux实际物理内存映射
事实上linux对物理内存的使用非常的抠门，一开始只是分配了虚拟内存的线性区，并没有分配实际的物理内存，只有推到最后使用的时候才分配具体的物理内存，即所谓的请求调页。如下图所示:     
![gc_do_page_fault](https://static.oschina.net/uploads/img/201801/08105218_fVOt.png "gc_do_page_fault")         
### 查看smaps进程内存使用信息
使用如下命令，查看

```
cat /proc/[pid]/smaps > smaps.txt
```
实际物理内存使用信息，如下所示:

```
7fa69a6d1000-7fa69a74f000 rwxp 00000000 00:00 0 
Size:                504 kB
Rss:                  92 kB
Pss:                  92 kB
Shared_Clean:          0 kB
Shared_Dirty:          0 kB
Private_Clean:         0 kB
Private_Dirty:        92 kB
Referenced:           92 kB
Anonymous:            92 kB
AnonHugePages:         0 kB
Swap:                  0 kB
KernelPageSize:        4 kB
MMUPageSize:           4 kB

7fa69a7d3000-7fa69a851000 rwxp 00000000 00:00 0 
Size:                504 kB
Rss:                 152 kB
Pss:                 152 kB
Shared_Clean:          0 kB
Shared_Dirty:          0 kB
Private_Clean:         0 kB
Private_Dirty:       152 kB
Referenced:          152 kB
Anonymous:           152 kB
AnonHugePages:         0 kB
Swap:                  0 kB
KernelPageSize:        4 kB
MMUPageSize:           4 kB
```
搜索下504KB，正好是12563个，对了12563个线程，其中Rss表示实际物理内存(含共享库)92KB，Pss表示实际物理内存(按比例共享库)92KB(由于没有共享库，所以Rss==Pss)，以第一个7fa69a6d1000-7fa69a74f000线性区来看，其映射了92KB的空间，第二个映射了152KB的空间。如下图所示:    
![gc_mem](https://static.oschina.net/uploads/img/201801/08105319_dPoG.png "gc_mem")        
挑出符合条件（即size是504K）的几十组看了下，基本都在92K-152K之间，再加上内核栈8K

```
(92+152)/2+8K=130K，由于是估算，取整为128K，即反映此应用平均线程栈大小。
```
注意，实际内存有波动的原因是由于环境不同，从而走了不同的分支，导致栈上的增长不同。   
### 重新进行内存计算
JVM一开始申请了

```
-Xmx1792m -Xms1792m
```
即1.8G的堆内内存，这里是即时分配，一开始就用物理页框填充。      
12563个线程，每个线程栈平均大小128K，即:

```
128K * 12563=1570M=1.5G的对外内存
```
取个整数128K，就能反映出平均水平。再拿这个128K * 12563 =1570M = 1.5G，加上JVM的1.8G，就已经达到了3.3G，再加上kernel和日志传输进程等使用的内存数量，确实已经接近了4G，这样内存就对应上了！(注:用于定量内存计算的环境是一台内存用量将近4G，但还没OOM的机器)
### 为什么在物理机上没有应用Down机
笔者登录了原来物理机，应用还在跑，发现其同样有堆外内存泄露的现象，其物理内存使用已经达到了5个多G!幸好物理机内存很大，而且此应用发布还比较频繁，所以没有被OOM。    
Dump了物理机上应用的线程，

```
一共有28737个线程，其中28626个线程等待在CachedBnsClient上。 
```   
同样用smaps查看进程实际内存信息，其平均大小依旧为

```
128K，因为是同一应用的原因
```
继续进行物理内存计算

```
1.8+(28737 * 128k)/1024K =(3.6+1.8)=5.4G
```
进一步验证了我们的推理。
### 这么多线程应用为什么没有卡顿
因为基本所有的线程都睡眠在

```
 Thread.sleep(60 * 1000);//一次睡眠60s
```
上。所以仅仅占用了内存，实际占用的CPU时间很少。
# 总结
查找Bug的时候，现场信息越多越好，同时定位Bug必须要有实质性的证据。例如内存泄露就要用你推测出的模型进行定量分析。在定量和实际对不上的时候，深挖下去，你会发现不一样的风景!
# 原文链接
https://my.oschina.net/alchemystar/blog/edit/1603817    