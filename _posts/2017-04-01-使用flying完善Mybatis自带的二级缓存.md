---
layout: post
title: 使用 flying 完善Mybatis 自带的二级缓存
description: 本节内容向您讲解如何使用 EnhancedCachingInterceptor 拦截器来改造Mybatis的二级缓存使其可用。
category: blog
---
<a id="Index"></a>
## [上手指南](#Index)
上一篇文章中我们介绍了使用 flying 解决 pojo 自动映射问题，在本篇文章中我们介绍如何使用 flying 优化后的 mybatis 自带二级缓存。通常我们会选择 Redis 等更为强大的第三方缓存，但如果您的系统用户数不到一千，您也不想为缓存配置额外的服务器，那您可以试试 mybatis 自带的二级缓存，因为它方便上手且成本极低。

mybatis 的二级缓存只存在于内存中，不会写入硬盘，所以服务容器一旦停止工作缓存就会消失，在使用前请确保您的业务可以接受这些特性。

为使用 flying 优化后的 mybatis 自带二级缓存，您只需要在 Configuration.xml 和 <i>pojo_mapper</i>.java 中进行简单的配置。在 Configuration.xml 中配置的内容是：
```
<settings>
	<setting name="cacheEnabled" value="true" />
	<setting name="lazyLoadingEnabled" value="false" />
	<setting name="aggressiveLazyLoading" value="false" />
	<setting name="localCacheScope" value="SESSION" />
	<setting name="autoMappingBehavior" value="PARTIAL" />
	<setting name="lazyLoadTriggerMethods" value="" />
</settings>
```
需要注意的是 `settings` 中第一行 `cacheEnabled` 的值为 `true`，其它内容都和 《为什么要开发mybatis.flying》 中介绍的配置完全一致。

在完成上述工作后，mybatis 的二级缓存就可以使用了。现在启动项目后，每次对数据库发起的查询请求都会被缓存进行拦截，如果在缓存中能找到结果（包括单个 pojo、pojo集合、数量等）就直接返回结果，不会对数据库产生压力；如果找不到结果再去数据库中进行查询，查询后返回结果的同时把此次结果记入缓存中，这样下次再进行相同条件查询时如果相关记录没进行过刷新型操作（如 update、delete），就会返回缓存中的结果；如果对某条数据进行了 update、delete 操作，会使得相关缓存失效，其中的机制在后面会有详细介绍。

由于服务容器刚启动时缓存中没有任何内容，因此所有查询第一次都会经过数据库，在此之后缓存将发挥作用。为了更好的说明，我们需要新建[一个账号表](#AccountTableCreater)和[一个角色表](#RoleTableCreater)、相关的 `account.xml`、`role.xml`、`AccountMapper.java`、`RoleMapper.java`、`AccountService.java`、`RoleService.java`、`Account.java` 以及 `Role.java`。`AccountMapper.java` 如下：
```
package myPackage;
public interface AccountMapper {
    public Account select(Object id);
    public Account selectOne(Account t);
    public Collection<Account> selectAll(Account t);
    public int count(Account t);
    public void insert(Account t);
    public int update(Account t);
    public int updatePersistent(Account t);
    public int delete(Account t);
}
```
`RoleMapper.java` 如下：
```
package myPackage;
public interface RoleMapper {
    public Role select(Object id);
    public Role selectOne(Role t);
    public Collection<Role> selectAll(Role t);
    public int count(Role t);
    public void insert(Role t);
    public int update(Role t);
    public int updatePersistent(Role t);
    public int delete(Role t);
}
```
`account.xml`、`role.xml`、`AccountService.java`、`RoleService.java`、`Account.java` 和 `Role.java` 与《使用 flying 解决 pojo 自动映射问题》中介绍的完全一致且不会在本文中进行修改，因此这里不再累述。

之后，我们再定义一个用于测试的数据集：
```
<dataset>
    <account account_id="1" fk_role_id="10" address="beijing" name="frank" />
    <account account_id="2" fk_role_id="11" address="tianjin" name="gale" />
    <account account_id="3" fk_role_id="11" address="beijing" name="hank" />
    <role role_id="10" role_name="user" />
    <role role_id="11" role_name="super_user" />
</dataset>
```
当我们第一次执行 `accountService.select(1)` 时，会从数据库中得到一个 `name` 为 “frank” 的 Account 对象，当再次执行 `accountService.select(1)` 时，会不经数据库而从缓存中得到这个 Account 对象。如果我们通过数据库管理工具直接修改了这条记录，比如将 `name` 改为 “iris”，缓存也不会知道我们做出了改动，再次执行 `accountService.select(1)` 得到的 pojo 的 `name` 依然为 “frank”。但接下来如果我们在程序中执行 `accountService.update(account)` 将此对象的 `name` 改为 “hank”，则在数据库中的记录变更后缓存中的对象也会刷新，下一次再执行 `accountService.select(1)` 得到的 Account 对象的 `name` 就是 “hank” 了。

最后，我们强烈不建议在开启缓存的项目正在运行的情况下，通过数据库管理工具直接修改数据，这会导致数据库和缓存不一致，就像上一段中演示的那样。当您得到缓存的好处时，您也只能使用已经定义好的方法来操作数据库。（实际上，我们不建议通过数据库管理工具直接修改任何正在运行中的系统，无论它是否使用了缓存。）
## [观察者 & 触发者](#Index)
在上一节中我们了解了 mybatis 二级缓存的原理，但它本身还有需要优化的地方。例如当您缓存了一个 Account 对象后，当这个对象对应的 Role 对象发生改变时，我们当然希望缓存中的 Account 对象会知道它的多对一关系 Role 对象已经失效，然后在查询时会到数据库中查找最新数据（因为查询子对象时会自动加载父对象）。然而遗憾的是，mybatis 原始的二级缓存并没有这个功能，但是 flying 提供了解决方案。您只需将 flying 优化缓存的插件配置好并了解 “观察者” 和 “触发者” 的概念，就可以解决这一问题。

首先在 `Configuration.xml` 中的 `plugins` 中加入以下内容：
```
<plugins>
    <plugin interceptor="indi.mybatis.flying.interceptors.EnhancedCachingInterceptor">
        <property name="cacheEnabled" value="true" />
        <property name="annotationPackage" value="myPackage" />
    </plugin>
</plugins>
```
其中 `cacheEnabled` 为 `true` 表示让优化插件生效，`annotationPackage` 的值则是放置您所有 <i>pojo_mapper</i>.java 接口的包路径。

然后我们来介绍<b>观察者</b>和<b>触发者</b>的概念。在有父子关系的两个 pojo 中，父对象自身的改变可以使子对象的缓存失效，因此父对象可以称作子对象的<b>触发者</b>，而因为子对象只能参考它的父对象，子对象又可称为父对象的<b>观察者</b>。值得一提的是，这两种称呼都是基于 pojo 的关系的。所以会出现这种情况：pojo A 是 B 的触发者，pojo B 是 A 的观察者；同时 pojo B 是 C 的触发者，pojo C 是 B 的观察者。触发-观察关系可以传递，因此可以称 pojo A 是 C 的<b>间接触发者</b>，pojo C 是 A 的<b>间接观察者</b>。（如果两个 pojo 间有 触发-观察 关系但又不是直接关联，那就称为 间接触发-间接观察 关系，稍后您就后明白为什么要这么定义）。

接下来我们还需要了解触发者有哪些方法可以“触发”观察者，由之前的文章可以知道，`update`、`delete` 会使观察者改变，`select` 不会改变观察者，而 `insert` 是增加新的父对象，也不会改变观察者。了解以上概念后，我们就需要让程序了解我们的对象关系模型，而 flying 选择在 <i>pojo_mapper</i>.java 接口中接收您描述的对象关系模型。

我们只需在 AccountMapper.java 中增加一些注解：
```
package myPackage;
import indi.mybatis.flying.annotations.CacheAnnotation;
import indi.mybatis.flying.annotations.CacheRoleAnnotation;
import indi.mybatis.flying.statics.CacheRoleType;

@CacheRoleAnnotation(ObserverClass = { Role.class }, TriggerClass = { Account.class })
public interface AccountMapper {

    @CacheAnnotation(role = CacheRoleType.Observer)
    public Account select(Object id);
    
    @CacheAnnotation(role = CacheRoleType.Observer)
    public Account selectOne(Account t);
    
    @CacheAnnotation(role = CacheRoleType.Observer)
    public Collection<Account> selectAll(Account t);
    
    @CacheAnnotation(role = CacheRoleType.Observer)
    public int count(Account t);
    
    public void insert(Account t);
    
    @CacheAnnotation(role = CacheRoleType.Trigger)
    public int update(Account t);
    
    @CacheAnnotation(role = CacheRoleType.Trigger)
    public int updatePersistent(Account t);
    
    @CacheAnnotation(role = CacheRoleType.Trigger)
    public int delete(Account t);
}
```
接下来我们对这些注解依次说明：
`@CacheRoleAnnotation` 只能放在类定义之上，它声明从当前这个 <i>pojo_mapper</i>.java 接口角度看待其它 pojo 的关系，具体来说，`ObserverClass` 里需要填写当前 pojo 的所有<b>直接触发者</b>的类名（无需填写间接触发者，因为 flying 在运行期可以推导出），`TriggerClass` 则<b>只需填写当前接口对应的 pojo 类名</b>。填好以后，`@CacheRoleAnnotation` 就配置完毕。

`@CacheAnnotation` 只能放在方法定义之上，它声明当前这个方法是具有“触发”能力还是具有“观察”能力。具有触发能力的方法可以刷新这个接口的观察者的缓存，具有观察能力的方法可以发现这个接口的触发者的缓存被刷新，因为 触发－观察 关系可以传递，最终所有 pojo 的缓存关系都会被 flying 所掌握。

在 flying 给定的方法中，`update`、`updatePersistent`、`delete` 具有触发能力，`select`、`selectOne`、`selectAll`、`count` 具有观察能力，`insert` 既不具备触发能力也不具备观察能力。如果您在 <i>pojo_mapper</i>.java 中还有自己定义的方法，需要看这个方法对应的 sql 语句类型，如果是 update 或 delete 类型就具有触发能力，如果是 select 类型就具有观察能力。

按以上情况配置好后，flying 优化 mybatis 二级缓存的工作就完成了，现在您可以放心的使用 mybatis 的自带缓存，而不用担心任何缓存与数据库不匹配的问题。

## [注意事项](#Index)


## [附录](#Index)
<a id="AccountTableCreater"></a>
### [account 表建表语句](#Index)
```
CREATE TABLE account (
  account_id int(11) NOT NULL AUTO_INCREMENT,
  name varchar(20) DEFAULT NULL,
  address varchar(100) DEFAULT NULL,
  fk_role_id int(11) DEFAULT NULL,
  fk_second_role_id int(11) DEFAULT NULL,
  PRIMARY KEY (account_id)
)
```

<a id="RoleTableCreater"></a>
### [role 表建表语句](#Index)
```
CREATE TABLE role (
  role_id int(11) NOT NULL AUTO_INCREMENT,
  role_name varchar(30) DEFAULT NULL,
  PRIMARY KEY (role_id)
)
```
