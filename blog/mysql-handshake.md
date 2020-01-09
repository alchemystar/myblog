## MySql协议详解-HandShake握手篇
各位有没有对Cobar、MyCat这些MySqlProxy感到新奇。反正笔者在遇到这些proxy时，感受到其对代码的无侵入兴感到大为惊奇。于是走上了研究MySql协议的不归路。现在我就在博客里面将其中所得分享出来，以飨大家。

### HandShake协议
下图是笔者整理的HandShake协议交互流程
![image](/Users/alchemystar/image/mysqlimage/handshake.png)
Step1:客户端向DB发起TCP握手。     
Step2:三次握手成功。与通常流程不同的是，由DB发送HandShake信息。这个Packet里面包含了MySql的能力、加密seed等信息。   
Step3:客户端根据HandShake包里面的加密seed对MySql登录密码进行摘要后，构造Auth认证包发送给DB。    
Step4:DB接收到客户端发过来的Auth包后会对密码摘要进行比对，从而确认是否能够登录。如果能，则发送Okay包返回。    
Step5:客户端与DB的连接至此完毕。  
### MySql报文图解
MySql协议与众多基于TCP的应用层一样，为了解决"粘包"问题，自定义了自己的帧格式。其主要通过Packet头部的length字段来确定整个报文的大小。 
#### MySql报文的分层   
MySql报文分为两层，一层是解决"粘包"的length-body.然后body中对应不同的格式有不同的字段含义。
![image](/Users/alchemystar/image/mysqlimage/mysqlpacket.png)
#### MySql报文外层
如上图所示:    
(1)前面3字节描述了整个报文中Body的长度。   
(2)第四个字节比较特殊，是为了防止串包用。机制是每收到一个报文都在其sequenceId上加1，并随着需要返回的信息返回回去。如果DB检测到sequenceId连续，则表明没有串包。如果不连续，则串包，DB会直接丢弃这个连接。   
(3)Body则是最终传递信息的地方，如上图中的handshake包。   
以下代码就是MySql报文的外层解包过程(基于Netty):    

``` 
    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
        // 4 bytes:3 length + 1 packetId
        if (in.readableBytes() < packetHeaderSize) {
            return;
        }
        in.markReaderIndex();
        int packetLength = ByteUtil.readUB3(in);
        // 过载保护
        if (packetLength > maxPacketSize) {
            throw new IllegalArgumentException("Packet size over the limit " + maxPacketSize);
        }
        byte packetId = in.readByte();
        if (in.readableBytes() < packetLength) {
            // 半包回溯
            in.resetReaderIndex();
            return;
        }
        BinaryPacket packet = new BinaryPacket();
        packet.packetLength = packetLength;
        packet.packetId = packetId;
        // data will not be accessed any more,so we can use this array safely
        packet.data = in.readBytes(packetLength).array();
        if (packet.data == null || packet.data.length == 0) {
            logger.error("get data errorMessage,packetLength=" + packet.packetLength);
        }
        out.add(packet);
    }
```
详情请见lancelot中的alchemystar.lancelot.common.net.codec.MySqlPacketDecoder类。
#### MySql报文内层-Body中的HandShake
HandShake类比较复杂，在此为了简便就写出其代码定义:   

```
public class HandshakePacket {
    private static final byte[] FILLER_13 = new byte[] {0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0};

    public byte protocolVersion;
    public byte[] serverVersion;
    public long threadId;
    public byte[] seed;
    public int serverCapabilities;
    public byte serverCharsetIndex;
    public int serverStatus;
    public byte[] restOfScrambleBuff;
 }
```
其中包含了协议版本号、服务器版本号、当前线程ID、加密种子等等。其都是通过对BodyDecode而来，Decode部分繁琐而无味,详情请看:
alchemystar.lancelot.common.net.proto.mysql.HandshakePacket;
#### 客户端的认证报文-AuthPacket

```
public class AuthPacket extends MySQLPacket{
    private static final byte[] FILLER = new byte[23];

    public long clientFlags;
    public long maxPacketSize;
    public int charsetIndex;
    public byte[] extra;// from FILLER(23)
    public String user;
    public byte[] password;
    public String database;
}
```
其构造必须按照MySql协议一个一个填写。其中最重要的字段位user和password。
#### password的摘要方式
AuthPacket的password是对原密码摘要后的byte流，其根据MySql协议版本的不同分为411和322两个大版本。即4.1.1版本协议和3.2.2版本协议所采用的加密方法。
##### 411-password摘要

```
    public static final byte[] scramble411(byte[] pass, byte[] seed) throws NoSuchAlgorithmException {
        MessageDigest md = MessageDigest.getInstance("SHA-1");
        byte[] pass1 = md.digest(pass);
        md.reset();
        byte[] pass2 = md.digest(pass1);
        md.reset();
        md.update(seed);
        byte[] pass3 = md.digest(pass2);
        for (int i = 0; i < pass3.length; i++) {
            pass3[i] = (byte) (pass3[i] ^ pass1[i]);
        }
        return pass3;
    }
```
##### 322-password摘要

```
   public static final String scramble323(String pass, String seed) {
        if ((pass == null) || (pass.length() == 0)) {
            return pass;
        }
        byte b;
        double d;
        long[] pw = hash(seed);
        long[] msg = hash(pass);
        long max = 0x3fffffffL;
        long seed1 = (pw[0] ^ msg[0]) % max;
        long seed2 = (pw[1] ^ msg[1]) % max;
        char[] chars = new char[seed.length()];
        for (int i = 0; i < seed.length(); i++) {
            seed1 = ((seed1 * 3) + seed2) % max;
            seed2 = (seed1 + seed2 + 33) % max;
            d = (double) seed1 / (double) max;
            b = (byte) java.lang.Math.floor((d * 31) + 64);
            chars[i] = (char) b;
        }
        seed1 = ((seed1 * 3) + seed2) % max;
        seed2 = (seed1 + seed2 + 33) % max;
        d = (double) seed1 / (double) max;
        b = (byte) java.lang.Math.floor(d * 31);
        for (int i = 0; i < seed.length(); i++) {
            chars[i] ^= (char) b;
        }
        return new String(chars);
    }
```
### 最后的Okay报文
Okay报文很简单，现在直接将其代码结构放出:

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
### MySql-HandShake Server端代码:
由于最终的代码是以Netty的形式呈现，所以不是很直观。读者可以慢慢分析

```
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        switch (state) {
            case (BackendConnState.BACKEND_NOT_AUTHED):
                // init source's ctx here
                source.setCtx(ctx);
                // 处理握手包并发送auth包
                auth(ctx, msg);
                // 推进连接状态
                state = BackendConnState.BACKEND_AUTHED;
                break;
            case (BackendConnState.BACKEND_AUTHED):
                authOk(ctx, msg);
                break;
            default:
                break;
        }
    }

    private void authOk(ChannelHandlerContext ctx, Object msg) {
        BinaryPacket bin = (BinaryPacket) msg;
        switch (bin.data[0]) {
            case OkPacket.FIELD_COUNT:
                afterSuccess();
                break;
            case ErrorPacket.FIELD_COUNT:
                ErrorPacket err = new ErrorPacket();
                err.read(bin);
                throw new ErrorPacketException("Auth not Okay");
            default:
                throw new UnknownPacketException(bin.toString());
        }
        // to wake up the start up thread
        source.countDown();
        // replace the commandHandler of Authenticator
        ctx.pipeline().replace(this, "BackendCommandHandler", new BackendCommandHandler(source));

    }
```
### Github链接
https://github.com/alchemystar/Lancelot.git 