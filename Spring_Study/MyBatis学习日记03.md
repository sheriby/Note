# MyBatis学习日记03

## 动态`SQL`

动态 SQL 是 MyBatis 的强大特性之一。如果你使用过 JDBC 或其它类似的框架，你应该能理解根据不同条件拼接 SQL 语句有多痛苦，例如拼接时要确保不能忘记添加必要的空格，还要注意去掉列表最后一个列名的逗号。利用动态 SQL，可以彻底摆脱这种痛苦。

### `if`标签

使用动态 SQL 最常见情景是根据条件包含 where 子句的一部分。比如：

```xml
<select id="findActiveBlogWithTitleLike"
     resultType="Blog">
  SELECT * FROM BLOG
  WHERE state = ‘ACTIVE’
  <if test="title != null">
    AND title like #{title}
  </if>
</select>
```

如果传入的对象中的`title`属性为空，查询的时候就不参考该属性。

使用多个条件

```xml
<select id="findActiveBlogLike"
     resultType="Blog">
  SELECT * FROM BLOG WHERE state = ‘ACTIVE’
  <if test="title != null">
    AND title like #{title}
  </if>
  <if test="author != null and author.name != null">
    AND author_name like #{author.name}
  </if>
</select>
```

### `choose when otherwise`标签

有时候，我们不想使用所有的条件，而只是想从多个条件中选择一个使用。这有点类似`switch`语句。

```xml
<select id="findActiveBlogLike"
     resultType="Blog">
  SELECT * FROM BLOG WHERE state = ‘ACTIVE’
  <choose>
    <when test="title != null">
      AND title like #{title}
    </when>
    <when test="author != null and author.name != null">
      AND author_name like #{author.name}
    </when>
    <otherwise>
      AND featured = 1
    </otherwise>
  </choose>
</select>
```

上面的动态`SQL`表示，如果传入了`title`就按照`title`查询。如果传入了`author`并且具有`name`就按照`author_name`查询。如果两者都没有传入就返回`featrued = 1`的数据。

### `where`标签

考虑下面的动态`SQL`语句。

```xml
<select id="findActiveBlogLike"
     resultType="Blog">
  SELECT * FROM BLOG
  WHERE
  <if test="state != null">
    state = #{state}
  </if>
  <if test="title != null">
    AND title like #{title}
  </if>
  <if test="author != null and author.name != null">
    AND author_name like #{author.name}
  </if>
</select>
```

如果我们什么都没有传入的话，生成的`SQL`语句会变成

```sql
SELECT * FROM BLOG 
WHERE
```

明显是一条错误的语句。

如果我们只是传入`title`的话，生成的`SQL`语句会变成

```sql
SELECT * FROM BLOG
WHERE
AND title like ‘someTitle‘
```

则明显也是错误的`SQL`语句。

可以使用`where`标签解决这个问题。

```xml
<select id="findActiveBlogLike"
     resultType="Blog">
  SELECT * FROM BLOG
  <where>
    <if test="state != null">
         state = #{state}
    </if>
    <if test="title != null">
        AND title like #{title}
    </if>
    <if test="author != null and author.name != null">
        AND author_name like #{author.name}
    </if>
  </where>
</select>
```

*where* 元素只会在子元素返回任何内容的情况下才插入 “WHERE” 子句。而且，若子句的开头为 “AND” 或 “OR”，*where* 元素也会将它们去除。

### `set`标签

还上面的`where`标签的使用方式相同。

```xml
<update id="updateAuthorIfNecessary">
  update Author
    <set>
      <if test="username != null">username=#{username},</if>
      <if test="password != null">password=#{password},</if>
      <if test="email != null">email=#{email},</if>
      <if test="bio != null">bio=#{bio}</if>
    </set>
  where id=#{id}
</update>
```

*set* 元素会动态地在行首插入 SET 关键字，并会删掉额外的逗号

### `trim`标签

使用`trim`标签可以自定义的定制和上面的`where`和`set`类似的功能。

如`where`标签等价于下面的`trim`

```xml
<trim prefix="WHERE" prefixOverrides="AND |OR ">
  ...
</trim>
```

`set`标签等价于下面的`trim`

```xml
<trim prefix="SET" suffixOverrides=",">
  ...
</trim>
```

### `foreach`标签

动态 SQL 的另一个常见使用场景是对集合进行遍历（尤其是在构建 IN 条件语句的时候）。比如：

```xml
<select id="selectPostIn" resultType="domain.blog.Post">
  SELECT *
  FROM POST P
  WHERE ID in
  <foreach item="item" index="index" collection="list"
      open="(" separator="," close=")">
        #{item}
  </foreach>
</select>
```

我们可以将`List Set Map`等一系列可以迭代的集合交由`foreach`标签进行迭代。`item`是迭代的元素，`index`是迭代的序号。对于`Map`对象来说，`index`是键，`item`是值。`open separator close`的使用也非常的易懂。

