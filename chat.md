## 消息中心客户端接入指南

### 设计

消息的产生源头分为以下几类：点赞（投稿或者评论）、收藏（投稿）、评论/回复、聊天消息。
消息类型定义如下：
- 1-点赞  
- 2-收藏
- 3-评论和@
- 4-关注 
- 5-系统通知 
- 6-聊天消息

消息包含触发者和接收者，触发者是发起该消息的用户，接收者是该消息需要通知到的用户。

比如，用户点赞投稿的行为中，点赞者是消息的触发者，而被点赞的投稿的作者就是该点赞消息的接收者。
又比如，在聊天场景中，发起聊天消息的用户为触发者，而消息发送对象就是接收者。

某一条消息对于接收者来说，有一个已读和未读的状态，当接收者读取该消息后，此消息对于接收者来说是已读了。App的推送以及消息中心的小红点数字，代表的是用户未读的消息。

当用户打开APP的消息中心界面，一般会展示如下信息：

![image](https://user-images.githubusercontent.com/1658418/128816411-3bf4347c-77a4-4541-8fc0-9e8d2b471811.png)

这里面会涉及到消息的已读和未读问题。一般APP端会存储已经接收到的消息，每次APP打开，只需要查询未读消息即可。

在我们的设计中，消息已读未读并非通过消息本身设置状态字段来表示，而是通过服务器记录用户读取某类消息的最新的时间戳来判断：当某条消息的产生时间晚于用户对此类消息的最新读取时间，则认为该消息对于某个用户来说是未读的状态。

用户读取最新一条消息的时间戳由客户端主动上报。

读取最新消息时间戳的上报时机：
- 点赞、收藏、评论和@场景：当用户点开点赞消息列表页面时，上报最新一条点赞的发生时间作为已读时间戳进行上报。收藏和评论类似。
- 聊天消息场景：
  - 当用户从其他界面进入到跟某个朋友的聊天界面，也就是用户进入聊天会话页时，上报该会话最新一条聊天消息的时间戳作为该会话的已读时间。
  - 当用户一直处于聊天会话中，可以根据性能要求，有两种做法：1）每次收到一条最新的消息，即时上报该消息的时间戳作为该会话的已读时间； 2）每隔固的时间判断一下是否距离上一次上报是否有新的消息，如果有，则发送最新消息的时间戳作为已读时间进行上报。

#### 消息接收机制
- 对于点赞、收藏、评论等非聊天消息，一律通过APP外推送，触达用户；
- 对于聊天消息，区分场景对待：
  - 如果用户不在聊天会话中，客户端无须连接聊天服务器(websocket)，聊天消息通过APP外部推送触达。
  - 如果用户界面在聊天会话中，客户端需要连接聊天服务器进行实时消息通讯，聊天消息通过web socket协议应用内实时送达。

### 核心接口和流程

#### 1. 点开我的消息界面，查询我的消息中心概览
GET /inbox/overview
查询我的消息中心概览，其中包含点赞数、收藏数、评论@数等统计，以及聊天会话列表。其中聊天会话列表中会显示最新的未读消息数。
返回结果案例：
```
{
    "totalNum": 8,
    "likeNum": 0,
    "commentNum": 7,
    "collectNum": 0,
    "chatSessions": [
        {
            "freshNum": 0,
            "userName": "アリア",
            "userAvatar": "https://tva1.sinaimg.cn/large/008i3skNgy1gqs8lrfn4pj303m03e749.jpg",
            "app": "pet",
            "uids": [
                2,
                2
            ],
            "createTime": 1627980891258,
            "id": "pet_2_2",
            "otherUid": 2
        },
        {
            "freshNum": 1,
            "freshMessage": {
                "app": "pet",
                "source": 3,
                "target": 2,
                "content": "msg3->2 you're welcom",
                "sendTime": 1628569459413,
                "session": "pet_2_3",
                "uids": [
                    2,
                    3
                ],
                "id": "X95NLnsBnYPJhva72kXa"
            },
            "userName": "コーラ好きメガネ",
            "userAvatar": "https://tva1.sinaimg.cn/large/008i3skNgy1gqs8oycjz2j304u040q2v.jpg",
            "app": "pet",
            "uids": [
                2,
                3
            ],
            "createTime": 1628569459413,
            "id": "pet_2_3",
            "otherUid": 3
        }
    ]
}
```

#### 2. 点击某类消息或者某个聊天会话，查询消息列表

当用于点击到某个消息类别（点赞、收藏、评论@)或者点击某个聊天会话，需要拉取该消息类别或者聊天会话中的消息列表（按时间倒序排序）。
GET /inbox/messages?type=<type>
当type=1,2,3（点赞、收藏、评论）时，请求为：
```
GET /inbox/messages?type=1&fresh=true
```
当type=6时，是查询聊天消息，则需要指定聊天会话id以及聊天对象的id，这两个字段在`GET /inbox/overview`接口中会返回，参见上述案例，其中chatSessions[x].id和chatSessions[x].otherUid分别表示聊天会话id和聊天对象id。作为消息查询的`chatSessionId`和`triggerUid`两个参数，此时查询请求为：
```
GET /inbox/messages?type=6&chatSessionId=pet_2_3&triggerUid=3&fresh=true
```
注意，参数`fresh`代表是否返回未读消息。

当用户进入到消息列表，表示用户已读，需要向服务器提交最新一条消息的时间戳作为已读时间：
```
PUT /inbox/state 
{
    "type":3,
    "readTime":1628303111086,
    "sourceUid":3
}
```
其中`sourceUid`是消息触发者，只有在聊天消息的已读设置请求中才需要填，点赞、收藏、评论等消息不需要传。

对于APP端，需要本地存储消息会话列表和消息列表，每次从服务器可以只拉取最新的消息即可。每次查询消息列表时，通过`fresh=true`参数实现。
如果APP端不保存任何消息，那么每次可以拉取最新的N条消息，并通过翻页参数来实现翻页。
  
#### 3. 当用户点进入聊天会话，连接聊天服务器

当用户点击进入某个聊天会话，此时需要连接聊天服务，接入方式参见下文聊天服务接入指南
当用户切换界面到非聊天界面，此时可以根据需要断开跟聊天服务器的连接以减少不必要的系统开销。



### 聊天服务接入指南(chat-service)
聊天服务对外启动两个服务，一个web服务和一个web socket，两个服务监听同一个端口。

- Web服务: Web服务提供聊天会话(`ChatSession`)和聊天消息(`ChatMessage`)的查询接口，具体参见swagger-ui的API文档定义。该Web服务不直接提供对给APP调用，仅用于给pet-service提供会话和消息的查询接口。
- Web Socket服务: Web Socket服务提供端到端的实时双向消息传输。

#### 客户端连接聊天服务

##### 浏览器中接入方式

```
<script src="${serverAddress}/socket.io/socket.io.js"></script>
<script>
  const socket = io("${serverAddress}", {
    auth: {
      token: authtoken
    }
  });
</script>
```
注意：
- 本地启动`serverAddress`为`http://127.0.0.1:10003`, 线上为`https://api-dev.nga-x.com/`。
- `authtoken`跟APP调用后台接口的统一登录的token (跟访问后台接口的HTTP header中的`authorization`值相同)

##### Nodejs接入方式

```
const io = require("socket.io-client");
// or with import syntax
// import { io } from "socket.io-client";

const socket = io("${serverAddress}", {
  auth: {
    token: authtoken
  }
});
```

##### 其他语言客户端接入方式

参见：https://socket.io/docs/v4

#### 客户端监听并处理接收消息

聊天服务采用Web Socket协议，支持双向通信，注册完监听之后能收到从服务转发过来的接收消息。
```
socket.on('msg', function(msg) {
  //处理msg
});
```
接收到的消息格式如下：
```
{
  "app":"pet",   //应用标识
  "source":2,   //发送者id
  "target":3,   //接收者id
  "content":"hello", //消息内容
  "sendTime":1628397863789,  //发送时间
  "session":"pet_2_3", //会话
  "uids":[2,3]  //消息收发用户id列表
}
```
#### 客户端发送消息
```
socket.emit('msg', {app: 'pet', target: 3, content:'hello'});
```
发送的消息格式如下：
```
{
    "app":"pet",  //应用标识
    "target":3,   //消息接收者的uid
    "content":"hello", //消息内容
}
```