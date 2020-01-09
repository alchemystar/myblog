# MySql协议详解-CRUD与Result篇
## Com_query报文
一般对DB的CRUD操作都由com_query报文封装并发送给DB。com_query报文如下图所示:
![image](/Users/alchemystar/image/mysqlimage/querypacket.png)
PacketLength:3byte表示body长度，防"粘包"。     
sequenceId:1byte防串包。  
body部分:是一个以0x00结尾的字符串，这个字符串就是想要执行的SQL,实际操作中即是一直读body直到读到一个0x00皆为的字符串即停止。   
## insert,update和delete
如果SQL是insert、update或者是delete,则返回的是对应的okay或者error报文。    
okay报文的类表示:  
 
```
public class OkPacket extends MySQLPacket {
    public static final byte FIELD_COUNT = 0x00;
    public static final byte[] OK = new byte[] { 7, 0, 0, 1, 0, 0, 0, 2, 0, 0, 0 };
    public static final byte[] AUTH_OK = new byte[] { 7, 0, 0, 2, 0, 0, 0, 2, 0, 0, 0 };
    public byte fieldCount = FIELD_COUNT;
    public long affectedRows;
    public long insertId;
    public int serverStatus;
    public int warningCount;
    public byte[] message;
}
```
error报文的类表示:

```
public class ErrorPacket extends MySQLPacket {
    public static final byte FIELD_COUNT = (byte) 0xff;
    private static final byte SQLSTATE_MARKER = (byte) '#';
    private static final byte[] DEFAULT_SQLSTATE = "HY000".getBytes();

    public byte fieldCount = FIELD_COUNT;
    public int errno;
    public byte mark = SQLSTATE_MARKER;
    public byte[] sqlState = DEFAULT_SQLSTATE;
    public byte[] message;
}
```
## Select和ResultSet报文
如果执行的SQL是select语句，则返回的报文比较复杂，不过笔者已经整理成图的形式。   
### Row报文
首先ResultSet是由很多行(Row)组成，每一行(Row)就表示了一条记录,Row格式如下所示:    
![image](/Users/alchemystar/image/mysqlimage/rowpacket.png)
每一行(row)又分好field_count个字段,这个field_count将会在比Row还高一层的Result格式中描述，下面有详解。  
每一个字段都是一个length-value对，length长度是3byte,其读取方法很特殊，现在直接用代码表述:   

```
    public int readUB3() {
        final byte[] b = this.data;
        int i = b[position++] & 0xff;
        i |= (b[position++] & 0xff) << 8;
        i |= (b[position++] & 0xff) << 16;
        return i;
    }
```
获取了length长度之后，则可以根据此读出来后面的value。value的类型也会在后面的Resutl格式中描述。
### ResultSet格式
严格来说ResultSet是由多个独立的报文以协议的形式组织起来，现直接放出ResultSet的协议格式图:
![image](/Users/alchemystar/image/mysqlimage/resultset.png)
从上图中可以看到，当客户端发送一个select的com_query包后，DB会按照下列步骤返回:    
Step1:返回一个ResultSetHeader报文，其中包含了fieldCount,在此图就不例出了，现只给出代码定义。

```
public class ResultSetHeaderPacket extends MySQLPacket {
    public int fieldCount;
    public long extra;
}
```
Step2:根据读取到的fieldCount来在接下来的byte流里面读取fieldCount个FiledPacket报文,FieldPacket报文的代码定义:

```
public class FieldPacket extends MySQLPacket {
    private static final byte[] DEFAULT_CATALOG = "def".getBytes();
    private static final byte[] FILLER = new byte[2];

    public byte[] catalog = DEFAULT_CATALOG;
    public byte[] db;
    public byte[] table;
    public byte[] orgTable;
    public byte[] name;
    public byte[] orgName;
    public int charsetIndex;
    public long length;
    public int type;
    public int flags;
    public byte decimals;
    public byte[] definition;
}
```
具体逻辑请参照github    
Step3:再读取一个eof包表示field包流的结束   
eof包的代码定义:
   
```
public class EOFPacket extends MySQLPacket {
    public static final byte FIELD_COUNT = (byte) 0xfe;

    public byte fieldCount = FIELD_COUNT;
    public int warningCount;
    public int status = 2;
```    
Step4:一直读Row,直到读到last eof位置，Row格式已经在上面给过。   
Step5:如果读到任何一个error包后，此此读取结束，抛出错误。   
Step6:值得注意的是，如果eof中的status & SERVER_MORE\_RESULT\_EXISTS不为0，表明还有ResultSet。则继续返回到fieldCount阶段进行下一步的读取。
Step7:至此，整个ResultSet读取完毕。    
# GitHub链接
https://github.com/alchemystar/Lancelot.git