# JSONDB

## Problem

设计一个单机数据库用于存储百亿级别的json对象。

这些json对象只有一层（即没有nested field），且fields的值只能是字符串，数值，布尔等原始类型（即不能是数组）。

这些json对象的fields不一定相同；即使两个json对象有一模一样的fields，其值的类型也不一定相同。下面是三个
json对象的例子。

```
json1

{
  name: "John",
  age: 45，
  gender: 'M'
}

json2

{
  title: "Software engineer",
  salary: 45000,
  age: "middle age"
}

json3

{
  title: "Github flavored markdown",
  author: "Tim",
  pages: 450
}
```

从上面的例子可以看出，json1和json3的fields完全不同，json1和json2虽然都有age这个field，但在json1中它是数值
型；在json2中，它是字符串。  

这个系统要能支持json对象的CRUD操作。在单个对象上的操作应该是原子的。

系统要能支持点查询和范围查询：

```
点查询

select * from collection where name="John"

范围查询

select * from collection where age > 30

```

第一个点查询将返回所有1)具有name这个field，2）name field的值是字符串，3）字符串的值为John的json对象。

第二个范围查询将返回所有1）具有age这个field，2）age field的值是数值，3）数值的值大于30的json对象。

