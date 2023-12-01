---
title: rebac/zanzibar介绍、实现及应用
tags:
- iam
date: 2023-12-01 13:00:00
---

# 介绍

ReBAC完整的英文是**Relational Based Access Control**，基于关系的访问控制。

这个概念是由Carrie E. Gates在2006年提出的，但他并没有给出如何实践，2019年，谷歌发表了一篇论文 [Zanzibar: Google’s Consistent, Global Authorization System](https://research.google/pubs/pub48190/)，他详细介绍了如何实践rebac，其中包括授权模型，提供的api接口，授权一致性方案，acl存储，acl索引，但并没有给出具体实现。

Zanzibar指出，谷歌的大量应用基于Zanzibar鉴权，包括但不局限于Calendar, Cloud, Drive, Maps, Photos,  YouTube，acl规模在万亿级别，每秒百万级请求量，三年可用性高达99.999%。

在传统的rbac中，只能限制操作权限，并不能限制带数据约束的操作权限，比如，rbac中无法表达`用户Bob拥有项目A的'查看'权限`，而rebac可以丝滑解决问题。rebac与rbac或abac并不是水火不容的，可以基于rebac构建rbac或abac。

## Zanzibar中的ReBAC

在Zanzibar中，每一条关联（也可称为acl）称为一个**Relation Tuple**，可以用伪代码表示为：

```
<tuple>    ::= <object>'#'<relation>'@'<user>
<obejct>   ::= <namespace>':'<object_id>
<user>     ::= <user_id> | <user_set>
<user_set> ::= <object>'#'<relation>
```

其中namespace和relation是在客户端预定义的，然后通过控制面接口上传至Zanzibar。

| 示例元组                           | 意义                          |
| ---------------------------------- | ----------------------------- |
| doc:readme#own@10                  | 用户10是readme文档的所有者    |
| group:eng#member@11                | 用户11是用户组eng的成员       |
| doc:readme#viewer@group:eng#member | 用户组eng是readme文档的查看者 |
| doc:readme#parent@folder:A         | 文件夹A是readme文档的父级     |

在计算权限时，则可以推测出，用户11有readme文档的查看权限，因为用户11是用户组eng的成员。

根据Zanzibar中的描述，我们可以将此权限模型表示为：

```
name: "doc"

relation { name: "parent" namespace: "folder"} // 论文中并没有表达出relation中存在namespace，但我推测应该会有此字段来显示声明userset类型，用来在acl保存时校验有效性

relation { name: "owner"}

relation {
	name: "editor"
	userset_rewrite: {
		union {
			child { _this {} }
			child { computed_userset { relation: "owner" } }
		}
	}
}

relation {
	name: "viewer"
	userset_rewrite: {
		union {
			child { _this {} }
			child { computed_userset { relation: "editor" } }
			child { 
				tuple_to_userset { 
					tupleset { relation "parent"}
					computed_userset {
						object: $TUPLE_USERSET_OBJECT
						relation: "viewer"
					}
				} 
			}
		}
	}
}

----------------------------------------------------------------
name "folder"

relation { name: "viewer"}
```

可以发现，schema中有一个重要的概念叫做userset_rewrite,貌似可以意译为**用户集重定向**。其意义是减少重复acl记录数量，翻译上述schema为：

1. 实体doc拥有三种权限：所有者，编辑者，查看者。
2. 所有者拥有编辑者的全部权限，编辑者可以查看实体doc
3. 如果用户或用户集拥有<实体doc的父级folder实体>的viewer权限，则用户或用户集拥有该实体doc的viewer权限

dsl中特殊语法解释：

1. _this： 代表所有有关联的用户
1. union： 或 
1. computed_userset：当前实体嵌套用户集
1. tuple_to_userset：其他实体嵌套用户集

# Zanzibar提供的api

Zanzibar中有一个重要的概念是zookie，他是一个encode后的全局时间戳，用来指定快照版本等，这部分在论文的一致性模型（2.2）里有详细流程，这里只关心api本身。

> acl与relation tuple同义

## read

通过定义的各种条件直接读取acl，应用层再将acl翻译成人类友好的元组，方便业务权限

## write

创建或修改或删除acl

## watch

可以通过watch api维护客户端二级索引

## check

请求指定 谁（user） 与 实体（object）是否存在关系（relation），返回运行或拒绝。

## expand

通常展开有两种方式，一种是返回与实体（object）存在关系（relation）的所有用户，另一种是返回与用户x有关系（relation）的所有实体（object）。使用场景比如：返回用户有权限的所有项目

# Zanzibar架构及实现

![](/images/zanzibar-arch.png)

1. aclserver会将复杂关联拆分，请求其他aclserver共同完成权限计算，当存在一个完成时则取消其他aclserver的计算
2. aclserver会有限流操作
3. Leopard是一个专门为Zanzibar实现的索引
4. 谷歌专门为Zanzibar开发了一个acl数据库Spanner

# 世面上开源的Zanzibar实现

## keto

第一款实现zanzibar的系统，但是他并没有schema的定义，namespace的定义是在服务端配置的，同理computed_rewrite也是在服务端配置，极度不灵活，而且文档模糊，花了很大的篇幅去说schema的定义，却不说这个定义要如何使用。

## openfga

cncf沙盒项目，可以使用dsl和json两种方式定义授权模型schema，但值得吐槽的事授权模型不可变，这对于多变的企业应用是非常不能接受的。

## permify

使用dsl定义授权模型，使用pg的txid和xid实现了一致性，所以数据库只能用pg

## warrant

使用json定义授权模型，api只有http

## SpiceDB

我不能理解他的定位，究竟是要做Spanner还是Zanzibar？看起来是要做Zanzibar



目前开源的实现都不是特别完整，理由如下：

1. 一致性模型实现的参差不齐
1. 对于ReBAC的理解各有不同
1. Leopard索引貌似都没实现（没看SpiceDB源码不知道他有没有实现）



# 最后

补一张表记录一下关于权限模型的对比

|              | casbin                           | keycloak                            | permify   | warrant               | openfga        | keto                  | cedar    | opa      |
| ------------ | -------------------------------- | ----------------------------------- | --------- | --------------------- | -------------- | --------------------- | -------- | -------- |
| acl          | √                                | √                                   | √         | √                     | √              | √                     | √        | √        |
| rbac         | √                                | √                                   | √         | √                     | √              | √                     | √        | √        |
| abac         | √                                | √                                   | √         | √                     | X              | X                     | √        | √        |
| rebac/fga    | X                                | X                                   | √         | √                     | √              | √                     | √        | √        |
| api类型      | 库                               | http                                | http,grpc | sdk(http)             | http,grpc      | sdk(http,grpc)        | 库       | 库       |
| 支持多租户   | √                                | √                                   | √         | √                     | √              | √                     | √        | √        |
| 权限模型定义 | dsl                              | 页面操作                            | dsl       | json                  | dsl,json       | NO                    | dsl      | dsl      |
| 基于zanzibar | X                                | X                                   | √         | √                     | √              | √                     | X        | X        |
| 后端存储     | 实现Adapters接口即可             | mariadb/mssql/mysql/oracle/postgres | postgres  | postgres/mysql/sqlite | postgres/mysql | postgres/mysql/sqlite | DIY      | DIY      |
| 开源支持     | 美籍华人                         | CNCF                                | 独立公司  | 独立公司              | CNCF           | ORY                   | AWS      | CNCF     |
| 开发语言     | 初版是go，之后实现了很多其他语言 | java                                | go        | go                    | go             | go                    | rust     | go       |
| 定位         | 权限引擎，需要实现上层控制面     | IAM                                 | authz     | authz                 | authz          | authz                 | 策略引擎 | 策略引擎 |

