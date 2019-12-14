---
title: "如何更简单方便地执行SQL操作?"
date: 2018/07/13 21:48:00
---
现在公司使用`mybatis`作为DAL层的框架.  
使用起来比较简单,使用xml进行SQL的书写,java代码使用接口执行.  
但在写一些简单SQL的时候会显得非常繁琐:  
1. xml和java分离(设计上为了解耦),一些字段是否设置等需要反复查看(虽然可以通过插件直达);
2.  原生无法热加载,修改xml后需要重启(可以使用三方实现);  
3. xml的动态SQL没有java灵活.    

上述这些"缺点"仅仅针对写简单的sql,特别是第一条.    
我对一张表的需求是简单的CRUD,那使用xml就会非常麻烦.  
  
最基本的需求:  
* 简单安全的SQL拼接方案;
* 方便的类型映射.  

探索了几个方案,接下来一一介绍.  
# JOOQ  
这是考虑的第一个方案,有个没好的愿景,类型安全的SQL,不用担心出现SQL的语法错误.    
但实际使用遇到了比较麻烦的事情,他的codegen.  
```xml
        <generate>
            <pojos>true</pojos>
            <daos>true</daos>
            <keys>false</keys>
            <indexes>false</indexes>
            <globalObjectReferences>false</globalObjectReferences>
        </generate>
```  
自己加了`pojo`和`dao`,一定会生成`Table`,`Record`,洋洋洒洒一堆文件摆在眼前.  
因为是互联网企业,业务也在发现会相当不稳定,管理这坨生成的代码会很麻烦,而且不简洁.  
当然也有不是用codegen的方式,但最后感觉还是过于庞大,也比之前直接用mybatis更麻烦了.  

# ActiveJDBC  
`active record`的实践.  
总体感觉非常简洁清晰,想起了当初用JFinal的感觉.  
当时看了文档就感觉是他了,不需要自己手写SQL,同时兼容多种方言,单测容易用`H2`替换运行.  
但实际使用发现了一个严峻的问题,严峻到不得不放弃他的问题... ...  
他需要`Instrument`的支持.(子类使用父类的静态方法但是要在字节码中改为子类调用(复制了那些方法到子类... ...))
且无论是动态方案还是静态方案都需要修改IDE配置.  
这意味着我只要在现有项目里使用就会有一群开发过来揍我  

# Mybatis SqlBuilder&@SelectProvider   
绕了一圈,回到了原点... ...  
但这个方案的确是满足了解决了上面3个问题.  
`org.apache.ibatis.jdbc.SQL`本身提供了几个方法方便拼写简单的SQL.  
例子如下:  
```java
new SQL()
    .SELECT(COLUMNS)
    .FROM("user AS u")
    .INNER_JOIN("user_info AS ui ON u.user_id = ui.user_id")
    .WHERE("u.user_type = #{userType}")
    .WHERE("u.deleted = 0")
    .ORDER("u.user_id")
    .toString()
    + " LIMIT #{start},#{num}";
```
LIMIT要自己拼,因为他不是标准SQL.  
接下来接口一样定义,用上对应注解(`@UpdateProvider`,`@InsertProvider`,`@DeleteProvider`)  
```java
    @SelectProvider(type = UserProvider.class, method = "getUserByType")
    UserInfo getUserByType(@Param("userType") int userType);
```  

# JdbcTemplate  
使用`NamedParameterJdbcTemplate`会感觉好一点,占位符可以用`:name`的形式.  
我们spring也一起用,对应的依赖也有.  
这个也可行,而且写起来还可以,不过他本身缺少SQL拼接方便的工具,适合写简单没有使用条件分支生成SQL的情况.  
这个网上资料很多,这里稍微列一些常用的需求.  

## 根据class自动映射  
可以使用`BeanPropertyRowMapper`,会按照属性进行SQL字段的关联,java的驼峰会变成小写和下划线的形式进行匹配.  
例子:
```java  
private static final BeanPropertyRowMapper<User> USER_MAPPER = BeanPropertyRowMapper.newInstance(User.class);  

... ...

MapSqlParameterSource params = new MapSqlParameterSource();
params.addValue("userId", userId);
List<ServiceApply> applies = mTemplate.query(
        "SELECT user_id,user_name,created FROM user WHERE user_id=:userId",
        params, USER_MAPPER);
    
```

## 获取插入ID
```java
KeyHolder key = new GeneratedKeyHolder();
mTemplate.update(
        "INSERT INTO user(user_name,age) VALUES (:name,:age)",
        params, key);
Number keyValue = key.getKey();
```  

## 批量插入  
```java
List<MapSqlParameterSource> batchParams = names
        .stream()
        .distinct()
        .map(e -> new MapSqlParameterSource("name", name))
        .collect(Collectors.toList());
mTemplate.batchUpdate(
        "INSERT IGNORE INTO user(user_name) VALUES (:name)",
        batchParams.toArray(new SqlParameterSource[0]));
```  

---  
参考链接:    
https://github.com/shakarelic/mybatis-reload  

https://www.jooq.org/  

http://javalite.io/activejdbc  

https://zh.wikipedia.org/wiki/Active_Record  

http://www.mybatis.org/mybatis-3/statement-builders.html  
  
https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/jdbc/core/namedparam/NamedParameterJdbcTemplate.html
