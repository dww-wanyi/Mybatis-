# **Mybatis：**

**sql:**`sql`元素用来定义可以重复使用 的的`SQL`代码段。参数可以静态的确定下来，在不同的`include`元素中定义不同的参数值。

```sql
<sql id="userColumns"> 
${alias}.id,${alias}.username,${alias}.password </sql>
```

在其他语句中使用：

```sql
<select id="selectUsers" resultType="map">
  select
    <include refid="userColumns"><property name="alias" value="t1"/></include>,
    <include refid="userColumns"><property name="alias" value="t2"/></include>
  from some_table t1
    cross join some_table t2
</select>
```

也可以在 include 元素的 （refid/提炼） 属性或内部语句中使用属性值，例如：

```sql
<sql id="sometable">
  ${prefix}Table
</sql>

<sql id="someinclude">
  from
    <include refid="${include_target}"/>
</sql>

<select id="select" resultType="map">
  select
    field1, field2, field3
  <include refid="someinclude">
    <property name="prefix" value="Some"/>
    <property name="include_target" value="sometable"/>
  </include>
</select>
```





**动态SQL: if、choose(when,otherwise)、trim(where,set)、foreach******

**if:**

```SQL
#在BLOG表中查询所有数据，条件：当state=ACTIVE时，（模糊查询）匹配title的参数值并对应
<select id="findActiveBlogWithTitleLike"
     resultType="Blog">
  SELECT * FROM BLOG
  WHERE state = ‘ACTIVE’
  <if test="title != null">
    AND title like #{title},
  </if>
</select>
#如果希望匹配两个参数，只需要再加上一个if
#在BLOG表中查询state=ACTIVE时，同时（模糊查询）匹配title，和author.name参数值的所有数据
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

**choose、when、otherwise**:

​	可以选择匹配的条件，传入条件A就按A查找，传入条件B就按B查找。都不传入时按 featured标记的返回。

```SQL
#在BLOG表中查询所有数据 state=‘ACTIVE’时，当"title != null"时匹配title，当"author != #null and author.name != null"时匹配author.name，两者都为null时，按featured标记的返回。
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

可能这里就有疑问了。当都满足条件时，又该匹配哪个条件查询呢？`featured`又表示什么呢？其实这里就是前面的选择了。使用`if`嵌套就好了。使用`choose`时一定是要在几个条件中取舍的。`featured`其实可以理解成我们要选择的第三个条件。

**trim、where、set**：

在实际开发中，查询操作所匹配的条件不一定是固态，也就是可能回事动态的。例如我们的第一个实例中，它可能就是：

```SQL
<select id="findActiveBlogWithTitleLike"
     resultType="Blog">
  SELECT * FROM BLOG
  WHERE
  <if test="state != null">
         state = #{state}
  </if>
  <if test="title != null">
    AND title like #{title},
  </if>
</select>
```

如果没有匹配的条件，他就会变成：

```sql
SELECT * FROM BLOG
WHERE
#没有结果
```

这时候可以选择`where`（注意区分和`SQL`中`WHERE`区别，一般为了方便辨认，我们将`SQL`语句大写表示。要注意`SQL`本身对大小写不敏感.

```sql
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
  </where>
</select>
```

#需要注意的是，`where`实现功能的原理是，只有在子元素有返回内容的情况下才会插入`WHERE`子句而且若子句的开头为`AND`和`OR`，`where`元素会将他们剔除。这是为了防止出现`if`元素里第一个条件不匹配，第二个条件匹配。就会变成这样：

```sql
SELECT * FROM BLOG
WHERE
AND title like ‘someTitle’
	#无意义的查询语句
```

`trim`元素可以自定义`where`元素的功能，例如：

这其实就是`where`元素的默认情况，覆盖开头为`AND\OR`的子句。需要注意的是`"AND |OR "`管道分隔符前的空格是必需，不能省略。

```SQL
<trim prefix="WHERE" prefixOverrides="AND |OR ">
  ...
</trim>
```

**`set`**元素可以解决动态更新语句的问题。用于动态包含需要更新的列，忽略其他不更新的列。

当id=#{id}时，动态更新Author表中的username，password，email，bio。即有更新时更新，没有更新时不管。

```sql
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

与`set`元素等价的自定义`trim`元素：

```sql
--自定义前缀值时SET,覆盖后缀值,
<trim prefix="SET" suffixOverrides=",">
...
</trim>
```

**`foreach:`**遍历元素，可用于对集合进行遍历。下面例子实在构建`IN`条件时：

```sql
<select id ="selectPostion" resultType="domain.blog.Post">
	SELECT*
	FROM POST P
	WHERE ID in
	<foreach item="item" index="index" collection="list"
		open="(" separator="," close=")">//指定开头与结尾的字符串、集合项迭代的分隔符。
			#{item}
	</foreach>
</select>
```

你可以将任何可迭代对象（如 List、Set 等）、Map 对象或者数组对象作为集合参数传递给 *foreach*。当使用可迭代对象或者数组时，index 是当前迭代的序号，item 的值是本次迭代获取到的元素。当使用 Map 对象（或者 Map.Entry 对象的集合）时，index 是键，item 是值。

