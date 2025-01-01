---
title: "sqlit-jdbc与mybatis使用造成类型转换错误bug"
description: 
date: 2024-12-29T11:24:38+08:00
image: hari-nandakumar-hh9OkRFj-Pw-unsplash.jpg
math: 
license: 
hidden: false
comments: true
draft: false
categories:
    - Java
tags:
    - sqlite
    - mybatis
---

最近在项目中用到sqlite数据库作为底层数据存储，驱动层顺理成章的采用了sqlite-jdbc+mybatis的组合，但在使用的过程中，偶现了一个非常奇异的bug.

#### 复现：

相关组件版本：

- sqlite-jdbc 3.46.1.0
- mybatis-spring-boot-starter 3.0.3
- sqlite 3.45.0

在sqlite数据库里创建test表，id(bigint) 和 name(text) 两列，插入两条记录：

```
id       name
22	test22
850361281191579648	 tt
```

接着建个简单工程，dao层mapper.xml，整个完整工程示例可参考：https://github.com/sailingsky/sqlite-test.git

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.sqlitetest.mapper.TestMapper">

    <select id="queryMaps" resultType="java.util.Map">
        select * from test
    </select>
</mapper>
```

接着controller直接调用mapper返回查询结果，本来正常的结果应该是：

```json
[{"name":"test22","id":22},{"name":" tt","id":850361281191579648}]
```

但实际获得的结果是:

```json
[{"name":"test22","id":22},{"name":" tt","id":1881903104}]
```



很明显，第2条记录的id值被改变了，bug就这样产生了。



#### 根源

追踪整个代码执行逻辑，关键链路如下：

DefaultResultSetHandler.java#createAutomaticMappings --->ResultSetWrapper.java#getTypeHandler

``` java
//解析列对应数据类型
final Class<?> javaType = resolveClass(classNames.get(index)); 
```

而这个`classNames`是在`ResultSetWrapper `被初始化的时候赋值：

``` java
  public ResultSetWrapper(ResultSet rs, Configuration configuration) throws SQLException {
    this.typeHandlerRegistry = configuration.getTypeHandlerRegistry();
    this.resultSet = rs;
    final ResultSetMetaData metaData = rs.getMetaData();
    final int columnCount = metaData.getColumnCount();
    for (int i = 1; i <= columnCount; i++) {
      columnNames.add(configuration.isUseColumnLabel() ? metaData.getColumnLabel(i) : metaData.getColumnName(i));
      jdbcTypes.add(JdbcType.forCode(metaData.getColumnType(i)));
      classNames.add(metaData.getColumnClassName(i));
    }
  }
```

`metaData.getColumnClassName(i)` 这个会去调用sqlite-jdbc的`JDBC3ResultSet.java#getColumnClassName`获取对应数据类型：

``` java
    public String getColumnClassName(int col) throws SQLException {
        switch (safeGetColumnType(markCol(col))) {
            case SQLITE_INTEGER:
                long val = getLong(col);
                if (val > Integer.MAX_VALUE || val < Integer.MIN_VALUE) {
                    return "java.lang.Long";
                } else {
                    return "java.lang.Integer";
                }
            case SQLITE_FLOAT:
                return "java.lang.Double";
            case SQLITE_BLOB:
            case SQLITE_NULL:
                return "java.lang.Object";
            case SQLITE_TEXT:
            default:
                return "java.lang.String";
        }
    }
```

如果是整数型，很明显这个会根据列的值范围去返回对应的类型，所以当这列的第一行id(22)的值为int范围内时，它会返回 "java.lang.Integer",而接下来处理类型处理器的时候，就会根据`classNames`中的类型返回对应列的处理就是"java.lang.Integer"

![image-20250101163314208](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202501011633367.png)

拿到typehandler后，会将其放入到`autoMappingCache`中缓存起来,后续再也不会去获取对应列的typeHandler了，也就是针对id这一列，已经默认了采用"java.lang.Integer"类型进行处理。所以才出现了第一行值返回正确，第二行值却错误的情况，因为Long型值被当做了Integer类型被处理了。

![image-20250101163726643](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202501011637819.png)

#### 解决方案：

针对这个问题，我向sqlite-jdbc和mybatis都提了个issue,https://github.com/xerial/sqlite-jdbc/issues/1177 和 https://github.com/mybatis/mybatis-3/issues/3242。最后得出的结论：在sqlite中，最好在mapper中resultType使用具体的pojo类型，而不是使用`java.util.map`

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.sqlitetest.mapper.TestMapper">

    <select id="queryMaps" resultType="com.example.sqlitetest.po.TestPO">
        select * from test
    </select>
</mapper>
```

