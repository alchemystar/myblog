#自己动手写SQL查询引擎-总篇     
本篇Blog在总体层面介绍了SQL查询引擎Rider的功能及设计，其细节部分将会在后面的篇章中一一道来。          
## 起因    
笔者在实际工作中经常需要解析文件，每次文件稍有变化，都得拷贝粘贴一堆代码。      
于是就想着能不能做一个通用的服务，通过配置的方式解析文件。         
### 配置通用     
最通用的方法就是自己定义一个文件描述语言，用语言去描述文件的组织结构。但如果自己定义一套新的语法，学习成本则太高。   
### 基于SQL       
于是就想到了数据库，数据库是通过create table来表示文件格式的，且通过sql来查询底层数据。     
这个create table和select操作和我的需求match,就这样SQL查询引擎Rider诞生了。   
### Rider代码灵感
Rider借鉴了不少项目的代码，例如MySql协议部分借鉴了Corbar。   
Sql解析部分借鉴了h2database,derby等。  
文件解析部分源于笔者写的大部分文件解析业务代码。   
在此向上述优秀的开源代码致敬。       
## SQL查询引擎Rider
Rider是一个基于Netty通讯框架的纯java写的Server,其不依赖其它任何服务。其主要功能如下图所示:         
![codegen](/Users/alchemystar/image/rider/rider_func.png)     
(1)Rider基于MySql协议和用户交互，用户可以使用mysqlClient、jdbc以及odbc等对Rider发送SQL命令   
(2)Rider支持select join where condition、create table等语法   
(3)Rider支持MyBatis   
## Rider总体设计   
![codegen](/Users/alchemystar/image/rider/rider_archetype.png)    
这里Rider主要分四层:   
(1)MySql协议层，负责通过MySql协议与用户的交互，详情可见:     
https://my.oschina.net/alchemystar/blog/834150    
(2)Sql解析层:负责对select以及create table等语法的解析     
(3)Access层:提供游标Cursor这个概念，供Sql解析层去遍历记录      
(4)Storage层:对很多中文件格式进行解析，统一封装成游标Cursor给上层调用,         
当前Storage还包含了视图的概念，这是Rider另一个特性，在后面的篇章中阐述。               
## Rider查询表的原理 
下图是Rider查询表的原理，    
![codegen](/Users/alchemystar/image/rider/rider_execute.png)    
Rider查询表的原理是通过将文件中所有记录读取出来并通过where或者join条件进行遍历，从而筛选出对应的记录。      
对于多表查询，则是通过将多个文件中的记录进行笛卡尔积的便利来筛选记录。      
## Rider文件配置的通用性    
### 文件列位置不定
详细描述:文件A,文件B包含相同的数据，只是列的位置不一样，例如:
文件A:

```
1,lancer,lancer_comment   
2,rider,rider_comment
```

文件B:

```
1.lancer_comment,lancer    
2,rider,rider_comment  
```  
在Rider中只需要在不同的schema中建立两张相同的表t_test,就可以在应用端代码复用，底层细节的Rider全包了。    

```
use schemaA;
create table t_test( 
  id BIGINT comment 'id test ', 
  name VARCHAR comment 'name',
  extension VARCHAR comment 'extension' 
)Engine='archer' SEP=',' comment='just for test';
use schemaB;
 create table t_test( 
  id BIGINT comment 'id test ', 
  extension VARCHAR comment 'extension' /*此处列位置调整*/
  name VARCHAR comment 'name',
)Engine='archer' SEP=',' comment='just for test'
```
这样客户端就可以不考虑文件列的位置了。     
## 文件格式不固定
考虑到三个文件，文件A、文件B以及文件C
文件A,以,分隔:    

```
1,lancer,lancer_comment   
2,rider,rider_comment
```

文件B,以|分隔:    

```
1|lancer|lancer_comment   
2|rider|rider_comment
```
文件C,XLSX格式    

```
use schemaA;
create table t_test( 
  id BIGINT comment 'id test ', 
  name VARCHAR comment 'name',
  extension VARCHAR comment 'extension' 
)Engine='archer' SEP=',' comment='just for test';
use schemaB;
 create table t_test( 
  id BIGINT comment 'id test ', 
  name VARCHAR comment 'name',
  extension VARCHAR comment 'extension' 
)Engine='archer' SEP='|' /*此处分隔符调整为|*/  comment='just for test'
use schemaC;
create table t_test( 
  id BIGINT comment 'id test ', 
  name VARCHAR comment 'name',
  extension VARCHAR comment 'extension' 
)Engine='XLSX'/*此处引擎调整为xlsx*/;
```
这样客户端也不需要考虑文件格式了。   
如果上述不直观的话，可以如下图所示:     
![codegen](/Users/alchemystar/image/rider/rider_file.png) 
## Rider性能
文件解析速度4W行/s,其只和java本身文件IO性能相关。  
## Rider截图 
![codegen](/Users/alchemystar/image/rider/rider_example.png)     

## github链接     
https://github.com/alchemystar/Rider
## 码云链接
http://git.oschina.net/alchemystar/Rider    