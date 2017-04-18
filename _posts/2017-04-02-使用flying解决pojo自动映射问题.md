---
layout: post
title: 使用flying解决pojo自动映射问题
description: 本节内容向您讲解如何使用AutoMapperInterceptor拦截器来实现pojo的自动映射。
category: blog
---

## Hello World

上一篇文章中我们介绍了flying的基本情况，在展示第一个demo之前还需要做一些额外的工作，即描述你想让mybatis托管的数据的表结构。
无论是否使用flying插件，对于每一个由mybatis托管的表，都要有一个<i>pojo_mapper.xml</i>来告诉mybatis这个表的基本信息。在以往这个配置文件可能会因为sql片段而变得非常复杂，但加入flying插件后，这个配置文件中将不需要sql片段，变得精简而统一。下面是一个有代表性的配置文件account.xml：
 
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
 
在以上配置文件中，我们描述了一个接口<i>myPackage.AccountMapper</i>，一个方法<i>select</i>，一个方法<i>selectOne</i>，一个对象实体<i>Account</i>，以及表结构<i>resultMap</i>。在<i>resultMap</i>中由于设置了`autoMapping="true"`，我们只需要写出主键（以及外键，在稍后的章节会讲到），其余字段mybatis会自动感知。

<i>myPackage.AccountMapper</i>接口是mybatis本身需要的，里面的内容和此配置文件中定义的方法相对应。如果你有使用mybatis的经验你就能猜到，<i>AccountMapper.java</i>中的内容是：

    package myPackage;
    public interface AccountMapper {
        public Account select(Object id);
	    public Account selectOne(Account t);
    }
    

到目前为止一切都和不使用flying时一模一样，你可能唯一奇怪的一点就是account.xml中的select方法描述中的<i>#{id}</i>，selectOne方法描述中的<i>#{cacheKey}</i>，以及具体的sql在哪里。不要急，马上在对象实体<i>Account</i>中我们就会认识到flying的存在。<i>Account.java</i>的代码如下：

    package myPackage;
    import org.apache.ibatis.type.JdbcType;
    import indi.mybatis.flying.annotations.FieldMapperAnnotation;
    import indi.mybatis.flying.annotations.TableMapperAnnotation;
    
    @TableMapperAnnotation(tableName = "account")
    public class Account {
        @FieldMapperAnnotation(dbFieldName = "id", jdbcType = JdbcType.INTEGER, isUniqueKey = true)
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
    
可见，和普通的pojo相比，<i>Account.java</i>只是多了以下3行注解而已：

     @TableMapperAnnotation(tableName = "account")
     @FieldMapperAnnotation(dbFieldName = "id", jdbcType = JdbcType.INTEGER, isUniqueKey = true)
     @FieldMapperAnnotation(dbFieldName = "name", jdbcType = JdbcType.VARCHAR) 

下面我们分别来解释它们的含义。

第1行@TableMapperAnnotation只能放在类定义之上，它声明这个类是一个表，它的属性tableName描述了这个表在数据库中的名字。

第2行@FieldMapperAnnotation只能放在变量定义之上，它声明这个变量是一个字段，它的属性dbFieldName描述了在数据库中这个字段的名称，属性jdbcType描述了在数据库中这个字段的类型，属性isUniqueKey为true描述了这个字段是主键。

第3行@FieldMapperAnnotation与第二行相同，它描述了另一个字段name，值得注意的是这个字段的类型是VARCHAR并且不是主键。

以上3个注解直白的描述了表account的数据结构，然后我们就可以使用<i>AccountService</i>非常方便的操纵数据库的读取了。（<i>AccountService</i>是<i>AccountMapper</i>的实现类，单独使用或在spring中使用都有多种方法进行配置，本文档不对此进行赘述）

使用以下代码，可以查询id为1的账户：

     Account account = accountService.select(1);
     
使用以下代码，可以查询name为andy的<b>1</b>条账户数据：

     Account accountCondition = new Account();
     accountCondition.setName("andy");
     Account account = accountService.selectOne(accountCondition);

与以往的方式相比，这种方式是不是变得优雅了很多？关于select和selectOne之间的区别，我们在后面的章节会讲到。

## insert & delete

在最基本的select之后，我们再看新增功能。但在此之前，需要先在<i>account.xml</i>中增加以下内容：

    <insert id="insert" useGeneratedKeys="true" keyProperty="id"></insert>

上面的`useGeneratedKeys="true"`表示主键自增，如果你不使用主键自增策略此处可以省略，上面的语句和一般mybatis映射文件的区别在于没有具体sql语句。

同样在<i>AccountMapper.java</i>中我们需要加入

    public void insert(Account t);

就可以了。例如使用以下代码，可以增加1条name为<i>bob</i>的账户数据（由于我们配置了主键自增，新增数据时不需要指定主键）：

    Account newAccount = new Account();
    newAccount.setName("bob");
    accountService.insert(newAccount);

然后我们再看删除功能。先在<i>account.xml</i>中增加以下内容：

    <delete id="delete"></delete>

然后在<i>AccountMapper.java</i>中加入

    public int delete(Account t);

就可以了。例如使用以下代码，可以删掉id与<i>accountToDelete</i>的id一致的数据。

    accountService.delete(accountToDelete);

delete方法的返回值代表执行sql后产生影响的条数，一般来说，返回值为0表示sql执行后没有效果，返回值为1表示sql执行成功，在代码中可以通过判断delete方法的返回值来实现更复杂的事务逻辑。

## update & updatePersistent

接下来我们看看更新功能，这里我们要介绍两个方法：update（更新）和updatePersistent（完全更新）。首先，在<i>account.xml</i>中增加以下内容：

    <update id="update"></update>
    <update id="updatePersistent"></update>

上面的语句和一般mybatis映射文件的区别在于没有具体sql语句。

然后在<i>AccountMapper.java</i>中加入

    public int update(Account t);
    public int updatePersistent(Account t);

就可以了。例如使用以下代码，可以将<i>accountToUpdate</i>的name更新为duke。

    accountToUpdate.setName("duke");
    accountService.update(accountToUpdate);

update和updatePersistent方法的返回值代表执行sql后产生影响的条数，一般来说，返回值为0表示sql执行后没有效果，返回值为1表示sql执行成功，在代码中可以通过判断update和updatePersistent方法的返回值来实现更复杂的事务逻辑。

下面我们来说明update和updatePersistent和关系。如果我们执行

    accountToUpdate.setName(null);
    accountService.update(accountToUpdate);

实际上数据库中这条数据的name字段不会改变，因为flying对为null的属性有保护措施。这在大多数情况下都是方便的，但如果我们真的需要在数据库中将这条数据的name字段设为null，updatePersistent就派上了用场。我们可以执行

    accountToUpdate.setName(null);
    accountService.updatePersistent(accountToUpdate);

这样数据库中这条数据就会发生变化。可见`updatePersistent`会把pojo中所有的属性都更新到数据库中，而`update`只更新不为null的属性。在实际使用`updatePersistent`时，需要特别小心慎重，因为你的pojo中当时为null的属性有可能比你想象的多！

## selectAll & count

为了能更清晰的展示，我们需要给<i>Account.java</i>再增加一个属性：

    @FieldMapperAnnotation(dbFieldName = "address", jdbcType = JdbcType.VARCHAR)
    private java.lang.String address;

（相关的getter和setter方法请自行补充）
