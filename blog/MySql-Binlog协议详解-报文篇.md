#MySql-Binlog协议详解-报文篇
紧接上篇流程篇，本篇主要将binlog的event报文。
##Event报文分层
event报文主要分三层。   
(1)MySql报文都有的length-body防粘包结构。  
(2)Event Header  
(2)Event Body  
总体结构如下图所示:  
![image](/Users/alchemystar/image/binlogimage/binlog-event.png)   
##EventHeader
Event Header结构如下图所示:
![image](/Users/alchemystar/image/binlogimage/event-header.png)   (1)前4比特，是当前binlogEvent发生的时间戳   
(2)1byte的event类型,详情见github    
(3)4byte的serverId,是发送event的主库标识    
(4)4byte的event_length,是指包含当前eventHeader的整个body的长度    
(5)最后2byte是标志位    
解EventHeader报文的代码如下所示:    

```
    public void read() {
        timestamp = mm.readUB4() * 1000;
        eventType = getEventType(mm.read());
        serverId = mm.readUB4();
        eventLength = mm.readUB4();
        nextPosition = mm.readUB4();
        flags = mm.readUB2();
    }
```
## EventBody
紧接着就是描述EventBody。EventBody根据类型分主要有:    
(1)RotateEventData:当MySql的binlog文件从file1滚动到file2的时候会发生此事件。   
(2)UpdateRowsEventData:当binlog格式设置的是Row|mixed且Row更新的时候会发生此事件。   
(3)QueryEventData:当binlog格式设置的是statement|mixed且做DB有了更新、插入或删除操作的时候会发生时间(例如修改Row,alter表等)。     
(4)WriteRowsEventData:当binlog格式设置的是Row|mixed且有insert操作时候，有此事件发生。     
(5)余下还有不少event格式，在此就不一一罗列了，具体见github
### RotateEventData
![image](/Users/alchemystar/image/binlogimage/rotate-event.png)  
(1)8byte的binlogPosition
(2)以0x00结尾的String,表示了当前binlog文件名
###UpdateRowsEventData  
![image](/Users/alchemystar/image/binlogimage/update-rows.png) 
(1)byte的tableId,表明唯一一张表   
(2)一个复杂的Bit集合，有更新的Row在更新之前的值在下面Row列表中的位置      
(3)一个复杂的Bit集合，有更新的Row在更新之后的值在下面Row列表中的位置         
(4)一个Row的列表，配合上面两个bitSet使用  
具体解析BitSet非常复杂，详情见github    
###QueryEventData
![image](/Users/alchemystar/image/binlogimage/query-event.png) 
(1)4bytes的线程id      
(2)4bytes的当前事件执行时间     
(3)1bytes的数据库名字长度     
(4)2bytes的errorCode    
(5)2bytes的statusVar长度     
(6)statusVar    
(7)以0x00结尾的数据库名称   
(8)以0x00皆为的执行SQL,例如update t_temp set name='123' where id=1     
###WriteRowsEventData
![image](/Users/alchemystar/image/binlogimage/write-rows.png)    
(1)8bytes的tableId   
(2)一个复杂的Bit集合,和下面的Row配合使用    
(3)一个Row的列表，表明了插入的行   
由于上述几个eventData解析都很复杂，详情请见github    
## github地址
https://github.com/alchemystar/Aroundight