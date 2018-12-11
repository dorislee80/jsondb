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

## 设计
### 思路一 MongoDB
可以模仿MongoDB的做法，每个json插入时赋予一个id，然后基于这个id建立一个基于B+-tree的数据库。

这种做法的问题是对上述点查询和范围查询，都必须扫描所有记录（因为对这两个域没有索引），性能会极差。也可以继续模仿mongodb的做法，允许
用户指定几个fields来建立索引。但由于这些json并不“同质”，用户的查询习惯也会随时间和人而改变，这个方法需要后续大量的维护成本。

### 思路二 Lucene
模仿Lucene的做法，为所有json对象中出现的所有fields都建立一个倒排索引。上面的例子可以得到下面的索引：

```
name
John -> 1

age
45 -> 1
"middle age" -> 2

gender
M -> 1

title
"Software engineer" -> 2
"Github flavored markdown" -> 3

salary
45000 -> 2

author
tim -> 3

pages
450 -> 3
```

有了上述倒排索引后，我们再用一个hash（Key为field name, value为倒排索引地址）进行索引。倒排索引内部，我们可以用b+-tree来对term进行索引。

对原始json对象，我们可以建立一个以id为索引(用hash）的存储。

具备上述索引以后，我们来看看第一个查询的执行。
1. 先根据hash-based field index找到name对应的倒排索引
2. 再通过b+-tree-based的term index找到"John"对应的Posting list
3. 再通过hash-based的id索引，一一读出原始json对象


对第二个范围查询，其步骤如下
1. 先根据hash-based field index找到age对应的倒排索引
2. 再通过b+-tree-based的term index发现所有值为数值且大于30的所有term，并求出这些posting lists的并集
3. 再通过hash-based的id索引，一一读出原始json对象

### 思路三 Azure CosmosDB
将所有的json都看作一棵小树。那么把所有这些树并在一起将得到一棵大树。这些小树和大树都是三层，最上一层是根，中间一层是fields，最下一层
是values。叶子节点包含一个指向Posting list的指针。

这颗大树实际上就是一个以path为term的覆盖所有小树的倒排索引。为在磁盘上存放这棵大树，我们采用如下方案：

每一条从根到叶子节点的路径被称为一个term，比如/age/45, /age/“middle age"。
搞一个B+-tree或者Bw-tree来索引这些term。

再有一个单独的key-value存储（比如采用rockdb）来存放id -> 原始Json。

现在看看第一个查询的执行过程：
1. 分析查询得到term "/name/John"
2. 搜索term b-tree索引找到/name/John，从而得到其对应的Posting list
3. 再通过key-value存储，以posting list中的id为key读取原始Json.

第二个查询的执行过程
1. 分析查询得到term "/age/30"
2. 搜索term b-tree索引遍历所有/age/nnn (nnn>30), 从而得到一个posting list的并集
3. 再通过key-value存储，以posting list中的id为key读取原始Json.

这种方案的好处是：1）所有json对象的所有fields都被自动索引了，不需要用户告诉系统哪些fields需要被索引；2）把field和value统一到一棵树中，
使得索引的设计更纯粹。

和方案二相比的空间分析？时间分析？CUD操作的具体过程？