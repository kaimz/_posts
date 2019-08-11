---
title: No Dialect mapping for JDBC type 的问题
date: 2017-12-11 18:38:03
tags: [java]
categories: 学习笔记
---

问题很简单，就是方言不能把该数据库类型映射到Java中， 方言不认识这个数据库类型。

#### 出现原因
1. 数据库字段类型和JAVA类型不匹配。
2. 错误地配置了数据库方言。

<!--more-->

#### 解决方案

首先查看java.sql.Types类型，以及各种错误代码。

序号 | 类型 | 状态码
---|---|--
1|	ARRAY|	2003
2|	BIGINT|	-5
3|	BINARY|	-2
4|	BIT|	-7
5|	BLOB|	2004
6|	BOOLEAN	|16
7|	CHAR|	1
8|	CLOB|	2005
9|	DATALINK|	70
10|	DATE|	91
11|	DECIMAL|	3
12|	DISTINCT|	2001
13|	DOUBLE|	8
14|	FLOAT|	6
15|	INTEGER|	4
16|	JAVA_OBJECT	|2000
17	|LONGNVARCHAR|	-16
18|	LONGVARBINARY|	-4
19|	LONGVARCHAR	|-1
20|	NCHAR|	-15
21|	NCLOB|	2011
22|	NULL|	0
23|	NUMERIC	|2
24|	NVARCHAR|	-9
25|	OTHER|	1111
26|	REAL|	7
27|	REF	|2006
28|	ROWID|	-8
29|	SMALLINT|	5
30|	SQLXML|	2009
31|	STRUCT|	2002
32|	TIME|	92
33|	TIMESTAMP	|93
34|	TINYINT|	-6
35|	VARBINARY|	-3
36|	VARCHAR	| 

##### 方案一
根据错误代码就可以找到我们是哪一种错误。

比如：我这次出现的错误的是No Dialect mapping for JDBC type: 1111
输入 序号25 中的 other 。
复制sql 执行查询发现，它查出的数据库类型是``unknown``。

现在知道了问题所在，可以执行sql 的时候指定数据类型，各种数据库都有相应的函数或方法转换。

例外，如果是方言配置错误，需要配置正确的方言。


##### 方案二
既然是方言的问题，我们可以取看下方言的实现，我使用的是postgresql，就以它为例，它的实现其实就是这样的。
```java
public class PostgreSQL81Dialect extends Dialect {

	/**
	 * Constructs a PostgreSQL81Dialect
	 */
	public PostgreSQL81Dialect() {
		super();
		registerColumnType( Types.BIT, "bool" );
		registerColumnType( Types.BIGINT, "int8" );
		registerColumnType( Types.SMALLINT, "int2" );
		registerColumnType( Types.TINYINT, "int2" );
		registerColumnType( Types.INTEGER, "int4" );
		registerColumnType( Types.CHAR, "char(1)" );
		registerColumnType( Types.VARCHAR, "varchar($l)" );
		registerColumnType( Types.FLOAT, "float4" );
		registerColumnType( Types.DOUBLE, "float8" );
		registerColumnType( Types.DATE, "date" );
		registerColumnType( Types.TIME, "time" );
		registerColumnType( Types.TIMESTAMP, "timestamp" );
		registerColumnType( Types.VARBINARY, "bytea" );
		registerColumnType( Types.BINARY, "bytea" );
		registerColumnType( Types.LONGVARCHAR, "text" );
		registerColumnType( Types.LONGVARBINARY, "bytea" );
		registerColumnType( Types.CLOB, "text" );
		registerColumnType( Types.BLOB, "oid" );
		registerColumnType( Types.NUMERIC, "numeric($p, $s)" );
		registerColumnType( Types.OTHER, "uuid" );

		registerFunction( "abs", new StandardSQLFunction("abs") );
		registerFunction( "sign", new StandardSQLFunction("sign", StandardBasicTypes.INTEGER) );
        
       //  ……太多了，
	}

```
去看下 java.sql.Types的定义
![image](https://ws1.sinaimg.cn/large/ece1c17dgy1fmwxz829cwj20ex0aw74o.jpg)

明白了它是我们上面表格中状态码定义的地方。

所以我们做一个方言，只要在构造函数中实现相应类型码的对应类类型就可以了，上面是``1111`` 出现错误，我只要将它的类型重新定义为``text``即可，就是它怎么写的我们就怎么写。
```java

import org.hibernate.dialect.PostgreSQLDialect;

import static java.sql.Types.OTHER;

/**
 * 重写方言，指定代码相应的数据库类型
 * @author Zhang Kai
 * @version 1.0
 * @since <pre>2017/12/11 14:28</pre>
 */
public class UserPostgreSQLDialect extends PostgreSQLDialect {

    public UserPostgreSQLDialect() {
        super();
        registerColumnType(OTHER, "text");
    }
}

```
然后在数据库的配置文件中，将方言的类改成我们自定义的就OK了。
