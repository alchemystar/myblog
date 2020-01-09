# MySql-Binlog协议详解-流程篇
MySql-Binlog在MySql主从不同方面发挥着不可或缺的作用，同时我们也能通过Binlog实时监控数据的变化。本系列就讲述了怎样接收并解析Binlog。本篇就主要对接收binlog的流程做了一下探讨。
## Binlog发送接收流程
流程如下图所示:
![image](/Users/alchemystar/image/binlogimage/binlog-flow.png)   
(1)第一步和上篇blog一样，通过HandShake协议进行Client和DB的握手认证
(2)握手成功以后,Client对DB发送show master status命令，此命令中回带回当前最新binlog存储在哪个文件，以及对应哪个偏移量。如果想从当前开始接收binglog,则在后面发送binlog dump命令的时候用这两个值就好。    
(3)发送show global variables like 'binlog\_checksum'命令，这是由于binlog event发送回来的时候需要,在最后获取event内容的时候，会增加4个额外字节做校验用。mysql5.6.5以后的版本中binlog\_checksum=crc32,而低版本都是binlog\_checksum=none。如果不想校验，可以使用set命令设置set binlog_checksum=none   
(4)最后终于到了发送Dump命令的阶段
## MySql-Binlog-Dump命令
Dump命令包图如下所示:
![image](/Users/alchemystar/image/binlogimage/dump-binlog.png)
如上图所示,在报文中塞入binlogPosition和binlogFileName即可让master从相应的位置发送binlog event
## MySql-Binlog-Event
一但发送了BinlogDump命令，master就会在数据库有变化的源源不断的推送binlog event到client。值得注意的是binlog的类型有三种:    
(1)Statement:每一条会修改数据的sql都会记录在binlog中。   
(2)Row:不记录sql语句上下文相关信息，仅保存哪条记录被修改。    
(3)Mixedlevel:以上两种Level的混合。    
在下一篇博客中，笔者会详细讲述binlog-event的格式。   
## github地址
https://github.com/alchemystar/Aroundight

