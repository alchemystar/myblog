# MySql协议讲解-事务协议
MySql事务协议主要是通过set autocommit、commit以及rollback这三个报文(命令)来实现的。
## MySql事务协议交互图
![image](/Users/alchemystar/image/mysqlimage/transaction.png)
##1.Client向DB发送set autocommit命令
autocommit,顾名思义，是否自动提交(事务)。如果设置为1，表明自动提交，设置为0，则是非自动提交,这样就隐式的开启了事务。   
值得注意的是，一但运行了set autocommit这个命令，不管设置为1或者0，都会自动提交前一个事务。
### MySql的set variable命令
此命令通常是用来设置局部/全局变量用。也是一种com_query包,如下图所示:
![image](/Users/alchemystar/image/mysqlimage/autocommit.png)
##2.DB向Client返回Okay包
Okay包已经在之前的博客中讲述过，在此不再赘述。
##3.Client向DB发送SQL语句
SQL语句即是Com_query包，如果是insert、update或者delete则返回okay包。在此的okay包比起前面的包多了一些信息，下面进行详解。
### insert、update或者delete
#### okay包会返回影响的行数
insert、update或者delete在执行后，都会返回影响的行数。这是通过在okay包中的affectedRows返回。

```
public class OkPacket extends MySQLPacket{
    public long affectedRows;
    public long insertId;
}
```
#### last insert id
在insert执行后，OkPacket在其中的insertId返回last insert id,但是有些ORM框架(例如MyBatis)是通过select LAST_INSERT_ID()方法来获取的。  
由于这个Id是存在mysql的session里面的，只要这个session没有被多线程复用，就不用担心last\_insert\_id被覆盖的问题。
### 
### select
SQL语句是select的话，则会返回ResultSet报文，这个在上一篇报文中已经讨论过。
### 重复上述过程
在一个事务内可以重复上述过程，知道有commit报文或者rollback报文被发送为止。
## Commit/Rollback报文
Commit/Rollback报文也是一种com_query报文,如下图所示:
![image](/Users/alchemystar/image/mysqlimage/commit_rollback.png)
Commit报文发送后，DB进行事务提交，并返回okay报文。  
Rollback报文发送后，DB进行事务回滚，并返回okay报文。
# GitHub链接
https://github.com/alchemystar/Lancelot.git