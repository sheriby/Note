# Neo4j CQL基础

## 常用命令

### `Create`

#### 创建没有属性的节点。

```cql
create (<node-name>:<label-name>)
```

例如

```neo4j
create (emp:Emp)
```

创建了一个节点，节点名为`emp`，其标签为`Emp`。一个节点可以有不止一个标签。

```CQL
create (emp:Emp:Student:Person)
```

该节点有三个不同的标签。

#### 创建拥有属性的节点

```cql
create (
	<node-name>:<label-name> {
		<property-name>:<property-value>,
		...
		<property-name>:<property-value>
	}
)
```

例如

```cql
create (sher:Emp:Student{uid:1, uname:"sher", deptid:10})
```

一个节点可以有多个标签，也可以有多个属性。

### `Match & Return`

`match`和`return`都不可以单独使用，两者需要配置使用。`match`用来查找和匹配数据，`return`用来返回数据。例如，查找数据库中所有标签为`Emp`的节点。

```cql
match (emp:Emp) return emp
```

也可以不返回节点，直接返回节点的中某些属性，如

```CQL
match (emp:Emp) return emp.uid
match (emp:Emp) return emp.uid, emp.uname
```

### 建立和查询关系

#### 现有节点之间建立关系

命令的大概格式如下

```cql
create (node) -[relationship]-> (node)
```

如

```cql
match (emp:Emp), (dept:Dept) create (emp) -[r:workFor]-> (dept) return r
```

上面这条语句首先是通过`match`命令，匹配到所有的`Emp`和`Dept`标签的节点，然后建立一个叫做`workFor`的关系。不过这条语句很有问题，因为一个`Emp`节点将和所有的`Dept`节点建立关系。

同样的，关系也是可以拥有属性的，写法和节点的属性相同。如

```cql
match (emp:Emp), (dept:Dept) create (emp) ->[r:workFor{Data:"2018/01/01"}] -> (dept) return r
```

#### 新建节点之间建立关系

命令大概格式和上面相同。如

```cql
create (emp:Emp{uid:1, uname:"sher"}) -[r:WorkFor]-> (dept:Dept{did=10, dname:"Tech"})
```

上面的指令在创建了两个节点的同时，建立了从emp到dept的关系。

#### 查询关系

查询当前的图数据库中的所有的`workFor`关系，可以使用一下的语句

```cql
match (emp) -[r:workFor]-> (dept) return emp, dept
```

### `Where`

和`SQL`语句一样，使用`Where`为查询添加限定条件。如

```cql
match (emp:Emp) where emp.name = "sher" return emp
```

上面的语句查询`Emp`标签中，`name`为`sher`的节点。

也可以使用`and or not`连接多个条件，如

```cql
match (emp:Emp) where emp.name = "sher" and emp.id <> 1 return emp
```

#### 配置使用`Where`语句建立关系

```cql
match (emp:Emp), (dept:Dept)
where emp.id = 1 and dept.id = 7
create (emp) -[r:workFor{time: "2020/01/01"}]-> (dept)
return r
```

### `Delete`

删除节点需要配合`Match`语句，只需要将`match`后面的`return`换成`delete`就行了。如删除`id`为1的`Emp`。

```cql
match (emp:Emp) where emp.id = 1 delete emp
```

删除关系也是一样的，如我们想要删除两个指定节点的所有的关系

```cql
match (emp:Emp), (dept:Dept)
where emp.id = 1 and dept.id = 10
match (emp) -[r]-> (dept)
delete r
```

### `Remove`

`remove`语句也是删除，和`delete`删除节点不同的是，`remove`语句用来删除节点的**属性和标签**。和`delete`相同，该语句必须要配合`match`语句一同使用。

如，删除节点的属性。

```cql
match (emp:Emp)
where emp.id = 1
remove emp.name
```

删除节点的标签

```cql
match (emp:Emp)
where emp.id = 1
remove emp:Emp
```

因为`emp`只有一个标签，删除该标签只有，该节点就没有标签了。

### `Set`

上面我们通过`Remove`语句删除了`id`为`1`的节点的`name`属性和`Emp`标签，通过`Set`语句可以添加回来。

添加或修改`name`属性。

```cql
match (emp)
where emp.id = 1
set emp.name = "sher"
```

添加属性`Emp`

```cql
match (emp)
where emp.id = 1
set emp:Emp
```

### `Order By`

和`SQL`一样，`CQL`也有`Order By`语句用于排序，该语句一般用于`return`的后面。

```cql
match (emp:Emp)
return emp.id, emp.name, emp.salary
order by emp.id
```

和`SQL`一样，默认是升序，添加`Desc`设置为降序。

```cql
match (emp:Emp)
return emp.id, emp.name, emp.salary
order by emp.salary desc
```

### `Union`

和`SQL`一样，`Union`将两组查询结果合并成一组查询结果。

如，下面的查询语句

```cql
match (cc:CreaditCard) return cc.id, cc.number
union
match (dc:DebitCard) return dc.id, dc.number
```

上面的查询语句其实是错误的，因为查询的结果前缀不一样，一个是`cc.id`另一个是`dc.id`。此时可以使用`as`解决这个问题。（和`SQL`一模一样）

正确的查询语句如下。

```cql
match (cc:CreaditCard) return cc.id as id, cc.number as number
union
match (dc:DebitCard) return dc.id as id, dc.number as number
```

需要注意的是，`union`语句会合并相同的结果，如果我们并不想合并相同的结果，可以将`union`改成`union all`

### `Limit`与`Skip`

用于`MySQL`的都知道`Limit`，用来限制返回结果的数量。在`CQL`中，`limit`和`skip`都用来限制返回的结果的数量。其中`limit`删去后面的数据，`skip`删去前面的数据。

返回前三条结果

```cql
match (emp:Emp)
return emp.name
limit 3
```

返回后三条结果

```cql
match (emp:Emp)
return emp.name
skip 3
```

### `Merge`

`Merge`语句和`Create`语句的语法基本一样，二者唯一的区别就是`Merge`只会将新的节点加入，如果节点在数据库中已经存在了，就不会增加。如

```cql
create (emp:Emp {id: 1, name: “sher"})
merge (emp:Emp {id: 1, name: "sher"})
```

如果数据库中没有数据相同的节点，上面的两个语句作用是相同的。如果再次执行该语句。`create`会继续添加一个相同的节点，而`merge`不会添加。

### `Null`

`Null`并不是一个指令，在`CQL`中我们使用和`SQL`相同的方式处理`null`值。判断一个值为`null`使用`is null`，非`null`使用`is not null`判断。如

```sql
match (emp:Emp)
where emp.id is not null
return emp.id, emp.name
```

### `In`

和`SQL`中使用方式相同，判断一个值是否在给定的集合中。如

```cql
match (emp:Emp)
where emp.id in [111, 222, 333]
return emp.id, emp.name
```

上面的语句就相当于

```cql
match (emp:Emp)
where emp.id = 111 or emp.id = 222 or emp.id = 333
return emp.id, emp.name
```

不过，明显上面的方式更简洁美观。

### `Id`

在`Neo4j`中，会为每个节点赋一个`id`属性，从`0`开始自增。但是这个`id`是不可以通过`xx.id`的方式获取的，是通过`id(xx)`的方式。如获取前三个`id`的节点的信息。

```cql
match (emp:Emp)
return emp.id, emp.name
order by id(emp)
limit 3
```

## 常用函数

### 字符串函数

-   `upper`： 将字符串变成大写

-   `lower`： 将字符串变成小写

-   `substring`: 获取字符串部分

    语法格式为： `substring(str, start, end)`

    索引从`0`开始，`end`参数可以省略，省略的话默认就是到末尾。

-   `replace`：用于字符串的替换

    语法格式为：`replace(str, before, after)`

    讲`before`字符串替换称为`after`

### 聚合函数

和`SQL`一样，`CQL`提供一下的聚合函数，`min max avg sum count`，具体作用不必多言。

### 关系函数

-   `startnode`：返回关系的开始节点

-   `endnode`：返回关系的结束节点

-   `id`：获取关系的`id`，这对于普通的图节点也是一样的，之前说过。

-   `type`：获取关系的类型，如上面提到的`workFor`关系，如下面的例子

    ```cql
    match (emp:Emp) -[r]-> (dept:Dept) return distinct type(r)
    ```

    返回`Emp`到`Dept`中的所有的关系，其中`distinct`用来对结果进行去重。

## 索引

我们可以对拥有相同的标签的节点的属性创建索引，以使用这些索引改进`match where in`这些运算符的操作。

### 创建索引

语法格式为：`create index on :label (property)`。如

```cql
create index on :Emp (name)
```

### 删除索引

语法格式为：`drop index on :label(property)`。如删除刚才创建的索引

```cql
drop index on :Emp (name)
```

## `Unique`约束

和`SQL`一样，使用`Unique`约束保证同一个标签下某一个属性的值唯一。

### 创建`Unique`约束

语法格式为`create constraint on (xx:label) assert xx.property is unique`

如要保证`emp`的`id`都是唯一的。

```cql
create constraint on (emp:Emp)
assert emp.id is unique
```

### 删除`Unique`约束

语法格式为`drop constraint on (xx:label) assert xx.property is unique`

如，删除上面创建的`unique`约束。

```cql
drop constraint on (emp:Emp)
assert emp.id is unique
```

