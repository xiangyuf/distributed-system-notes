# Paxos

### Paxos渐进模型

- Basic Paxos
	- 多个节点都可以给出提议
	- 系统只能同意一个提议
	
	 
- Multi-Paxos
	- 结合多个Basic Paxos的操作来同意一系列的提议

### Paxos的角色

- Proposers
	- 处理客户端请求
	- 主动：可以发起并推举提议

- Acceptors
	- 被动：回复来自proposer的消息
	- 投票选举提议
	- 存储提交后的提议和推举的状态
	- 需要知道哪些提议通过

### Proposal Number

- 每一个Proposal有一个唯一编号：RoundNumber + ServerId
	- ProposalNumber数值越高，权重越高
	- Proposer需要有办法生成比已知ProposalNumber更高的ProposalNumber
	- 每个节点保存自己已知最大的RoundNumber，作为maxRound
	- maxRound需要被持久化到硬盘上

### Basic Paxos

- 两阶段提交
	- 阶段1：Prepare
		- Proposer：生成新的ProposalNumber(n)
		- Proposer：广播Prepare(n)给所有节点
		- Acceptor：if n > minProposal，则minProposal = n，返回(acceptedProposal, acceptedValue)
	- 阶段2：Accept
		- Proposer：当收到majority的prepare response，广播Accept(n, value)给所有节点
		- Acceptor：if n > minProposal，则acceptedProposal = minProposal = n，acceptedValue = value，返回(minProposal)
		- Proposer：有收到rejections时返回第一步，否则表明value被chosen

- Acceptors必须持久化minProposal, acceptedProposal和acceptedValue到硬盘上
- 只有Proposal知道提议是否成功，其它节点需要知道的话也需要执行自己的Proposal
- 为避免多个Proposer节点发生竞争死锁的情况，每次重新启动Paxos时需要有随机的延迟

## Multi-Paxos

一次Multi-Paxos通常包含多次Basic Paxos

1. 客户端发送命令到某个节点
2. 节点使用Paxos来选择一个未被选中的命令作为提议
3. 节点等待之前提议都通过后，在状态机中执行新的命令
4. 把状态机的执行结果返回给服务端

### Multi-Paxos需要解决的问题

- 发起哪个提议
- 性能优化
- 信息全披露（全复制，提议通过后广播）
- 客户端协议
- 配置变更

### 选择需要发起的提议

当收到来自客户端的请求时：

- 找到第一条未被提交的提议
- 使用Paxos在该条提议中执行客户端命令
- 如果提议被告知有acceptedValue，则执行之前的acceptedValue，并从头开始执行
- 如果提议被告知没有acceptedValue，则执行客户端的命令

### Multi-Paxos的性能优化

1. 选取Leader，任一时间只有一个Proposer
2. 减少Prepare请求，大多数请求只需要发送Accept调用

### Leader选举

1. 有最高ID的节点作为Leader
2. 每个节点每T毫秒发送心跳给其它节点
3. 如果一个节点在2T毫秒内没有收到来自其它节点的心跳，它就变成Leader

### 减少Prepare请求

- prepare的用处：block之前的请求，找到可能确认的提议
- 如果已经是最新的提议，直接noMoreAccepted
- 之后的Basic Paxos可以省略Prepare环节，直接发送Accept请求

### 信息全披露

1. 重试AcceptRPC，直到所有acceptors都成功返回
2. 每个节点维护自己的firstUnchosenIndex，并mark已通过的提案为极大值
3. Proposer告诉Acceptors自己的firstUnchosenIndex
	- 如果 i < request.firstUnchosenIndex && acceptedProposal[i] = request.proposal
	- 标记提案i为极大值
4. Acceptor返回它的firstUnchosenIndex给Proposer，如果proposer.firstUnchosenIndex > acceptor.firstUnchosenIndex，则proposer发送Success RPC
5. Success(index, v)请求通知acceptor：
	- acceptedValue[index] = v
	- acceptedProposal[index] = ∞
	- return firstUnchosenIndex

### 客户端协议

和raft类似，通过唯一的command id来保证exactly-once semantics