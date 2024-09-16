---
layout: post
title:  Type conversion between Spring and DB
categories: [spring-boot]
tags: [spring-boot, DB]
description: Automatic type conversion in ORM frameworks and SQL Mappers is convenient, but explicit type handling is safer to prevent errors.
---

It is a column that is clearly set to the _int_ type in the DB table, but when the value of the column was sent as a _String_ type in the Where condition of the SQL statement, the search was successful.

To give an example, IDX is clearly a column created in the DB as an _int_ type, but even when searched using Mybatis as shown below, it selects normally.

```sql
@Select(value = "SELECT name FROM testTBL WHERE IDX = #{idx}")
String test(@Param("idx") String idx);
```

**The reason it was searched normally was because of the type conversion function provided by the database driver, ORM framework, and SQL Mapper.**

Most database drivers, ORM frameworks, and SQL Mappers can perform type conversion by default when importing data. 

Thanks to this, they automatically convert the type if the query result data type and the method return type are different.

It is a very convenient feature, but of course there are some things to be careful about.

First, if the result type of the query and the type used in the application code are different (of course, it is best to match them the same during development), it is best to explicitly declare type conversion. 

**It is always risky to entrust your code to something that does it automatically.**

You should also be careful about conversion errors. 

For example, during automatic conversion, if a large number is converted to a string, the data may be truncated and sent due to string length restrictions.

In conclusion, data access technologies such as ORM frameworks and SQL Mappers provide a convenience function called automatic type conversion, but it is recommended to manage them explicitly if possible.