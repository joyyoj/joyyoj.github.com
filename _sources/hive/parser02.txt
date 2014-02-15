***************************************
HIVE的数据类型/数据模型/对象模型
***************************************

[HIVE API的JavaDoc](http://docs.hortonworks.com/HDPDocuments/HDP1/HDP-Win-1.1/ds_Hive/api/index.html)

数据模型
===================

HIVE有四种数据模型（表）：(内部)表、外部表、分区表和桶表。下面把thrift的重要接口定义罗列说明一下。

Database
------------------

Database是Table的命名空间，包含一组Table ::

    struct Database {
      1: string name,
      2: string description,
      3: string locationUri,
      4: map<string, string> parameters, // properties associated with the database
      5: optional PrincipalPrivilegeSet privileges
    }

Table
-----------------------
::

    // table information
    struct Table {
      1: string tableName,// name of the table
      2: string dbName,   // database name ('default')
      3: string owner,// owner of this table
      4: i32 createTime,   // creation time of the table
      5: i32 lastAccessTime,   // last access time (usually this will be filled from HDFS and shouldn't be relied on)
      6: i32 retention,// retention time
      7: StorageDescriptor sd,// storage descriptor of the table
      8: list<FieldSchema> partitionKeys, // partition keys of the table. only primitive types are supported
      9: map<string, string> parameters,   // to store comments or any other user level parameters
      10: string viewOriginalText, // original view text, null for non-view
      11: string viewExpandedText, // expanded view text, null for non-view
      12: string tableType, // table type enum, e.g. EXTERNAL_TABLE
      13: optional PrincipalPrivilegeSet privileges,
    }
    
Partition
--------------
::
    struct Partition {
      1: list<string> values // string value is converted to appropriate partition key type
      2: string   dbName,
      3: string   tableName,
      4: i32  createTime,
      5: i32  lastAccessTime,
      6: StorageDescriptor   sd,
      7: map<string, string> parameters,
      8: optional PrincipalPrivilegeSet privileges
    }
    
Index
---------------
::
    struct Index {
      1: string   indexName, // unique with in the whole database namespace
      2: string   indexHandlerClass, // reserved
      3: string   dbName,
      4: string   origTableName,
      5: i32  createTime,
      6: i32  lastAccessTime,
      7: string   indexTableName,
      8: StorageDescriptor   sd,
      9: map<string, string> parameters,
      10: bool deferredRebuild
    }

StorageDescriptor
---------------------

描述一张表的物理存储 ::

    struct StorageDescriptor {
      1: list<FieldSchema> cols,  // required (refer to types defined above)
      2: string location, // defaults to <warehouse loc>/<db loc>/tablename
      3: string inputFormat,  // SequenceFileInputFormat (binary) or TextInputFormat`  or custom format
      4: string outputFormat, // SequenceFileOutputFormat (binary) or IgnoreKeyTextOutputFormat or custom format
      5: bool   compressed,   // compressed or not
      6: i32numBuckets,   // this must be specified if there are any dimension columns
      7: SerDeInfoserdeInfo,  // serialization and deserialization information
      8: list<string> bucketCols, // reducer grouping columns and clustering columns and bucketing columns`
      9: list<Order>  sortCols,   // sort order of the data in each bucket
      10: map<string, string> parameters, // any user supplied key value hash
      11: optional SkewedInfo skewedInfo, // skewed information
      12: optional bool   storedAsSubDirectories   // stored as subdirectories or not
    }


FieldSchema
-----------------
::

    struct FieldSchema {
      1: string name, // name of the field
      2: string type, 
      3: string comment
    }

SerdeInfo
----------------
// This object holds the information needed by SerDes ::

    struct SerDeInfo {
      1: string name,   // name of the serde, table name by default
      2: string serializationLib,   // usually the class that implements the extractor & loader
      3: map<string, string> parameters // initialization parameters
    }


Column支持哪些数据类型
=======================

Hive使用TypeInfo描述数据类型，目前支持5种类型.

1. Primitive
2. List
3. Map
4. Struct
5. Union

具体的定义如下（只列出get接口） ::

    public abstract class TypeInfo implements Serializable {  
      public abstract Category getCategory(); 
      public abstract String getTypeName();
    }
    public final class PrimitiveTypeInfo extends TypeInfo implements Serializable {
      public PrimitiveCategory getPrimitiveCategory();
      public Class<?> getPrimitiveWritableClass();
      public Class<?> getPrimitiveJavaClass();
    }
	public final class MapTypeInfo extends TypeInfo implements Serializable { 
      public TypeInfo getMapKeyTypeInfo()；
      public TypeInfo getMapValueTypeInfo()；
    }
    public final class ListTypeInfo extends TypeInfo implements Serializable {
      public TypeInfo getListElementTypeInfo();
    }
    public final class StructTypeInfo extends TypeInfo implements Serializable {
      public ArrayList<String> getAllStructFieldNames();
      public ArrayList<TypeInfo> getAllStructFieldTypeInfos();
      public TypeInfo getStructFieldTypeInfo(String field);
    }
    public class UnionTypeInfo extends TypeInfo implements Serializable {
      public List<TypeInfo> getAllUnionObjectTypeInfos();
    }
        
union类型稍微提一下，union与c的union相似，即在同一时间点，只能有一个指定的数据类型 ::

    CREATE TABLE union_test(value UNIONTYPE<int, array<string>, struct<a:int,b:string>, map<string, string>>);
    SELECT value FROM union_test;    
    {0:1}
    {1:["val1","val2"]}
	{2:{"a":1,"b":"val"}}
	{3:{"key":"value","x":"y"}}

对于这些复合类型，有相应的UDF来创建，包括：array,map,struct,named_struct,create_union。

复杂数据类型的支持
=====================

从数据类型可以看出，hive支持嵌套struct、array、map等复杂结构。那么HIVE是如何实现的？
例如：`select a.b, key[0] from src; select a+b from src group by a+b;...`

类型检查
----------------

TypeCheck

::

	// select a+b, count(1) from T group by a+b;
	// a+b在group by时，处理过，因此RowResolver时已经包含a+b
	processGroupby
	if (.) {
		Column.Const ExprFieldDesc
		ColumnInfo info = get(tableAlias);
	}

如何生成相应的ExprNodeDesc？

2    => ExprConstantDesc

a    => ExprColumnDesc

a.b  => ExprFieldDesc

a[0] => ExprGenericFuncDesc

**问题**

1. 语法解析，构建AST？
2. 语义分析中，如何处理[]?
3. 语义分析中，如何.？与dbName.tableName.columnName与嵌套结构的struct如何区分？
4. 运行时如何获取这些类型信息？ 

问题4先解答：
编译OK之后，进行运行，那么如何传递TypeInfo信息给Task？
HIVE内部有个自己的TypeInfoParser来解析TYPE字符串；

serdeParams.columnTypes -> columnType -> 
columTypes从FieldSchema中来。

对于中间结果，它的FieldSchema又如何生成？
PlanUtils.getFieldSchemasFromRowSchema(parent.getSchema(), "temporarycol")

那么关键的问题在于RowSchema如何生成？

每个Operator都有一个RowSchema；RS的RowSchema

显然，如果是select算子，则找出outputCols的则它的RowSchema？
如何知道OutputCols有哪些？

#### Parser

首先，为了支持上述的语法解析，Hive.g的部分定义如下：

	precedenceFieldExpression
    :
    atomExpression ((LSQUARE^ expression RSQUARE!) | (DOT^ Identifier))*
    ;

    atomExpression
    :
    KW_NULL -> TOK_NULL
    | constant
    | function
    | castExpression
    | caseExpression
    | whenExpression
    | tableOrColumn
    | LPAREN! expression RPAREN!
    ;


类型检查、转换
================

HIVE引入几个数据结构。

FieldSchema
ColumnInfo
RowSchema
RowResolver
ColumnInfo

::

    class ColumnInfo {
    	String internalName;
		ObjectInspector objectInspector;
    	String tabAlias;
    	boolean isVirtualCol;
    	boolean isHiddenVirtualCol;
    }
	class RowSchema implements Serializable { 
    	ArrayList<ColumnInfo> signature;
    }
   
在类型检查时，TypeCheckProcFactory提供了几个Processor来处理：

NullExprProcessor;
NumberExprProcessor;
StringExprProcessor;
BoolExprProcessor;
ColumnExprProcessor;
DefaultExprProcessor；//处理function

**问题**

1. 到此为止，我们知道给定输入的RowSchema，得到相应的ExprNodeDesc。例如select算子，我们知道要选择字段的类型，但是对于Operator之间如何传递？
1. RowSchema和RowResolver什么关系？

RowSchema可以从RowResolver中生成。RowResolver维护了tableAlias,columnAlias,ColumnInfo
internalName -> alias的关系维护等；
在算子层：操作符之间的出入schema由RowResolver完成传递；
RowResolver如何维护？显然从genTablePlan时，获取得到Table对应的ObjectInspector(此时必然是StructObjectInspector），获取每一列的类型信息；
【不妥之处：从SerdeParams中获取】

getOpParseCtx
Operator的ParseContext中包含了每个Operator与RowResolver的映射关系；


如果是FileSinkOperator、CommonJoinOperator？

public ColumnInfo(String internalName, TypeInfo type, String tabAlias, boolean isVirtualCol)

事实上HIVE不支持dbname.tablename.columnName，即只支持columnOrXPath, tableName.columnOrXPath；


在函数层：操作符之间的出入的schema由ObjectInspector；

如何检查函数的参数是否正确？由UDF来保证，UDF能够返回ObjectInspector；

描述符：ExprNodeDesc

HIVE支持数据类型的隐式转换。如:

    insert into table string_table 
    select i32 as key from src; 

HIVE的union all如何处理？

TypeInfo
----------------------

对象模型
####################

ObjectInspector
======================

HIVE的运行时支持多种对象模型，ObjectInspector的引入的主要意义在于提供了对Object的统一访问方式，换句话说，它屏蔽了不同对象的存储细节。目前有两种对象模型的实现：

1. Java的对象模型（Thrift或者原生Java）
2. Hadoop的对象模型（Writable)

接口定义 ::

	public interface ObjectInspector {
		public static enum Category {
    		PRIMITIVE, LIST, MAP, STRUCT, UNION
  		};
  		String getTypeName();
		Category getCategory();
	}

PrimitiveObjectInspector
=========================

::

    public interface PrimitiveObjectInspector extends ObjectInspector {
      public static enum PrimitiveCategory {
    VOID, BOOLEAN, BYTE, SHORT, INT, LONG, FLOAT, DOUBLE, STRING, TIMESTAMP, BINARY, UNKNOWN
      };
    
      PrimitiveCategory getPrimitiveCategory();

      /**
       * Get the Primitive Writable class which is the return type of
       * getPrimitiveWritableObject() and copyToPrimitiveWritableObject().
       */
      Class<?> getPrimitiveWritableClass();
    
      /**
       * Return the data in an instance of primitive writable Object. If the Object
       * is already a primitive writable Object, just return o.
       */
      Object getPrimitiveWritableObject(Object o);
    
      /**
       * Get the Java Primitive class which is the return type of
       * getJavaPrimitiveObject().
       */
      Class<?> getJavaPrimitiveClass();
    
      /**
       * Get the Java Primitive object.
       */
      Object getPrimitiveJavaObject(Object o);
    
      /**
       * Get a copy of the Object in the same class, so the return value can be
       * stored independently of the parameter.
       *
       * If the Object is a Primitive Java Object, we just return the parameter
       * since Primitive Java Object is immutable.
       */
      Object copyObject(Object o);
    
      /**
       * Whether the ObjectInspector prefers to return a Primitive Writable Object
       * instead of a Primitive Java Object. This can be useful for determining the
       * most efficient way to getting data out of the Object.
       */
      boolean preferWritable();
    }

StructObjectInspector

::

  public abstract class StructObjectInspector implements ObjectInspector {
    //Returns all the fields.
    public abstract List<? extends StructField> getAllStructFieldRefs();

    //Look up a field.
    public abstract StructField getStructFieldRef(String fieldName);

    public abstract Object getStructFieldData(Object data, StructField fieldRef);
    public abstract List<Object> getStructFieldsDataAsList(Object data);
  }

  public interface StructField {
    //Get the name of the field. The name should be always in lower-case.
    String getFieldName();

    //Get the ObjectInspector for the field.
    ObjectInspector getFieldObjectInspector();

    String getFieldComment();
  }

MapObjectInspector

::

    public interface MapObjectInspector extends ObjectInspector {
      ObjectInspector getMapKeyObjectInspector(); 
      ObjectInspector getMapValueObjectInspector(); 
      Object getMapValueElement(Object data, Object key);
    
      Map<?, ?> getMap(Object data);
      
      int getMapSize(Object data);
    }


ListObjectInspector

::

    public interface ListObjectInspector extends ObjectInspector { 
      // Methods that does not need a data object **
      ObjectInspector getListElementObjectInspector(); 
      Object getListElement(Object data, int index); 
      int getListLength(Object data);
    
      List<?> getList(Object data);
    }

运行时如何使用ObjectInspector
==================================

::

	//反序列化得到行
	Object row = serDe.deserialize(t);
	StructObjectInspector oi = (StructObjectInspector) serDe 
        .getObjectInspector();
 	List<? extends StructField> fieldRefs = oi.getAllStructFieldRefs();
	//获取每一列的信息 
	for (int i = 0; i < fieldRefs.size(); i++) { 
      Object fieldData = oi.getStructFieldData(row, fieldRefs.get(i)); 
      ...
  	}
从这个例子中，不难出，Hive将对行中列的读取和行的存储方式解耦和了，只有ObjectInspector清楚行的结构，但使用者并不知道存储的细节。 对于数据的使用者来说，只需要行的Object和相应的ObjectInspector，就能读取出每一列的对象。 

ObjectInspector有什么好处
------------------------------

除了上文说的，ObjectInspector提供对象的统一访问接口，还负责获取TypeInfo。另外，可以实现惰性反序列化。

编译时也用了ObjectInspector
-----------------------------

HIVE支持用户自己配置serde lib，例如protobuf。列信息在serde的class中 ::

	static List<FieldSchema> getFieldsFromDeserializer(String tableName, Deserializer deserializer) {
	 	获取ObjectInspector；
		根据ObjectInspector的类型获取Field的OI；
		返回FieldSchema；
 	}

ObjectInspector的信息:
TypeInfo getTypeInfoFromObjectInspector();


genTablePlan
================

tab.getDeserializer().getObjectInspector();

待续...
