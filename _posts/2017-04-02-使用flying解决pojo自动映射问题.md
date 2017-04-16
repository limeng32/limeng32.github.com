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
	    <select id="select" resultMap="result"><i>#{id}</i></select>
	    <resultMap id="result" type="Account" autoMapping="true">
            <id property="id" column="account_id" />
        </resultMap>
    </mapper>
 
在以上配置文件中，我们描述了一个接口<i>myPackage.AccountMapper</i>，一个方法<i>select</i>，以及一个对象实体<i>Account</i>。
<i>myPackage.AccountMapper</i>接口是mybatis本身需要的，里面的内容和此配置文件中定义的方法相对应。如果你有使用mybatis的经验你就能预想到，<i>AccountMapper</i>中的内容是：

    package myPackage;
    public interface AccountMapper {
            public Account select(Object id);
    }
    
    到目前为止一切都和不使用flying时一模一样，你可能唯一奇怪的一点就是account.xml中的<select id="select" resultMap="result"><i>#{id}</i></select>，
