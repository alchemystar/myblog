# 解Bug之路-dubbo应用无法重连zookeeper
## 前言
dubbo是一个成熟且被广泛运用的框架。饶是如此,在某些极端条件下基于dubbo的应用还会出现无法重连zookeeper的问题。由于此问题容易导致比较大的故障，所以笔者费了一番功夫去定位，现将排查过程写成博文分享出来。
## Bug现场
这是一起在测试环境出现的故障。起因是网工做交换机切换演练，可能由于姿势不对，使得断网的时间从预估的秒级达到了分钟级。等网络恢复后，测试环境就炸开了锅，基本上所有应用再也无法提供服务，在dubbo控制台上也看不到任何提供者，他们和zk的连接都断开而且似乎完全没有重连的迹象。如下图所示:
![codegen](/Users/alchemystar/image/dubbo_no_zk/dubbo_no_zk.png)  
## 无法快速恢复
为了不影响测试的进度，运维同学紧急进行了重启，但坑爹的是大部分系统都有启动依赖，盲目的重启只会因为xxx provider不存在而无法启动。只能从最基础的服务开始重启，慢慢恢复。如下图所示:
![codegen](/Users/alchemystar/image/dubbo_no_zk/start_dependency.png)  
还好只是测试环境，但为了不让产线出现这种问题，必须一查到底，把这个Bug揪出来。
## 着手排查
### 模拟zookeeper连接断开
测试环境的好处是我们可以用各种手段去模拟复现，而不用和处理产线一样到处寻找蛛丝马迹然后进行逻辑推理(推理是一个非常烧脑的过程)。于是笔者联系了SA同学，通过iptables进行线下的断网模拟。命令如下所示:

```
// 禁用本机和zk三台机器的流量进出
iptables -A INPUT -s zk-1-ip/32 -j DROP
iptables -A INPUT -s zk-2-ip/32 -j DROP
iptables -A INPUT -s zk-3-ip/32 -j DROP

iptables -A OUTPUT -s zk-1-ip/32 -j DROP
iptables -A OUTPUT -s zk-2-ip/32 -j DROP
iptables -A OUTPUT -s zk-3-ip/32 -j DROP
```
拓扑图如下:
![codegen](/Users/alchemystar/image/dubbo_no_zk/iptables_disable_zk.png)
发现在drop对zk的包之后，不管等待多长时间，只要连接一放开，立马就能重连zk!
看来dubbo对zookeeper的重连还是非常靠谱的。
### 同时模拟DNS断开
由于模拟zk断开不会导致无法重连的现象。于是笔者开始思考，是否交换机异常的时候导致了所有的包都无法发送/接收,而导致重连出问题的并不是对zookeeper发起连接。于是笔者看了看配置，是否还有其它和重连有关联的点,仔细观察下这个配置:

```
// 这其中有一个不容易注意到的点，就是域名解析也需要网络包的交互
dubbo.registry.address=zookeeper://dubbo-1.com?back=dubbo-2.com,dubbo-3.com
```
难道是DNS访问不到导致了这一问题？反正测试环境，继续模拟一发，命令如下所示:

```
// 禁用本机和zk三台机器的流量进出
iptables -A INPUT -s zk-1-ip/32 -j DROP
iptables -A INPUT -s zk-2-ip/32 -j DROP
iptables -A INPUT -s zk-3-ip/32 -j DROP
iptables -A OUTPUT -s zk-1-ip/32 -j DROP
iptables -A OUTPUT -s zk-2-ip/32 -j DROP
iptables -A OUTPUT -s zk-3-ip/32 -j DROP
// 禁用本机和DNS两台机器的流量进出
iptables -A INPUT -s dns-ip/32 -j DROP
iptables -A INPUT -s dns-ip/32 -j DROP
iptables -A OUTPUT -s dns-ip/32 -j DROP
iptables -A OUTPUT -s dns-ip/32 -j DROP
```
网络拓扑如下:
![codegen](/Users/alchemystar/image/dubbo_no_zk/iptables_disable_zk_dns.png)
这次我们在禁用流量后，故意先放开对zk的流量，再放开对DNS的流量，如下图所示:
![codegen](/Users/alchemystar/image/dubbo_no_zk/first_zk_then_dns.png)
看来在dubbo对zookeeper重连过程中，如果DNS也无法响应，是会出现网络恢复后也再也无法重连的现象。但是，我们并不能下判断交换机的故障导致的无法重连肯定是这个Bug引起。需要找到证据来证明这一点！
## 它山之石，可以攻玉
有了DNS这个信息后，先google一下，看看能否有其它人遇到过这个坑。于是找到了这个链接

```
https://github.com/sgroschupf/zkclient/issues/23

```
![codegen](/Users/alchemystar/image/dubbo_no_zk/unknown_host_exception.png)
按照github上的描述，zkclient在UnknownHostException抛出之后再也无法重连zookeeper。不过他是在Kafka中遇到的，但他推断所有用低版本org.apache.zookeper的都会有这个问题。按照上面给出的Bug Fix链接

```
https://issues.apache.org/jira/browse/ZOOKEEPER-1576
```

笔者发现其在3.5.0修复 
![codegen](/Users/alchemystar/image/dubbo_no_zk/zookeeper_fix.png)
于是将对应的应用的org.apache.zookeeper版本升级到3.5.5版本，重新实验后，发现问题解决了!     
这里有个小技巧，我们可以通过

```
zip -d xxx.jar WEB-INF/lib/zookeeper-3.4.8.jar
zip -r 0 xxx.jar WEB-INF/lib/zookeeper-3.5.5.jar
// 以及zip -r 其它zookeeper-3.5.5新依赖的包
```
使得不用重新编译打包的方式即可修改应用使用的jar包版本,这样在快速验证的时候就不需要通知对应的开发修改依赖了。
## 寻找支持jdk1.6的zookeeper jar包
由于笔者所在的产线环境有很多老系统用的jdk1.6,而zookeeper-3.5.5之支持1.8及以上版本，所以需要寻找能够给jdk1.6使用的包。此时，笔者的同事由于负责kafka,其对kafa做过混沌测试，坚信kafka没有这个问题，于是笔者就用kafka依赖的zookeeper-3.4.13包继续进行测试，发现zookeeper-3.4.13也是okay的，具体代码改动将会在下面讲到。
## 搜索日志寻找证据
由于可能是UnknownHostException导致了这一问题，笔者在出问题的应用里面寻找，确实发现了UnknownHostException。日志如下所示:

```
// 下面过滤了一大堆disconnected连接断开日志,只列出核心有关的
2020-03-19 21:06:28.926 [DubboZkclientConnector-EventThread] zookeeper state changed (Disconnected)
2020-03-19 21:06:28.926 [ZkClient-EventThread-101-dubbo.com] Zookeeper失去连接
2020-03-19 21:06:49.758 [DubboZkclientConnector-EventThread] zookeeper state changed (Expired)
2020-03-19 21:06:49:759 [DubboZkclientConnecto-SendThread] Unable to reconnect to ZooKeeper sercice ,session 0xXXXXX has expired
2020-03-19 21:07:29.793 [DubboZkclientConnector-EventThread] ERROR ClientCxnn - Error while calling watcher
java.lang.RuntimeException: Exception while restarting zk client
	......
	......
Caused by: java.net.UnknownHostException: dubbo-1.com
	at...lookupAllHostAddr...
	......
	at...StaticHostProvier...
	......
	at...reconnect...[zookeeper-3.4.8.jar:3.4.8--1]
	
```
上面日志反应出在zookeeper session expired之后重新建立session的过程中如果抛出java.net.UnknownHostException后，zkclient对应的线程就再也不会有其它动作。
## 代码分析
### 旧版本代码逻辑
上面的证据再配合实验的结果基本就能确定这个DNS异常会导致dubbo无法重连zookeeper的现象。于是笔者开发翻阅代码,首先我们看下dubbo重连的逻辑:

```
public class ZkclientZookeeperClient extends AbstractZookeeperClient<IZkChildListener> {
	......
    public ZkclientZookeeperClient(URL url) {
        super(url);
        client = new ZkClientWrapper(url.getBackupAddress(), 30000);
        client.addListener(new IZkStateListener() {
            public void handleStateChanged(KeeperState state) throws Exception {
                ZkclientZookeeperClient.this.state = state;
                if (state == KeeperState.Disconnected) {
                    stateChanged(StateListener.DISCONNECTED);
                } else if (state == KeeperState.SyncConnected) {
                    stateChanged(StateListener.CONNECTED);
                }
            }
			  // 这边是session重建的过程
            public void handleNewSession() throws Exception {
                stateChanged(StateListener.RECONNECTED);
            }
        });
        client.start();
    }
	......
}
// StateListener.RECONNECTED的处理在zookeeperResigry.java中
public class ZookeeperRegistry extends FailbackRegistry{
	......
        zkClient.addStateListener(new StateListener() {
        	  // 在收到RECONNECTED事件后，进行recover也就是恢复的过程
            public void stateChanged(int state) {
                if (state == RECONNECTED) {
                    try {
                        recover();
                    } catch (Exception e) {
                        logger.error(e.getMessage(), e);
                    }
                }
            }
        });
   ......
}
```
由上面的代码我们可以得知，在session expired之后内部会重建session，在新建session之后，dubbo的Statelistener会发送reconnected事件从而执行恢复的过程，如下图所示:
![codegen](/Users/alchemystar/image/dubbo_no_zk/expired_okay.png) 
那么我们看下UnknownHostException抛出后会导致什么现象,代码如下所示:

```
public class ZkClient implements Watcher {
......
    private void processStateChanged(WatchedEvent event) {
    	  // 这边对应于session expired日志
        LOG.info("zookeeper state changed (" + event.getState() + ")");
        setCurrentState(event.getState());
        if (getShutdownTrigger()) {
            return;
        }
        try {
            fireStateChangedEvent(event.getState());

            if (event.getState() == KeeperState.Expired) {
				// UnknownHostException就是在这抛出。
                reconnect();
                fireNewSessionEvents();
            }
        } catch (final Exception e) {
			// 这边对应于Error while calling watcher日志
            throw new RuntimeException("Exception while restarting zk client", e);
        }
    }

......
}

```
异常是在reconnect抛出的，异常抛出后，就不会运行fireNewSessionEvents这个逻辑，也就不会执行listener中的handleNewSession逻辑，进而不会recover,从而导致dubbo无法重连！如下图所示:
![codegen](/Users/alchemystar/image/dubbo_no_zk/expired_fail.png) 
### 新版本如何修复
由于UnknownHostException是在StaticHostProviver中触发，在这边笔者给出了新旧版本的对应代码，旧版本zookeeper-3.4.8

```
public final class StaticHostProvider implements HostProvider{
	......
    public StaticHostProvider(Collection<InetSocketAddress> serverAddresses)
            throws UnknownHostException {
        for (InetSocketAddress address : serverAddresses) {
            InetAddress ia = address.getAddress();
            // 这边没有抓住UnknownHostException异常
            InetAddress resolvedAddresses[] = InetAddress.getAllByName((ia!=null) ? ia.getHostAddress():
                address.getHostName());
            for (InetAddress resolvedAddress : resolvedAddresses) {
 				 	......
                if (resolvedAddress.toString().startsWith("/") 
                        && resolvedAddress.getAddress() != null) {
                    this.serverAddresses.add(
                            new InetSocketAddress(InetAddress.getByAddress(
                                    address.getHostName(),
                                    resolvedAddress.getAddress()), 
                                    address.getPort()));
                } else {
                    this.serverAddresses.add(new InetSocketAddress(resolvedAddress.getHostAddress(), address.getPort()));
                }  
            }
        }
        
        if (this.serverAddresses.isEmpty()) {
            throw new IllegalArgumentException(
                    "A HostProvider may not be empty!");
        }
        Collections.shuffle(this.serverAddresses);
    }
   ......
}

```
新版本zookeeper-3.4.13小小的重构了一下，将DNS的逻辑放到next函数里面并抓住了UnknownHostException异常。

```
public final class StaticHostProvider implements HostProvider{
	......
    public InetSocketAddress next(long spinDelay) {
        currentIndex = ++currentIndex % serverAddresses.size();
        if (currentIndex == lastIndex && spinDelay > 0) {
            try {
                Thread.sleep(spinDelay);
            } catch (InterruptedException e) {
                LOG.warn("Unexpected exception", e);
            }
        } else if (lastIndex == -1) {
            // We don't want to sleep on the first ever connect attempt.
            lastIndex = 0;
        }

        InetSocketAddress curAddr = serverAddresses.get(currentIndex);
        try {
            String curHostString = getHostString(curAddr);
            List<InetAddress> resolvedAddresses = new ArrayList<InetAddress>(Arrays.asList(this.resolver.getAllByName(curHostString)));
            if (resolvedAddresses.isEmpty()) {
                return curAddr;
            }
            Collections.shuffle(resolvedAddresses);
            return new InetSocketAddress(resolvedAddresses.get(0), curAddr.getPort());
        } catch (UnknownHostException e) {
        	// 就是这边抓住了UnknownHostException进而修复了这一问题
            return curAddr;
        }
    }	
	......
}
```
我们可以看到新版zookeeper-3.4.13抓住了这个UnknownHostException进而修复了这一问题。
## BUG触发条件复盘
在与zookeeper服务连接异常并session expired(默认30s)，DNS缓存也超时(默认30s)同时用的低版本zookeeper jar包就很容易达成上述Bug的情况。而之前测试环境由于交换机切换的某些原因使得网络断开超过了30s从而诱发了这一问题。而且不仅仅是Dubbo,任何用zookeeper低版本jar包都有可能出现这个问题！
## 总结
海恩法则指出,每一起严重事故的背后，必然有29次轻微事故和300起未遂先兆以及1000起事故隐患。所以对测试环境的任何问题都要引起重视，从而把问题消灭在萌芽阶段。
## 公众号       
关注笔者公众号，获取更多干货文章:        
![](https://oscimg.oschina.net/oscnet/up-03e8bdd592b3eb9dec0a50fa5ff56192df0.JPEG)       
## 原文链接




