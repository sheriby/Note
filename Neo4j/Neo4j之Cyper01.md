# Neo4j之Cyper01

## `Case`表达式

和`SQL`当中的`Case`表达式基本上是一模一样的。

### 简单的`Case`表达式

语法格式为

```cypher
case test // test为表达式
when value then result // value为与test进行比较的值，相等返回resul，否则继续
[when ...] // 可以有多层 when xxx then xxx
[else default] // 如果上面都没有匹配到，返回默认值default
end
```

如

```cypher
match (n)
return 
    case n
    when 'blue' then 1
    when 'brown' then 2
    else 3
    end
as result
```

### 一般的`Case`表达式

语法格式为

```cypher
case
when predicate then result // 如果predicate为true，返回result，否则继续
[when ...] // 可以有多层的判断
[else default] // 上面的判断都不成立，返回默认值defalut
end
```

如

```cypher
match (n)
return 
       case 
       when n.eyes = 'blue' then 1
       when n.eyes = 'brown' then 2
       else 3
       end
as result
```

