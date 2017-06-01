---
layout: post
title: 使用 flying 解决 pojo 自动映射问题
description: 本节内容向您讲解如何使用 AutoMapperInterceptor 拦截器来实现pojo的自动映射。
category: blog
---

## Hello World
上一篇文章中我们介绍了 flying 的基本情况，在展示第一个 demo 之前还需要做一些额外的工作，即描述您想让 mybatis 管理的数据的表结构。

无论是否使用 flying 插件，对于每一个由 mybatis 托管的表，都要有一个 <i>pojo_mapper</i>.xml 来告诉 mybatis 这个表的基本信息。在以往这个配置文件可能会因为 sql 片段而变得非常复杂，但加入 flying 插件后，这个配置文件中将不需要 sql 片段，变得精简而统一。下面是一个有代表性的配置文件 account.xml ：
``` 
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"  "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="myPackage.AccountMapper">
    <cache />
    <select id="select" resultMap="result">#{id}</select>
    <select id="selectOne" resultMap="result">#{cacheKey}</select>
    <resultMap id="result" type="Account" autoMapping="true">
        <id property="id" column="account_id" />
    </resultMap>
</mapper>
``` 
在以上配置文件中，我们描述了一个接口 myPackage.AccountMapper，一个方法 select ，一个方法 selectOne，一个对象实体 Account，以及数据库表结构 resultMap。在 resultMap 中由于设置了 `autoMapping="true"`，我们只需要写出主键（以及外键，在稍后的章节会讲到），其余字段 mybatis 会自动感知。

myPackage.AccountMapper 接口是 mybatis 本身需要的，里面的内容和 account.xml 中定义的方法相对应。如果您有使用 mybatis 的经验您就能立刻想到， AccountMapper.java 中的内容是：
```
package myPackage;
public interface AccountMapper {
    public Account select(Object id);
    public Account selectOne(Account t);
}
```
到目前为止一切都和不使用 flying 时一模一样，您可能奇怪的几个地方是：account.xml 中的 select 方法描述中的 #{id} 和 selectOne 方法描述中的 #{cacheKey}是什么、以及具体的 sql 在哪里。不要急这些问题在附录中会有解答。马上我们在对象实体 Account 中就会意识到 flying 的存在，Account.java 的代码如下：
```
package myPackage;
import org.apache.ibatis.type.JdbcType;
import indi.mybatis.flying.annotations.FieldMapperAnnotation;
import indi.mybatis.flying.annotations.TableMapperAnnotation;
    
@TableMapperAnnotation(tableName = "account")
public class Account {
    @FieldMapperAnnotation(dbFieldName = "account_id", jdbcType = JdbcType.INTEGER, isUniqueKey = true)
    private Integer id;
	    
    @FieldMapperAnnotation(dbFieldName = "name", jdbcType = JdbcType.VARCHAR)
    private java.lang.String name;
	    
    public Integer getId() {
	    return id;
    }
    public void setId(Integer id) {
		this.id = id;
    }
    public String getName() {
		return name;
    }
    public void setName(String name) {
		this.name = name;
    }
}
```    
可见，和普通的 pojo 相比， Account.java 只是多了以下3行注解而已：
```
    @TableMapperAnnotation(tableName = "account")
    
    @FieldMapperAnnotation(dbFieldName = "id", jdbcType = JdbcType.INTEGER, isUniqueKey = true)
    
    @FieldMapperAnnotation(dbFieldName = "name", jdbcType = JdbcType.VARCHAR) 
```
下面我们分别来解释它们的含义。

第1行 `@TableMapperAnnotation` 只能放在类定义之上，它声明这个类是一个表，它的属性 `tableName` 描述了这个表在数据库中的名字。

第2行 `@FieldMapperAnnotation` 只能放在变量定义之上，它声明这个变量是一个字段，它的属性 `dbFieldName` 描述了在数据库中这个字段的名称，它的属性 `jdbcType` 描述了在数据库中这个字段的类型，它的属性 `isUniqueKey = true` 描述了这个字段是一个主键。

第3行 `@FieldMapperAnnotation` 与第二行相同，它描述了另一个字段 name，值得注意的是这个字段的类型是 varchar 并且不是主键。

以上 3 个注解描述了表 account 的数据结构，然后我们就可以使用 AccountService 非常方便的操纵数据库的读取了。（AccountService 是 AccountMapper 的实现类，单独使用或在 spring 中使用都有多种方法进行配置，本文档在附录部分提供了一种配置方法）

使用以下代码，可以查询 id 为 1 的账户：
```
Account account = accountService.select(1);
```     
使用以下代码，可以查询 name 为 andy 的 1 条账户数据：
```
Account accountCondition = new Account();
accountCondition.setName("andy");
Account account = accountService.selectOne(accountCondition);
```
与以往的方式相比，这种方式是不是变得优雅了很多？关于 select 和 selectOne 之间的区别，我们在后面的章节会讲到。

## insert & delete
在最基本的 select 之后，我们再看新增功能。但在此之前，需要先在 account.xml 中增加以下内容：
```
<insert id="insert" useGeneratedKeys="true" keyProperty="id" />
```
上面的 `useGeneratedKeys="true"` 表示主键自增，如果您不使用主键自增策略此处可以省略，上面的语句和一般 mybatis 映射文件的区别在于没有具体 sql 语句。

同样在 AccountMapper.java 中我们需要加入：
```
public void insert(Account t);
```
然后使用以下代码，可以增加 1 条 name 为 “bob” 的账户数据（由于我们配置了主键自增，新增数据时不需要指定主键）：
```
Account newAccount = new Account();
newAccount.setName("bob");
accountService.insert(newAccount);
```
然后我们再看删除功能。先在 account.xml 中增加以下内容：
```
<delete id="delete" />
```
然后在 `AccountMapper.java` 中加入：
```
public int delete(Account t);
```
然后使用以下代码，可以删掉 id 与 accountToDelete 的 id 一致的数据。
```
accountService.delete(accountToDelete);
```
delete 方法的返回值代表执行 sql 后产生影响的条数，一般来说，返回值为 0 表示 sql 执行后没有效果，返回值为 1 表示 sql 执行成功，在代码中可以通过判断 delete 方法的返回值来实现更复杂的事务逻辑。

## update & updatePersistent
接下来我们看看更新功能，这里我们要介绍两个方法：update（更新）和 updatePersistent（完全更新）。首先，在 `account.xml` 中增加以下内容：
```
<update id="update" />
<update id="updatePersistent" />
```
上面的语句和一般 mybatis 映射文件的区别在于没有具体 sql 语句。

然后在 `AccountMapper.java` 中加入：
```
public int update(Account t);
public int updatePersistent(Account t);
```
然后使用以下代码，可以将 accountToUpdate 的 name 更新为 “duke” 。
```
accountToUpdate.setName("duke");
accountService.update(accountToUpdate);
```
update 和 updatePersistent 方法的返回值代表执行 sql 后产生影响的条数，一般来说，返回值为 0 表示 sql 执行后没有效果，返回值为 1 表示 sql 执行成功，在代码中可以通过判断 update 和 updatePersistent 方法的返回值来实现更复杂的事务逻辑。

下面我们来说明 update 和 updatePersistent 和关系。如果我们执行
```
accountToUpdate.setName(null);
accountService.update(accountToUpdate);
```
实际上数据库中这条数据的 name 字段不会改变，因为 flying 对为 null 的属性有保护措施。这在大多数情况下都是合理的，但如果我们真的需要在数据库中将这条数据的 name 字段设为 null，updatePersistent 就派上了用场。我们可以执行：
```
accountToUpdate.setName(null);
accountService.updatePersistent(accountToUpdate);
```
这样数据库中这条数据的 name 字段就会变为 null。可见 updatePersistent 会把 pojo 中所有的属性都更新到数据库中，而 update 只更新不为 null 的属性。在实际使用 updatePersistent 时，您需要特别小心慎重，因为当时 pojo 中为 null 的属性有可能比您想象的多。

## selectAll & count
在之前学习 select 和 selectOne 时，细心的您可能已经发现，这两个方法要完成的工作似乎是相同的。的确 select 和 selectOne 都返回 1 个绑定了数据的 pojo，但它们接受的参数不同：select 接受主键参数；selectOne 接受 pojo 参数，这个 pojo 中的所有被 `@FieldMapperAnnotation` 标记过的属性都会作为“相等”条件传递到 sql 语句中。之所以要这么设计，是因为我们有时会需要按照一组条件返回多条数据或者数量，即 selectAll 方法与 count 方法，这个时候以 pojo 作为入参最为合适。为了更清晰的讲述，我们先给 `Account.java` 再增加一个属性 address：
```
@FieldMapperAnnotation(dbFieldName = "address", jdbcType = JdbcType.VARCHAR)
private java.lang.String address;
/*相关的getter和setter方法请自行补充*/
```
然后我们在 `account.xml` 中增加以下内容：
```
<select id="selectAll" resultMap="result">#{cacheKey}</select>
<select id="count" resultType="int">#{cacheKey}</select>
```
再在 `AccountMapper.java` 中加入
```
public Collection<Account> selectAll(Account t);
public int count(Account t);
```
就可以了。例如使用以下代码，可以查询所有 address 为 “beijing” 的数据和数量：
```
Account condition = new Account();
condition.setAddress("beijing");
Collection<Account> accountCollection = accountService.selectAll(condition);
int accountNumber = accountService.count(condition);
```
（当然一般来说执行 selectAll 后就不需要执行 count 了，我们取结果集的 size 即可，但如果我们只关心数量不关心具体数据集时，执行 count 比执行 selectAll 更节省时间）

如果我们想查询所有 address 为 “shanghai” 同时 name 为 “ella” 的账户，则执行以下代码：
```
Account condition = new Account();
condition.setAddress("shanghai");
condition.setName("ella");
Collection<Account> accountCollection = accountService.selectAll(condition);
```
如果我们知道 address 为 "shanghai" 同时  name 为 "ella" 的账户只有一个，并想直接返回这个数据绑定的 pojo，可以执行： 
```
Account account = accountService.selectOne(condition);
```
由此可见 selectOne 可以称作是 selectAll 的特殊形式，它只会返回一个 pojo 而不是 pojo 的集合。如果确实有多条数据符合给定的 codition ，也只会返回查询结果中排在最前面的数据，这一点用户在使用 selectOne 时需要了解。尽管如此，在合适的地方使用 selectOne 代替 selectAll，会让您的程序获得极大便利。

## foreign key
一般来说我们的 pojo 都是业务相关的，而这些相关性归纳起来无外乎一对一、一对多和多对多。其中一对一是一对多的特殊形式，多对多本质上是由两个一对多组成，所以我们只需要着重解决一对多关系，而 flying 完全就是为此而生。

首先我们定义一个新的 pojo：角色（role）。角色和账户是一对多关系，即一个账户只能拥有一个角色，一个角色可以被多个账户拥有。为此我们要新建 `role.xml`、`RoleMapper.java` 以及 `Role.java`。`role.xml` 如下：
``` 
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"  "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="myPackage.RoleMapper">
    <cache />
    <select id="select" resultMap="result">#{id}</select>
    <select id="selectOne" resultMap="result">#{cacheKey}</select>
    <select id="selectAll" resultMap="result">#{cacheKey}</select>
    <select id="count" resultType="int">#{cacheKey}</select>
    <insert id="insert" useGeneratedKeys="true" keyProperty="id" />
    <update id="update" />
    <update id="updatePersistent" />
    <delete id="delete" />
    <resultMap id="result" type="Role" autoMapping="true">
        <id property="id" column="role_id" />
    </resultMap>
</mapper>
```
`RoleMapper.java` 如下：
``` 
package myPackage;
public interface RoleMapper {
    public Role select(Object id);
    public Role selectOne(Role t);
    public Collection<Role> selectAll(Role t);
    public void insert(Role t);
    public int update(Role t);
    public int updatePersistent(Role t);
    public int delete(Role t);
    public int count(Role t);
}
```
`Role.java` 如下：
```  
package myPackage;
import org.apache.ibatis.type.JdbcType;
import indi.mybatis.flying.annotations.FieldMapperAnnotation;
import indi.mybatis.flying.annotations.TableMapperAnnotation;
    
@TableMapperAnnotation(tableName = "role")
public class Role {

    @FieldMapperAnnotation(dbFieldName = "role_id", jdbcType = JdbcType.INTEGER, isUniqueKey = true)
    private Integer id;
	    
    @FieldMapperAnnotation(dbFieldName = "role_name", jdbcType = JdbcType.VARCHAR)
    private String roleName;
    /*相关的getter和setter方法请自行补充*/
}
```
然后在 `Account.java` 中，加入以下内容：
```   
@FieldMapperAnnotation(dbFieldName = "fk_role_id", jdbcType = JdbcType.INTEGER, dbAssociationUniqueKey = "role_id")
private Role role;   
/*相关的getter和setter方法请自行补充*/
```
以上代码中，`dbFieldName` 的值为数据库表 account 中指向表 role 的外键名，`jdbcType` 的值为这个外键的类型，`dbAssociationUniqueKey` 的值为此外键对应的另一表的主键的名称，写出以上信息后，flying 在代码层面已经完全理解了数据结构。

最后在 `account.xml` 的 `resultMap` 元素中，加入以下内容
```   
<association property="role" javaType="Role" select="myPackage.RoleMapper.select" column="fk_role_id" /> 
```
写出以上信息后，flying 在配置文件层面已经完全理解了数据结构。

最后总结一下，完整版的 `account.xml` 如下：
``` 
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"  "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="myPackage.AccountMapper">
    <cache />
    <select id="select" resultMap="result">#{id}</select>
    <select id="selectOne" resultMap="result">#{cacheKey}</select>
    <select id="selectAll" resultMap="result">#{cacheKey}</select>
    <select id="count" resultType="int">#{cacheKey}</select>
    <insert id="insert" useGeneratedKeys="true" keyProperty="id" />
    <update id="update" />
    <update id="updatePersistent" />
    <delete id="delete" />
    <resultMap id="result" type="Account" autoMapping="true">
        <id property="id" column="account_id" />
        <association property="role" javaType="Role" select="myPackage.RoleMapper.select" column="fk_role_id" />
    </resultMap>
</mapper>
```
在写完以上代码后，我们看看 flying 能做到什么。首先多对一关系中的<b>一</b>（也即父对象），是可以在多对一关系中的<b>多</b>（也即子对象）查询时自动查询的。为了说明接下来的例子，我们先以 dataset 的方式定义一个数据集
```
<dataset>
    <account account_id="1" fk_role_id="10" address="beijing" name="frank" />
    <account account_id="2" fk_role_id="11" address="tianjin" name="gale" />
    <account account_id="3" fk_role_id="11" address="beijing" name="hank" />
    <role role_id="10" role_name="user" />
    <role role_id="11" role_name="super_user" />
</dataset>
``` 
我们使用这个数据集进行测试，当我们输入以下代码时：
```
Account account1 = accountService.select(1);
/*此时account1的role属性也已经加载了真实数据*/
Role role1 = account1.getRole();
/*role1.getId()为10，role1.getRoleName()为"user"*/
```
这种传递是可以迭代的，即如果 Role 自己也有父对象，则 Role 的父对象也会一并加载，只要它的配置文件和代码正确。

不仅如此，我们可以在入参 pojo 中加入父对象，比如下面的代码查询的是角色名为 “super_user” 的所有帐户：
```
Role roleCondition = new Role();
roleCondition.setRoleName("super_user");
Account accountCondition = new Account();
accountCondition.setRole(roleCondition);
Collection<Account> accounts = accountService.selectAll(accountCondition);
/*accounts.seiz()为 2，里面包含的对象的 account_id 是 2 和 3*/

/*我们再给入参pojo加一个address限制*/
accountCondition.setAddress("beijing");
Collection<Account> accounts2 = accountService.selectAll(accountCondition);
/*accounts.size()为 1，里面包含的对象的 account_id 是 3，这说明 account 的条件和父对象 role 的条件同时都生效了*/
```
这个特性在 selectOne、count 中同样存在

最后，父对象同样可以参与子对象的 insert、update、updatePersistent，代码如下：
```
Account newAccount = new Account();

/*我们新建一个姓名为 iris，角色名称为 "user" 的账号*/
newAccount.setName("iris");

/*角色名称为 "user" 的数据的 role_id 是 10，由变量 role1 来加载它*/
Role role1 = roleService.select(10);
newAccount.setRole(role1);
accountService.insert(newAccount);
/*一个姓名为iris，角色名称为"user"的账号建立完成*/

/*我们用update方法将iris的角色变为"super_user"*/
/*角色名称为"super_user"的数据的role_id是11，由变量role2来加载了它*/
Role role2 = roleService.select(11);
newAccount.setRole(role2);
accountService.update(newAccount);
/*现在newAccount.getRole().getId()为11，newAccount.getRole().getRoleName为"super_user"*/

/*我们用updatePersistent方法将iris的角色变为null，即与Role对象不再关联*/
newAccount.setRole(null);
accountService.updatePersistent(newAccount);
/*现在 newAccount.getRole()为 null，在数据库中也不再有关联（注意在这里 update 方法起不到这种效果，因为 update 会忽略 null）*/
```

## complex condition
之前我们展示的例子中，条件只有“相等”一种，但在实际情况中我们会遇到各种各样的条件：大于、不等于、like、in、is not null 等等。这些情况 flying 也是能够处理的，但首先我们要引入一个“条件对象”的概念。条件对象是实体对象的子类，但它只为查询而存在，它拥有实体对象的全部属性，同时它还有一些专为查询服务的属性。例如下面是 Account 对象的条件对象 AccountCondition 的代码：
```
package myPackage;
import java.util.Collection;
import java.util.List;
import indi.mybatis.flying.annotations.ConditionMapperAnnotation;
import indi.mybatis.flying.annotations.QueryMapperAnnotation;
import indi.mybatis.flying.models.Conditionable;
import indi.mybatis.flying.statics.ConditionType;
@QueryMapperAnnotation(tableName = "account")
public class AccountCondition extends Account implements Conditionable {

    @ConditionMapperAnnotation(dbFieldName = "name", conditionType = ConditionType.Like)
    /*用作 name 全匹配的值*/
    private String nameLike;
	
    @ConditionMapperAnnotation(dbFieldName = "address", conditionType = ConditionType.HeadLike)
    /*用作 address 开头匹配的值*/
    private String addressHeadLike;
	
    @ConditionMapperAnnotation(dbFieldName = "address", conditionType = ConditionType.TailLike)
    /*用作 address 结尾匹配的值*/
    private String addressTailLike;
	
    @ConditionMapperAnnotation(dbFieldName = "address", conditionType = ConditionType.MultiLikeAND)
    /*用作 address 需要同时匹配的若干个值的集合（类型只能为List）*/
    private List<String> addressMultiLikeAND;
	
    @ConditionMapperAnnotation(dbFieldName = "address", conditionType = ConditionType.MultiLikeOR)
    /*用作 address*/ 需要至少匹配之一的若干个值的集合（类型只能为List）
    private List<String> addressMultiLikeOR;
	
    @ConditionMapperAnnotation(dbFieldName = "address", conditionType = ConditionType.In)
    /*用作 address*/ 可能等于的若干个值的集合（类型可为任意Collection）
    private Collection<String> addressIn;
	
    @ConditionMapperAnnotation(dbFieldName = "address", conditionType = ConditionType.NotIn)
    /*用作 address 不可能等于的若干个值的集合（类型可为任意Collection）*/
    private Collection<String> addressNotIn;
	
    @ConditionMapperAnnotation(dbFieldName = "address", conditionType = ConditionType.NullOrNot)
    /*用作 address 是否为 null 的判断（类型只能为Boolean）*/
    private Boolean addressIsNull;
	
    /*相关的getter和setter方法请自行补充*/
	
    /*以下四个方法是实现 Conditionable 接口后必须要定义的方法，我们这里只写出默认实现，在下一节中我们会详细介绍它们*/
    @Override
    public Limitable getLimiter() {
        return null;
    }
    @Override
    public void setLimiter(Limitable limiter) {
    }
    @Override
    public Sortable getSorter() {
        return null;
    }
    @Override
    public void setSorter(Sortable sorter) {
    }
}
```
以上各种条件并非要全部写出，您可以只写出业务需要的条件（变量名可以是任意的，只要条件标注准确即可）。在 flying 中进行复杂条件查询前需要先按需求写一些条件代码，但请您相信，这种做法的回报率是相当高的。然后我们可以进行测试：
```
/*查询名称中带有"a"的帐户数量*/
AccountCondition condition1 = new AccountCondition();
condition1.setNameLike("a");
int count1 = accountService.count(condition1);

/*查询地址以"bei"开头的帐户的数量*/
AccountCondition condition2 = new AccountCondition();
condition2.setAddressHeadLike("bei");
int count2 = accountService.count(condition2);

/*查询地址以"jing"结尾的帐户的数量*/
AccountCondition condition3 = new AccountCondition();
condition3.setAddressTailLike("jing");
int count3 = accountService.count(condition3);

/*查询地址同时包含"e"和"i"的账户的数量*/
List<String> listAddressMultiLikeAND = new ArrayList<>();
listAddressMultiLikeAND.add("e");
listAddressMultiLikeAND.add("i");
AccountCondition condition4 = new AccountCondition();
condition4.setAddressMultiLikeAND(listAddressMultiLikeAND);
int count4 = accountService.count(condition4);

/*查询地址至少包含"e"或"i"的账户的数量*/
List<String> listAddressMultiLikeOR = new ArrayList<>();
listAddressMultiLikeOR.add("e");
listAddressMultiLikeOR.add("i");
AccountCondition condition5 = new AccountCondition();
condition5.setAddressMultiLikeOR(listAddressMultiLikeOR);
int count5 = accountService.count(condition5);

/*查询地址等于"beijing"或"shanghai"中的一个的账户的数量*/
List<String> listAddressIn = new ArrayList<>();
listAddressIn.add("beijing");
listAddressIn.add("shanghai");
AccountCondition condition6 = new AccountCondition();
condition6.setAddressIn(listAddressIn);
int count6 = accountService.count(condition6);

/*查询地址不等于"beijing"或"shanghai"的账户的数量*/
List<String> listAddressNotIn = new ArrayList<>();
listAddressNotIn.add("beijing");
listAddressNotIn.add("shanghai");
AccountCondition condition7 = new AccountCondition();
condition7.setAddressNotIn(listAddressNotIn);
int count7 = accountService.count(condition7);

/*查询地址为null的账户的数量*/
AccountCondition condition8 = new AccountCondition();
condition8.setAddressIsNull(true);
int count8 = accountService.count(condition8);

/*最后我们查询名称中带有"a"且地址以"bei"开头的帐户的数量*/
AccountCondition conditionX = new AccountCondition();
conditionX.setNameLike("a");
conditionX.setAddressHeadLike("bei");
int countX = accountService.count(conditionX);
/*这个用例说明所有条件变量都是可以组合使用的*/
```
## limiter & sorter
在之前的 selectAll 查询中我们都是取符合条件的所有值，但在实际业务需求中很少会这样做，更多的情况是我们会有一个数量限制。同时我们还会希望结果集是经过某种条件排序，甚至是经过多种条件排序的，幸运的是 flying 已经为此做好了准备。

一个可限制数量并可排序的查询也是由条件对象来实现的，代码如下：
```
package myPackage;
import indi.mybatis.flying.annotations.QueryMapperAnnotation;
import indi.mybatis.flying.models.Conditionable;
import indi.mybatis.flying.models.Limitable;
import indi.mybatis.flying.models.Sortable;
@QueryMapperAnnotation(tableName = "account")
public class AccountCondition extends Account implements Conditionable {
    private Limitable limiter;
    private Sortable sorter;
    @Override
    public Limitable getLimiter() {
        return limiter;
    }
    @Override
    public void setLimiter(Limitable limiter) {
        this.limiter = limiter;
    }
    @Override
    public Sortable getSorter() {
        return sorter;
    }
    @Override
    public void setSorter(Sortable sorter) {
        this.sorter = sorter;
    }
}
```
以上 limiter 和 sorter 变量名并非固定，只要类引入了 Conditionable 接口并实现相关方法，且在相关方法中对应上您定义的 limiter 和 sorter 即可。
然后可以采用如下代码进行测试：
```
import indi.mybatis.flying.models.Conditionable;
import indi.mybatis.flying.pagination.Order;
import indi.mybatis.flying.pagination.PageParam;
import indi.mybatis.flying.pagination.SortParam;

/*查询 account 表在默认排序下前 10 条数据*/
AccountCondition condition1 = new AccountCondition();
/*PageParam 的构造函数中第一个参数为起始页数，第二个参数为每页容量，new PageParam(0,10)即从头开始取 10 条数据*/
condition1.setLimiter(new PageParam(0, 10));
Collection<Account> collection1 = accountService.selectAll(codition1);

/*查询 account 表在默认排序下第 8 条数据*/
AccountCondition condition2 = new AccountCondition();
/*new PageParam(7,1)即从第 7 条开始取 1 条数据*/
condition2.setLimiter(new PageParam(7, 1));
/*因为结果只需要一条数据，我们可以使用 selectOne 方法*/
Account account2 = accountService.selectOne(condition2);

/*查询 account 表在 name 正序排序下的所有数据*/
AccountCondition condition3 = new AccountCondition();
/*new Order()的第一个参数是被排序的字段名，第二个参数是正序或倒序*/
condition3.setSorter(new SortParam(new Order("name", Conditionable.Sequence.asc)));
Collection<Account> collection3 = accountService.selectAll(codition3);

/*查询 account 表先在 name 正序排序，然后在 address 倒序排序下的所有数据*/
AccountCondition condition4 = new AccountCondition();
/*在new SortParam()中可以接受不定数量的 Order 参数，因此我们先新建一个 name 正序，再新建一个 address 倒序*/
condition4.setSorter(new SortParam(new Order("name", Conditionable.Sequence.asc),new Order("address", Conditionable.Sequence.desc)));
Collection<Account> collection4 = accountService.selectAll(codition4);

/*最后我们查询在 name 正序排序下的第 11 到 20 条数据*/
AccountCondition conditionX = new AccountCondition();
conditionX.setSorter(new SortParam(new Order("name", Conditionable.Sequence.asc)));
conditionX.setLimiter(new PageParam(1, 10));
Collection<Account> collectionX = accountService.selectAll(coditionX);
/*这个用例说明 limiter 和 sorter 是可以组合使用的*/
```
因为 limiter 和 sorter 也是以条件对象的方式定义，所以可以和复杂查询一起使用，只要在条件对象中既包含条件标注又包含 Limitable 和 Sortable 类型的变量即可。
## 分页
在大多数实际业务需求中，我们的 limiter 和 sorter 都是为分页服务。在 flying 中，我们提供了一种泛型 Page&lt;?&gt; 来封装查询出的数据。使用 Page&lt;?&gt; 的好处是，它除了提供数据内容（pageItems）外还提供了全部数量（totalCount）、最大页数（maxPageNum）、当前页数（pageNo）等信息，这都是数据接收端希望了解的信息。并且这些数量信息是 flying 自动获取的，您只需执行下面这样的代码即可：
```
import indi.mybatis.flying.pagination.Page;

AccountCondition condition = new AccountCondition();
condition.setLimiter(new PageParam(0, 10));
Collection<Account> collection = accountService.selectAll(condition);

/*下面这句代码就将查询结果封装为了 Page<?> 对象*/
Page<Account> page = new Page<>(collection, condition.getLimiter());
/*需要注意的是上面的入参 condition.getLimiter() 是不能用其它 PageParam 对象代替的，因为在之前执行 selectAll 时会将一些信息保存到 condition.getLimiter() 中*/
```
假设总的数据有 21 条，则 `page.getTotalCount()` 为 21，`pagegetMaxPageNum()` 为 3，`page.getPageNo()` 为 1，`page.getPageItems()` 为第一到第十条数据的集合。
## 乐观锁
乐观锁是实际应用的数据库设计中重要的一环，而 flying 在设计之初就考虑到了这一点，
目前 flying 只支持版本号型乐观锁。在 flying 中使用乐观锁的方法如下：
在数据结构中增加一个表示乐观锁的 Integer 型字段 opLock 即可：
```
@FieldMapperAnnotation(dbFieldName = "opLock", jdbcType = JdbcType.INTEGER, opLockType = OpLockType.Version)
private Integer opLock;
/*乐观锁可以增加 getter 方法，不建议增加 setter 方法*/
```
以上实际上是给 `@FieldMapperAnnotation` 中的 `opLockType` 上赋予了 `OpLockType.Version`，这样 flying 就会明白这是一个起乐观锁作用的字段。当含有乐观锁的表 account 更新时，实际 sql 会变为：
```
update account ... and opLock = opLock + 1 where id = '${id}' and opLock = '${opLock}'
```
（上面 ... 中的内容是给其它的字段赋值）

每次更新时都会加入 opLock 的判断，并且更新数据时 opLock 自增 1 ，这样就可以保证多个线程对同一个 account 执行 update 或 updatePersistent 时只有一个能执行成功，即达到了我们需要的锁效果。

当含有乐观锁的表 account 删除时，实际 sql 会变为：
```
delete from account where id = '${id}' and opLock = '${opLock}'
```
即只有 opLock 和 id 都符合时才能被删除，这里乐观锁起到了保护数据的作用。

在实际应用中，可以借助 update、updatePersistent、delete 方法的返回值来判断是否变动了数据（一般来说返回 0 表示没变动，1 表示有变动），继而判断锁是否有效，是否合法（符合业务逻辑），最后决定整个事务是提交还是回滚。

最后我们再来谈谈为什么不建议给乐观锁字段加上 setter 方法。首先在代码中直接修改一个 pojo 的乐观锁值是很危险的事情，它会导致事务逻辑的不可靠；其次乐观锁不参与 select、selectAll、selectOne 方法，即便给它赋值在查询时也不会出现；最后乐观锁不参与 insert 方法，无论给它赋什么值在新增数据中此字段的值都是零，即乐观锁总是从零开始增长。
## 其它
### 忽略选择
有时候，我们希望在查询结果中时隐藏某个字段的值，但在作为查询条件和更新时要用到这个字段。一个典型的例子是 password 字段，出于安全考虑我们不想在 select 方法返回的结果中看到它的值，但我们需要在查询条件（如判断登录）和更新（如修改密码）时使用到它，这时我们可以在 Account.java 中加入以下代码：
```
@FieldMapperAnnotation(dbFieldName = "password", jdbcType = JdbcType.VARCHAR, ignoredSelect = true)
private String password;
/*相关的getter和setter方法请自行补充*/
```
这样在查询 account 表时就不会再查找 password 字段，但作为查询条件和更新数据时 password 字段都可以参与进来，如下所示：
```
/*查找 name 为 "user" 且 password 为 "123456" 的一个账户*/
Account condition = new Account();
condition.setName("user");
condition.setPassword("123456");
Account account = accountService.selectOne(condition);
/*用以上方式是可以查出 passeord 为 "123456" 的账户的，然而结果中 account.getPassword()为 null*/

/* 但是仍然可以更新 password 的值 */
account.setPassword("654321");
accountService.update(account);
/*现在 account 对应的数据库中数据的 password 字段值变为 "654321"*/
```
### 复数外键
有时候一个数据实体会有多个多对一关系指向另一个数据实体，例如考虑下面的情况：我们假设每个账户都有一个兼职角色，这样 account 表中就需要另一个字段 fk_second_role_id，而这个字段也是指向 role 表。为了满足这个需要，首先我们要在 account.xml 的 resultMap元素中，加入以下内容：
[AccountService](#AccountService)
```
<association property="secondRole" javaType="Role" select="myPackage.RoleMapper.select" column="fk_second_role_id" />
```
然后在 Account.java 中还需要加入以下代码：
```
@FieldMapperAnnotation(dbFieldName = "fk_second_role_id", jdbcType = JdbcType.INTEGER, dbAssociationUniqueKey = "role_id")
private Role secondRole;   
/*相关的getter和setter方法请自行补充*/
```
如此一来表 account 和表 role 就构成了复数外键关系。flying 支持复数外键，您可以像操作普通外键一样操作它，代码如下：
```
/*查询角色名称为 "user",同时兼职角色名称为 "super_user" 的账户*/
Account condition = new Account();
Role role = new Role(), secondRole = new Role();
role.setName("user");
condition.setRole(role);
secondRole.setName("super_user");
condition.setSecondRole(secondRole);
Collection<Account> accounts = accountService.selectAll(condition);
```
可见，复数外键的增删改查等操作与普通外键是类似的，只需要注意虽然 secondRole 的类型为Role，但它的 getter、setter 是 getSecondRole()、setSecondRole()，而不是 getRole()、setRole()即可。
## 附录
### 常见问题（Frequently Asked Questions）
1、<i>pojo_mapper</i>.xml 中的 #{id} 和 #{cacheKey} 是什么？

A：这是 flying 内部的约定方法，您只需原封不动的复制粘贴即可。

2、为何<i>pojo_mapper</i>.xml 中没有 sql 语句细节？

A：flying 的 sql 语句是动态生成的，只要您指定了正确的字段名，就绝对不会出现 sql 书写上的问题。并且 flying 采用了缓存机制，您无需担心动态生成 sql 的效率问题。

<a id="AccountService"></a>
### AccountService 的实现方式
在 spring 3.x 及更高版本中，可以按以下方式构建一个 <i>pojoService</i>.java 类：
```
package myPackage;

import java.util.Collection;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class AccountService implements AccountMapper {

	@Autowired
	private AccountMapper mapper;

	@Override
	public Account select(Object id) {
		return mapper.select(id);
	}

	@Override
	public Account selectOne(Account t) {
		return mapper.selectOne(t);
	}
	
	@Override
	public Collection<Account> selectAll(Account t) {
		return mapper.selectAll(t);
	}
	
	@Override
	public void insert(Account t) {
		mapper.insert(t);
	}

	@Override
	public int update(Account t) {
		return mapper.update(t);
	}

	@Override
	public int updatePersistent(Account t) {
		return mapper.updatePersistent(t);
	}

	@Override
	public int delete(Account t) {
		return mapper.delete(t);
	}

	@Override
	public int count(Account t) {
		return mapper.count(t);
	}
}
```
