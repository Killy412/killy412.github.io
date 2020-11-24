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

### MyBatis中使用自定义转换器处理json类型

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


### MySql常用Json函数

- json是数组的话,使用`$[0]`来确定元素
- json是对象的话,使用`$.today`来确定元素
  
```sql
# json数组移除某个元素
select json_remove('[{"today": "周二", "tomorrow": "周三"},  {"today": "周四", "tomorrow": "周六"}]','$[0]')
# json对象移除某个键
select json_remove('{"today": "周四", "tomorrow": "周六"}','$.today')
# json数组移除某个元素中的键
select json_remove('[{"today": "周二", "tomorrow": "周三"}, {"today": "周四", "tomorrow": "周六"}]','$[0].today')
# json 对象合并
select json_merge('[1,2,3]','[4,5]')
# 是否包含某个值,包含返回1,否则返回0
select json_contains('{"today": "周四", "tomorrow": "周六"}','{"today":"周无"}')
# 是否包含某个路径,第二个参数可以为`all`/`one` all表示所有路径必须包含,one可以只包含一个
select json_contains_path('{"today": "周四", "tomorrow": "周六"}','all','$.today','$.tomorrow');
# 提取json某个值
select json_extract('[{"today": "周四", "tomorrow": "周六"}]','$[0].today');
# 添加json键,如果已经存在不添加
select json_insert('{"today": "周四", "tomorrow": "周六"}','$.desc','周二');
# 添加json元素
select json_array_append('[{"today": "周四", "tomorrow": "周六"}]','$','[{"today": "周四", "tomorrow": 周六}]')
# 添加json元素
select json_array_append('[{"today": "周四", "tomorrow": "周六"}]','$[0].today','周日')
```

更多函数见[mysql json函数官方文档](https://dev.mysql.com/doc/refman/5.7/en/json-function-reference.html)