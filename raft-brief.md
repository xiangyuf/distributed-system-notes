# Raft读书笔记

## Raft总览

1. Leader选举
2. Leader基本操作：日志复制
3. Leader改变后的安全性和一致性
4. 废除旧的Leader
5. 配置变更，集群增减节点

## Raft的几个概念

### 节点角色
- Leader：负责所有的客户端交互，日志复制，有且只有一个
- Follower：被动角色，不会主动发起请求，负责响应发来的请求
- Candidate：Leader的候选人

### Term（轮次）
- 时间切分的最小单位，包括选举和普通操作
- 一个轮次最多只能有一个Leader
- 用来确认过期信息
- 每个节点维护当前的轮次数

### 心跳和超时
- Leader必须要给Followers发送心跳来维持领导权
- 如果Leader发送心跳的时间超时，Follower默认Leader崩溃了，Follower发起新的选

## Raft过程

### Raft选举
1. 增加当前轮次
2. 改变到候选人状态
3. 给自己投票
4. 发送投票请求（RequestVote）给其它节点：
	-	如果收到大多数节点的投票则变成Leader，同时给其它节点发送心跳
	- 	收到来自有效Leader的回复，则变成Follower状态
	-  选举超时，则增加轮次，开始新的选举

### 日志结构
- 日志条目 = 日志位置索引 + 轮次数 + 指令
- 日志存储在持久硬盘上，避免崩溃
- 日志条目在大多数节点上确认后在Leader节点上被确认提交

### Leader基本操作
1. 客户端发送命令到Leader
2. Leader添加命令到本地日志
3. Leader发送AppendEntries的请求给Followers
4. 收到Followers添加命令成功的反馈后：
	- Leaders把命令发送给本地的状态机，并返回结果给客户端
	- Leader通过AppendEntries命令通知Followers，Followers把命令放到状态机里面
5. Leader会重试请求，直到Follower成功

### 日志一致性
1. 日志的位置索引和轮次可以唯一标识一条日志
2. 不同服务器上同一位置的日志如果相等，代表前置位置的日志也都会相等
3. 如果某一位置的日志被提交了，代表前置位置的日志也都提交了

### AppendEntries一致性请求
1. 每一个AppendEntries的请求都包含之前条目的位置索引和轮次数
2. Follower必须校验前置条目的正确性，拒绝不匹配的
3. 删除不匹配的日志

### Leader更改
1. 老的Leader可能有未被复制的日志
2. 新的Leader直接开始普通操作，以新Leader的日志为准
3. 最终让Follower的日志跟Leader的日志保持一致

### 安全性要求
日志一旦在某个Leader上被确认提交：

1. 不能被篡改
2. 	保证出现在之后所有的Leader上面
3. 必须确认提交后才能进入状态机

### Leader选举
1. 候选者把日志信息包含在RequestVote中，请求给其它投票节点
2. 当投票节点发现自己的轮次更高，或者轮次相等但索引位置更大，则拒绝投票
3. Leader需要有所有候选节点中总最完整的日志记录
4. 新的Leader会让Follower的日志跟自己保持一致
5. Leader记录每个Follower的nextIndex，当一致性检查失败时，nextIndex减一

### 客户端协议
1. 客户端发送请求给任意节点，如果节点不是Leader则转发给Leader
2. Leader接受请求后，需要等到日志被提交并且并状态机执行后才能回复
3. 如果请求超时，客户端会发送请求到新的节点直到找到新的Leader

### 配置变更
1. 不能直接从旧配置切换到新配置，会出现conflicting majorities
2. 需要中间步骤，由新老配置的majorities组成joint consensus
3. 新配置的日志在本地提交后，老配置的leader step down，变成新配置Leader的Follower