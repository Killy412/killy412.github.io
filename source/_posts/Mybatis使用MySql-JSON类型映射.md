---
title: Mybatis使用MySql JSON类型映射
date: 2020-11-23 19:33:44
tags:
  - "Mybatis"
  - "MySql"
  - "JSON"
categories:
  - "Mybatis"
---

mysql在5.7.8版本以后就支持了json这种数据类型,在程序中处理的方法有很多,可以直接用string去接收json类型,然后再转. 也可以使用自定义转换器,需要继承`BaseTypeHandler<>`类,然后在配置文件中加个扫描路径,在使用的地方指定转换器.

1. 自定义转换器

```java
package com.abc.common.handler;
/**
 * @MappedTypes和BaseTypeHandler的类型都是需要使用哪个数据类型去接受mysql中的json串.
 */
@MappedTypes(JSONArray.class)
@MappedJdbcTypes(JdbcType.OTHER)
public class JsonArrayHandler extends BaseTypeHandler<JSONArray> {
    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, JSONArray objects, JdbcType jdbcType) throws SQLException {
        ps.setString(i, String.valueOf(objects.toJSONString()));
    }

    @Override
    public JSONArray getNullableResult(ResultSet resultSet, String s) throws SQLException {
        String sqlJson = resultSet.getString(s);
        if (null != sqlJson) {
            return JSONArray.parseArray(sqlJson);
        }
        return null;
    }

    @Override
    public JSONArray getNullableResult(ResultSet resultSet, int columnIndex) throws SQLException {
        String sqlJson = resultSet.getString(columnIndex);
        if (null != sqlJson) {
            return JSONArray.parseArray(sqlJson);
        }
        return null;
    }

    @Override
    public JSONArray getNullableResult(CallableStatement callableStatement, int columnIndex) throws SQLException {
        String sqlJson = callableStatement.getString(columnIndex);
        if (null != sqlJson) {
            return JSONArray.parseArray(sqlJson);
        }
        return null;
    }
}
```

2. 在配置文件中加上类型处理包
```xml
# MyBatis配置
mybatis:
  type-handlers-package: com.abc.common.handler
```

3. 使用处指定类型转换器

```xml
<!-- resultMap标签指定处理器 -->
<resultMap type="GxInquiryRecord" id="GxInquiryRecordResult">
    <result property="inquiryDetail" column="inquiry_detail"
            typeHandler="com.abc.common.handler.JsonArrayHandler"/>
</resultMap>

<!-- 新增语句时的处理 -->
<insert id="insertGxInquiryRecord" parameterType="GxInquiryRecord">
    insert into gx_inquiry_record(inquiry_detail) 
    values ( #{inquiryDetail,jdbcType=OTHER,typeHandler=com.abc.common.handler.JsonArrayHandler})
</insert>
```