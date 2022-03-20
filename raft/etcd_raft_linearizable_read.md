# ETCD Raft Linearizable Read

## 线性一致性读的策略

所谓 Linearizable Read 线性一致性读即读请求需要读取到**最新的**、**已经 commit 的**数据。 

目前在 Raft 算法里实现线性一致性的策略主要有三种：

| **策略**         | **具体实现**                                                 | **安全性**                                                   | **效率**                                    |
| ---------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------- |
| 封装成 log entry | 将读请求与写请求同样对待，封装成 log entry ，走一遍复制流程再读取数据。 | 最安全的策略                                                 | 效率差                                      |
| ReadIndex        | ETCD 默认的线性一致性读的策略。记录下当前读请求时的 commit index ，并在集群中广播心跳以确认自己还是 leader 。若还是 leader 则等待直到 apply index 大于等于该 commit index 便可唤醒读请求进行读取数据。 | 安全性介于两者之间                                           | 效率介于两者之间（比起第一种省去复制步骤）  |
| LeaseRead        | 使用 lease 机制，即在一个选举超时周期内无需再确认 leader 身份，等待直到 apply index 大于等于该 commit index 就可唤醒读请求进行读取数据。 | 最不安全的策略（如果系统的时钟偏差严重，该策略是存在问题的） | 效率最高（比起第二种省去广播心跳的网络 IO） |

 ETCD 对 ReadIndex 和 LeaseRead 都进行了实现。下面来看 ETCD 是如何实现两种策略的。



## Read 流程

首先需要在初始化 Raft 时，在 Config 中的选择需要的读策略：

```Go
type ReadOnlyOption int

const (
   ReadOnlySafe ReadOnlyOption = iota
   
   ReadOnlyLeaseBased
)

type Config struct {
    // ...

    ReadOnlyOption ReadOnlyOption
    
    // ...
}
```

其中 ReadOnlySafe 即 ReadIndex 策略，ReadOnlyLeaseBased 即 LeaseRead 策略。如果未进行设置，**默认为 ReadIndex 策略**。

> Note ：当使用 ReadOnlyLeaseBased 策略时，必须将 Config 中的 CheckQuorum 设置为 true 。

然后 user 调用 `Node.ReadIndex()` 接口，该接口的默认实现将读请求封装成一个 log entry 输入到 Raft 中：

```Go
func (n *node) ReadIndex(ctx context.Context, rctx []byte) error {
   return n.step(ctx, pb.Message{Type: pb.MsgReadIndex, Entries: []pb.Entry{{Data: rctx}}})
}
```

> 其中 rctx 是客户端读请求的唯一标识 RequestCtx 。

如果接收该 log entry 的节点为 follower ，则该 follower 将会把此 log entry 转发给 leader 。

系统经过执行初始化选定的一致性读策略后，得到的结果 **ReadState** 将会放在 Ready 结构体里，供上层应用使用：

```Go
type ReadState struct {
   Index      uint64 // 请求到来时的 CommitIndex
   RequestCtx []byte // 客户端读请求的唯一标识
}
func newReady(r *raft, prevSoftSt *SoftState, prevHardSt pb.HardState) Ready {
   rd := Ready{
      // ...
   }
   // ...
   
   if len(r.readStates) != 0 {
      rd.ReadStates = r.readStates
   }
   // ...
   
   return rd
}
```

并且在上层应用接受本次 Ready 后将 Raft 的 readStates 清空：

```Go
func (rn *RawNode) acceptReady(rd Ready) {
   // ...
   
   if len(rd.ReadStates) != 0 {
      rn.raft.readStates = nil
   }
   // ...
}
```

下面以 leader 视角为例具体介绍 ETCD 是如何实现这两种读策略的。



## readOnly 结构体

```Go
type readIndexStatus struct {
   req   pb.Message // 调用 ReadIndex 接口时封装的 message
   index uint64     // 请求到来时的 CommitIndex
 
   // key 为 serverID ，value 始终为 true ，表示集群的节点对该 ReadIndex 请求
   // 的确认，用于 ReadIndex 策略的确认自身为 leader (len(acks) >= Quorum)
   acks map[uint64]bool
}

type readOnly struct {
   option           ReadOnlyOption // 读一致性策略
   // 待处理的 ReadIndex 请求
   // key 为 RequestCtx ，value 即为对应的 readIndexStatus
   pendingReadIndex map[string]*readIndexStatus
   // queue 内容是根据时间顺序由早到晚的 RequestCtx
   // 与 pendingReadIndex 的 key 相对应
   readIndexQueue   []string
}
```



## ReadOnlySafe

ReadIndex 请求首先在 stepLeader 函数里进行处理：

```Go
case pb.MsgReadIndex:
   // 如果集群中只有一个 voter 则直接生成 ReadState 添加到本地等待消费即可
   if r.prs.IsSingleton() {
      if resp := r.responseToReadIndexReq(m, r.raftLog.committed); resp.To != None {
         r.send(resp)
      }
      return nil
   }

   // 如果 leader 未在当前 term 提交过 log entry ，则需要等到提交完当前
   // term 的 log entry 才能再处理 ReadIndex 请求
   if !r.committedEntryInCurrentTerm() {
      r.pendingReadIndexMessages = append(r.pendingReadIndexMessages, m)
      return nil
   }

   sendMsgReadIndexResponse(r, m)

   return nil
}
```

然后进入到 `sendMsgReadIndexResponse()` 函数，对请求进行封装，同时并确认自身的 leader 身份：

```Go
func sendMsgReadIndexResponse(r *raft, m pb.Message) {
   switch r.readOnly.option {
   case ReadOnlySafe:
      // 将该请求封装为 readIndexStatus 并添加至 readOnly 中
      r.readOnly.addRequest(r.raftLog.committed, m)
      // 当前节点直接确认该请求
      r.readOnly.recvAck(r.id, m.Entries[0].Data)
      // 广播心跳以确认自己身份仍为 leader
      r.bcastHeartbeatWithCtx(m.Entries[0].Data)
   case ReadOnlyLeaseBased:
      // ...
   }
}
```
```Go
func (ro *readOnly) addRequest(index uint64, m pb.Message) {
   s := string(m.Entries[0].Data)
   if _, ok := ro.pendingReadIndex[s]; ok {
      return
   }
   ro.pendingReadIndex[s] = &readIndexStatus{index: index, req: m, acks: make(map[uint64]bool)}
   ro.readIndexQueue = append(ro.readIndexQueue, s)
}

func (ro *readOnly) recvAck(id uint64, context []byte) map[uint64]bool {
   rs, ok := ro.pendingReadIndex[string(context)]
   if !ok {
      return nil
   }

   rs.acks[id] = true
   return rs.acks
}
```

follower 收到带有 ctx 的消息后，确认并返回给 leader ，所以 leader 在处理 heartbeat response 时需要对 readOnly 进行推进：

```Go
case pb.MsgHeartbeatResp:
   // ...
   
   if r.readOnly.option != ReadOnlySafe || len(m.Context) == 0 {
      return nil
   }

   // 如果集群中大多数节点确认了该请求，则可以推进 ReadIndex 请求
   if r.prs.Voters.VoteResult(r.readOnly.recvAck(m.From, m.Context)) != quorum.VoteWon {
      return nil
   }

   // 推进当前 ReadIndex 请求
   rss := r.readOnly.advance(m)
   for _, rs := range rss {
      if resp := r.responseToReadIndexReq(rs.req, rs.index); resp.To != None {
         r.send(resp)
      }
   }
```
```Go
func (ro *readOnly) advance(m pb.Message) []*readIndexStatus {
   var (
      i     int
      found bool
   )

   ctx := string(m.Context)
   rss := []*readIndexStatus{}

   for _, okctx := range ro.readIndexQueue {
      i++
      rs, ok := ro.pendingReadIndex[okctx]
      if !ok {
         panic( cannot find corresponding read state from pending map )
      }
      // 取出包括当前 RequestCtx 之前的所有 readIndexStatus
      rss = append(rss, rs)
      if okctx == ctx {
         found = true
         break
      }
   }

   if found {
      // truncate 包括当前 RequestCtx 之前的所有 RequestCtx
      ro.readIndexQueue = ro.readIndexQueue[i:]
      for _, rs := range rss {
         // 删除 包括当前 RequestCtx 之前的所有 readIndexStatus
         delete(ro.pendingReadIndex, string(rs.req.Entries[0].Data))
      }
      return rss
   }

   return nil
}
```

> 问题：在 advance 方法里如果在当前 request 前还有其他的 request 未被集群中大多数节点确认就被取出会存在问题吗？
>
> 不会有问题。以我个人见解如果现在能确认身份为 leader 就能确认之前仍是 leader ，所以是可以对之前的 request 进行操作的。

最后将 ReadState 放入本地待上层应用获取：

```Go
func (r *raft) responseToReadIndexReq(req pb.Message, readIndex uint64) pb.Message {
   if req.From == None || req.From == r.id {
      r.readStates = append(r.readStates, ReadState{
         Index:      readIndex,
         RequestCtx: req.Entries[0].Data,
      })
      return pb.Message{}
   }
   // 用于转发回给 follower
   return pb.Message{
      Type:    pb.MsgReadIndexResp,
      To:      req.From,
      Index:   readIndex,
      Entries: req.Entries,
   }
}
```



## ReadOnlyLeaseBased

前面与 ReadOnlySafe 方式相同，直到 `sendMsgReadIndexResponse()` 函数，直接将该请求封装为 ReadState 加入本地即可完成此次请求：

```Go
func sendMsgReadIndexResponse(r *raft, m pb.Message) {
   switch r.readOnly.option {
   case ReadOnlySafe:
      // ...
   case ReadOnlyLeaseBased:
      if resp := r.responseToReadIndexReq(m, r.raftLog.committed); resp.To != None {
         r.send(resp)
      }
   }
}
```

lease 体现在哪里呢？答案是在 Step 方法中，通过表明自身仍处于选举超时周期之内来拒绝其他 vote 请求来达到 lease 的目的：

```Go
func (r *raft) Step(m pb.Message) error {
   // ...
   
   case m.Term > r.Term:
      if m.Type == pb.MsgVote || m.Type == pb.MsgPreVote {
         force := bytes.Equal(m.Context, []byte(campaignTransfer))
         inLease := r.checkQuorum && r.lead != None && r.electionElapsed < r.electionTimeout
         if !force && inLease {
            // If a server receives a RequestVote request within the minimum election timeout
            // of hearing from a current leader, it does not update its term or grant its vote
            r.logger.Infof( %x [logterm: %d, index: %d, vote: %x] ignored %s from %x [logterm: %d, index: %d] at term %d: lease is not expired (remaining ticks: %d) ,
               r.id, r.raftLog.lastTerm(), r.raftLog.lastIndex(), r.Vote, m.Type, m.From, m.LogTerm, m.Index, r.Term, r.electionTimeout-r.electionElapsed)
            return nil
         }
      }
```

所以通过 `r.checkQuorum && r.lead != None && r.electionElapsed < r.electionTimeout` 来判断当前节点是否处于 lease 内。下面简单介绍一下 CheckQuorum 机制。



### CheckQuorum

CheckQuorum 是用于检测集群中大多数节点是否活跃，同时检测 leader 是否出现了网络分区，这是 lease 机制的基础。

leader 会在广播心跳前检测当前是否超过了一个选举超时周期，如果已超过，就会向集群广播 CheckQuorum 消息，通过 QuorumActive 方法以检测集群中大多数节点的 RecentActive 状态为是否为 true ：

- 若集群中大多数节点的 RecentActive 为 true ，则将每个节点的 RecentActive 状态设置为 false ，后续 leader 将在收到 append entries response 和 heartbeat response 时将其设置为 true ；

- 若集群中大多数节点的 RecentActive 不为 true ，说明可能 leader 自身出现网络分区，则需要立刻 step down 。