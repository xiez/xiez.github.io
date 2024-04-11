---
title: Raft 共识算法 - Leader 选举
classes: wide
categories:
  - 2024-04
words_per_minute: 60
tags:
  - raft
  - python
---

分布式系统的「共识问题」（consensus）是指多个节点共享数据，并通过共识算法来保证数据的一致性。

假设一个单节点的 key-value 系统，它不需要共识算法就能确保数据的一致性，但是存在单点问题，所以需要增加多个节点来增加可靠性。而增加节点会带来数据管理和一致性的问题，目前的常见方案有两种：

1. 数据分片，例如 memcached/redis 集群，每个节点只管理一部分数据，再通过一致性哈希算法来确保在节点添加或删除时，数据的重新分布最小化。

2. 所有节点共享数据，从外部看，跟单节点系统没有差别，也就是所谓的「线性化」（Linearizability），需要通过共识算法来保证，例如 Gossip，Paxos，Raft等。

共识算法的实现也分为两类，分别是：

1. 以 Gossip 为代表的「无主」（Leaderless）算法，各个节点平等接收请求，需要进行冲突处理、合并，由于需要等所有节点达成共识，决策过程相对较慢，可以归类为“高可用，弱一致性算法”。

2. 以 Paxos, Raft 为代表的「有主」（Leader-based）算法，系统中存在一个特定的 Leader 节点，负责接收和处理请求，其他节点都是从属节点。由于有 Leader 的存在，决策过程相对简单高效，但是需要进行选举，在 Leader 选出来之前，系统短暂处于不可用状态，所以可以归类为“较高可用，强一致算法”。

## Raft 概念

Raft 虽然相比 Paxos 简化不少，但依旧属于比较难的算法，包含有不少概念。

### 任期（term）

term 在 Raft 中扮演“逻辑时钟”的作用。每一个节点维护一个从 0 开始单调递增的整数标识当前的轮次（currentTerm），用来判定哪些信息已经过期。

每个任期都会有选举，最终会产生一个领导者。如果由于票数分裂（split vote）导致没有领导者产生，自动转入下一任的选举。例如，5 节点的集群由于某种原因，领导者下线了，剩下的 4 个节点投票时，可能产生 2v2 的情况，无法选出领导者，此时就会进入下一个选举轮次。

所以，为了降低票数分裂的概率，集群通常包含奇数个节点，其中 5 节点比较常见。

### 状态（state）

集群节点有3类状态，99% 的时间只有跟随者和领导者。只有当领导者下线的情况下，会有部分跟随者转变为候选者，一旦某个跟随者成为领导者，剩下的节点又会恢复到跟随者状态。

**跟随者（Follower）**: 节点启动时的初始状态，跟随者不对外发送 RPC 请求，只接收来自领导者或候选者的消息。

**领导者（Leader）**: 正常集群有且只有一个领导者，负责跟客户沟通。领导者定期发送消息给到所有的跟随者，告诉他们自己的存在。

**候选者（Candidate）**: 当跟随者一段时间内没有收到领导者的消息，就会假定领导者已经下线，此时转变为候选者，发起选举投票。

3类状态的转换如下图：

![state-machine](../../assets/images/2024/04/state-machine.png "state-machine")

## 工作原理

领导者选举的大致原理如下：

### 选举触发

- 跟随者依靠来自领导者的心跳消息来确认其存活状态。

- 如果跟随者在超时时间段（`electionTimeout`）内没有收到心跳，它将通过转换为候选者角色来启动领导者选举。

- 跟随者每任期只向一个候选者投票（先到先得）。他们只有在候选者的任期号高于他们当前任期号时才会投票，这用来确保跟随者只支持最新的选举。

### 候选者竞选

- 作为候选者，服务器会增加当前任期（`currentTerm`）并向集群中的所有其他服务器发送投票请求消息，消息里包括候选者的当前任期号。

- 在由于网络分区或并发选举导致没有候选者获得多数的情况下，选举会超时，所有服务器都会重新开始选举过程，并增加任期号。

### 成为领导者

- 如果候选者收到来自集群中大多数服务器（超过一半）的投票，则该候选者将成为领导者。

- 一旦领导者被选举，它会定期向跟随者发送心跳消息来维持其领导地位并防止不必要的选举。

### 任期提升

- 如果服务器收到任期号高于其当前任期号的投票请求，则它会承认领导者并更新自己的任期号。这确保服务器识别新领导者的权威。

## 算法实现

### 初始化

```
STATE_F = 'follower'
STATE_C = 'candidate'
STATE_L = 'leader'

class ConsensusModule:
    def __init__(self, server):
        self._lock = threading.Lock()
        self.server_id = server.server_id
        self.server = server

        self.state = STATE_F
        self.election_reset_event = -1

        self.current_term = 0
        self.vote_for = -1
```

共识模块初始时，状态为“follower”，当前任期为0。


### 跟随者定时器

```
class ConsensusModule:

    ...

    def set_follower_timer(self):
        """Follower timer, also known as election timer. The timer exits either:

        - we find the election timer is no longer needed, or
        - the timer expires and this CM becomes a candidate

        """
        def ticker():
            is_running = True
            while is_running:
                time.sleep(0.01 / PLAY_SPEED)
                self.debug(f'follower timer ticking ... term={self.current_term}')

                self._lock.acquire()
                if self.state != STATE_F:
                    self.info(f"follower timer stopped, reason: current state is {self.state}, term={self.current_term}")
                    self._lock.release()
                    is_running = False
                    continue

                if self.current_term != term_started:
                    self.info(f"follower timer stopped, reason: term changed from {term_started} to {self.current_term}")
                    self._lock.release()
                    is_running = False
                    continue
 
                if time.time() - self.election_reset_event > timeout_duration:
                    self._lock.release()
                    self.start_election()
                    is_running = False
                    continue
                self._lock.release()

        with self._lock:
            timeout_duration = self.election_timeout
            term_started = self.current_term
            self.election_reset_event = time.time()

        threading.Thread(target=ticker).start()
        self.info(f'follower timer started, timeout={timeout_duration}, term={term_started}')

    @property
    def election_timeout(self):
        return random.randint(150, 300) / 1000 / PLAY_SPEED              # seconds

```

节点初始化完成后，设置一个定时器，过期时间（`election_timeout`）为 150ms-300ms 之间的随机值。

1. 在定时器过期时间内，如果当前状态或者任期发生变化，则停止定时器。

2. 如果定时器过期了，则触发选举。

**要点**：定时器和任期绑定，当前任期可能会改变，在定时器开始前，需要保存下 `currentTerm`, 如果发现任期变了，需要把定时器取消。


### 候选者竞选

```
class ConsensusModule:

    ...

    def start_election(self):
        self.info(f'election timer timed out, no visible leader, start leader election')

        # To begin an election, a follower increments its current term and transitions to candidate state.
        with self._lock:
            self.state = STATE_C
            self.current_term += 1
            self.vote_for = self.server_id
            self.info(f'became Candidate at term: {self.current_term}, and vote for self')

        votes_received = 1
        votes_lock = threading.Lock()

        # It then votes for itself and issues RequestVote RPCs in parallel to each of the other servers in the cluster.
        # A candidate continues in this state until one of three things happens:
        # (a) it wins the election,
        # (b) another server establishes itself as leader, or
        # (c) a period of time goes by with no winner.
        
        def request_vote(pid, sp):
            self.info(f'sending RequestVote to server#{pid} with args: term={self.current_term}, server_id={self.server_id}')
            try:
                reply = sp.request_vote(self.current_term, self.server_id)
                if self.current_term < reply['term']:
                    # current term out of date, become follower
                    self.become_follower(reply['term'])
                else:
                    if reply['vote_granted'] is True:
                        if self.state in [STATE_F, STATE_L]:
                            return
                        
                        nonlocal votes_received
                        with votes_lock:
                            votes_received += 1
                            if votes_received * 2 > len(self.server.peer_servers) + 1:
                                self.info(f'won election with {votes_received} votes')
                                self.become_leader()
            except Exception as e:
                err = f"RV RPC error to server#{pid}: {e}"
                print(err)
                self.info(err)

        for pid, sp in self.server.peer_clients.items():
            threading.Thread(target=request_vote, args=(pid, sp)).start()

        self.set_candidate_timer()

    def set_candidate_timer(self):
        """Almost same as follower timer.

        Maybe we should combine those two timers ?
        """
        def ticker():
            is_running = True
            while is_running:
                time.sleep(0.01 / PLAY_SPEED)
                self.debug(f'candidate timer ticking ... term={self.current_term}')

                self._lock.acquire()
                if self.state != STATE_C:
                    self.info(f"candidate timer stoped, reason: current state is not Candidate, term={self.current_term}")
                    self._lock.release()
                    is_running = False
                    continue

                if self.current_term != term_started:
                    self.info(f"candidate timer stoped, reason: term changed from {term_started} to {self.current_term}")
                    self._lock.release()
                    is_running = False
                    continue

                if time.time() - self.election_reset_event > timeout_duration:
                    self._lock.release()
                    self.start_election()
                    is_running = False
                    continue
                self._lock.release()

        with self._lock:
            timeout_duration = self.election_timeout
            term_started = self.current_term
            self.election_reset_event = time.time()
                    
        threading.Thread(target=ticker).start()
        self.info(f'candidate timer started with timeout {timeout_duration}, term={term_started}')

```

触发选举后，节点状态置为“candidate”后，先投自己一票，再向所有节点发送投票请求，最后再设置一个定时器，过期时间可以跟 `electionTimeout` 相同。

1. 在定时器过期时间内，收到超过半数的投票，则成为领导者。

2. 如果定时器过期了，则重置定时器，开始下一轮投票。

### 成为领导者

```
class ConsensusModule:

    ...

    def become_leader(self):
        if self.state == STATE_L:
            return

        self.state = STATE_L
        self.info(f"became Leader at term:{self.current_term}, sending heartbeats ...")

        def ticker():
            is_running = True
            while is_running:
                self.send_heartbeats()

                time.sleep(0.05 / PLAY_SPEED)
                if self.state != STATE_L:
                    is_running = False

        threading.Thread(target=ticker).start()

    def send_heartbeats(self):
        def ae(pid, sp):
            self.info(f'sending AppendEntries to server#{pid} with args: term={self.current_term}, leader_id={self.server_id}')
            try:
                reply = sp.append_entries(self.current_term, self.server_id)
                if self.current_term < reply['term']:
                    # stop heartbeats and step down
                    self.become_follower(reply['term'])
                else:
                    ...
            except Exception as e:
                err = f"AE RPC error to server#{pid}: {e}"
                print(err)
                self.info(err)

        for pid, sp in self.server.peer_clients.items():
            threading.Thread(target=ae, args=(pid, sp)).start()
```

成为领导后，为了维护领导权威，需要每隔 50ms 向其他节点发送心跳消息。


### RPC

#### 请求投票

```
class ConsensusModule:

    ...

    def request_vote(self, term, server_id):
        self.info(f"got RequestVote from server#{server_id} at term: {term}")

        if self.current_term < term:
            self.info(f"... current_term is out of date, current: {self.current_term}, request: {term}")
            if self.state in [STATE_L, STATE_C, STATE_F]:
                self.become_follower(term)

        if self.current_term == term:
            if self.vote_for == -1:
                self.vote_for = server_id
                reply = RequestVoteReply(term, True)
            else:
                reply = RequestVoteReply(term, False)
        elif self.current_term > term:
            reply = RequestVoteReply(self.current_term, False)
        else:
            ...

        self.info(f'... RequestVote reply {reply}')
        return reply

```

节点在收到候选者的请求投票消息后，先检查自己当前任期是否过期，如果过期了，降级成跟随者；否则，开始投票，一个任期只能给投一次票。

#### 心跳消息

```
class ConsensusModule:

    ...

    def append_entries(self, term, leader_id):
        self.info(f"got AppendEntries from server#{leader_id} at term: {term}")

        if self.current_term < term:
            self.info(f"... current_term is out of date, current: {self.current_term}, request: {term}")
            if self.state in [STATE_L, STATE_C, STATE_F]:
                self.become_follower(term)

        reply = AppendEntriesReply(term, False, self.server_id)
        if self.current_term == term:
            reply.success = True
            if self.state == STATE_F:
                # reset timer when got heartbeats from Leader
                self.election_reset_event = time.time()
                self.info(f"reset election event")
        elif self.current_term > term:
            reply = AppendEntriesReply(self.current_term, False, self.server_id)
        else:
            ...

        self.debug(f'... AppendEntries reply {reply}')
        return reply
```

节点在收到领导者的心跳消息后，先检查自己当前任期是否过期，如果过期了，降级成跟随者；如果已经是追随者了，就重置定时器；其他状态节点可以忽略该消息。

### 日志可视化

程序运行会产生比较多的日志，同时多个节点的日志相互交织，不容易排查问题，把日志以表格的形式展现会比较直观。

![raft log](../../assets/images/2024/04/raft-log.png "raft log")

![raft log viewer](../../assets/images/2024/04/raft-log-viewer.png "raft log viewer")


通过可视化的日志，再结合一些测试脚本，就能模拟 leader 选举，leader 下线后选举新 leader，网络异常这些场景，从而对整个选举过程有比较深入的理解。

完整代码请参考：[https://github.com/xiez/learn-to-raft/tree/leader-election/py](https://github.com/xiez/learn-to-raft/tree/leader-election/py)

## 特殊情况

目前的选举算法虽然能够在各种场景下选举出了 leader，但在某些场景，会产生运行正常的 leader 被下线并重新选举的情况。

![new leader](../../assets/images/2024/04/raft-new-leader.png "new leader")

如上图，A，B，C 3个节点，其中 A 为 leader，各自的任期都为 1。此时，网络异常导致 B 从集群中消失，A 和 C 正常运行，集群也能正常的接收客户的消息。但是 B 会进入选举状态成为「候选者」不停的请求投票，一段时间后，B 的任期递增到了 10。此时，网络异常恢复，B 又回到了集群，这个时候，就有可能产生一种情况：

由于 B 的任期较高，拒绝了来自「领导者」 A 的心跳消息，开始请求投票；A 在收到 B 的消息后，发现自己的任期过期了，自动降级成了「跟随者」，最后 B 成为了新的「领导者」。

这种多余的选举目前无法避免，在下一篇文章里引入了「日志复制」（log replication）后，该问题可以有效解决。

## 参考资料

- [https://raft.github.io/raft.pdf](https://raft.github.io/raft.pdf)

- [https://www.youtube.com/watch?v=YbZ3zDzDnrw&t=235s](https://www.youtube.com/watch?v=YbZ3zDzDnrw&t=235s)

- [https://eli.thegreenplace.net/2020/implementing-raft-part-1-elections/](https://eli.thegreenplace.net/2020/implementing-raft-part-1-elections/)

