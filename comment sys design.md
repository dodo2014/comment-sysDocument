





















[TOC]

# 评论系统设计文档

## 1、目标

实现通用的评论管理方案，为各个不同的社交平台提供通用评论接口，实现第三方的评论管理。

- 提升评论概率，提升web、移动断的用户体验

- 解决审核问题，提供稳定、低成本的评论管理功能

## 2、系统设计

评论系统采用B/S结构，提供社交平台接入的API数据接口，支持web、移动客户端。
评论系统包括：评论过滤、评论内容管理、用户管理(预留)、评论审计


## 3、实现策略

### 3.1结构体系
B/S 结构

### 3.2数据存储
系统数据存储采用couchbase集群和redis集群

## 4、运行环境
centOS、nginx、tornado

## 5、数据模型
`业务平台根据统计，暂定为三类，分别为：电子商务类，社交信息类，新闻资讯类`

**Platrorm: 接入业务平台**
```
appid : 分配给接入平台的appid
domain : 平台域名
apptoken: 平台校验token
```
_ _ _

**User: 发表评论用户**
```
userId : 用户id
userName : 用户名称
```
_ _ _

**Topic: 话题**
不同平台，由于业务不同，所以可以发表评论对象的类型有区别。评论对象统一抽象为topic(暂定，只是概念上的定义)。
topic可以存储原有topic的简要信息
topic有分类，分类之间具有层级关系
topic的分类暂定为：10000：电子商务类、20000：社交信息类、30000：新闻资讯类
	topic的类型，暂定三大类，如果有扩展需要，可以在各自分类下增加字段定义，例如电子商务类1001下扩展微商10001、商城10002等，新闻资讯30000扩展为汽车板块30001、时事板块30002等

```
domain : 所属的业务平台
topictId ：评论对象在原有平台的id
topicUuid : 评论对象uuid
block : 板块。话题所属的板块(扩展字段)
subject ：专题。话题所属的专题(扩展字段)
ttype ： 分类，10000：电子商务类、20000：社交信息类、30000：新闻资讯类
parentType : 父分类. 默认-1，即type为三大分类
detail : 话题简要内容
likes : 喜欢/赞 计数
comments : 评论 计数
```
_ _ _

**Comment: 评论**
评论的回复只提供一个层级，不提供多级回复。
不同大类topic下的comment数据结构有不同的字段
```
topicUuid : 评论对象uuid
commentUuid : 评论uuid
userId : 发表评论的用户id
anonymous ： 匿名评价。0：匿名，1：不匿名
status : 评论状态，0:正常，1:异常, 异常包括：被举报 等情况
value_level : （商品用）评价等级，0：全部，1：好评，2：中评，3：差评
	商品评价等级在评论中显示为星级，1星为差评，2、3星为中评，4、5星为好评
parentId : 父id，默认值 -1
releaseTime : 评论时间
detail : 评论内容--文本内容，包括文本，图片，链接等
likes : 喜欢/赞/有用 计数
unlikes : 踩 计数
comments : 评论 计数 (被评论的数量)
```
_ _ _

**Operation: 评论操作**
用户对某条评论的操作，包括：喜欢/赞、评分、举报
```
topicUuid : topicUuid
commentUuid : 评论uuid
userId ：发起操作的userid
operatedUserId : 评论对象的用户id
operateType : 操作类型， 0：喜欢/赞/顶，1：踩，2：评分，3：举报
operateContent ：操作的具体内容。
	- 操作类型为0, 字段保存值0；
	- 操作类型为1，字段保存值1；
	- 操作类型为2，字段保存内容{"score":0,"reason":"很给力！"}；
	- 操作类型为3，字段保存内容{"reason":"其他"}

```

report:举报
```
reason: 举报理由。 1、色情淫秽 2、骚扰谩骂 3、恶意广告 4、反动言论 5、其他
```

score:评分
```
score: 评分
reason : 评分理由、默认为空。1、很给力！ 2、赞一个！ 3、淡定 4、神马都是浮云 5、山寨、6、自定义
```
_ _ _

**conf: 评论设置**

针对不同的平台、提供不同的内容设置。暂时根据不同属性的业务平台，提供不同的默认设置。  

*平铺模式* `平铺模式下，所有评论同级展示，如果评论a是对评论b的评论，则评论a 中显示 b 的简要信息。a不允许被评论`  
*嵌套模式* `嵌套模式下，第一级展示第一级评论，点击显示第二级评论，第二级不允许被再次评论`  
*盖楼模式* `以盖楼模式显示评论内容`  
```
displayMode : 显示方式
	0：平铺模式。 
    1：嵌套模式，最多嵌套 1 层
    2：盖楼模式
checkRule : 审核规则
	0：默认全部通过
    1：全部人工审核(预留)
anonymous ： 允许匿名
	0：允许匿名
    1：不允许匿名
filter : 评论内容过滤
	0：过滤
    1：不过滤
banpicture : 禁止发图片
	0: 禁止
    1：不禁止
banexpression : 禁止发表情
	0：禁止
    1：不禁止
bandown : 禁止踩
	0: 禁止
    1: 不禁止
```

## 6、系统集成架构

### 6.1 评论列表服务  
向平台提供评论的展示、操作等服务  
####6.1.1 获取文章(评论对象)评论数 接口  

**接口名称**  
/topic/counts  
**接口说明**  
获取当前文章的评论数

**URL**

http://comments.qbao.com/topic/counts

**HTTP请求方式**

GET

**请求参数**  
`domain`  `string` `not null` 分配给平台的二级域名。  
`appid`   `string` `not null` 分配给接入平台的appid， 用于接入校验  
`topics` `string` `not null` 需要获取的文章的id，即评论对象在接入平台中的id，可以传入多个id,用逗号分割，例如：1,2,3(或者4ff160411318693231000006, 4ff160411318693231000007)

**请求示例**  
http://comments.qbao.com/topic/counts?domain=microStore&appid=TqJyenJSN8T4N4Er41Kei4G2pULzQJSbZVZjXTzF&topics=1,2,3

**返回数据实例**
```
{
    "result": [
        {
            "topicId": 1, // 接入平台的topicid
            "topicUuid": "TqJyenJSN8T4N4Er41Kei4G2pULzQJSbZVZjXTzF", // 评论系统中的topic uuid
            "comments": 389, // 评论数
            "likes": 3, // 喜欢数
        }
    ]
}
```
####6.1.2 获取设置 接口
**接口名称**  
/comment/conf
**接口说明**  
获取评论设置。不同类型的平台，有不同的配置。先期不允许自定义配置项，配置项由评论系统提供默认值。暂定三种类型的平台，各自的配置不相同。  
**URL**  
http://comments.qbao.com/comment/conf  
**HTTP请求方式**  
GET  
**请求参数**  
`domain` `string` `not null` 分配给平台的二级域名.  
`appid` `string` `not null` 分配给接入平台的appid.  
**请求示例**  
**返回数据示例**  
**1. 电子商务类 返回数据示例**  
```
{
    "display_mode" : 0,//显示方式，0：平铺模式，1：嵌套模式，最多嵌套1层，2：盖楼模式
    "check_rule" : 0, //审核规则，0：通过，1：人工审核。默认全部通过
    "anonymous" : 0, //是否允许匿名，0：允许匿名，1：不允许匿名。
    "filter" : 0, //评论内容过滤，0：过滤，1：不过滤。默认过滤
    "banpicture" : 1, //禁止发图片，0：禁止，1：不禁止
    "banexpression" : 0, //禁止发表情，0：禁止，1：不禁止
    "bandowm" : 0, //禁止踩， 0：禁止，1：不禁止
}
```
**2. 社交信息类 返回数据示例**  
```
{
    "display_mode" : 0,//显示方式，0：平铺模式，1：嵌套模式，最多嵌套1层，2：盖楼模式
    "check_rule" : 0, //审核规则，0：通过，1：人工审核。默认全部通过
    "anonymous" : 0, //是否允许匿名，0：允许匿名，1：不允许匿名。
    "filter" : 0, //评论内容过滤，0：过滤，1：不过滤。默认过滤
    "banpicture" : 1, //禁止发图片，0：禁止，1：不禁止
    "banexpression" : 1, //禁止发表情，0：禁止，1：不禁止
    "bandowm" : 1, //禁止踩， 0：禁止，1：不禁止
}
```
**3. 新闻资讯类 返回数据示例**  
```
{
    "display_mode" : 1,//显示方式，0：平铺模式，1：嵌套模式，最多嵌套1层，2：盖楼模式
    "check_rule" : 0, //审核规则，0：通过，1：人工审核。默认全部通过
    "anonymous" : 0, //是否允许匿名，0：允许匿名，1：不允许匿名。
    "filter" : 0, //评论内容过滤，0：过滤，1：不过滤。默认过滤
    "banpicture" : 0, //禁止发图片，0：禁止，1：不禁止
    "banexpression" : 0, //禁止发表情，0：禁止，1：不禁止
    "bandowm" : 1, //禁止踩， 0：禁止，1：不禁止
}
```
**返回数据参数说明**  
详见返回数据示例中的字段注释

####6.1.3 获取文章(评论对象)评论 接口
**接口名称**  
/comment/list  
**接口说明**  
获取文章(评论对象)的评论列表信息。  
  评论内容的返回值，与业务平台的分类对应，分为三类。不同业务平台评论内容的字段定义不同

**URL**  
http://comments.qbao.com/comment/list  
**HTTP请求方式**  
GET  
**请求参数**  
`domain`  `string` `not null` 分配给平台的二级域名。  
`appid` `string` `not null` 分配给接入平台的appid.  
`topicid` `string` `not null` 文章的id  
`limit` `int` `null` 每页显示条数，默认20  
`page_no` `int` `null` 当前页数，默认0  
**请求示例**  

**返回数据示例**  

**1. 电子商务类返回数据示例**
```
{
    "topicid" : 324, //id
    "comment_count" : 5432, //评论计数
    //此处的 likes, 商品类表示为好评率, 不返回，app端自行计算
    "good_raps" : 984, // 好评
    "mid_raps" : 35, //中评
    "bad_raps" : 8, //差评
    "pic_raps" : 53, //有图
    "comments" : [ //评论内容列表
        {
            "comment" : { //评论
                "commentUuid" : "TqJyenJSN8T4N4Er41Kei4G2pULzQJSbZVZjXTzF", //评论uuid
                "status" : 0, //评论状态, 0:正常，1:非正常。商品status默认为0
                "value_level" : 1, //评价等级，1：好评，2：中评，3：差评。默认 1
                "releaseTime" : "2014-04-21 08:05:00", //发表时间
                "detail" : "hello world", //评论具体内容
                "likes" : 5, //有用计数
                "comments" : 0, //回复计数
                /**
                "children" : [
                    {
                        "userid" : "3214",
                        "user_name" : "张三",
                        "releaseTime" : "2015-05-04 10:59:00",
                        "commentuuid" : "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXX",
                        "detail" : "................",
                    },
                ],
                */
            },
            "user_info" : { //用户信息
                "userId" : 324, //用户id
                "userName" : "tom", //用户名称
                "userlevel" : 3, //用户等级，比如普通用户，铁牌用户，银牌用户，金牌用户等
            }
        },
        {}
    ]
}
```
**返回数据参数说明**
*topicid* `话题id`  
*comment_count* `话题被评论的次数，即评论计数`  
*good_raps* `好评计数`  
*mid_raps* `中评计数`  
*bad_raps* `差评计数`  
*pic_raps* `有图片的评论计数`  
*comment-* *commentUuid* `评论的uuid`  
*comment-* *status* `评论的状态。0：正常，1：非正常。 电商评论的状态，默认为0，没有其他操作`  
*comment-* *value_level* `评论等级。1：好评，2：中评，3：差评。默认好评1`  
*comment-* *releaseTime* `评论发表的时间`  
*comment-* *detail* `评论具体内容`  
*comment-* *likes* `评论的有用计数`  
*comment-* *comments* `评论的回复计数`  
  
*children* `评论的回复。平铺模式下，回复不直接显示在评论列表中，通过点击‘回复’获取评论的回复内容列表。`  
*children-* *userid* `回复的用户id`  
*children-* *user_name* `回复的用户名称`  
*children-* *releaseTime* `回复发表的时间`  
*children-* *commentuuid* `回复的uuid`  
*children-* *detail* `回复的内容`  
  
*user_info* `发表评论的用户信息`  
*user_info-* *userId* `发表评论的用户id`  
*user_info-* *userName* `发表评论的用户名称`  
*user_info-* *userlevel* `(预留字段)发表评论的用户等级。电商平台的用户等级，比如普通用户，铁牌用户，银牌用户，金牌用户等。这个字段等级的定义以及具体用途暂时未定，属于预留字段`  

**2. 社交信息类返回数据示例**
```
{
    "topicid" : 324, //文章id
    "comment_count" : 5432, //评论计数
    "likes" : 2, //喜欢(赞) 计数
    "comments" : [ //评论内容列表
        {
            "comment" : { //评论
                "commentUuid" : "TqJyenJSN8T4N4Er41Kei4G2pULzQJSbZVZjXTzF", //评论uuid
                "status" : 0, //评论状态, 0:正常，1:非正常。 默认为1
                "releaseTime" : "2014-04-21 08:05:00", //发表时间
                "detail" : "hello world", //评论具体内容
                "likes" : 5, //喜欢(赞、顶)计数
                "unlikes" : 0, //踩 计数
                "comments" : 0, //评论计数
                //parent_info信息预留接口
                "parent_info" : { //父评论信息
                    "parentId" : "TqJyenJSN8T4N4Er41Kei4G2pULzQJSbZVZjXTzF",
                    "userName" : "jack",
                    "releaseTime" : "2014-04-20 08:05:00",
                    "detail" : "天天向上..." //父评论的详细信息会截取部分长度
                    },
                "children" : [
                    {
                        "releaseTime" : "2015-05-04 10:59:00",
                        "userid" : "",
                        "user_name" : "",
                        "commentuuid" : "",
                        "detail" : "",

                        "likes" : 0, //顶 计数
                        "unlikes" : 0, //踩 计数
                        "comments" : 0 //评论 计数
                    },
                ],
            },
            "user_info" : { //用户信息
                "userId" : 324, //用户id
                "userName" : "tom", //用户名称
                "userlevel" : "3" //用户等级，例如普通用户、vip用户，皇冠用户等
            }
        }
    ]
}
```
**返回数据参数说明**
*topicid* `话题id`  
*comment_count* `话题被评论的次数，即评论计数`  
*likes* `喜欢/赞 计数。 话题下评论的喜欢/赞 的计数，并入话题的喜欢/赞 计数`  
*comment-* *commentUuid* `评论的uuid`  
*comment-* *status* `评论的状态。0：正常，1：非正常。 默认为0，如果被举报，则状态变为1。踩 操作，不改变此字段的值`  
*comment-* *value_level* `评论等级。1：好评，2：中评，3：差评。社交信息此字段默认好评1，没有其他值`  
*comment-* *releaseTime* `评论发表的时间`  
*comment-* *detail* `评论具体内容`  
*comment-* *likes* `评论的 喜欢/赞 计数`  
*comment-* *unlikes* `评论的 踩 计数`  
*comment-* *comments* `评论的回复计数`  

*parent_info* `评论的父信息`  
*parent_info-* *parentId* `父评论的uuid`  
*parent_info-* *userName* `发表父评论的username`  
*parent_info-* *releaseTime* `父评论发表的日期`  
*parent_info-* *detail* `父评论的内容`  

*children* `评论的回复。平铺模式下，回复不直接显示在评论列表中，通过点击 评论内容框 获取评论的回复内容列表`  
*children-* *userid* `回复的用户id`  
*children-* *user_name* `回复的用户名称`  
*children-* *releaseTime* `回复发表的时间`  
*children-* *commentuuid* `回复的uuid`  
*children-* *detail* `回复的内容`  
*children-* *likes* `回复被点赞的计数`  
*children-* *unlikes* `回复被踩的计数`  
*children-* *comments* `被回复的计数`  

*user_info* `发表评论的用户信息`  
*user_info-* *userId* `发表评论的用户id`  
*user_info-* *userName* `发表评论的用户名称`  
*user_info-* *userlevel* `(预留字段)发表评论的用户等级。社交平台的用户等级，比如普通用户，vip用户，皇冠用户，大V用户等。这个字段等级的定义以及具体用途暂时未定，属于预留字段`  


**3. 新闻资讯类返回数据示例**
```
{
    "topicid" : 324, //文章id
    "comment_count" : 5432, //评论计数
    "likes" : 2, //喜欢(赞) 计数
    "hotcomments" : [//热门评论
        //内容结构与comments一样
    ],
    "comments" : [ //评论内容列表
        {
            "comment" : { //评论
                "commentUuid" : "TqJyenJSN8T4N4Er41Kei4G2pULzQJSbZVZjXTzF", //评论uuid
                "status" : 0, //评论状态, 0:正常，1:非正常
                "releaseTime" : "2014-04-21 08:05:00", //发表时间
                "detail" : "hello world", //评论具体内容
                "likes" : 5, //喜欢(赞、顶)计数
                "comments" : 0, //评论计数
                "parent_info" : { //父评论信息
                    "parentId" : "TqJyenJSN8T4N4Er41Kei4G2pULzQJSbZVZjXTzF",
                    "userName" : "jack",
                    "releaseTime" : "2014-04-20 08:05:00",
                    "detail" : "天天向上..." //父评论的详细信息会截取部分长度
                    },
                "children" : [
                    {
                        "status" : 0,
                        "releaseTime" : "2015-05-04 10:59:00",
                        "commentuuid" : "",
                        "detail" : "",
                        "parent" : "",
                        "userid" : "",
                        "user_name" : "",
                        "likes" : "",
                        "unlikes" : 0,
                        "comments" : 0
                    },
                ],

            },
            "user_info" : { //用户信息
                "userId" : 324, //用户id
                "userName" : "tom", //用户名称
                "userlevel" : "3"
            }
        }
    ]
}
```
**返回数据参数说明**


####6.1.4 发表评论 接口
**接口名称**  
/comment/submit  
**接口说明**  
发表评论, 评论目标分为两类：topic，comment  
topic的评论数+1。非盖楼模式下，不允许二次评论  
**URL**  
http://comments.qbao.com/comment/submit  
**HTTP请求方式**  
POST  
**请求参数**  
`domain`  `string` `not null` 分配给平台的二级域名。  
`appid` `string` `not null` 分配给平台的appid  
`topictId` `long` `not null` 话题id  
`detail` `string` `not null` 评论内容  
`userId` `long`	`not null` 发表评论的用户id  
`parentId` `string` `null` 被评论的comment的uuid, 如果被评论的目标是一个comment, 则传入commentuuid, 默认为空。  
`userName` `string` `null` 发表评论个用户名或者昵称  


**请求示例**  
**返回数据示例**  
{"commentUuid" : "5ac09eef1e3141958fd3618410f5cd96"}  
**返回数据参数说明**  
`commentUuid` `string` 评论的id  

####6.1.5 顶赞喜欢/踩 接口
**接口名称**  
/comment/action  
**接口说明**  
对评论内容做 （顶/赞/喜欢）,踩 等操作，后继可扩展.  
是否允许 踩 操作，参看各平台的默认配置.  
**URL**  
http://comments.qbao.com/comment/action  
**HTTP请求方式**  
POST
**请求参数**  
`domain`  `string` `not null` 分配给平台的二级域名。  
`appid` `string` `not null` 分配给平台的appid  
`topictid` `long` `not null` 话题id  
`commentuuid` `string` `not null` 评论uuid  
`userId` `string` `not null` 发表动作的用户id  
`action_type` `int` `not null` 操作的类型。0:顶/赞/喜欢， 1:踩，  
**请求示例**  
**返回数据示例**  
**返回数据参数说明**  

####6.1.6 举报 接口
**接口名称**  
/comment/report  
**接口说明**  
举报接口  
**URL**  
http://comments.qbao.com/comment/report  
**HTTP请求方式**  
POST  
**请求参数**  
`domain`  `string` `not null` 分配给平台的二级域名。  
`appid` `string` `not null` 分配给平台的appid  
`topictId` `long` `not null` 话题id  
`commentUuid` `string` `not null` 评论uuid  
`userId` `string` `not null` 发起举报的用户id  
`reportedUserId` `string` `not null` 被举报评论的用户id  
`reason` `string` `not null` 举报理由  
**请求示例**  
**返回数据示例**  
**返回数据参数说明**  

### 6.2 统计服务
*统计服务详细设计有待完善*  

#### 6.2.1 统计单位时间评论趋势 接口
**接口名称**  
/comment/countInPeriods  
**接口说明**  
获取单位时间的评论数， 统计指标：评论总数，评论人数  
**URL**  
http://comments.qbao.com/comment/countInPeriods  
**HTTP请求方式**  
GET
**请求参数**  
`domain`  `string` `not null` 分配给平台的二级域名。  
`appid` `string` `not null` 分配给平台的appid  
`starttime` `time` `not null` 起始时间  
`endtime` `time` `null` 结束时间。结束时间可为空，如果为空，则统计starttime当天的24小时的趋势  

**请求示例**  
**返回数据示例**  
```
{"time1" ： 1, "time2" : 2,...}
```

**返回数据参数说明**

#### 6.2.2 统计热门文章 接口
**接口名称**  
/comment/countHotTopic  
**接口说明**  
统计热门文章  
**URL**  
http://comments.qbao.com/comment/countHotTopic  
**HTTP请求方式**  
GET  
**请求参数**  
`domain`  `string` `not null` 分配给平台的二级域名。  
`appid` `string` `not null` 分配给平台的appid  
`date` `time` `not null` 日期  
`limit` `int` `null` 每页显示条数， 默认20  
`page_no` `int` `null` 当前页，默认0  
**请求示例**  
**返回数据示例**
```
{
    "page_no" : 0,
    "result" : [
        {
            "topicUuid" : "TqJyenJSN8T4N4Er41Kei4G2pULzQJSbZVZjXTzF",
            "count" : 434
        },
        {}
    ]
}
```
**返回数据参数说明**

###6.3 后继开发工作
后继开发工作规划，包括评论管理(评论审核、删除、标记)、配置项自定义、统计功能完善、用户管理(用户登陆、用户信息编辑、权限管理)