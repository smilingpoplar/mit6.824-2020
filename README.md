## MIT 6.824 分布式系统 [(2022)](http://nil.csail.mit.edu/6.824/2022/schedule.html)

### [Lab1](http://nil.csail.mit.edu/6.824/2022/labs/lab-mr.html)-MapReduce

#### /mr 实现思路

coordinator和多个worker通过RPC通信，worker以plugin方式加载/mrapps下的Map()和Reduce()函数实现

worker向coordinator发心跳要任务：
- 若是Map任务，收到fileName读取文件kv，mapf处理后输出成json格式的中间文件mr-fileId-reduceId
- 若是Reduce任务，将mr-XX-reduceId文件合并后输出mr-out-reduceId

coordinator管理整体任务状态：
- 在Map全部完成后才开始Reduce
- 给worker任务10s后检查完成情况，没完成就重新标记为待做

### [Lab2](http://nil.csail.mit.edu/6.824/2022/labs/lab-raft.html)-Raft

Raft共识算法的[论文](https://pdos.csail.mit.edu/6.824/papers/raft-extended.pdf)与[翻译](https://github.com/maemual/raft-zh_cn/blob/master/raft-zh_cn.md)，Figure2总结了算法过程

#### Part2A 选举和心跳
- currentTerm作为逻辑时钟，空entries[]的AppendEntries RPC作为心跳
- 三个角色follower/candidate/leader的状态机转化
- 无论什么角色，收到的request或reply中对方的term比自己大，马上变成follower

粗略过程如下：
1. 刚开始都是follower，超时没有收到leader心跳变为candidate
2. candidate广播RequestVote RPC请求投票
3. RequestVote处理方对一个term只能投一张票
4. candidate收到">1/2多数"选票变为leader
5. leader广播AppendEntries RPC心跳阻止其他选举产生

#### Part2B 日志项的复制

粗略过程如下：
1. client用Start()发送需本地状态机执行的command
2. leader在本地日志log[]中添加LogEntry{需本地状态机执行的command, 创建时的term}
3. 通过AppendEntries心跳通知所有follower复制log[]
4. follower[i]回复成功后，leader更新matchIndex[i]，matchIndex[i]是已成功复制到follower[i]的log[]最大索引
5. 用matchIndex[]找出满足">1/2多数"的commitIndex，则log[:commitIndex]已成功复制到多数follower，可应用log[:commitIndex]到本地状态机
6. leader的commitIndex在心跳中传给follower，follower也可应用log[:commitIndex]到本地状态机

第3步通过心跳复制leader日志到follower[i]，关键是找到leader与follower[i]日志匹配的最大索引lastMatch，然后将follower[i]的log[lastMatch+1:]全部覆盖。leader用prevLogIndex从最后位置（nextIndex[i]-1）往前探查lastMatch。日志匹配，这里检查某index位置的term相同。也有加速查找lastMatch的[回退优化](https://thesquareplanet.com/blog/students-guide-to-raft/#an-aside-on-optimizations)。

#### /raft调bug
[go-test-many](https://gist.github.com/smilingpoplar/793cb88ff29bff65bdb1a78d49f4cfdd)可多核运行测试。
`go-test-many.sh 20 8 2A`：共20次用8核运行`go test -run 2A`