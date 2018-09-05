# Windows SDK代码

## 核心对象

### SDKData

成员：

1. dnsHost （服务域名）
2. databasePath \(本地数据库路径）
3. appKey（应用AK）
4. userAcc（账户）
5. loginTicket \(App签名）
6. genTs\(签名时间）
7. genNonce\(签名随机串\)
8. onlineStatus\(在线状态\)
9. uid
10. token（这个是什么？）
11. chatSignUrl （这个是什么？）

### SDKMgr \(API入口，设置回调函数，发起Call调用\)

### AccountMgr

成员：账户到UID的映射关系\(map\) :  map&lt;std::string, uint64\_t&gt; m\_acc2Uid

消息响应处理映射：

1. OnGetUidByAccsRes （根据账户获取UID\)
2. OnGetAccByUidRes \(根据UID获取账户）

功能

1. GetUidByAcc \(根据uid获取账户信息：内存，数据库\)
2. GetAccByUid \(根据账户获取uid信息：内存，数据库\)
3. GetAccByUidTNet \(根据ui从服务端获取账户信息\)
4. GetUidByAccTNet\(根据账户从服务端获取uid）
5. \_GetUidByDataBase（查询数据库，获取uid）
6. \_GetAccByDataBase（查询数据库，获取账户）
7.  StoreAcctoUid （保存账户和uid的映射关系，并持久化）
8. OnGetUidByAccsRes （获取到uid信息，触发Chat去发送消息）
9. OnGetAccByUidRes（获取到账户信息，触发消息接收到了通知ui）

### LoginMgr

成员：

1. 登陆状态（connecting, connected, logining, Annoy\_Logined, TicketAuthen\_Logined, CIM\_Logined, LoginFailed, Disconnect, Invalid\_LoginState\)
2. 连接信息：当前连接，连接池
3. 连接定时器：定期发起连接（异步连接）
4. PingAp定时器 ： 定期发起Ping AP操作。
5. 异步任务线程：

消息响应映射：

1. OnApLoginRes （登陆成功消息处理）
2. OnAppTicketAuthentificationRes（带签名登陆成功消息处理\)
3. OnPAPRouter
4. OnAPPing （Ping AP成功消息处理\)

功能：

1. 登陆（ThirdPartLogin, ThirdPartReLogin, Login\)
   1. 通过LBSLinkMgr2 获取AP信息（异步）
   2. 建立连接（异步）
2. 连接
   1. CreateTcpLink 
   2. onConnected
   3. OnNetEvent
   4. OnConnectedNetEvent
   5. SendAnonyLoginAPRequest （ 发起匿名登陆\)
3. 匿名登陆
   1. 创建PCS\_APLogin（账户，AppKey，消息ID）
   2. 添加到消息重发队列
   3. 发送消息\(SendPacket ： 异步\)
   4. OnApLoginRes \(匿名登陆成功，发起签名登陆\)
4. 签名登陆
   1. 创建PCS\_AppTicketLogin（账户，ticket）
   2. 添加到消息重发队列
   3. 发送消息\(SendPacket : 异步）
   4. OnApLoginRes \(签名登陆成功：设置token和uid，发起正常登陆\)
5. 正常登陆
   1. 创建PCS\_APLogin（账户，uid，AppKey，AppToken，消息ID）
   2. 添加到消息重发队列
   3. 发送消息（SendPacket：异步）
   4. OnApLoginRes\(正常登陆成功，产生windows IM事件，同时UI线程，初始化登陆环境：发起定时PingAP，拉取离线消息\)

### ImChat

   消息签名：SP2PMsgSignInfo \(签名的时间戳，签名的随机串，签名\)

   消息：SP2PMsgInfo \(消息类型，消息内容，消息ID，消息的接收者账号\)

   成员:

       1\) m\_acc2MsgSignInfo : 消息的签名信息\(map\) : 管理了账号到消息签名的映射关系，存储在本地数据库中。

       2\) m\_acc2ToSendMsg : 正在发送的消息\(map\) : 管理了账号到发送的消息的映射关系

       3\) m\_msgId2Info : 消息信息\(map\) : 管理了消息ID到正在发送的消息的映射关系

       4\) m\_acc2Uid : 账号信息\(map\) : 管理了账号到uid的映射关系

       5\) m\_sourceId2PushSeq : 离线消息推送进度\(map\) : 管理了数据源机房ID到离线消息推送进展的映射关系

       6\) m\_msgId2PullState : 推送消息的拉取状态\(map\) : 管理了消息ID到消息拉取状态的映射关系

       7\) m\_fromUid2Msgs :

       8\) m\_pTaskThread : 异步任务处理线程 ：

   RPC消息处理

       1\) OnSendIMCloudCheckAppSignatureRes : 检查应用程序签名

       2\) OnRecvNewP2PMsgNotify : 新的消息通知

       3\) OnSendImCloudP2PMsgRes : 消息发送

       4\) OnPullImCloudP2PMsgRes : 拉取消息

       5\) OnSetCommPushFlagRes : 设置推送标记

   功能

   1. InitiativePullP2PMsg \(登陆成功后，通过此函数获取离线消息\)

      1\) 从本地数据库中获取push的seqId，

      2\) 从seqId开始，从云端拉取消息

   2. \_PullImCloudMsg \(从云端拉取消息\)

      1\) 创建拉取请求 ： PCS\_IMCloudCommPull ：任务ID，拉取条目，拉取起始ID，请求的目标IDC，已经收到的Push消息的ID

      2\) 将请求添加到消息重试队列中 : CImSdkMgr::GetMsgResendMgrInstance\(\)-&gt;AddResendMsg

      3\) 发送请求\(异步\) : SendPacket

      4）如果发送失败: 则打印日志

      5\) 如果发送成功：设置本次请求任务的拉取状态 SPullP2PMsgState，添加到m\_msgId2PullState中

   3. OnPullImCloudP2PMsgRes \(从云端拉取消息的RPC消息处理函数\)

      1\) 解析拉取结果

      2\) 设置消息Push的Flag ： \_SetCommPushFlag

      3\) 如果拉取请求对应的消息拉取状态中保存的推送消息SeqID大于本次拉取结果的最大SeqID，则继续发起\_PullImCloudMsg

   4. \_SetCommPushFlag \(TODO, 确认这个命令是干什么的\)

      1\) 创建PCS\_CIMSetCommPushFlag请求：任务ID，起始SeqID，结束SeqID，flag

      2\) 将请求添加到消息重试队列中

      3\) 发送请求（异步\)

   5. OnSetCommPushFlagRes

      1\) 消息回复去重判断

      2\) 从消息重发队列中删除

      3\) 检查返回结果，打印日志

   6. \_CanSendImCloudP2PMsg \(检查该账户当前是否能够发送消息\)

      1\) 检查是否消息签名信息

      2\) 如果没有签名信息，则从服务端端获取签名信息, 函数执行结束, 返回不能发送消息

      3\) 如果有签名信息：则使用账户系统，根据账户信息，获取uid

      4\) 如果没有uid，则使用账户系统, 根据账户信息，从云端获取uid, 返回不能发送消息

      5\) 如果有，则返回能够发送消息

   7. \_HaveMsgSignInfo \(检查是否有消息签名信息\)

      1\) 从m\_acc2MsgSignInfo中检查消息是否有签名信息：根据账户信息

      2\) 如果存在，则返回。

      3\) 如果不存在，则从本地数据库中获取账户对应的签名信息，并填充m\_acc2MsgSignInfo。

   8. \_GetMsgSignInfoByHttp \(从服务端端获取对应的签名信息\)

      1\) 发送Http请求

   9. \_ParseMsgSignInfo \(从服务端获取了消息的签名信息后，进行结果解析\)

      1\) 解析：ticket\(签名串\) , nonce\(随机串）, server\_ts\(签名时间戳\), to\(目标账户\)

      2\) 向IM云发起签名检查请求: SendIMCloudCheckAppSignature

   10. SendIMCloudCheckAppSignature \(请求IM云检查应用的签名信息\)

      1\) 创建PCS\_CIMCheckOpAppSign请求 \(任务ID， 时间戳，签名串，随机串，发送者账户，接受者账户\)

      2\) 加入到消息重发队列

      3\) 发送消息:SendPacket\(异步\)

   11. SendImCloudP2PMsg（向IM云发送单聊消息\)

      1\) 检查是否能发送消息 ： \_CanSendImCloudP2PMsg

      2\) 如果不能发送：则添加到发送队列中（m\_acc2ToSendMsg\), 等待后续发送， 然后返回。

      3\) 如果能发送：则创建 PCS\_ImCloudP2PMsgWithSign 请求 （消息ID，消息类型，消息内容, 签名信息）

      4\) 加入到消息重发队列中

      5\) 发送消息：SendPacket（异步\)

      6\) 如果发送成功，并且是文本类型消息，则保存消息信息到：m\_msgId2Info

   12. OnSendImCloudP2PMsgRes \(发送消息RPC的回调处理函数\)

      1\) 回复消息去重 （根据消息ID 和 重发消息队列\) , 如果不存在\(消息已经处理\)，则返回， 如果存在，则继续。

      2\) 从重发消息队列中删除对应消息任务

      3\) 如果是文本消息

      4\) 检查消息是否存在\(m\_msgId2Info\), 如果不存在，则打印出错信息，返回, 如果存在，则继续。

      5\) 向windows产生一个IM\_EVENT, 通知用户UI线程。

      6\) 从正在处理的消息中删除该消息\(m\_msgId2Info\)

   13. OnRecvNewP2PMsgNotify \(消息通知的回调处理函数\)

      1\) 判断消息是否需要确认，如果需要，则SendPacket进行确认 \(携带push的消息通知对应的消息ID）

      2\) 判断是不是有效通知，如果不是，则打印错误日志，然后返回，如果有效，则继续

      3\) 获取本地保存的消息通知的SeqID

      4\) 向IM云发起消息拉取请求：从本地的SeqID到推送过来的本条通知中的SeqID

   14. \_ParsePullP2PMsgs \(解析拉取到的消息\)

      1\) 本地数据库存储消息 \(\_StoreP2PMsg2DataBase\)

      2\) 通知UI消息已经收到\(Notify2UiRecvP2PMsgs : 需要根据UID获取发送者的账户信息\)

   15. vOnAccToUidUpdate \(当AccountMgr根据账户获取到UID信息后，会调用该函数, 触发消息重发\)

   16. vOnUidToAccUpdate \(当AccountMgr根据UID获取到账户信息后，会调用该函数, 触发UI事件通知，消息已接收\)

## 回调函数

主动请求对应的响应（IM Server处理请求完成，应答请求方\)

1. 登陆----On登陆成功
2. 添加好友-----On添加好友成功
3. 删除好友-----On删除好友成功
4. 屏蔽好友-----On屏蔽好友成功
5. 取消屏蔽------On取消屏蔽
6. 获取好友列表-----On获取好友列表
7. 获取好友在线状态-----On获取好友在线状态
8. 单聊私信------On消息发送成功
9. 同意或者拒绝添加好友------On同意或者拒绝添加好友
10. 拉取消息-----On拉取消息成功
11. 获取屏蔽列表------On获取屏蔽列表成功

被动推送的响应

1. 收到添加好友请求
2. 收到删除好友请求
3. 收到消息通知
4. 收到屏蔽好友请求
5. 收到取消屏蔽好友请求
6. 收到消息？

## 错误码定义



