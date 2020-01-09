#MySql-Proxy之多路结果集归并
笔者觉得Cobar之类的分库分表最神奇的部分就是靠一条sql查询不同schema下(甚至不同实例下)的不同的表。例如

```
select * from t_test; // 映射为
	|------select * from schema1.t_test
	|------select * from schema2.t_test
ResultSet // 返回结果集为两者的归并	
	|--schema1.t_test.ResultSet
	|--schema2.t_test.ResultSet
```
以笔者这种刨根到底的性格当然要把这个过程DIY出来。  
由于Cobar对MySql的连接是BIO的。而笔者喜欢NIO,于是用NIO将Corbar的多节点查询全部重写(基于Netty)。NIO的难度更大，性能也更好，这个重写的过程就记录成博客，以飨读者。
#多路归并原理
#多节点发送select语句
![codegen](/Users/alchemystar/image/lancelot/lancelot_select.png) 
当客户端发送给select * from test后，Lancelot会根据配置将语句将当前语句路由到多个不同的DB实例上，如上图所示。  
FrontEnd:用来和client交互，一个FrontEnd可以对应多个Backend   
BackEnd:用来和DB交互     
#多节点归并结果集
![codegen](/Users/alchemystar/image/lancelot/lancelot_resultset.png)
每条语句在一个DB实例上面执行后，都会返回一个ResultSet结果集,在此需要将多个结果集归并成一个统一的结果集，然后返回给client,这样client就感觉像查询一个DB实例一样。   
如上图所示,归并过程在下面讲解。
##归并ResultSet结果集
在讲如何归并前，我们需要重温一下MySql返回结果集的结构,
其详细描述见笔者博客:  

```  
https://my.oschina.net/alchemystar/blog/834150
```
其协议格式如下所示:  
![image](/Users/alchemystar/image/mysqlimage/resultset.png)    
由上图可见，   
其中的Row才是真正的数据内容。而其余的例如,field\_count、fields
、eof以及last_eof则仅仅是携带数据格式的信息。    
如果要多路归并成一路的话，field\_count、fields、eof以及last\_eof这些只需要返回给client一份即可。
##去掉多余的结构描述信息
现在根据协议结构将Frontend归并结果集的代码阶段分为三个:    
(1)fieldList阶段: 
由于field\_count、fields、eof这三个阶段是连续的，于是将其合并成一个状态。   
(2)Row阶段:顾名思义，接收DB返回的数据阶段。    
(3)LastEof阶段:最后的收尾阶段，每个结果集的last\_eof表示此结果集的结束，只有所有的last_eof都收到之后才能表示结果的结束。      
###fieldList阶段的处理:   
首先每个Backend都接收field\_count,fields,eof。当其接收到eof之后，收到row之前，向Frontend提交这些信息。如下图所示:    
![codegen](/Users/alchemystar/image/lancelot/lancelot_fields.png)   
当Frontend获取到Backend1的feilds信息之后，就开始接收Row,并丢弃其余Backend的fields信息。代码如下:    

```
public void fieldListResponse(List<BinaryPacket> fieldList) {
    lock.lock();
    try {
        if(!isFailed.get()) {
            // 如果还没有传过fieldList的话,则传递
            if (!fieldEofReturned) {
                writeFiledList(fieldList);
                fieldEofReturned = true;
            }
        }
    } finally {
        lock.unlock();
    }
}
```
###Row阶段的处理
当Frontend进入Row阶段之后，处理比较简单，Backend发送的任何Row都向前段传输，如果是Backend的fields信息则丢弃。如下图所示:    
![codegen](/Users/alchemystar/image/lancelot/lancelot_row.png)    
###LastEof阶段
每当一个Backend收到last\_eof之后，表明当前Backend的结果集已经结束。Frontend需要等所有的Backend结果集结束之后，再发送一个last\_eof告诉client，所有的结果已经完了，如下图所示:    
![codegen](/Users/alchemystar/image/lancelot/lancelot_last_eof.png)     
代码如下所示:   

```
// last eof response
public void lastEofResponse(BinaryPacket bin) {
    lock.lock();
    try {
        logger.info("last eof ");
        if (decrementCountBy()) {
            if (!isFailed.get()) {
                bin.packetId = ++packetId;
                logger.info("write eof okay");
                bin.write(session.getCtx());
                // 如果是自动提交,则释放session
                if(session.getSource().isAutocommit()){
                    session.release();
                }
            }else{
                notifyFailure();
            }
        }
    } finally {
        lock.unlock();
    }
}
```
#例子
运行lancelot中的LanceLotServer的main命令，其就自动连接了我本机的MySql。
配置之类的在SystemConfig中进行修改(现在还没有做到配置文件化)。     
我用mysqlclient连接到lancetlot,然后运行select * from test命令。结果如下图所示:     
![codegen](/Users/alchemystar/image/lancelot/lancelot_example.png) 
#GitHub链接
https://github.com/alchemystar/Lancelot    
#码云链接
https://git.oschina.net/alchemystar/Lancelot
#原文链接


