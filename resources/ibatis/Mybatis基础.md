# Mybatis基础

## 一、JDBC基础

一个简单的JDBC Demo如下所示。

```java
        Connection conn=null;
        PreparedStatement ps=null;
        ResultSet s=null;
        try
        {
            //在Driver的static块中将Driver本身注册进了DriverManager中
            // Class.forName("com.mysql.jdbc.Driver")
             conn = DriverManager.
                    getConnection("jdbc:mysql://localhost:3306/subtitle_search?Unicode=true&characterEncoding=UTF-8", "root", "888888");

            String sql = "select s_zh from subtitle where s_zh like ? ";

            ps = conn.prepareStatement(sql);

            ps.setString(1, "\'%");
            ps.execute();
             s= ps.getResultSet();
            s.next();
            System.out.println(s.getString("s_zh"));
        }catch (Exception e)
        {
            e.printStackTrace();
        }
        finally
        {
            conn.close();
            ps.close();
            s.close();
        }
```

主要流程：

1. 通过`DriverManager`获得连接。该调用是SPI调用，内部经历了`Driver`类的加载和将`Driver`类实例注册进`DriverManager`的流程，最后通过`Driver`获得`Connection`。
2. 创建`Statement`或者`PreparedStatement`接口，执行SQL语句
3. 通过`ResultSet`处理结果
4. 关闭三个资源

### Ⅰ、JDBC事务

通过JDBC执行事务其实非常简单。调用`conn.setAutoCommit(false)`之后就等于开启了事务,通过`PrepareStatement`与数据库交互完成后,在调用`conn.commit()`即可进行事务提交。

### Ⅱ、Statement批处理

其中`Statement`和`PrepareStatement`都支持批处理,我就拿`PrepareStatement`当例子。

当需要向数据库发送一批SQL语句执行时，应避免向数据库一条条的发送执行，而应采用JDBC的批处理机制，以提升执行效率，这其实相当于redis的pipeline机制。

```java
  String sql = "insert into testbatch(id,name) values(?,?)";
            ps = conn.prepareStatement(sql);

            for (int i = 1; i <= 10; i++) {
                ps.setString(1, i + "");
                ps.setString(2, "aa" + i);
                ps.addBatch();
            }
            //返回int[] 第一条sql影响数据库几行，第二条sql影响数据库几行
            ps.executeBatch();
            //发给数据库之后，就要清理批处理
            ps.clearBatch();
```

在对SQL设置参数后,不立刻调用`ps.execute()`,而是调用`ps.addBatch()`将该条SQL放入批处理中。在最后调用

`ps.executeBatch()`一次性将SQL发送至数据库,提升效率。

### Ⅲ、⭐⭐⭐⭐`PrepareStatement`和`Statement`的区别

#### ①、用法上的区别

`Statement`通过`connection.createStatement()`获取对象

`PrepareStatement`通过`connection.prepareStatement()`获取对象

#### ②、参数上的区别

`Statement`直接通过字符串拼接来完成变量植入。

```java
  Statement s = c.createStatement();
  String sql = "delete from category where id = " + id ;
  s.execute(sql);
```

而`PrepareStatement`用?作为占位符代替变量,之后通过`ps.setInt()`和`ps.setString()`等方法将参数设置进SQL中。`ps.setString()`是重要的预防SQL注入的方法。

```java
String sql = "select file from file where name = ?";
PreparedStatement preparedStatement = connection.prepareStatement(sql);
preparedStatement.setString(1, param);
prepareStatement.execute();
```

#### ③、原理上的区别

 先回顾一下MySQL的查询执行步骤:

1. 客户端发送一条查询给服务器
2. 服务器先检查查询缓存，如果命中了缓存，则立刻返回存储在缓存中的结果（mysql8删除了查询缓存）。否则进入下一阶段。
3. 服务器端进行SQL解析、预处理（检查解析后生成的解析树是否合法 ），再由优化器生成对应的执行计划。
4. mysql根据优化器生成的执行计划，设置运行的参数并调用存储引擎的api来执行查询
5. 将结果返回给客户端。

使用`Statement.executeQuery()`时,MySQL每次都会重新解析这条SQL,也就是从第一步开始执行。

而`PrepareStatement`的预编译其实就是在`connection.prepareStatement()`时将SQL先发送给MySQL执行前面三步步骤,之后在调用`prepareStatement.executeQuery()`时MySQL就可以直接获取已经提前生成好的执行计划设置运行的参数，**从而复用了MySQL编译后的查询语句。**

### Ⅳ、⭐⭐`PrepareStatement`如何预防SQL注入

1. `Statement`

   如下所示：

   ```java
   String name="教父 ' -- ";
   String id="1";
   String sql="select * from subtitle where v_name='"+name+"' and s_id="+id;
   Statement statement=conn.createStatement();
   s=statement.executeQuery(sql);
   ```

   在使用`Statement`的代码中,由我们自己来实现字符串拼接。上面这种简单的直接`append()`的拼接方式就是SL注入的源头,从前端中传入这个电影名`教父 ' -- `,就可以将`and s_id`这个条件完全注释掉。此时这条sql将返回给用户许多不该让他看到的数据。

这个问题的根源就在于我们自己做拼接的时候，并没有对一些敏感字符进行过滤导致任意字符都能传给数据库。而`prepareStatement.setString()`就提供了转义与过滤的功能。

2. `PrepareStatement`

   如下所示:

   ```java
   String name="教父 ' -- ";
   int id=1;
   String sql="select * from subtitle where v_name=? and s_id=?";
   PreparedStatement preparedStatement=conn.prepareStatement(sql);
   preparedStatement.setString(1,name);
   preparedStatement.setInt(2,id);
   s=preparedStatement.executeQuery();
   ```

   根据MySQL的查询日志可以得出本次的查询语句:

   ```sql
   select * from subtitle where v_name='教父 \' -- ' and s_id=1
   ```

   `setString()`方法在内部将整个参数用`''`包括了起来,**并对`'`符号在其前面加上了`\`将其转义,这样就使得MySQL在对这句SQL进行解析时不将用户写的`'`当成字符串结束的标志,而是当成查询的参数，于是MySQL会寻找v_name为`教父 ' -- `的行。**（注：MySQL在执行查询时会消除掉转义字符前的`\`）

## 二、Mybatis标签

### Ⅰ、 定义 sql 语句
#### ①、`<select>` 标签
属性介绍:

* id :唯一的标识符.

* parameterType:传给此语句的参数的全路径名或别名 例:com.test.poso.User 或 user
  resultType :语句返回值类型或别名。注意，如果是集合，那么这里填写的是集合的泛型，而不是集合本身（resultType 与 resultMap 不能并用）

```xml
<select id="selectByPrimaryKey" resultMap="BaseResultMap" parameterType="Object">
      select * from student where id=#{id}
</select>
```

#### ②、`<insert>` 标签

属性介绍:

- id :唯一的标识符
- parameterType:传给此语句的参数的全路径名或别名 例:com.test.poso.User

```xml
<insert id="insert" parameterType="Object">
    insert into student
    <trim prefix="(" suffix=")" suffixOverrides=",">
        <if test="name != null"> NAME, </if>
    </trim>
    <trim prefix="values(" suffix=")" suffixOverrides=",">
        <if test="name != null"> #{name}, </if>
     </trim>
</insert>

```

#### ③、`<delete>` 标签

属性同 `<insert>`

```xml
<delete id="deleteByPrimaryKey" parameterType="Object">
    delete from student where id=#{id}
</delete>

```

#### ③、`<update>` 标签

属性同 `<insert>`

### Ⅱ、 配置 JAVA 对象属性与查询结果集中列名对应关系

`<resultMap>` 标签的使用
基本作用：

- 建立 SQL 查询结果字段与实体属性的映射关系信息
- 查询的结果集转换为 java 对象，方便进一步操作。
- 将结果集中的列与 java 对象中的属性对应起来并将值填充进去

注意：与 java 对象对应的列不是数据库中表的列名，而是查询后结果集的列名

```xml
<resultMap id="BaseResultMap" type="com.online.charge.platform.student.model.Student">
    <id property="id" column="id" />
    <result column="NAME" property="name" />
    <result column="HOBBY" property="hobby" />
    <result column="MAJOR" property="major" />
    <result column="BIRTHDAY" property="birthday" />
    <result column="AGE" property="age" />
</resultMap>

<!--查询时resultMap引用该resultMap -->
<select id="selectByPrimaryKey" resultMap="BaseResultMap" parameterType="Object">
    select id,name,hobby,major,birthday,age from student where id=#{id}
</select>

```

主标签：

- id:该 resultMap 的标志
- type：返回值的类名，此例中返回 Studnet 类

子标签：

- id:用于设置主键字段与领域模型属性的映射关系，此处主键为 ID，对应 id。
- result：用于设置普通字段与领域模型属性的映射关系

### Ⅲ、动态 sql 拼接

#### ①、`<if>`标签

if 标签通常用于 WHERE 语句、UPDATE 语句、INSERT 语句中，通过判断参数值来决定是否使用某个查询条件、判断是否更新某一个字段、判断是否插入某个字段的值。

```xml
<if test="name != null and name != ''">
    and NAME = #{name}
</if>
```

#### ②、`<foreach>` 标签

foreach 标签主要用于构建 in 条件，可在 sql 中对集合进行迭代。也常用到批量删除、添加等操作中。

```xml

<!-- in查询所有，不分页 -->
<select id="selectIn" resultMap="BaseResultMap">
    select name,hobby from student where id in
    <foreach item="item" index="index" collection="list" open="(" separator="," close=")">
        #{item}
    </foreach>
</select>

```

属性介绍：

`collection`：collection 属性的值有三个分别是 list、array、map 三种，分别对应的参数类型为：List、数组、map 集合。
`item` ：表示在迭代过程中每一个元素的别名
`index` ：表示在迭代过程中每次迭代到的位置（下标）
`open` ：前缀
`close` ：后缀
`separator `：分隔符，表示迭代时每个元素之间以什么分隔

#### ③、`<choose><when>` 标签

有时候我们并不想应用所有的条件，而只是想从多个选项中选择一个。MyBatis 提供了 choose 元素，按顺序判断 when 中的条件出否成立，如果有一个成立，则 choose 结束。当 choose 中所有 when
的条件都不满则时，则执行 otherwise 中的 sql。类似于 Java 的 switch 语句，choose 为 switch，when 为 case，otherwise 则为 default。

if 是与(and)的关系，而 choose 是或（or）的关系。

```xml
<select id="getStudentListChoose" parameterType="Student" resultMap="BaseResultMap">
    SELECT * from STUDENT WHERE 1=1
    <where>
        <choose>
            <when test="Name!=null and student!='' ">
                AND name LIKE CONCAT(CONCAT('%', #{student}),'%')
            </when>
            <when test="hobby!= null and hobby!= '' ">
                AND hobby = #{hobby}
            </when>
            <otherwise>
                AND AGE = 15
            </otherwise>
        </choose>
    </where>
</select>

```

### Ⅳ、格式化输出

#### ①、`<where><if>` 标签

当 if 标签较多时，这样的组合可能会导致错误。 如下：

```xml

<select id="getStudentListWhere" parameterType="Object" resultMap="BaseResultMap">
    SELECT * from STUDENT WHERE
    <if test="name!=null and name!='' ">
        NAME LIKE CONCAT(CONCAT('%', #{name}),'%')
    </if>
    <if test="hobby!= null and hobby!= '' ">
        AND hobby = #{hobby}
    </if>
</select>
```

当 name 值为 null 时，查询语句会出现 “WHERE AND” 的情况，解决该情况除了将"WHERE"改为“WHERE 1=1”之外，还可以利用 where
标签。这个“where”标签会知道如果它包含的标签中有返回值的话，它就插入一个‘where’。此外，如果标签返回的内容是以 AND 或 OR 开头的，则它会剔除掉。

```xml
<select id="getStudentListWhere" parameterType="Object" resultMap="BaseResultMap">
    SELECT * from STUDENT
    <where>
        <if test="name!=null and name!='' ">
            NAME LIKE CONCAT(CONCAT('%', #{name}),'%')
        </if>
        <if test="hobby!= null and hobby!= '' ">
            AND hobby = #{hobby}
        </if>
    </where>
</select>

```

#### ②、`<set><if>` 标签

有使用 if 标签时，如果有一个参数为 null，都会导致错误。当在 update 语句中使用 if 标签时，如果最后的 if 没有执行，则或导致逗号多余错误。使用 set 标签可以将动态的配置 set关键字，和剔除追加到条件末尾的任何不相关的逗号
```xml
<update id="updateStudent" parameterType="Object">
    UPDATE STUDENT
    SET NAME = #{name},
    MAJOR = #{major},
    HOBBY = #{hobby}
    WHERE ID = #{id};
</update>

<update id="updateStudent" parameterType="Object">
    UPDATE STUDENT SET
    <if test="name!=null and name!='' ">
        NAME = #{name},
    </if>
    <if test="hobby!=null and hobby!='' ">
        MAJOR = #{major},
    </if>
    <if test="hobby!=null and hobby!='' ">
        HOBBY = #{hobby}
    </if>
    WHERE ID = #{id};
</update>
```
使用 set+if 标签修改后，如果某项为 null 则不进行更新，而是保持数据库原值。
```xml
<update id="updateStudent" parameterType="Object">
    UPDATE STUDENT
    <set>
        <if test="name!=null and name!='' ">
            NAME = #{name},
        </if>
        <if test="hobby!=null and hobby!='' ">
            MAJOR = #{major},
        </if>
        <if test="hobby!=null and hobby!='' ">
            HOBBY = #{hobby}
        </if>
    </set>
    WHERE ID = #{id};
</update>
```
#### ③、`<trim>`标签

trim标记是一个格式化的标记，主要用于拼接sql的条件语句（前缀或后缀的添加或忽略），可以完成set或者是where标记的功能。

trim属性主要有以下四个:

* prefix：在trim标签内sql语句加上前缀

* suffix：在trim标签内sql语句加上后缀

* prefixOverrides：指定去除多余的前缀内容，如：prefixOverrides=“AND | OR”，去除trim标签内sql语句多余的前缀"and"或者"or"。

* suffixOverrides：指定去除多余的后缀内容。
  

例如在update中

```xml
<update id="updateByPrimaryKey" parameterType="Object">
	update student set 
	<trim  suffixOverrides=",">
		<if test="name != null">
		    NAME=#{name},
		</if>
		<if test="hobby != null">
		    HOBBY=#{hobby},
		</if>
	</trim> 
	where id=#{id}
</update>
```
用在insert中
```xml
<insert id="insert" parameterType="Object">
    insert into student 
	<trim prefix="(" suffix=")" suffixOverrides=",">
		<if test="name != null">
			NAME,
		</if>
		<if test="hobby != null  ">
			HOBBY,
		</if>
	</trim>
	<trim prefix="values(" suffix=")" suffixOverrides=",">
		<if test="name != null  ">
			#{name},
		</if>
		<if test="hobby != null  ">
			#{hobby},
		</if>
	</trim>
</insert>
```
### Ⅴ、⭐关联关系

当SQL要做多表关联查询，这时返回的结果数据的POJO对象映射就不能只是一个简单的对象了，而是要对POJO对象中使用关联映射。

如有多部电影属于同一个类型，而每部电影都有其自己的字幕。当我想查询某类型的所有电影字幕，这个时候就需要电影类型表和字幕表关联查询，这个时候就需要用到关联映射。

```sql
SELECT * FROM subtitle inner join(select v_name from type where v_type='剧情')as t USING(v_name);
```

若业务需求是将不同电影名的字幕放在不同的对象中,此时就需要用到一对多映射。

```java
//DAO接口
List<NameSubtitleVO> selectVideoByType(@Param("v_type")String videoType);
```

```java

public class NameSubtitleVO implements Serializable
{
    
    private String vName;
	//⭐一个电影对应着多条字幕
    private List<Subtitle> subtitleList;
}

public class Subtitle implements Serializable
{
    private static final long serialVersionUID=1L;
    private int sId;
    private String sSubtitleZh;
    private String sTime;
    private String vName;
    private String vType;
    private String image;
    private String sSubtitleEn;
}
```

在关联映射中,`<resultMap>`的编写需要两个标签`<association>`和`<collection>`。`<association>`用于一对一的映射(如一个商品订单对应着一个商品信息,如果想将这两个表的信息分解成两个对象,就要用`<association>`),而`<collection>`用于一对多和多对多映射。

```xml
<resultMap id="BaseResultMap" type="com.robin.pojo.NameSubtitleVO">

         <id column="v_name" jdbcType="VARCHAR" property="vName"/>

        <collection  property="subtitleList"  ofType="com.robin.pojo.Subtitle" >
            <id column="s_id" property="sId"/>
            <result column="v_name" property="vName"/>
            <result column="s_zh" property="sSubtitleZh"/>
             <result column="s_en" property="sSubtitleEn"/>
             <result column="s_time" property="sTime"/>
             <result column="v_type" property="vName"/>
             <result column="image" property="image"/>
        </collection>

 </resultMap>
```

注意一下标签的属性：

1. `property`：将关联查询到多条记录映射到Orders哪个属性
2. `ofType`：指定映射到list集合属性中pojo的类型**(如果是一对一使用`<association>`,由于不是一对多不需要list存储数据,所以使用的是`javaType`直接执行映射到的pojo类型,这是两者很重要的区别)**

### Ⅵ、定义常量与引用

#### ①、`<sql>`标签

当多种类型的查询语句的查询字段或者查询条件相同时，可以将其定义为常量，方便调用。为求 <select> 结构清晰也可将 sql 语句分解。

```xml
<!-- 查询字段 -->
<sql id="Base_Column_List">
    ID,MAJOR,BIRTHDAY,AGE,NAME,HOBBY
</sql>

<!-- 查询条件 -->
<sql id="Example_Where_Clause">
    where 1=1
    <trim suffixOverrides=",">
        <if test="id != null and id !=''">
            and id = #{id}
        </if>
        <if test="major != null and major != ''">
            and MAJOR = #{major}
        </if>
        <if test="birthday != null ">
            and BIRTHDAY = #{birthday}
        </if>
        <if test="age != null ">
            and AGE = #{age}
        </if>
        <if test="name != null and name != ''">
            and NAME = #{name}
        </if>
        <if test="hobby != null and hobby != ''">
            and HOBBY = #{hobby}
        </if>
        <if test="sorting != null">
            order by #{sorting}
        </if>
        <if test="sort!= null and sort != ''">
            order by ${sort} ${order}
        </if>
    </trim>
</sql>
```
#### ②、`<include>`标签

用于引用定义的常量
```xml
<!-- 查询所有，不分页 -->
<select id="selectAll" resultMap="BaseResultMap">
    SELECT
    <include refid="Base_Column_List" />
    FROM student
    <include refid="Example_Where_Clause" />
</select>


<!-- 分页查询 -->
<select id="select" resultMap="BaseResultMap">
    select * from (
        select tt.*,rownum as rowno from
        (
            SELECT
            <include refid="Base_Column_List" />
            FROM student
            <include refid="Example_Where_Clause" />
            ) tt
            <where>
                <if test="pageNum != null and rows != null">
                    and rownum
                    <![CDATA[<=]]>#{page}*#{rows}
                </if>
            </where>
        ) table_alias
    where table_alias.rowno>#{pageNum}
</select>


<!-- 根据条件删除 -->
<delete id="deleteByEntity" parameterType="java.util.Map">
    DELETE FROM student
    <include refid="Example_Where_Clause" />
</delete>
```
## 三、Mybatis配置文件

* properties（属性）

  * settings（全局配置参数）
    * setting
  * typeAliases（类型别名，用来替换namespace和各种type）
  * typeHandlers（类型处理器）
  * objectFactory（对象工厂，在经过`TypeHandler`转换完参数之后要创建结果对象时,若想在这个创建过程做一些动作,可以使用objectFactory）
  * plugins（插件）
* environments（环境集合属性对象）
  
  * environment（环境子属性对象）
  
    * transactionManager（事务管理）
  
    * dataSource（数据源）
  * mappers（映射器）
    * mapper（mapper文件路径）
    * package（dao接口路径）

## 四、延迟加载

延迟加载又叫懒加载，也叫按需加载。延迟加载的应用场景主要是在关联查询中,先查询主表的信息,当需要其他表的行数据时才根据查询过的主表字段信息对副表进行关联的单表查询。这样做的好处非常多,具体请看MySQL的分解关联查询小节。

先直接看一下延迟加载的demo。

```xml
<!--Mybatis-config.xml-->
<configuration>
    <settings>
        <setting name="lazyLoadingEnabled" value="true"/>
        <setting name="aggressiveLazyLoading" value="false"/>
        <setting name="proxyFactory" value="cglib"/>

    </settings>
    <mappers>
        <mapper resource="mapper/SubtitleMapper.xml"/>
        <package name="com.robin.dao.SubtitleMapper"/>
    </mappers>
</configuration>
```

```xml
<!--SubtitleMapper.xml-->
<mapper namespace="com.robin.dao.SubtitleMapper">
    <cache></cache>
 <resultMap id="BaseResultMap" type="com.robin.pojo.NameSubtitleVO">
         <id column="v_name" jdbcType="VARCHAR" property="vName"/>
        <collection fetchType="lazy" property="subtitleList" column="v_name" ofType="com.robin.pojo.Subtitle" select="selectSubtitleByVname">
            <id column="s_id" property="sId"/>
            <result column="v_name" property="vName"/>
            <result column="s_zh" property="sSubtitleZh"/>
             <result column="s_en" property="sSubtitleEn"/>
             <result column="s_time" property="sTime"/>
             <result column="v_type" property="vName"/>
             <result column="image" property="image"/>
        </collection>

 </resultMap>
    <resultMap id="LazyLoadMap" type="com.robin.pojo.Subtitle">
        <id column="s_id" property="sId"/>
        <result column="v_name" property="vName"/>
        <result column="s_zh" property="sSubtitleZh"/>
        <result column="s_en" property="sSubtitleEn"/>
        <result column="s_time" property="sTime"/>
        <result column="v_type" property="vName"/>
        <result column="image" property="image"/>
    </resultMap>
<select id="selectVideoByType" resultMap="BaseResultMap" >
    select * from type where v_type=#{v_type} 
</select>
<!--注意延迟加载的sql要重新指定resultMap,不要指定了BaseResultMap-->
    <select id="selectSubtitleByVname" resultMap="LazyLoadMap">
select  * from subtitle where v_name=#{v_name}

    </select>
</mapper>
```

```java
public interface SubtitleMapper
{
    public List<NameSubtitleVO> selectVideoByType(@Param("v_type")String videoType);
}
//Test.java
 	@org.junit.Test
    public void test() throws Exception
    {
        SqlSessionFactory ssf = new SqlSessionFactoryBuilder().build(new FileInputStream("D:\\Projects\\TransationalTest\\src\\test\\resources\\mybatis-config.xml"));
        SqlSession sqlSession = ssf.openSession();
        SubtitleMapper subtitleMapper=sqlSession.getMapper(SubtitleMapper.class);
        List<NameSubtitleVO> nameSubtitleVOS = subtitleMapper.selectVideoByType("喜剧");
        for (NameSubtitleVO n : nameSubtitleVOS)
        {
            System.out.println(n.getvName());
            System.out.println("-----------延迟加载-------------");
            System.out.println(n.getSubtitleList().get(0).getsSubtitleEn());
        }



    }
```

在这里,我将

```sql
select * from subtitle inner join(select v_name from type where v_type='传记')AS t USING(v_name);
```

这条语句分成了两部分查询,第一部分是通过`v_type`查找`v_name`,第二部分是通过第一部分找到的`v_name`从而寻找对应的`subtitle`。

### Ⅰ、延迟加载原理

延迟加载实际上是对第一部分`subtitleMapper.selectVideoByType("喜剧");`的返回集合做了一个CGLIB动态代理。之后一旦调用`nameSubtitleVO.getSubtitleList()`,此时调用栈就会进入到CGLIB动态代理从而自动的根据`NameSubtitleVO`对象的`vName`值使用第二部分SQL对MySQL做查询。

### Ⅱ、配置解析

首先来讲Mybatis的配置。

#### 1、Mybatis配置

##### ①、lazyLoadingEnabled

默认为 `false`, 也就是不使用懒加载. 所以如果 association 和 collection 使用了 `select`, 那么 MyBatis 会一次性执行所有的查询. 如果 accociation 和 collection 中的 `fetchType` 指定为 `lazy`, 那么即使 `lazyLoadingEnabled` 为 false, MyBatis 也会使用懒加载.

##### ②、aggressiveLazyLoading

默认为 `true`, 也就是说当你开启了懒加载之后, 只要调用返回的对象中的 *任何一个方法*, 那么 MyBatis 就会加载该对象的所有的懒加载的属性（要注意是该对象的，比如第一部分返回了许多`vName`,但是在调用`vName`为"逍遥法外"的集合方法时,就只会对`vName`为"逍遥法外"的集合的所有的懒加载的属性进行加载,不会影响其他影片）, 即执行你配置的 `select` 语句.

##### ③、lazyLoadTriggerMethods

默认值为 `equals,clone,hashCode,toString`, 当你调用这几个方法时, MyBatis 会加载所有懒加载的属性.

#### 2、Mapper配置

由于是关联查询涉及多张表，那么就必定会有一个一对一或一对多的关系。Mybatis就是依赖这个结果集关联映射做成的延迟加载。

##### ①、`<collection>`

* colomn:colomn是用来指定关联查询时用作关联的那个字段。
* select：指定的是懒加载时所需触发的SQL的Statement Id

## 五、与hibernate比较

1. hibernate是全自动，而mybatis是半自动。

hibernate完全可以通过对象关系模型实现对数据库的操作，拥有完整的JavaBean对象与数据库的映射结构来自动生成sql。而mybatis仅有基本的字段映射，对象数据以及对象实际关系仍然需要通过手写sql来实现和管理。

2. hibernate数据库移植性远大于mybatis。

hibernate通过它强大的映射结构和hql语言，大大降低了对象与数据库（oracle、mysql等）的耦合性，而mybatis由于需要手写sql，因此与数据库的耦合性直接取决于程序员写sql的方法，如果sql不具通用性而用了很多某数据库特性的sql语句的话，移植性也会随之降低很多，成本很高。

3. hibernate拥有完整的日志系统，mybatis则欠缺一些。

hibernate日志系统非常健全，涉及广泛，包括：sql记录、关系异常、优化警告、缓存提示、脏数据警告等；而mybatis则除了基本记录功能外，功能薄弱很多。

4. sql直接优化上，mybatis要比hibernate方便很多

由于mybatis的sql都是写在xml里，因此优化sql比hibernate方便很多。而hibernate的sql很多都是自动生成的，无法直接维护sql；虽有hql，但功能还是不及sql强大，见到报表等变态需求时，hql也歇菜，也就是说hql是有局限的；hibernate虽然也支持原生sql，但开发模式上却与orm不同，需要转换思维，因此使用上不是非常方便。总之写sql的灵活度上hibernate不及mybatis。


> 参考资料
>
> https://www.cnblogs.com/roostinghawk/p/9703806.html
>
> Mybatis标签全部引用自
>
> https://blog.csdn.net/qq_39623058/article/details/88779242
>
> 
>
> https://www.cnblogs.com/yjc1605961523/p/11671803.html
>
> [https://yoncise.com/2016/11/05/MyBatis-%E8%AE%B0%E5%BD%95%E4%BA%8C-lazy-loading/](https://yoncise.com/2016/11/05/MyBatis-记录二-lazy-loading/)