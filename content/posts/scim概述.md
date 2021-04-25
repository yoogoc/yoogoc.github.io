---
title: scim 翻译
tags:
- iam
date: 2021-04-16 16:19:00
---

![scim-logo](/images/scim-logo.png)

# 跨域身份管理系统

SCIM 2，用于管理身份的开放API现已完成，并在IETF下发布。

## 概述

跨域身份管理系统(SCIM)规范旨在使基于云的应用和服务中的用户身份管理更加容易。该规范套件寻求建立在现有模式和部署的经验基础上，特别强调开发和集成的简单性，同时应用现有的认证、授权和隐私模型。其目的是通过提供一个通用的用户模式和扩展模型，以及提供使用标准协议交换该模式的绑定文档，来降低用户管理操作的成本和复杂性。实质上：让用户快速、廉价、轻松地将用户移入、移出和在云中移动。

本概览页上的信息不具有规范性。



## 模型

SCIM 2.0建立在一个对象模型上，其中资源是共同点，所有SCIM对象都是由它派生出来的。它有id、externalId和meta作为属性，RFC7643定义了User、Group和EnterpriseUser，扩展了共同属性。

![scim-model](/images/scim-model.png)

## User 示例

这是一个如何将用户数据编码为JSON中的SCIM对象的例子。

虽然这个例子不包含完整的可用属性集，但请注意可以用来创建SCIM对象的不同数据类型。简单类型，如id、用户名等的字符串。复杂类型，即有子属性的属性，如姓名和地址。多值类型，如电子邮件、电话号码、地址等。

```json
{
  "schemas": ["urn:ietf:params:scim:schemas:core:2.0:User"],
  "id":"2819c223-7f76-453a-919d-413861904646",
  "externalId":"dschrute",
  "meta":{
    "resourceType": "User",
    "created":"2011-08-01T18:29:49.793Z",
    "lastModified":"2011-08-01T18:29:49.793Z",
    "location":"https://example.com/v2/Users/2819c223...",
    "version":"W\/\"f250dd84f0671c3\""
  },
  "name":{
    "formatted": "Mr. Dwight K Schrute, III",
    "familyName": "Schrute",
    "givenName": "Dwight",
    "middleName": "Kurt",
    "honorificPrefix": "Mr.",
    "honorificSuffix": "III"
  },
  "userName":"dschrute",
  "phoneNumbers":[
    {
      "value":"555-555-8377",
      "type":"work"
    }
  ],
  "emails":[
    {
      "value":"dschrute@example.com",
      "type":"work",
      "primary": true
    }
  ]
}
```

## Group 示例

除了用户，SCIM还包括组的定义。组用于模拟所提供资源的组织结构。组可以包含用户或其他组。

```json
{
  "schemas": ["urn:ietf:params:scim:schemas:core:2.0:Group"],
  "id":"e9e30dba-f08f-4109-8486-d5c6a331660a",
  "displayName": "Sales Reps",
  "members":[
    {
      "value": "2819c223-7f76-453a-919d-413861904646",
      "$ref": "https://example.com/v2/Users/2819c223-7f76-453a-919d-413861904646",
      "display": "Dwight Schrute"
    },
    {
      "value": "902c246b-6245-4190-8e05-00816be7344a",
      "$ref": "https://example.com/v2/Users/902c246b-6245-4190-8e05-00816be7344a",
      "display": "Jim Halpert"
    }
  ],
  "meta": {
    "resourceType": "Group",
    "created": "2010-01-23T04:56:22Z",
    "lastModified": "2011-05-13T04:42:34Z",
    "version": "W\/\"3694e05e9dff592\"",
    "location": "https://example.com/v2/Groups/e9e30dba-f08f-4109-8486-d5c6a331660a"
  }
}
```

## rest操作

对于资源的操作，SCIM提供了一个REST API，具有丰富但简单的操作集，支持从对特定用户的特定属性进行修补到进行大规模的批量更新。

- **Create:** POST https://example.com/{v}/{resource}
- **Read:** GET https://example.com/{v}/{resource}/{id}
- **Replace:** PUT https://example.com/{v}/{resource}/{id}
- **Delete:** DELETE https://example.com/{v}/{resource}/{id}
- **Update:** PATCH https://example.com/{v}/{resource}/{id}
- **Search:** GET https://example.com/{v}/{resource}?ﬁlter={attribute}{op}{value}&sortBy={attributeName}&sortOrder={ascending|descending}
- **Bulk:** POST https://example.com/{v}/Bulk

## Discovery

为了简化互操作性，SCIM提供了三个端点来发现支持的功能和具体的属性细节。

- GET /ServiceProviderConfig  规范遵守、认证方案、数据模型。
- GET /ResourceTypes 用于发现可用资源类型的端点。
- GET /Schemas 介绍资源和属性扩展。

## Create Request

要创建一个资源，请向资源的相应端点发送一个HTTP POST请求。在下面的例子中，我们看到的是一个用户的创建。

在这个例子和后面的例子中可以看到，URL包含一个版本号，这样SCIM API的不同版本可以共存。可用版本可以通过 ServiceProviderConﬁg 端点动态发现。

```
POST /v2/Users  HTTP/1.1
Accept: application/json
Authorization: Bearer h480djs93hd8
Host: example.com
Content-Length: ...
Content-Type: application/json

{
  "schemas":["urn:ietf:params:scim:schemas:core:2.0:User"],
  "externalId":"dschrute",
  "userName":"dschrute",
  "name":{
    "familyName":"Schrute",
    "givenName":"Dwight"
  }
}
```

## Create Response

响应中包含创建的资源和HTTP代码201，表示资源创建成功。需要注意的是，返回的用户包含的数据比发布的多，id和meta数据已经被服务商添加到了一个完整的User资源中。位置头表示在后续请求中可以在哪里获取资源。

```
HTTP/1.1 201 Created
Content-Type: application/scim+json
Location: https://example.com/v2/Users/2819c223-7f76-453a-919d-413861904646
ETag: W/"e180ee84f0671b1"

{
  "schemas":["urn:ietf:params:scim:schemas:core:2.0:User"],
  "id":"2819c223-7f76-453a-919d-413861904646",
  "externalId":"dschrute",
  "meta":{
    "resourceType":"User",
    "created":"2011-08-01T21:32:44.882Z",
    "lastModified":"2011-08-01T21:32:44.882Z",
    "location": "https://example.com/v2/Users/2819c223-7f76-453a-919d-413861904646",
    "version":"W\/\"e180ee84f0671b1\""
  },
  "name":{
    "familyName":"Schrute",
    "givenName":"Dwight"
  },
  "userName":"dschrute"
}
```

## Get Request

获取资源是通过向所需的资源端点发送HTTP GET请求来完成的，如本例。

```
GET /v2/Users/2819c223-7f76-453a-919d-413861904646 HTTP/1.1
Host: example.com
Accept: application/scim+json
Authorization: Bearer h480djs93hd8
```

## Get Response

GET的响应包含资源。在后续请求中，Etag头可用于防止资源的并发修改。

```
HTTP/1.1 200 OK HTTP/1.1
Content-Type: application/scim+json
Location: https://example.com/v2/Users/2819c223-7f76-453a-919d-413861904646
ETag: W/"f250dd84f0671c3"

{
  "schemas": ["urn:ietf:params:scim:schemas:core:2.0:User"],
  "id":"2819c223-7f76-453a-919d-413861904646",
  "externalId":"dschrute",
  "meta":{
    "resourceType": "User",
    "created":"2011-08-01T18:29:49.793Z",
    "lastModified":"2011-08-01T18:29:49.793Z",
    "location":"https://example.com/v2/Users/2819c223...",
    "version":"W\/\"f250dd84f0671c3\""
  },
  "name":{
    "formatted": "Mr. Dwight K Schrute, III",
    "familyName": "Schrute",
    "givenName": "Dwight",
    "middleName": "Kurt",
    "honorificPrefix": "Mr.",
    "honorificSuffix": "III"
  },
  "userName":"dschrute",
  "phoneNumbers":[
    {
      "value":"555-555-8377",
      "type":"work"
    }
  ],
  "emails":[
    {
      "value":"dschrute@example.com",
      "type":"work",
      "primary": true
    }
  ]
}
```

## Filter Request

除了获取单个资源外，还可以通过查询资源端点来获取资源集，而无需特定资源的id。通常情况下，获取请求会包含一个应用于资源的过滤器。SCIM 支持等价、包含、开始与等过滤操作。除了过滤响应外，还可以要求服务提供商对响应中的资源进行排序。

除了过滤响应外，还可以要求服务提供商对响应中的资源进行排序，返回资源的特定属性，以及只返回资源的子集。

- https://example.com/{resource}?ﬁlter={attribute} {op} {value} & sortBy={attributeName}&sortOrder={ascending|descending}&attributes={attributes}
- https://example.com/Users?ﬁlter=title pr and userType eq “Employee”&sortBy=title&sortOrder=ascending&attributes=title,username

## Filter Response

响应是一个匹配资源的列表。

```
{
  "schemas":["urn:ietf:params:scim:api:messages:2.0:ListResponse"],
  "totalResults":2,
  "Resources":[
    {
      "id":"c3a26dd3-27a0-4dec-a2ac-ce211e105f97",
      "title":"Assistant VP",
      "userName":"dschrute"
    },
    {
      "id":"a4a25dd3-17a0-4dac-a2ac-ce211e125f57",
      "title":"VP",
      "userName":"jsmith"
    }
  ]
}
```

# 规范

## SCIM 2.0

SCIM 2.0于2015年9月在[IETF](https://tools.ietf.org/wg/scim/)下以RFC7642、RFC7643和RFC7644的形式发布。

- [RFC7643 - SCIM](https://tools.ietf.org/html/rfc7643)：核心模式。
  核心模式提供了一个平台中立的模式和扩展模型来表示用户和组。

- [RFC7644 - SCIM](https://tools.ietf.org/html/rfc7644)：协议
SCIM协议是一个应用级的REST协议，用于在网络上提供和管理身份数据。

- [RFC7642 - SCIM](https://tools.ietf.org/html/rfc7642)：定义、概述、概念和要求。
本文档列出了跨域身份管理系统（SCIM）的用户场景和使用案例。

相关文档和扩展。

- [草稿-ansari-scim-soft-delete(软删除)](https://tools.ietf.org/id/draft-ansari-scim-soft-delete-00.txt)
本文档指定了一个配置文件，用于处理 SCIM 服务提供商的用户软删除。

- [草案-猎取-SCIM-通知](https://tools.ietf.org/id/draft-hunt-scim-notify-00.txt)
在SCIM环境中，多方可能会要求更改资源。随着时间的推移，感兴趣的用户可能希望被告知SCIM服务提供商正在发生的资源状态变化。本规范定义了一个中心通知服务，可用于向感兴趣的注册用户发布和分发事件。

- [草稿-猎取-SCIM-口令-管理服务](https://tools.ietf.org/id/draft-hunt-scim-password-mgmt-00.txt)
本规范定义了一组密码和账户状态扩展，用于管理密码和密码使用（如失败）以及其他相关会话数据。该规范定义了新的ResourceTypes，可以实现对密码和账户恢复功能的管理。

- [草案-wahl-scim-jit-profile](https://tools.ietf.org/id/draft-wahl-scim-jit-profile-02.txt)
本文件规定了跨域身份管理协议系统（SCIM）的配置文件。实现SAML或OpenID Connect等协议的服务器通过这些协议接收用户身份，并经常对其进行缓存，SCIM的这个配置文件定义了身份提供者如何将用户账户的变化通知SCIM服务器。

- [SCIM和vCard映射](https://tools.ietf.org/id/draft-greevenbosch-scim-vcard-mapping-04.txt)
该文档定义了SCIM和vCard之间的映射。

- [草案-Grizzle-SCIM-PAM-EXT](https://tools.ietf.org/id/draft-grizzle-scim-pam-ext-01.txt)
本文档包含SCIM 2.0对特权访问管理的扩展，其中包括对核心用户和组对象的扩展，以及标准特权访问管理结构的新资源类型和模式。该扩展旨在为PAM软件和客户端之间提供更强的互操作性、PAM概念的通用语言以及可进一步扩展以支持更复杂的PAM要求的基线。

