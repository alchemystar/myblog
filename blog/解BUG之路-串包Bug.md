#解Bug之路-串包Bug
笔者很热衷于解决Bug,同时比较擅长(网络/协议)部分，所以经常被唤去解决一些网络IO方面的Bug。现在就挑一个案例出来，写出分析思路，以飨读者，希望读者在以后的工作中能够少踩点坑。
#串包Bug现场
##前置故障Redis超时
由于某个系统大量的hget、hset操作将Redis拖垮，通过监控发现Redis的CPU和IO有大量的尖刺,CPU示意图下图所示:
![codegen](/Users/alchemystar/image/redis_bug/cpu_high.png)    
CPU达到了100%,导致很多Redis请求处理不及时，其它业务系统都频繁爆出readTimeOut。此时，紧急将这个做大量hget、hset的系统kill,过了一段时间，Redis的CPU恢复平稳。
##一波未平，一波又起    
就在我们以为事件平息的时候，线上爆出登录后的用户名称不正确。同时错误日志里面也有大量的Redis返回不正确的报错。尤为奇葩的是，系统获取一个已经存在的key,例如get User123456Name,返回的竟然是redis的成功返回OK。示意图如下:    
  
```
Jedis.sendCommand:get User123456Name
Jedis.return:OK
	or
Jedis.sendCommand:get User123456Name
Jedis.return:user789
```
我们发现此情况时，联系op将Redis集群的所有Key紧急delete,当时监控示意图:    
![codegen](/Users/alchemystar/image/redis_bug/del_key.png)     
当重启后，我们再去线上观察的时候，发现错误依然存在，神奇的是，这种错误发生的频率会随着时间的增加而递减。到最后刷个10分钟页面才会出现这种错,示意图如下所示:    
![codegen](/Users/alchemystar/image/redis_bug/error_frequency.png)   
既然如此，那只能祭出重启大法，把出错的业务系统全部重启了一遍。    
重启之后，线上恢复正常，一切Okay。     
##Bug复盘    
此次Bug是由Redis本身Server负载太高超时引起的。Bug的现象是通过Jedis去取对应的Key值，得不到预期的结果，简而言之包乱了，串包了。    
##缩小Bug范围  
首先:Redis是全球久经考验的系统，这样的串包不应该是Redis的问题。   
第二:Redis刷新了key后Bug依然存在，而业务系统重启了之后Okay。  
第三:笔者在错误日志中发现一个现象，A系统只可能打印出属于A系统的json串结构(redis存的是json)。        
很明显，是业务系统的问题，如果是Redis本身的问题，那么在很大概率上A系统会接收到B系统的json串结构。      
##业务系统问题定位    
业务系统用的是Jedis,这同样也是一个久经考验的库，出现此问题的可能性不大。那么问题肯定是出在运用Jedis的姿势上。     
于是笔者找到了下面一段代码:  
 
```
public Object invoke(Object proxy,Method method,Object[] args) throws Throwable{
	JedisClient jedisClient = jedisPool.getResource();   
	try{
	  return method.invoke(jedisClient,args);  
	} catch(Exception e){
	  logger.error("invoke redis error",e);   
	  throw e;   
	}finally {
		if(jedisClient != null){
			// 问题处在下面这句
			jedisPool.returnResource(jedisClient);
		}
	}
}
```
当时我就觉得很奇怪，笔者自己写的，阅读过的连接池的代码都没有将抛异常的连接放回池里。就以Druid为例，如果是网络IO等fatal级别的异常，直接抛弃连接。这里把jedisClient连接返回去感觉就是出问题的关键。 
##Bug推理   
笔者意识到，之所以串包可能是由于jedisClient里面可能有残余的数据，导致读取的时候读取到此数据，从而造成串包的现象。
###串包原因
####正常情况下的redis交互
先上Jedis源码  

```
public String get(final String key) {
	checkIsInMulti();
	client.sendCommand(Protocol.Command.GET, key);
	return client.getBulkReply();
}
```
Jedis本身用的是Bio,上述源码的过程示意图如下:   
![codegen](/Users/alchemystar/image/redis_bug/normal_jedis.png)    
####出错的业务系统的redis交互    
![codegen](/Users/alchemystar/image/redis_bug/error_jedis.png)    
由于Redis本身在高负载状态，导致没能及时相应command请求，从而导致readTimeOut异常。    
####复用这个出错链接导致出错   
在Redis响应了上一个command后，把数据传到了对应command的socket，进而被inputream给buffer起来。而这个command由于超时失败了。      
![codegen](/Users/alchemystar/image/redis_bug/redis_back_pool.png)     
这样，inputStream里面就有个上个命令留下来的数据。   
下一次业务操作在此拿到这个连接的时候，就会出现下面的情况。    
![codegen](/Users/alchemystar/image/redis_bug/redis_pool_get_again.png)   
再下面的命令get user789Key会拿到get user456Key的结果，依次类推，则出现串包的现象。    
####串包过程图   
![codegen](/Users/alchemystar/image/redis_bug/redis_bug_process.png)     
上图中相同颜色的矩形对应的数据是一致的。但是由于Redis超时，导致数据串了。   
####为什么get操作返回OK
上图很明显的解释了为什么一个get操作会返回OK的现象。因为其上一个操作是set操作，它返回的OK被get操作读取到，于是就有了这种现象。  
####为什么会随着时间的收敛而频率降低   
因为在调用Redis出错后，业务系统有一层拦截器会拦截到业务层的出错，同时给这个JedisClient的错误个数+1,当错误个数>3的时候，会将其从池中踢掉。这样这种串了的连接会越来越少，导致Bug原来越难以出现。          
####在每次调用之前清理下inputstream可行否   
不行，因为Redis可能在你清理inputstream后，你下次读取前把数据给传回来。
####怎么避免这种现象?
抛出这种IO异常的连接直接给扔掉，不要放到池子里面。    
####怎么从协议层面避免这种现象   
对每次发送的命令都加一个随机的packetId,然后结果返回回来的时候将这个packetId带回来。在客户端每次接收到数据的时候，获取包中的packetId和之前发出的packetId相比较,如下代码所示:   


```
if(oldPacketId != packetIdFromData){
	throw Exception("串包");
}
```
##总结
至少在笔者遇到的场景中，出现IO异常的连接都必须被抛掉废弃，因为你永远不知道在你复用的那一刻，socket或者inputstream的buffer中到底有没有上一次命令遗留的数据。    
当然如果刻意的去构造协议，能够通过packetId之类的手段把收发状态重新调整为一致也是可以的,这无疑增加了很高的复杂度。所以废弃连接重建是最简单有效的方法。      
##原文链接
      










 