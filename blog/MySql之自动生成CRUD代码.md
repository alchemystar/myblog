#MySql之自动生成CRUD代码
MyBatis能够通过获取MySql中的information_schema从而获取表的字段等信息，最后通过这些信息生成代码。  
笔者受此启发，将MyBatis-Generator中的核心结构体剥离出来，写成了能自动生成简单CRUD的工具。
##自动生成代码原理图
![codegen](/Users/alchemystar/image/codegen/codegen-main.png) 
##information_schema
mysql本身存在一个information_schema,记录了所有的元数据信息，主要的几个有:  
schema表：当前mysql实例中所有数据库的信息。    
COLUMNS表：关联了所有表和其中列的信息。    
TABLES表：提供了关于数据库中的表的信息。    
......
##jdbc中的MetaData
jdbc提供了非常方便的工具帮助我们获取这些元数据信息，就是MetaData。获取MetaData的代码如下:

```
 Connection connection = getConnection();   
 DatabaseMetaData metaData = connection.getMetaData();
```
###metaData获取一张表中的所有字段
通过metaData.getColumns方法在指定了schema和table后可以很方便的获取一张表中的所有字段,代码如下: 
   
```

    private void caculateColumns(DatabaseMetaData metaData) {

        ResultSet rs = null;
        try {
            rs = metaData.getColumns("", introspectedTable.getIntrospectedSchema(),
                    introspectedTable.getIntrospectedTableName(), null);

            while (rs.next()) {
                IntrospectedColumn introspectedColumn = new IntrospectedColumn();
                introspectedColumn.setJdbcType(rs.getInt("DATA_TYPE")); //$NON-NLS-1$
                introspectedColumn.setLength(rs.getInt("COLUMN_SIZE")); //$NON-NLS-1$
                introspectedColumn.setActualColumnName(rs.getString("COLUMN_NAME")); //$NON-NLS-1$
                introspectedColumn.setNullable(rs.getInt("NULLABLE") == DatabaseMetaData.columnNullable); //$NON-NLS-1$
                introspectedColumn.setScale(rs.getInt("DECIMAL_DIGITS")); //$NON-NLS-1$
                introspectedColumn.setRemarks(rs.getString("REMARKS")); //$NON-NLS-1$
                introspectedColumn.setDefaultValue(rs.getString("COLUMN_DEF")); //$NON-NLS-1$

        } catch (SQLException e) {
            e.printStackTrace();
        }

    }
```     
##metaData获取表中的主键
同样的通过getPrimaryKeys可以很轻松的计算其主键，注意主键可能是联合主键:
对应的代码如下:

```
       ResultSet rs = null;
        String cataLog;

        try {
            rs = metaData.getPrimaryKeys("", introspectedTable.getIntrospectedSchema(),
                    introspectedTable.getIntrospectedTableName());
            Map<Short, String> keyColumns = new TreeMap<Short, String>();
            while (rs.next()) {
                String columnName = rs.getString("COLUMN_NAME");
                short keySeq = rs.getShort("KEY_SEQ");
                keyColumns.put(keySeq, columnName);
            }
            for (String columnName : keyColumns.values()) {
                introspectedTable.addPrimaryKeyColumn(columnName);
            }
        } catch (SQLException e) {
            e.printStackTrace();
            System.exit(1);
        } finally {
            closeResultSet(rs);
        }
```
##将meta信息组织成结构体
有了上面的信息我们就可以将一张表中的所有信息用java结构体表现出来:   
Table表Java结构体: 
  
```
public class IntrospectedTable {

    protected List<IntrospectedColumn> primaryKeyColumns;
    protected List<IntrospectedColumn> baseColumns;
    protected List<IntrospectedColumn> blobColumns;

    protected String introspectedSchema;
    private String introspectedCatalog;
    private String introspectedTableName;
```
字段表Java结构体属性过多，在此不一一赘述，详情请见github
## Velocity渲染:
有了上述的元数据结构体后，就可以用Velocity渲染，例如如下的Velocity模板: 

```
    public ${name} get${name}(${primaryType} ${primaryKey}) {

        return mapper.selectByPrimaryKey(${primaryKey});

    }
```
就会被渲染成(假设primaryKey是id,tableName是Archer,appname是codegen):

```
package com.alchemystar.codegen.dal.dao.auto;
import com.alchemystar.codegen.dal.mapper.auto.ArcherMapper;

public class AutoArcherDao {
    public Archer getArcher(Long id) {

        return mapper.selectByPrimaryKey(id);
     }
}
```
入法炮制，就可以通过元数据信息生成不同的方法。从而生成想要的代码。
##github链接
https://github.com/alchemystar/codegen