---
title: "keystone详解"
date: 2020-03-20T17:41:16+08:00
draft: false
---
# keystone详解

## keystone架构

 ![这里写图片描述](https://img-blog.csdn.net/20180515212224345?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2R5bGxvdmV5b3U=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70) 

 ![技术分享](E:\文档\笔记\keystone详解.assets\20180110220412160294.png) 

## 基本概念

　　官方介绍

　　　　1.User：has account credentials, is associated with one or more projects or domains

　　　　　　　　 user是账户凭证，是与一个或多个项目或相关的域

　　　　2.Group: a collection of users, is associated with one or more projects or domains

　　　　       group就是用户的一个集合，与一个或多个项目或相关的域

　　　　3.Project(Tenant)： unit of ownership in OpenStack, contains one or more users

　　　　　　　　project(tenant)是一个单位，指在openstack中可用的资源，它包含一个或多个用户

　　　　4.Domain：unit of ownership in OpenStack, contains users, groups and projects

　　　　　　　　domain是一个单位，它包含用户、组和项目

　　　　5.Role：a first-class piece of metadata associated with many user-project pairs

　　　　　　　　role一个用户与project相关联的元素据

　　　　6.Token：identifying credential associated with a user or user and project

　　　　　　　　Token鉴定凭证关联到用户或者用户和项目

　　　　7.Extras：bucket of key-value metadata associated with a user-project pair

　　　　　　　　extras关于用户-项目的关系，还可以设置一些其他的属性

　　　　8.Rule:describes a set of requirements for performing an action

　　　　　　　　rule描述了一组要求 执行一个动作

### User

User 指代任何使用 OpenStack 的实体，可以是真正的用户，其他系统或者服务。当 User 请求访问 OpenStack 时，Keystone 会对其进行验证。

Horizon 在 “身份管理->用户” 管理 User： 
除了 admin 和 demo，OpenStack 也为 nova、cinder、glance、neutron 等服务创建了相应的 User。 admin 也可以管理这些 User。 
 ![这里写图片描述](https://img-blog.csdn.net/20180515212343559?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2R5bGxvdmV5b3U=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70) 

###  **Credentials** 

 Credentials 是 User 用来证明自己身份的信息。可以是： （1） 用户名/密码 （2） Token （3） API Key （4） 其他高级方式。 

```
	   （1）：用户名和密码

　　　　（2）：用户名跟API Kye（秘钥）　　　　　　　　　　　　　　#（1）（2）用户第一次确认身份的方法

　　　　（3）：一个keystone分配的身份的token　　　　　　　　　　  #（3）用户已经确认身份后的方法 （token是有时间限制的）
```

###  **Authentication** 

Authentication 是 Keystone 验证 User 身份的过程。User 访问 OpenStack 时向 Keystone 提交用户名和密码形式的 Credentials，Keystone 验证通过后会给 User 签发一个 Token 作为后续访问的 Credential。

###  **Token** 

Token 是由数字和字母组成的字符串，User 成功 Authentication 后由 Keystone 分配给 User。Token 用做访问 Service 的 Credential，Service 会通过 Keystone 验证 Token 的有效性。

Token 还有 **scope** 的概念，表明这个 Token 作用在什么范围内的资源，例如某个 Project 范围或者某个 Domain 范围，Token 只能用于认证用户对指定范围内资源的操作。Token 的**有效期**默认是 24 小时。

Keystone 提供如下几种 Token，可以根据需要配置使用某种 Token：

`UUID token`：服务 API 收到带有 UUID token 的请求时，必须到 Keystone 进行验证token 的合法性，验证通过之后才能响应用户请求。随着集群规模的扩大，Keystone 需处理大量验证 UUID token 的请求，在高并发下容易出现性能问题。

`PKI token`：携带更多用户信息并附上了数字签名，服务 API收 到 PKI token 时无需再去Keystone 验证，但是 PKI token 所携带可能随着 OpenStack Region 增多而变得非常长，很容易超出 HTTP Server 允许的最大 HTTP Header（默认为 8 KB），导致 HTTP 请求失败。

`PKIZ token`：PKI token 的压缩版，但压缩效果有限，无法良好的处理 token 长度过大问题。

`Fernet token`：携带了少量的用户信息，大小约为 255 Byte，采用了对称加密，无需存于数据库中。前三种 token 都会持久性存于数据库，与日俱增积累的大量 token 引起数据库性能下降，所以用户需经常清理数据库的 token；Fernet token没有这样的需要。

>  **在Ocata版本中 Fernet 成为默认的 Token Provider。 PKI 和 PKIz token provider 被移除 。** 

###  **Project** 

Project 用于将 OpenStack 的资源（计算、存储和网络）进行分组和隔离。在企业私有云中，Project 可以是一个部门或者项目组，和公有云的 VPC（虚拟私有网络）概念类似。资源的所有权是属于 Project 的，而不是 User。每个 User（包括 admin）必须挂在 Project 里才能访问该 Project 的资源，一个 User 可以属于多个 Project。

```
（1）：Project（Tenant）是一个人或服务所拥有的资源集合。不同的Project之间资源是隔离的，资源可以设置配额

（2）：Project（Tenant）中可以有多个User，每一个User会根据权限的划分来使用Project（Tenant）中的资源

（3）：User在使用Project（Tenant）的资源前，必须要与这个Project关联，并且制定User在Project下的Role，一个assignment(关联) 即：Project-User-Role
```

Horizon 在 “身份管理->项目” 中管理 Project： 
可以通过 “管理成员” 将 User 添加到 Project 中。 
 ![可以通过 “管理成员” 将 User 添加到 Project 中](E:\文档\笔记\keystone详解.assets\20180515212706136.png) 

###  **Service** 

OpenStack 的 Service 包括 Compute (Nova)、Block Storage (Cinder)、Object Storage (Swift)、Image Service (Glance) 、Networking Service (Neutron) 等。每个 Service 都会提供若干个 Endpoint，User 通过 Endpoint 访问资源和执行操作。

### Endpoint

 Endpoint 是一个网络上可访问的地址，通常是一个 URL。Service 通过 Endpoint 暴露自己的 API。Keystone 负责管理和维护每个 Service 的 Endpoint。  

 ![这里写图片描述](E:\文档\笔记\keystone详解.assets\20180515212813296.png) 

###  **Role** 

安全包含两部分：Authentication（认证）和 Authorization（鉴权）。Authentication 解决的是“你是谁？”的问题， Authorization 解决的是“你能干什么？”的问题。Keystone 是借助 Role 来实现 Authorization 的。Role 是全局（global）的，因此在一个 keystone 管辖范围内其名称必须唯一。

```
　　　　（1）：本身是一堆ACL集合，主要用于权限的划分。

　　　　（2）：可以给User指定Role，是user获得role对应的操作权限。

　　　　（3）：系统默认使用管理Role的角色 管理员用户：admin 普通用户：member（老版本） user（新版本）

　　　　（4）：user验证的时候必须带有Project。老版本叫（Tenant）
```

Horizon 在 “身份管理->角色” 中管理 Role： 
可以为 User 分配一个或多个 Role。 
 ![这里写图片描述](E:\文档\笔记\keystone详解.assets\20180515212936310.png) 

Service 决定每个 Role 能做什么事情。Service 通过各自的 policy.json 文件对 Role 进行访问控制。Role 的名称没有意义，其意义在于 policy.json 文件根据 role 的名称所指定的允许进行的操作。 

上面是Nova 服务 /etc/nova/policy.json 中的示例，配置含义为：

![image-20191101150414187](E:\文档\笔记\keystone详解.assets\image-20191101150414187.png)

对于 create、attach_network 和 attach_volume 操作，任何 Role 的 User 都可以执行； 但只有 admin 这个 Role 的 User 才能执行 forced_host 操作。
OpenStack 默认配置只区分 admin 和非 admin Role。 如果需要对特定的 Role 进行授权，可以修改 `policy.json`。

### Group

Group 是一个 domain 部分 user 的集合，其目的是为了方便分配 role。给一个 group 分配 role，结果会给 group 内的所有 users 分配这个 role。

Horizon 在 “身份管理->组” 中管理 Group： 

 ![这里写图片描述](E:\文档\笔记\keystone详解.assets\20180515213218485.png) 

### Domain

Domain 表示 Project、Group 和 User 的集合，在公有云或者私有云中常常表示一个客户，和 VDC（虚拟机数据中心）的概念类似。Domain 可以看做一个命名空间，就像域名一样，全局唯一。在一个 Domain 内，Project、Group、User 的名称不可以重复，但是在两个不同的 Domain 内，它们的名称可以重复。因此，在确定这些元素时，需要同时使用它们的名称和它们的 Domain 的 id 或者 name。

下图表示了 Domain、Project、User、Group、Role 之间的关系： 
![image-20191101150720511](E:\文档\笔记\keystone详解.assets\image-20191101150720511.png)

 下面是一个整体关系图：  

 ![这里写图片描述](E:\文档\笔记\keystone详解.assets\20180515213316730.png) 

### Policy

　　　　（1）：对于keystone service 来说，Policy就是一个JSON文件，rpm安装默认是在/etc/keyston/policy.json。通过配置这个文件，keystone实现了对User基于Role的权限管理（User <-- Role（ACL） <--Policy）

　　　　（2）：Policy就是用来控制User对Project（tenant）中资源的操作权限

## 交互流程

  ![img](E:\文档\笔记\keystone详解.assets\1092539-20170118160810140-898855570.png)  

- 首先用户向 Keystone 提供自己的身份验证信息，如用户名和密码。
- Keystone 会从数据库中读取数据对其验证，如验证通过，会向用户返回一个 token，此后用户所有的请求都会使用该 token 进行身份验证。

如用户向 Nova 申请虚拟机服务，nova 会将用户提供的 token 发给 Keystone 进行验证，Keystone 会根据 token 判断用户是否拥有进行此项操作的权限，若验证通过那么 nova 会向其提供相对应的服务。其它组件和 Keystone 的交互也是如此。

 ![img](E:\文档\笔记\keystone详解.assets\1092539-20170203095839933-2134357353.png) 

比如说，某公司年会组织跟团去旅游（公司相当于一个group，公司的员工相当于User）。到了晚上要住店，首先要先到前台登记（前台就相当于Keystone），对前台（keystone）来说，你要住店要拿出你的证明（对keystone来说就是要证明你是你）。

　　　　怎么办？拿出身份证，这里的身份证就相当于Credentials（用户名和密码），前台（keystone）会进行验证你的身份信息（Authentication），验证成功后，前台（Keystone）会给你一个房卡（Token），并且有不同的房卡（比如：普通卡，会员卡，白金卡等），不同的卡有不同的权限（Role），并且拿到房卡后，前台（keystone）会给你一个导航图（Endpoint）让你找到你的房间。并且一个酒店不光会有住宿服务，可能还有别的服务（service），像餐饮，娱乐，按摩等等，比如说要去吃饭，不知道路线怎么走，看一下导航图（endpoint）就知道了，到餐饮部门（service）会有三个路线（Endpoint）可以走。为什么会有三个，领导层通道 --> 走后门（admin），内部员工通道 -->（internal），客人通道  -->（public）。知道如何去，也有了权限（Token/Role）到了餐饮部门，当你点餐的时候，会让你刷上你的会员卡（这个步骤就是service像keystone确认你有没有权限），验证成功后，你就可以点餐吃饭

## REST API 调用

 示例 ：User admin 要查看 Project 中的 image。 

###  第一步，获取Token。 

URL http://controller:35357/v3/auth/tokens 
传入的参数包括 user id，user password，project id，示例如下： 

```json
{
    "auth": {
        "identity": {
            "methods": [
                "password"
            ],
            "password": {
                "user": {
                    "id": "ee4dfb6e5540447cb3741905149d9b6e",
                    "password": "admin"
                }
            }
        },
        "scope": {
            "project": {
                "id": "a6944d763bf64ee6a275f1263fae0352"
            }
        }
    }
}
```

 执行成功后，会返回 Token 信息。  

###  第二步，根据token获取image列表。  

 URL http://controller:9292/v2/images 
其中请求的 Header 需要加上上一步返回的 Token。 

![image-20191101152228171](E:\文档\笔记\keystone详解.assets\image-20191101152228171.png)

 下面是返回的 images 列表信息（JSON 格式）：  

![image-20191101152316055](E:\文档\笔记\keystone详解.assets\image-20191101152316055.png)

## 架构分析

### 组件

####  **Keystone API**

 Keystone API与Openstack其他服务的API类似，也是基于ReSTFul HTTP实现的。 Keystone API划分为Admin API和Public API： Public API不仅实现获取版本以及相应扩展信息的操作，同时包括获取Token以及Token租户信息的操作； Admin API主要提供服务开发者使用，不仅可以完成Public API的操作，同时对User、Tenant、Role和Service Endpoint进行管理操作。 

#### **Router**

Keystone Router主要实现上层API和底层服务的映射和转换功能，包括四种Router类型。 
（1） AdminRouter 
　　负责将Admin API请求映射为相应的行为操作并转发至底层相应的服务执行； 
（2） PublicRouter 
　　与AdminRouter类似； 
（3） PublicVersionRouter 
　　对系统版本的请求API进行映射操作； 
（4） AdminVersionRouter 
　　与PublicVersionRouter类似。 

#### **Services**

Keystone Service接收上层不同Router发来的操作请求，并根据不同后端驱动完成相应操作，主要包括四种类型； 
**（1） Identity Service** 

Identity Service提供关于用户和用户组的授权认证及相关数据。 Keystone-10.0.0支持ldap.core.Identity,Sql.Identity两种后端驱动，系统默认的是Sql.Identity； 

- users和groups  

**（2） Resource Service**   

Resouse服务提供关于projects和domains的数据

- projects和domains 

- 在v3版本中的唯一性概念 

**（3） Assignment Service** 

Assignment Service提供role及role assignments的数据

- roles和role assignments 

**（4） Token Service** 
Token Service提供认证和管理令牌token的功能，用户的credentials认证通过后就得到token Keystone-10.0.0对于Token Service 
支持Kvs.Token，Memcache.Token，Memcache_pool.Token,Sql.Token四种后端驱动，系统默认的是kvs.Token
**（5） Catalog Service** 
Catalog Service提供service和Endpoint相关的管理操作（service即openstack所有服务，endpont即访问每个服务的url） keystone-10.0.0对Catalog Service支持两种后端驱动：Sql.Catalog、Templated.Catalog两种后端驱动，系统默认的是templated.Catalog；

**（6） Policy Service** 
Policy Service提供一个基于规则的授权驱动和规则管理 keystone-10.0.0对Policy Service支持两种后端驱动：rules.Policy,sql.Policy，默认使用sql.Policy

#### **Backend Driver** 

Backend Driver具有多种类型，不同的Service选择不同的Backend Driver。

 **官方http://docs.openstack.org/developer/keystone/architecture.html#groups** 

### keystone管理这些概念的方法  

| **组件名称** | **管理对象**                                    | **生成方法**              | **保存方式**        | **配置项**                                                   |
| ------------ | ----------------------------------------------- | ------------------------- | ------------------- | ------------------------------------------------------------ |
| identity     | user，以及 user group                           | -                         | **sql**, ldap.core  | [identity] <br />driver = keystone.identity.backends.[**sql**\|ldap.core].Identity |
| token        | 用户的临时 token                                | pki，pkiz，uuid           | sql, kvs,memcached  | [token] <br />driver = keystone.token.persistence.backends.[**sql**\|kvs\|memcache\|memcache_pool].Token<br />provider=keystone.token.providers.[pkiz\|pki\|**uuid**].Provider |
| credential   | EC2 credential                                  | -                         | **sql**\| templated | [catalog] driver = keystone.catalog.backends.[**sql**\| templated].Catelog |
| assignment   | tenant，domain，role 以及它们与 user 之间的关系 | external, password, token | sql                 | [assignment] <br />methods = external, password, token<br /> password = keystone.auth.plugins.password.Password |
| trust        | trust                                           | -                         | sql                 | [trust] driver = keystone.trust.backends.[**sql**].Trust     |
| policy       | Keystone service 的用户鉴权策略                 | -                         | ruels\| sql         | [default]<br /> policy_file = policy.json <br />[policy] <br />driver = keystone.policy.backends.[ruels\| sql].Policy |



### keystone-10.0.0代码结构展示 

keystone-manage 是个 CLI 工具，它通过和 Keystone service 交互来做一些无法使用 Keystone REST API 来进行的操作，包括： 

`db_sync`: Sync the database. 

`db_version`: Print the current migration version of the database. 

`mapping_purge`: Purge the identity mapping table. 

`pki_setup`: Initialize the certificates used to sign tokens. 

`saml_idp_metadata`: Generate identity provider metadata. 

`ssl_setup`: Generate certificates for SSL. 

`token_flush`: Purge expired tokens.   每个Keystone 组件，比如 catalog， token 等都有一个单独的目录。
每个组件目录中：
routes.py  定义了该组件的 routes 。其中identity 和 assignment 分别定义了 admin 和 public 使用的 routes，分别供 admin service 和 public service 使用。 
controller.py 文件定义了该组件所管理的对象，比如 assignment 的controller.py 文件定义了 Tenant、Role、Project 等类。
core.py 定了了两个类 Manager 和 Driver。Manager 类对外提供该组件操作具体对象的方法入口； Driver 类定义了该组件需要其Driver实现类所提供的接口。
backend 目录下的每个文件是每个具体driver 的实现

###  keystone服务启动

> Keystone is an HTTP front-end to several services. Like other  OpenStack applications, this is done using python WSGI interfaces and  applications are configured together using [Paste](http://pythonpaste.org/). The application’s HTTP endpoints are made up of pipelines of WSGI middleware。。。 详见：http://docs.openstack.org/developer/keystone/architecture.html  



/usr/bin/keystone-all 会启动 keystone 的两个service：admin and main，它们分别对应 /etc/keystone/keystone-paste.ini 文件中的两个composite：

 ![技术分享](E:\文档\笔记\keystone详解.assets\20180110220412434719.png) 

可见 admin service 提供给administrator 使用；main 提供给 public 使用。它们分别都有 V2.0 和 V3 版本，只是目前的 keystone Cli 只支持 V2.0.

#### 比较下 admin 和 public： 

| **名称** | **middlewares**                                              | **factory**                         |
| -------- | ------------------------------------------------------------ | ----------------------------------- |
| admin    | 比 public 多 s3_extension                                    | keystone.service:public_app_factory |
| public   | sizelimit url_normalize <br />build_auth_context <br />token_auth <br />admin_token_auth <br />xml_body_v2 <br />json_body ec2_extension <br />user_crud_extension | keystone.service:admin_app_factory  |

  **功能区别** : 从 factory 函数来看， admin service 比 public service 多了 identity 管理功能， 以及 assignment 的admin/public 区别： 1. admin 多了对 GET /users/{user_id} 的支持，多了 get_all_projects， get_project，get_user_roles 等功能 2. keystone 对 admin service 提供 admin extensions, 比如 [OS-KSADM](http://docs.rackspace.com/openstack-extensions/auth/OS-KSADM-admin-devguide/content/Admin_API_Service_Developer_Operations-d1e1357.html) 等；对 public service 提供 public extensions。 

**简单总结一下， public service 主要提供了身份验证和目录服务功能；admin service 增加了 tenant、user、role、user group 的管理功能。** 

/usr/bin/keystone-all 会启动 admin 和 public 两个 service，分别绑定不同 host 和  端口。默认的话，绑定host 都是 0.0.0.0； admin 的绑定端口是 35357 （admin_port）， public  的绑定端口是 5000 （public_port）。因此，给 admin 使用的 OS_AUTH_URL 为  http://controller:35357/v2.0， 给 public  使用的 OS_AUTH_URL=http://controller:5000/v2.0  

#### keystone详细说明 

WSGI server的父进程(50511号进程)开启两个socket去分别监听本环境的5000和35357号端口，
其中5000号端口是为main的WSGI server提供的，35357号端口为admin的WSGI server提供的。
即WSGI server的父进程接收到5000号端口的HTTP请求时，则将把该请求转发给为main开启的WSGI server去处理，
而WSGI server的父进程接收到35357号端口的HTTP请求时，则将把该请求转发给为admin开启的WSGI server去处理。

vim /etc/keystone/keystone-paste.ini

![技术分享](E:\文档\笔记\keystone详解.assets\20180110220412458157.png) 日志

> 2016-09-14 11:53:01.037 12698 INFO keystone.common.wsgi [req-07b28d5b-084c-467e-b45a-a4c8a52b7e96
> 9ff041112e454cca9b54bf117a80ca29 15426931fe4746d08736c5e5c1da6b1c 
> \- 6e495643fb014e5e8a3992c69d80d234 6e495643fb014e5e8a3992c69d80d234] 
> GET http://controller02:35357/v3/auth/tokens

(1) type = composite 

这个类型的section会把URL请求分发到对应的Application，use表明具体的分发方式，比如”egg:Paste#urlmap”表示使用Paste包中的urlmap模块，这个section里的其他形式如”key = value”的行是使用urlmap进行分发时的参数。

 (2) type = app 

一个app就是一个具体的WSGI Application。

 (3) type = filter-app 

接收一个请求后，会首先调用filter-app中的use所指定的app进行过滤，如果这个请求没有被过滤，就会转发到next所指定的app进行下一步的处理。

 (4) type = filter 

与filter-app类型的区别只是没有next。

 (5) type = pipeline 

pipeline由一系列filter组成。这个filter链条的末尾是一个app。pipeline类型主要是对filter-app进行了简化，
否则，如果多个filter，就需要多个filter-app，然后使用next进行连接。OpenStack的paste的deploy的配置文件主要采用的pipeline的方式。
因为url为http://192.168.118.1:5000/v2.0/tokens，因为基本url的后面接的信息为/v2.0，所以将到public_api的section作相应操作。

## 源码分析

###   **Policy策略** 

Policy机制就是用来控制某一个User在某个Tenant中某个操作的权限。对于Keystone服务来说，policy就是一个json文件，默认是/etc/keystone/policy.json。

```json
{
    "admin_required": "role:admin",
    "cloud_admin": "rule:admin_required and domain_id:admin_domain_id",
    "service_role": "role:service",
    "service_or_admin": "rule:admin_required or rule:service_role",
    "owner": "user_id:%(user_id)s or user_id:%(target.token.user_id)s",
    "admin_or_owner": "(rule:admin_required and domain_id:%(target.token.user.domain.id)s) or rule:owner",
    "admin_or_cloud_admin": "rule:admin_required or rule:cloud_admin",
    "admin_and_matching_domain_id": "rule:admin_required and domain_id:%(domain_id)s",
    "service_admin_or_owner": "rule:service_or_admin or rule:owner",
}
```

通过配置这个文件，Keystone Service实现了对User的基于用户角色的权限管理。

### keystone目录下：

（1）每个Keystone 组件，比如 catalog， token 等都有一个单独的目录。

（2）每个组件目录中：

1> routes.py

> 定义了该组件的 routes

其中，identity 和 assignment 分别定义了 admin 和 public 使用的 routes，分别供 admin service 和 public service 使用。 

2> controller.py

文件定义了该组件所管理的对象，比如 assignment 的controller.py 文件定义了 Tenant、Role、Project 等类。

3> core.py 

定义了两个类 Manager 和 Driver。

Manager 类对外提供该组件操作具体对象的方法入口；

Driver 类定义了该组件需要其Driver实现类所提供的接口。

4> backend 目录下的每个文件是每个具体driver 的实现

5> contrib 目录包括resource extension 和 extension resource 的类文件

### admin_api 流水线

在上一章中， 我们讲了WEB服务器的创建及使用。这一章我们将进入Keystone的实际业务认证流程。

我们以admin_api做为一个实例，从Paste Deployment配置文件的流水线开始。

> 处理流程是pipeline = 
>
> access_log
>
>  sizelimit 
>
> url_normalize 
>
> token_auth 
>
> admin_token_auth 
>
> xml_body 
>
> json_body 
>
> ec2_extension 
>
> user_crud_extension 
>
> public_service

- access_log实现对请求的log记录

-  sizelimit  判断request大小，如果大于最大值，则raise Excetion报异常，否则继续向下传递， 把内容读出来放到req.body_file中, 这样body_file就存放了HTTP请求的实际内容 

  ```python
  class RequestBodySizeLimiter(wsgi.Middleware):
      """Limit the size of an incoming request."""
   
      def __init__(self, *args, **kwargs):
          super(RequestBodySizeLimiter, self).__init__(*args, **kwargs)
   
      @webob.dec.wsgify(RequestClass=wsgi.Request)
      def __call__(self, req):
   
          if req.content_length > CONF.max_request_body_size:
              raise exception.RequestTooLarge()
          if req.content_length is None and req.is_body_readable:
              limiter = utils.LimitingReader(req.body_file,
                                             CONF.max_request_body_size)
              req.body_file = limiter
          return self.application
  ```

- url_normalize 这个没有直接在NormalizingFilter内部实现__call__功能，而是重载了process_request函数覆盖其基类Middleware的process_request函数并调用其__call__函数，如下，这个函数的功能应该是重新定义request的路径，使其规范化。 并继续向下调用，

  ```python
  class NormalizingFilter(wsgi.Middleware):
      """Middleware filter to handle URL normalization."""
   
      def process_request(self, request):
          """Normalizes URLs."""
          # Removes a trailing slash from the given path, if any.
          if (len(request.environ['PATH_INFO']) > 1 and
                  request.environ['PATH_INFO'][-1] == '/'):
              request.environ['PATH_INFO'] = request.environ['PATH_INFO'][:-1]
          # Rewrites path to root if no path is given.
          elif not request.environ['PATH_INFO']:
              request.environ['PATH_INFO'] = '/'
  ————————————————————————————————————————————————————————————————————————————————
  @webob.dec.wsgify(RequestClass=Request)
      def __call__(self, request):
          try:
              response = self.process_request(request)
              if response:
                  return response
              response = request.get_response(self.application)
              return self.process_response(request, response)
          except exception.Error as e:
              LOG.warning(e)
              return render_exception(e,
                                      user_locale=request.best_match_language())
          except TypeError as e:
              LOG.exception(e)
              return render_exception(exception.ValidationError(e),
                                      user_locale=request.best_match_language())
          except Exception as e:
              LOG.exception(e)
              return render_exception(exception.UnexpectedError(exception=e),
                                      user_locale=request.best_match_language())
  ```

- token_auth，一样的，处理request的public auth数据 

  ```python
  class TokenAuthMiddleware(wsgi.Middleware):
      def process_request(self, request):
          token = request.headers.get(AUTH_TOKEN_HEADER)
          context = request.environ.get(CONTEXT_ENV, {})
          context['token_id'] = token
          if SUBJECT_TOKEN_HEADER in request.headers:
              context['subject_token_id'] = (
                  request.headers.get(SUBJECT_TOKEN_HEADER))
          request.environ[CONTEXT_ENV] = context
  ```

-  再往下是admin_token_auth,处理admin auth数据 

  ```python
  class AdminTokenAuthMiddleware(wsgi.Middleware):
      """A trivial filter that checks for a pre-defined admin token.
      Sets 'is_admin' to true in the context, expected to be checked by
      methods that are admin-only.
      """
   
      def process_request(self, request):
          token = request.headers.get(AUTH_TOKEN_HEADER)
          context = request.environ.get(CONTEXT_ENV, {})
          context['is_admin'] = (token == CONF.admin_token)
          request.environ[CONTEXT_ENV] = context
  ```

-  然后xml_body，xml to json 

  ```python
  class XmlBodyMiddleware(wsgi.Middleware):
      """De/serializes XML to/from JSON."""
   
      def process_request(self, request):
          """Transform the request from XML to JSON."""
          incoming_xml = 'application/xml' in str(request.content_type)
          if incoming_xml and request.body:
              request.content_type = 'application/json'
              try:
                  request.body = jsonutils.dumps(
                      serializer.from_xml(request.body))
              except Exception:
                  LOG.exception('Serializer failed')
                  e = exception.ValidationError(attribute='valid XML',
                                                target='request body')
                  return wsgi.render_exception(e)
  ```

-  然后 json_body，对post的json格式数据进行解析。联合之前的xml处理，可以得出， 如果不是XML或是JSON的格式。那么Keystone就不会进行处理， 换句话说， 它只处理XML和JSON的类型。  把JSON类型转换出来后，就把结果保存在request.environ[PARAMS_ENV]中， 后面的应该就会更方便的使用。 

  ```python
  class JsonBodyMiddleware(wsgi.Middleware):
      """Middleware to allow method arguments to be passed as serialized JSON.
      Accepting arguments as JSON is useful for accepting data that may be more
      complex than simple primitives.
      In this case we accept it as urlencoded data under the key 'json' as in
      json=<urlencoded_json> but this could be extended to accept raw JSON
      in the POST body.
      Filters out the parameters `self`, `context` and anything beginning with
      an underscore.
      """
      def process_request(self, request):
          # Abort early if we don't have any work to do
          params_json = request.body
          if not params_json:
              return
   
          # Reject unrecognized content types. Empty string indicates
          # the client did not explicitly set the header
          if request.content_type not in ('application/json', ''):
              e = exception.ValidationError(attribute='application/json',
                                            target='Content-Type header')
              return wsgi.render_exception(e)
   
          params_parsed = {}
          try:
              params_parsed = jsonutils.loads(params_json)
          except ValueError:
              e = exception.ValidationError(attribute='valid JSON',
                                            target='request body')
              return wsgi.render_exception(e)
          finally:
              if not params_parsed:
                  params_parsed = {}
   
          params = {}
          for k, v in params_parsed.iteritems():
              if k in ('self', 'context'):
                  continue
              if k.startswith('_'):
                  continue
              params[k] = v
   
          request.environ[PARAMS_ENV] = params
  ```

-  然后ec2_extension，s3_extension，这两个过滤filter实现的是Routes的机制，完成路径的匹配，如果在路径中没有找到，再调用self.application,如果找到，直接返回结果，在这个中间件中， 可以看出它并没有定义process_request和process_response方法。 但是细心的你也应该发现这个中间件的父类也已经不是Middleware了， 而是ExtensionRouter， 这里我们先不管这个类的具体实现。从名字大概可以猜出这个中间件的作用就是Ec2接口增加路由，然后使用ec2_controller来处理EC2风格的认证请求。

  ```python
  class Ec2Extension(wsgi.ExtensionRouter):
      def add_routes(self, mapper):
          ec2_controller = controllers.Ec2Controller()
          # validation
          mapper.connect(
              '/ec2tokens',
              controller=ec2_controller,
              action='authenticate',
              conditions=dict(method=['POST']))
          # crud
          mapper.connect(
              '/users/{user_id}/credentials/OS-EC2',
              controller=ec2_controller,
              action='create_credential',
              conditions=dict(method=['POST']))
          mapper.connect(
              '/users/{user_id}/credentials/OS-EC2',
              controller=ec2_controller,
              action='get_credentials',
              conditions=dict(method=['GET']))
          mapper.connect(
              '/users/{user_id}/credentials/OS-EC2/{credential_id}',
              controller=ec2_controller,
              action='get_credential',
              conditions=dict(method=['GET']))
          mapper.connect(
              '/users/{user_id}/credentials/OS-EC2/{credential_id}',
              controller=ec2_controller,
              action='delete_credential',
              conditions=dict(method=['DELETE']))
  ```

- 如果在这些路径中没有找到，则在其基类ExtensionRouter中继续向下传递，如下所示：mapper.connect(xxxxxxxxxx,contoller=self.application),也实现了factory和上面一样调用自身的__call__，在其基类Router中返回self._router.

  ```python
  class ExtensionRouter(Router):
      """A router that allows extensions to supplement or overwrite routes.
      Expects to be subclassed.
      """
      def __init__(self, application, mapper=None):
          if mapper is None:
              mapper = routes.Mapper()
          self.application = application
          self.add_routes(mapper)
          mapper.connect('{path_info:.*}', controller=self.application)
          super(ExtensionRouter, self).__init__(mapper)
   
      def add_routes(self, mapper):
          pass
   
      @classmethod
      def factory(cls, global_config, **local_config):
          """Used for paste app factories in paste.deploy config files.
          Any local configuration (that is, values under the [filter:APPNAME]
          section of the paste config) will be passed into the `__init__` method
          as kwargs.
          A hypothetical configuration would look like:
              [filter:analytics]
              redis_host = 127.0.0.1
              paste.filter_factory = keystone.analytics:Analytics.factory
          which would result in a call to the `Analytics` class as
              import keystone.analytics
              keystone.analytics.Analytics(app, redis_host='127.0.0.1')
          You could of course re-implement the `factory` method in subclasses,
          but using the kwarg passing it shouldn't be necessary.
          """
          def _factory(app):
              conf = global_config.copy()
              conf.update(local_config)
              return cls(app, **local_config)
          return _factory
  ```

- ec2_extension除了实现上述功能，在其core函数中实现了如下代码：

  ```python
  EXTENSION_DATA = {
      'name': 'OpenStack EC2 API',
      'namespace': 'http://docs.openstack.org/identity/api/ext/'
                   'OS-EC2/v1.0',
      'alias': 'OS-EC2',
      'updated': '2013-07-07T12:00:0-00:00',
      'description': 'OpenStack EC2 Credentials backend.',
      'links': [
          {
              'rel': 'describedby',
              # TODO(ayoung): needs a description
              'type': 'text/html',
              'href': 'https://github.com/openstack/identity-api',
          }
      ]}
  extension.register_admin_extension(EXTENSION_DATA['alias'], EXTENSION_DATA)
  extension.register_public_extension(EXTENSION_DATA['alias'], EXTENSION_DATA)
  ————————————————
  注册这个是有用的，在pipeline的最后一步会遇到。这里先放在着。
  ```

- S3_extension和ec2也是类似的实现了一个S3Extension，S3Controller和一个extension.registerxxxx()，user_crud也是类似。

  最后一步是public_app_factroy，OK，找到他，在service.py下面

  ```python
  @fail_gracefully
  def public_app_factory(global_conf, **local_conf):
      controllers.register_version('v2.0')
      conf = global_conf.copy()
      conf.update(local_conf)
      return wsgi.ComposingRouter(routes.Mapper(),
                                  [identity.routers.Public(),
                                   token.routers.Router(),
                                   routers.VersionV2('public'),
                                   routers.Extension(False)])
  ```

- 配置完成后，返回了ComposingRouter对象，这个类下面做了什么呢，如下 

  ```python
  class ComposingRouter(Router):
      def __init__(self, mapper=None, routers=None):
          if mapper is None:
              mapper = routes.Mapper()
          if routers is None:
              routers = []
          for router in routers:
              #主要就是这一句，routers.add_routers(mapper)，就是调用indentity.routers.Public(), 
              router.add_routes(mapper)
          super(ComposingRouter, self).__init__(mapper)
  ```

  -  token.routers.Router(),routers.VersionV2('public'),routers.Extension(False)类下的add_routes函数，并把mapper作为参数传进去了。继续看，这里注意routers.Extension这个类，跟进去 

  ```python
  class Extension(wsgi.ComposableRouter):
      def __init__(self, is_admin=True):
          if is_admin:
              self.controller = controllers.AdminExtensions()
          else:
              self.controller = controllers.PublicExtensions()
   
      def add_routes(self, mapper):
          extensions_controller = self.controller
          mapper.connect('/extensions',
                         controller=extensions_controller,
                         action='get_extensions_info',
                         conditions=dict(method=['GET']))
          mapper.connect('/extensions/{extension_alias}',
                         controller=extensions_controller,
                         action='get_extension_info',
                         conditions=dict(method=['GET']))
  ```

  -  add_routes实现了Routes，这不奇怪，其他的三个类也是实现了上述功能，关键是他的controller，跟到PublicExtension下面 

  ```python
  class PublicExtensions(Extensions):
      @property
      def extensions(self):
          return extension.PUBLIC_EXTENSIONS
  ___________________
  extension.register_public_extension(EXTENSION_DATA['alias'], EXTENSION_DATA)
  ```

-  就是给PUBLI_EXTENSION赋值，因此，这里指向了前面的EC2Controller,S3Controller,CRUDController。OK，这样就可以根据注册的服务，来完成映射的分发，调用public_api的流程基本完成。 

  ```python
  def register_public_extension(url_prefix, extension_data):
      """Same as register_admin_extension but for public extensions."""
   
      PUBLIC_EXTENSIONS[url_prefix] = extension_data
  ```

  ```python
[filter:crud_extension]
  paste.filter_factory = keystone.contrib.admin_crud:CrudExtension.factory
  ```

- 这个类的作用很明显， 就是给内部对象增加RESTful 风格的CRUD操作路由。 其中有五个控制器(tenant_controller, user_controller, role_controller, service_controller, endpoint_controller)。

  ```python
  class CrudExtension(wsgi.ExtensionRouter):
      """Previously known as the OS-KSADM extension.
  
      Provides a bunch of CRUD operations for internal data types.
  
      """
  
      def add_routes(self, mapper):
          tenant_controller = identity.controllers.Tenant()
          user_controller = identity.controllers.User()
          role_controller = identity.controllers.Role()
          service_controller = catalog.controllers.Service()
          endpoint_controller = catalog.controllers.Endpoint()
  
          # Tenant Operations
          mapper.connect(
              '/tenants',
              controller=tenant_controller,
              action='create_project',
              conditions=dict(method=['POST']))
          mapper.connect(
              '/tenants/{tenant_id}',
              controller=tenant_controller,
              action='update_project',
              conditions=dict(method=['PUT', 'POST']))
          mapper.connect(
              '/tenants/{tenant_id}',
              controller=tenant_controller,
              action='delete_project',
              conditions=dict(method=['DELETE']))
          mapper.connect(
              '/tenants/{tenant_id}/users',
              controller=tenant_controller,
              action='get_project_users',
              conditions=dict(method=['GET']))
  
   ...
  ```

- ```python
  [app:admin_service] paste.app_factory = keystone.service:admin_app_factory 
  ```

  这个factory产生了一个ComposingRouter的对象，这是流水线的最后一级，也是业务的实际处理点。

  ```python
  @fail_gracefully
  def admin_app_factory(global_conf, **local_conf):
      conf = global_conf.copy()
      conf.update(local_conf)
      return wsgi.ComposingRouter(routes.Mapper(),
                                  [identity.routers.Admin(),
                                      token.routers.Router(),
                                      routers.VersionV2('admin'),
                                      routers.Extension()])
  ————————————————
  
  ```

### 路由分发

路由是MVC架构中非常重要的一个关键节点， 可以说， 如果没有路由，就不可能去实现一个很好的MVC架构， 路由也是整个系统中最先处理的节点。如果想弄清楚Keystone的整体架构， 路由是必须先搞清楚的。

看具体实现之前， 我们先看看路由的具体用法。比如说有如下一条路由：

假设Keystone的服务器为127.0.0.1:35357, 那么如果我们访问如下地址时: 
http://127.0.0.1:35357/ec2tokens 
经过路由分发， 代码就会跑到类ec2_controller的authenticate方法中。

```python
class ExtensionRouter(Router):
    def __init__(self, application, mapper=None):
        if mapper is None:
            mapper = routes.Mapper()
        self.application = application
        self.add_routes(mapper)
        mapper.connect('{path_info:.*}', controller=self.application)
        super(ExtensionRouter, self).__init__(mapper)

    def add_routes(self, mapper):
        pass

    @classmethod
    def factory(cls, global_config, **local_config):
        def _factory(app):
            conf = global_config.copy()
            conf.update(local_config)
            return cls(app, **local_config)
        return _factory
```

 在ExtensionRouter的`__init__` 方法中， routes.Mapper创建了一个Mappper对象

 至此，可以看到， 在ExtensionRouter的factory方法中，创建一个ExtensionRouter的一个实例， 并在其它构造方法中，创建Mapper对象，并把所有的路由信息全部增加到Mapper对象中，然后调用其它父类的构造方法。继续看看Router类的构造方法。  

```python
class Router(object):
    def __init__(self, mapper):
        if CONF.debug:
            logging.getLogger('routes.middleware')

        self.map = mapper
        self._router = routes.middleware.RoutesMiddleware(self._dispatch,self.map)

    @webob.dec.wsgify(RequestClass=Request)
    def __call__(self, req):
        return self._router

    @staticmethod
    @webob.dec.wsgify(RequestClass=Request)
    def _dispatch(req):
        match = req.environ['wsgiorg.routing_args'][1]
        if not match:
            return render_exception(
                exception.NotFound(_('The resource could not be found.')),
                user_locale=req.best_match_language())
        app = match['controller']
        return app
```

 在Route类的__init__ 中使用， 使用Router._dispath方法作为一个中间件的应用程序初始化一个中间件。找到这个中间件的实现方法。在[routes.middleware.py](http://pydoc.net/Python/Routes/2.1/routes.middleware/) 中， 代码看起来有点多， 我们只要找关键部分及连接部分。首先这是个可调用的类。在其__call__ 中， 调用其app, 并返回其返回值。 
但是这样，肯定是不够的， 因为在之前的步骤中， 我们把所有的路由信息全部交给了它来处理， 也就是说在这里，我们需要找到URL请求中，所对应的controller及action. 仔细查找，可以看到如下代码：

```python
if self.singleton:
            config = request_config()
            config.mapper = self.mapper
            config.environ = environ
            match = config.mapper_dict
            route = config.route
else:
    results = self.mapper.routematch(environ=environ)
    if results:
        match, route = results[0], results[1]
    else:
        match = route = None
...
url = URLGenerator(self.mapper, environ)
environ['wsgiorg.routing_args'] = ((url), match)
environ['routes.route'] = route
environ['routes.url'] = url
```

 这里有两种方法来查找路由。在Keystone的实现中，是通过`routes.middleware.RoutesMiddleware(self._dispatch,self.map)` 来实现的， 所以self.singleton的值为默认值True. 

  那么这里又通过config = request_config()创建了一个新的对象。继续找到其实现, 在[routes](http://pydoc.net/Python/Routes/2.1/routes/) 的`__init__.py`中， 代码如下： 

```python
def request_config(original=False):
        obj = _RequestConfig()
    try:
        if obj.request_local and original is False:
            return getattr(obj, 'request_local')()
    except AttributeError:
        obj.request_local = False
        obj.using_request_local = False
    return _RequestConfig()
```

 它创建了一个_RequestConfig对象，在同一个文件中， 找到这个类的实现。 

```python
class _RequestConfig(object):
    __shared_state = threading.local()

    def __getattr__(self, name):
        return getattr(self.__shared_state, name)

    def __setattr__(self, name, value):
        """
        If the name is environ, load the wsgi envion with load_wsgi_environ
        and set the environ
        """
        if name == 'environ':
            self.load_wsgi_environ(value)
            return self.__shared_state.__setattr__(name, value)
        return self.__shared_state.__setattr__(name, value)

    def __delattr__(self, name):
        delattr(self.__shared_state, name)

    def load_wsgi_environ(self, environ):
        """
        Load the protocol/server info from the environ and store it.
        Also, match the incoming URL if there's already a mapper, and
        store the resulting match dict in mapper_dict.
        """
        if 'HTTPS' in environ or environ.get('wsgi.url_scheme') == 'https' \
           or environ.get('HTTP_X_FORWARDED_PROTO') == 'https':
            self.__shared_state.protocol = 'https'
        else:
            self.__shared_state.protocol = 'http'
        try:
            self.mapper.environ = environ
        except AttributeError:
            pass

        # Wrap in try/except as common case is that there is a mapper
        # attached to self
        try:
            if 'PATH_INFO' in environ:
                mapper = self.mapper
                path = environ['PATH_INFO']
                result = mapper.routematch(path)
                if result is not None:
                    self.__shared_state.mapper_dict = result[0]
                    self.__shared_state.route = result[1]
                else:
                    self.__shared_state.mapper_dict = None
                    self.__shared_state.route = None
        except AttributeError:
            pass

        if 'HTTP_X_FORWARDED_HOST' in environ:
            # Apache will add multiple comma separated values to
            # X-Forwarded-Host if there are multiple reverse proxies
            self.__shared_state.host = \
                environ['HTTP_X_FORWARDED_HOST'].split(', ', 1)[0]
        elif 'HTTP_HOST' in environ:
            self.__shared_state.host = environ['HTTP_HOST']
        else:
            self.__shared_state.host = environ['SERVER_NAME']
            if environ['wsgi.url_scheme'] == 'https':
                if environ['SERVER_PORT'] != '443':
                    self.__shared_state.host += ':' + environ['SERVER_PORT']
            else:
                if environ['SERVER_PORT'] != '80':
                    self.__shared_state.host += ':' + environ['SERVER_PORT']
```



