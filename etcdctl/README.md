etcdctl
========

`etcdctl` is a command line client for [etcd][etcd].

The v3 API is used by default on master branch. For the v2 API, make sure to set environment variable `ETCDCTL_API=2`. See also [READMEv2][READMEv2].

If using released versions earlier than v3.4, set `ETCDCTL_API=3` to use v3 API.

Global flags (e.g., `dial-timeout`, `--cacert`, `--cert`, `--key`) can be set with environment variables:

```
ETCDCTL_DIAL_TIMEOUT=3s
ETCDCTL_CACERT=/tmp/ca.pem
ETCDCTL_CERT=/tmp/cert.pem
ETCDCTL_KEY=/tmp/key.pem
```

Prefix flag strings with `ETCDCTL_`, convert all letters to upper-case, and replace dash(`-`) with underscore(`_`).

## Key-value commands

### PUT [options] \<key\> \<value\>

PUT assigns the specified value with the specified key. If key already holds a value, it is overwritten.

RPC: Put

#### Options

- lease -- lease ID (in hexadecimal) to attach to the key.

- prev-kv -- return the previous key-value pair before modification.

- ignore-value -- updates the key using its current value.

- ignore-lease -- updates the key using its current lease.

#### Output

`OK`

#### Examples

```bash
./etcdctl put foo bar --lease=1234abcd
# OK
./etcdctl get foo
# foo
# bar
./etcdctl put foo --ignore-value # to detache lease
# OK
```

```bash
./etcdctl put foo bar --lease=1234abcd
# OK
./etcdctl put foo bar1 --ignore-lease # to use existing lease 1234abcd
# OK
./etcdctl get foo
# foo
# bar1
```

```bash
./etcdctl put foo bar1 --prev-kv
# OK
# foo
# bar
./etcdctl get foo
# foo
# bar1
```

#### Remarks

If \<value\> isn't given as command line argument, this command tries to read the value from standard input.

When \<value\> begins with '-', \<value\> is interpreted as a flag.
Insert '--' for workaround:

```bash
./etcdctl put <key> -- <value>
./etcdctl put -- <key> <value>
```

Providing \<value\> in a new line after using `carriage return` is not supported and etcdctl may hang in that case. For example, following case is not supported:

```bash
./etcdctl put <key>\r
<value>
```

A \<value\> can have multiple lines or spaces but it must be provided with a double-quote as demonstrated below:

```bash
./etcdctl put foo "bar1 2 3"
```

### GET [options] \<key\> [range_end]

GET gets the key or a range of keys [key, range_end) if range_end is given.

RPC: Range

#### Options

- hex -- print out key and value as hex encode string

- limit -- maximum number of results

- prefix -- get keys by matching prefix

- order -- order of results; ASCEND or DESCEND

- sort-by -- sort target; CREATE, KEY, MODIFY, VALUE, or VERSION

- rev -- specify the kv revision

- print-value-only -- print only value when used with write-out=simple

- consistency -- Linearizable(l) or Serializable(s)

- from-key -- Get keys that are greater than or equal to the given key using byte compare

- keys-only -- Get only the keys

#### Output

\<key\>\n\<value\>\n\<next_key\>\n\<next_value\>...

#### Examples

First, populate etcd with some keys:

```bash
./etcdctl put foo bar
# OK
./etcdctl put foo1 bar1
# OK
./etcdctl put foo2 bar2
# OK
./etcdctl put foo3 bar3
# OK
```

Get the key named `foo`:

```bash
./etcdctl get foo
# foo
# bar
```

Get all keys:

```bash
./etcdctl get --from-key ''
# foo
# bar
# foo1
# bar1
# foo2
# foo2
# foo3
# bar3
```

Get all keys with names greater than or equal to `foo1`:

```bash
./etcdctl get --from-key foo1
# foo1
# bar1
# foo2
# bar2
# foo3
# bar3
```

Get keys with names greater than or equal to `foo1` and less than `foo3`:

```bash
./etcdctl get foo1 foo3
# foo1
# bar1
# foo2
# bar2
```

#### Remarks

If any key or value contains non-printable characters or control characters, simple formatted output can be ambiguous due to new lines. To resolve this issue, set `--hex` to hex encode all strings.

### DEL [options] \<key\> [range_end]

删除指定的key, 如果给出 range_end，删除[key, range_end)之间的所有key。
RPC: DeleteRange

#### Options

- prefix -- 通过匹配前缀删除键

- prev-kv -- 返回已删除的key-value键值对

- from-key -- 删除大于等于指定key的所有key，使用字节排序

#### 输出

如果 DEL 成功，则以十进制打印删除的key的数量。

#### 样例

```bash
./etcdctl put foo bar
# OK
./etcdctl del foo
# 1
./etcdctl get foo
```

```bash
./etcdctl put key val
# OK
./etcdctl del --prev-kv key
# 1
# key
# val
./etcdctl get key
```

```bash
./etcdctl put a 123
# OK
./etcdctl put b 456
# OK
./etcdctl put z 789
# OK
./etcdctl del --from-key a
# 3
./etcdctl get --from-key a
```

```bash
./etcdctl put zoo val
# OK
./etcdctl put zoo1 val1
# OK
./etcdctl put zoo2 val2
# OK
./etcdctl del --prefix zoo
# 3
./etcdctl get zoo2
```

### TXN [options]

TXN 从标准输入读取多个 etcd 请求并在一个原子性事务中执行他们。
事务由条件列表、所有条件为真时要执行的请求列表以及任意条件为假时要执行的请求列表组成。

RPC: Txn

#### Options

- hex -- 将key和value以十六进制编码字符串打印。

- interactive -- 交互式输入事务。

#### 输入格式
```ebnf
<Txn> ::= <CMP>* "\n" <THEN> "\n" <ELSE> "\n"
<CMP> ::= (<CMPCREATE>|<CMPMOD>|<CMPVAL>|<CMPVER>|<CMPLEASE>) "\n"
<CMPOP> ::= "<" | "=" | ">"
<CMPCREATE> := ("c"|"create")"("<KEY>")" <CMPOP> <REVISION>
<CMPMOD> ::= ("m"|"mod")"("<KEY>")" <CMPOP> <REVISION>
<CMPVAL> ::= ("val"|"value")"("<KEY>")" <CMPOP> <VALUE>
<CMPVER> ::= ("ver"|"version")"("<KEY>")" <CMPOP> <VERSION>
<CMPLEASE> ::= "lease("<KEY>")" <CMPOP> <LEASE>
<THEN> ::= <OP>*
<ELSE> ::= <OP>*
<OP> ::= ((see put, get, del etcdctl command syntax)) "\n"
<KEY> ::= (%q formatted string)
<VALUE> ::= (%q formatted string)
<REVISION> ::= "\""[0-9]+"\""
<VERSION> ::= "\""[0-9]+"\""
<LEASE> ::= "\""[0-9]+\""
```

#### 输出

`SUCCESS` 表示etcd处理了事务成功列表, `FAILURE` 表示etcd处理了事务成功列表. 打印每个请求的输出，以空行分隔。

#### 样例

txn交互式:
```bash
./etcdctl txn -i
# compares:
mod("key1") > "0"

# success requests (get, put, delete):
put key1 "overwrote-key1"

# failure requests (get, put, delete):
put key1 "created-key1"
put key2 "some extra key"

# FAILURE

# OK

# OK
```

txn in non-interactive mode:
```bash
./etcdctl txn <<<'mod("key1") > "0"

put key1 "overwrote-key1"

put key1 "created-key1"
put key2 "some extra key"

'

# FAILURE

# OK

# OK
```

#### Remarks

在TXN命令中使用多行值时，换行符必须表示为`\n`。 文字换行符将导致解析失败。 这与其他命令（例如 PUT）不同，在这些命令中，shell 将为我们转换文字换行符。 例如：

```bash
./etcdctl txn <<<'mod("key1") > "0"

put key1 "overwrote-key1"

put key1 "created-key1"
put key2 "this is\na multi-line\nvalue"

'

# FAILURE

# OK

# OK
```

### COMPACTION [options] \<revision\>

COMPACTION 丢弃指定版本之前的所有etcd事件历史记录。 由于 etcd 使用多版本并发控制(mvcc)模式，
它将所有key的更新保留为事件历史记录。 当不再需要某些修订的事件历史记录时，
可以压缩所有被取代的键以回收etcd后端数据库中的存储空间。

RPC: Compact

#### Options

- physical -- 'true' 等待物理上移除所有旧版本的压缩

#### Output

输出已压缩的版本号

#### Example
```bash
./etcdctl compaction 1234
# compacted revision 1234
```

### WATCH [options] [key or prefix] [range_end] [--] [exec-command arg1 arg2 ...]

Watch 在多个key、 多个前缀、如果指定了range_end，在[key or prefix, range_end) 范围内watch事件流。watch 命令会一直运行，直到遇到错误或被用户终止。 如果给出 range_end，它必须按字典顺序大于key或"\x00"。

RPC: Watch

#### Options

- hex -- 以十六进制编码字符串打印key和value

- interactive -- 开启交互式watch会话

- prefix -- 如果前缀设置了，在此前缀上watch

- prev-kv -- 在事件发生前，获取上一个kye-value键值对

- rev -- 从某个版本开始watch。想观察过去的事件指定版本是很有用的。

#### 输入格式

仅在交互式下接受输入。

```
watch [options] <key or prefix>\n
```

#### 输出

\<event\>[\n\<old_key\>\n\<old_value\>]\n\<key\>\n\<value\>\n\<event\>\n\<next_key\>\n\<next_value\>\n...

#### 样例

##### 非交互式

```bash
./etcdctl watch foo
# PUT
# foo
# bar
```

```bash
ETCDCTL_WATCH_KEY=foo ./etcdctl watch
# PUT
# foo
# bar
```

接收到事件时执行 `echo watch event received`命令:

```bash
./etcdctl watch foo -- echo watch event received
# PUT
# foo
# bar
# watch event received
```

通过设置`ETCD_WATCH_*`环境变量观察响应:

```bash
./etcdctl watch foo -- sh -c "env | grep ETCD_WATCH_"

# PUT
# foo
# bar
# ETCD_WATCH_REVISION=11
# ETCD_WATCH_KEY="foo"
# ETCD_WATCH_EVENT_TYPE="PUT"
# ETCD_WATCH_VALUE="bar"
```

使用环境变量Watch并执行`echo watch event received`:

```bash
export ETCDCTL_WATCH_KEY=foo
./etcdctl watch -- echo watch event received
# PUT
# foo
# bar
# watch event received
```

```bash
export ETCDCTL_WATCH_KEY=foo
export ETCDCTL_WATCH_RANGE_END=foox
./etcdctl watch -- echo watch event received
# PUT
# fob
# bar
# watch event received
```

##### 交互式

```bash
./etcdctl watch -i
watch foo
watch foo
# PUT
# foo
# bar
# PUT
# foo
# bar
```

接收到事件时执行 `echo watch event received`:

```bash
./etcdctl watch -i
watch foo -- echo watch event received
# PUT
# foo
# bar
# watch event received
```

使用环境变量Watch并执行`echo watch event received`:

```bash
export ETCDCTL_WATCH_KEY=foo
./etcdctl watch -i
watch -- echo watch event received
# PUT
# foo
# bar
# watch event received
```

```bash
export ETCDCTL_WATCH_KEY=foo
export ETCDCTL_WATCH_RANGE_END=foox
./etcdctl watch -i
watch -- echo watch event received
# PUT
# fob
# bar
# watch event received
```

### LEASE \<subcommand\>

LEASE 提供了key的租约管理命令。

### LEASE GRANT \<ttl\>

LEASE GRANT 创建一个新的租约，服务器设置其生存时间（单位秒）大于或等于请求的 TTL 值。

RPC: LeaseGrant

#### 输出

输出带有租约ID的信息

#### 样例

```bash
./etcdctl lease grant 10
# lease 32695410dcc0ca06 granted with TTL(10s)
```

### LEASE REVOKE \<leaseID\>

LEASE REVOKE 废除给定的租约，也删除所有附加的key。

RPC: LeaseRevoke

#### 输出

输出租约已被废除的信息

#### 样例

```bash
./etcdctl lease revoke 32695410dcc0ca06
# lease 32695410dcc0ca06 revoked
```

### LEASE TIMETOLIVE \<leaseID\> [options]

LEASE TIMETOLIVE 使用给定的租约id查找租约信息

RPC: LeaseTimeToLive

#### Options

- keys -- 获取附加在该租约上的key

#### Output

输出租约信息。

#### Example

```bash
./etcdctl lease grant 500
# lease 2d8257079fa1bc0c granted with TTL(500s)

./etcdctl put foo1 bar --lease=2d8257079fa1bc0c
# OK

./etcdctl put foo2 bar --lease=2d8257079fa1bc0c
# OK

./etcdctl lease timetolive 2d8257079fa1bc0c
# lease 2d8257079fa1bc0c granted with TTL(500s), remaining(481s)

./etcdctl lease timetolive 2d8257079fa1bc0c --keys
# lease 2d8257079fa1bc0c granted with TTL(500s), remaining(472s), attached keys([foo2 foo1])

./etcdctl lease timetolive 2d8257079fa1bc0c --write-out=json
# {"cluster_id":17186838941855831277,"member_id":4845372305070271874,"revision":3,"raft_term":2,"id":3279279168933706764,"ttl":465,"granted-ttl":500,"keys":null}

./etcdctl lease timetolive 2d8257079fa1bc0c --write-out=json --keys
# {"cluster_id":17186838941855831277,"member_id":4845372305070271874,"revision":3,"raft_term":2,"id":3279279168933706764,"ttl":459,"granted-ttl":500,"keys":["Zm9vMQ==","Zm9vMg=="]}

./etcdctl lease timetolive 2d8257079fa1bc0c
# lease 2d8257079fa1bc0c already expired
```

### LEASE LIST

LEASE LIST 列出所有活动中的租约

RPC: LeaseLeases

#### Output

打印活动中的租约信息。

#### 样例

```bash
./etcdctl lease grant 10
# lease 32695410dcc0ca06 granted with TTL(10s)

./etcdctl lease list
32695410dcc0ca06
```

### LEASE KEEP-ALIVE \<leaseID\>

LEASE KEEP-ALIVE 定期刷新租约，使其不会过期。

RPC: LeaseKeepAlive

#### 输出

输出每个keey-alive的发送信息或者租约已过期的信息。

#### 样例
```bash
./etcdctl lease keep-alive 32695410dcc0ca0
# lease 32695410dcc0ca0 keepalived with TTL(100)
# lease 32695410dcc0ca0 keepalived with TTL(100)
# lease 32695410dcc0ca0 keepalived with TTL(100)
...
```

## 集群维护命令

### MEMBER \<subcommand\>

MEMBER 提供了管理 etcd 集群成员的命令。

### MEMBER ADD \<memberName\> [options]

MEMBER ADD 给etcd添加新的成员。

RPC: MemberAdd

#### Options

- peer-urls -- 新成员的url列表，使用逗号分隔。

#### Output

输出新成员的成员ID和集群ID。

#### Example

```bash
./etcdctl member add newMember --peer-urls=https://127.0.0.1:12345

Member ced000fda4d05edf added to cluster 8c4281cc65c7b112

ETCD_NAME="newMember"
ETCD_INITIAL_CLUSTER="newMember=https://127.0.0.1:12345,default=http://10.0.0.30:2380"
ETCD_INITIAL_CLUSTER_STATE="existing"
```

### MEMBER UPDATE \<memberID\> [options]

MEMBER UPDATE 设置etcd集群中已存在成员的url。

RPC: MemberUpdate

#### Options

- peer-urls -- 需更新成员的url列表，以逗号分隔。
#### 输出

打印已更新成员的成员ID和集群ID。

#### 样例

```bash
./etcdctl member update 2be1eb8f84b7f63e --peer-urls=https://127.0.0.1:11112
# Member 2be1eb8f84b7f63e updated in cluster ef37ad9dc622a7c4
```

### MEMBER REMOVE \<memberID\>

MEMBER REMOVE 从etcd集群中移除一个成员。

RPC: MemberRemove

#### 输出

打印已移除成员的成员ID和集群ID。

#### 样例

```bash
./etcdctl member remove 2be1eb8f84b7f63e
# Member 2be1eb8f84b7f63e removed from cluster ef37ad9dc622a7c4
```

### MEMBER LIST

MEMBER LIST 打印etcd集群中所有成员的详细信息。

RPC: MemberList

#### Output

打印成员ID、状态、名称、peer地址和客户端地址的可读表格。

#### Examples

```bash
./etcdctl member list
# 8211f1d0f64f3269, started, infra1, http://127.0.0.1:12380, http://127.0.0.1:2379
# 91bc3c398fb3c146, started, infra2, http://127.0.0.1:22380, http://127.0.0.1:22379
# fd422379fda50e48, started, infra3, http://127.0.0.1:32380, http://127.0.0.1:32379
```

```bash
./etcdctl -w json member list
# {"header":{"cluster_id":17237436991929493444,"member_id":9372538179322589801,"raft_term":2},"members":[{"ID":9372538179322589801,"name":"infra1","peerURLs":["http://127.0.0.1:12380"],"clientURLs":["http://127.0.0.1:2379"]},{"ID":10501334649042878790,"name":"infra2","peerURLs":["http://127.0.0.1:22380"],"clientURLs":["http://127.0.0.1:22379"]},{"ID":18249187646912138824,"name":"infra3","peerURLs":["http://127.0.0.1:32380"],"clientURLs":["http://127.0.0.1:32379"]}]}
```

```bash
./etcdctl -w table member list
+------------------+---------+--------+------------------------+------------------------+
|        ID        | STATUS  |  NAME  |       PEER ADDRS       |      CLIENT ADDRS      |
+------------------+---------+--------+------------------------+------------------------+
| 8211f1d0f64f3269 | started | infra1 | http://127.0.0.1:12380 | http://127.0.0.1:2379  |
| 91bc3c398fb3c146 | started | infra2 | http://127.0.0.1:22380 | http://127.0.0.1:22379 |
| fd422379fda50e48 | started | infra3 | http://127.0.0.1:32380 | http://127.0.0.1:32379 |
+------------------+---------+--------+------------------------+------------------------+
```

### ENDPOINT \<subcommand\>

ENDPOINT 提供了查询单个endpoint的命令。

#### Options

- cluster -- 从etcd集群成员列表中获取并使用所有endpoint

### ENDPOINT HEALTH

ENDPOINT HEALTH 检查集群所有endpoint的健康状态。如果endpoint不健康，表示它无法参与到集群其他endpoint的共识算法中。

#### Output

如果端点可以参与共识算法，则打印endpoint健康的。 如果端点未能参与共识，则打印endpoint不健康。
If an endpoint can participate in consensus, prints a message indicating the endpoint is healthy. If an endpoint fails to participate in consensus, prints a message indicating the endpoint is unhealthy.

#### Example

检查默认endpoint的健康状态：

```bash
./etcdctl endpoint health
# 127.0.0.1:2379 is healthy: successfully committed proposal: took = 2.095242ms
```

检查与默认endpoint关联的集群的所有endpoint：

```bash
./etcdctl endpoint --cluster health
# http://127.0.0.1:2379 is healthy: successfully committed proposal: took = 1.060091ms
# http://127.0.0.1:22379 is healthy: successfully committed proposal: took = 903.138µs
# http://127.0.0.1:32379 is healthy: successfully committed proposal: took = 1.113848ms
```

### ENDPOINT STATUS

ENDPOINT STATUS 查询给定endpoint列表中每个endpoint的状态。

#### Output

##### Simple format

打印每个endpoint的URL、ID、版本、数据库大小、leader状态、raft任期和raft状态的可读表格。

##### JSON format

打印每个endpoint的URL、ID、版本、数据库大小、leader状态、raft任期和raft状态的一行JSON编码。

#### Examples

获取默认endpoint的状态：

```bash
./etcdctl endpoint status
# 127.0.0.1:2379, 8211f1d0f64f3269, 3.0.0, 25 kB, false, 2, 63
```

以 JSON 形式获取默认endpoint的状态：

```bash
./etcdctl -w json endpoint status
# [{"Endpoint":"127.0.0.1:2379","Status":{"header":{"cluster_id":17237436991929493444,"member_id":9372538179322589801,"revision":2,"raft_term":2},"version":"3.0.0","dbSize":24576,"leader":18249187646912138824,"raftIndex":32623,"raftTerm":2}}]
```
获取集群中与默认endpoint关联的所有endpoint的状态：

```bash
./etcdctl -w table endpoint --cluster status
+------------------------+------------------+----------------+---------+-----------+-----------+------------+
|        ENDPOINT        |        ID        |    VERSION     | DB SIZE | IS LEADER | RAFT TERM | RAFT INDEX |
+------------------------+------------------+----------------+---------+-----------+-----------+------------+
| http://127.0.0.1:2379  | 8211f1d0f64f3269 | 3.2.0-rc.1+git |   25 kB |     false |         2 |          8 |
| http://127.0.0.1:22379 | 91bc3c398fb3c146 | 3.2.0-rc.1+git |   25 kB |     false |         2 |          8 |
| http://127.0.0.1:32379 | fd422379fda50e48 | 3.2.0-rc.1+git |   25 kB |      true |         2 |          8 |
+------------------------+------------------+----------------+---------+-----------+-----------+------------+
```

### ENDPOINT HASHKV

ENDPOINT HASHKV 获取一个endPoint的key-value的hash值。

#### Output

##### Simple format

打印每个endpoint的URL和KV历史hash值。

##### JSON format

以JSON格式打印每个endpoint的URL和KV历史hash值。

#### Examples

获取默认endpoint的hash：

```bash
./etcdctl endpoint hashkv
# 127.0.0.1:2379, 1084519789
```

以 JSON 形式获取默认endpoint的状态：

```bash
./etcdctl -w json endpoint hashkv
# [{"Endpoint":"127.0.0.1:2379","Hash":{"header":{"cluster_id":14841639068965178418,"member_id":10276657743932975437,"revision":1,"raft_term":3},"hash":1084519789,"compact_revision":-1}}]
```

获取集群中与默认endpoint关联的所有endpoint的hash：

```bash
./etcdctl -w table endpoint --cluster hashkv
+------------------------+------------+
|        ENDPOINT        |    HASH    |
+------------------------+------------+
| http://127.0.0.1:2379  | 1084519789 |
| http://127.0.0.1:22379 | 1084519789 |
| http://127.0.0.1:32379 | 1084519789 |
+------------------------+------------+
```

### ALARM \<subcommand\>

提供了告警相关命令

### ALARM DISARM

`alarm disarm` 解除所有告警

RPC: Alarm

#### Output

如果存在告警并解除则输出`alarm:<alarm type>`。

#### Examples

```bash
./etcdctl alarm disarm
```

如果存在NOSPACE告警：

```bash
./etcdctl alarm disarm
# alarm:NOSPACE
```

### ALARM LIST

`alarm list` 列出所有告警。

RPC: Alarm

#### Output

如果存在告警则输出`alarm:<alarm type>` 。如果不存在则输出空字符串

#### Examples

```bash
./etcdctl alarm list
```

如果存在NOSPACE告警：

```bash
./etcdctl alarm list
# alarm:NOSPACE
```

### DEFRAG [options]

DEFRAG 在etcd运行时对一组endpoint的后端数据库文件进行碎片整理，或者在etcd未运行时直接对etcd数据目录进行碎片整理。 当etcd成员从已删除和压缩的键中回收存储空间时，该空间将保留在空闲列表中，并且数据库文件的大小保持不变。通过对数据库进行碎片整理，etcd成员将空闲空间释放给文件系统。

**请注意，对活动成员进行碎片整理会重建其状态，会阻塞读取和写入数据。**

**请注意，碎片整理请求不会复制到集群中其他节点。 也就是说，请求只应用于本地节点。 可以在`--endpoints` 中指定所有成员，或者使用 `--cluster`自动查找所有集群成员。**

#### Options

- data-dir -- 可选的。 如果存在，则对etcd未使用的数据目录进行碎片整理。

#### Output

对于每个endpoint，打印一条消息，指示endpoint是否已成功进行碎片整理。

#### Example

```bash
./etcdctl --endpoints=localhost:2379,badendpoint:2379 defrag
# Finished defragmenting etcd member[localhost:2379]
# Failed to defragment etcd member[badendpoint:2379] (grpc: timed out trying to connect)
```

为与默认endpoint关联的集群中的所有endpoint运行碎片整理操作：

```bash
./etcdctl defrag --cluster
Finished defragmenting etcd member[http://127.0.0.1:2379]
Finished defragmenting etcd member[http://127.0.0.1:22379]
Finished defragmenting etcd member[http://127.0.0.1:32379]
```

要直接对数据目录进行碎片整理，请使用 `--data-dir`：

``` bash
# Defragment while etcd is not running
./etcdctl defrag --data-dir default.etcd
# success (exit status 0)
# Error: cannot open database at default.etcd/member/snap/db
```

#### Remarks

DEFRAG 仅当对所有给定endpoint成功进行碎片整理时才返回零退出代码。

### SNAPSHOT \<subcommand\>

SNAPSHOT 提供将正在运行的etcd服务器的快照恢复到新集群中的命令。

### SNAPSHOT SAVE \<filename\>

SNAPSHOT SAVE 将etcd后端数据库的快照写入到一个文件中。

#### Output

后端快照写入到指定的文件路径。

#### Example

保存快照到 "snapshot.db":
```
./etcdctl snapshot save snapshot.db
```

### SNAPSHOT RESTORE [options] \<filename\>

SNAPSHOT RESTORE 使用后端数据库的快照和新集群的配置创建一个etcd数据库目录。 从后端数据库快照和新的集群配置为 etcd 集群成员创建一个 etcd 数据目录。 新集群的每个成员都使用快照恢复，新集群配置将初始化一个新的由快照预加载的etcd集群。

#### Options

恢复快照的选项与`etcd` 命令中用来定义集群选项非常相似。

- data-dir -- 数据目录的路径。 如果没有给出，则使用 \<name\>.etcd。

- wal-dir -- WAL 目录的路径。 如果没有给出，则使用数据目录。

- initial-cluster -- 恢复的 etcd 集群的初始集群配置。

- initial-cluster-token -- 已恢复的etcd集群的初始集群token。

- initial-advertise-peer-urls -- 正在恢复的成员的peer URL列表。

- name -- 正在恢复的etcd集群成员的名称。

- skip-hash-check -- 忽略快照完整性哈希值（如果快照是从数据目录复制的则需要）

#### Output

使用快照初始化的新etcd数据目录。

#### Example

保存快照，恢复到新的3节点集群，然后启动集群：
```
./etcdctl snapshot save snapshot.db

# 恢复成员
bin/etcdctl snapshot restore snapshot.db --initial-cluster-token etcd-cluster-1 --initial-advertise-peer-urls http://127.0.0.1:12380  --name sshot1 --initial-cluster 'sshot1=http://127.0.0.1:12380,sshot2=http://127.0.0.1:22380,sshot3=http://127.0.0.1:32380'
bin/etcdctl snapshot restore snapshot.db --initial-cluster-token etcd-cluster-1 --initial-advertise-peer-urls http://127.0.0.1:22380  --name sshot2 --initial-cluster 'sshot1=http://127.0.0.1:12380,sshot2=http://127.0.0.1:22380,sshot3=http://127.0.0.1:32380'
bin/etcdctl snapshot restore snapshot.db --initial-cluster-token etcd-cluster-1 --initial-advertise-peer-urls http://127.0.0.1:32380  --name sshot3 --initial-cluster 'sshot1=http://127.0.0.1:12380,sshot2=http://127.0.0.1:22380,sshot3=http://127.0.0.1:32380'

# 运行成员
bin/etcd --name sshot1 --listen-client-urls http://127.0.0.1:2379 --advertise-client-urls http://127.0.0.1:2379 --listen-peer-urls http://127.0.0.1:12380 &
bin/etcd --name sshot2 --listen-client-urls http://127.0.0.1:22379 --advertise-client-urls http://127.0.0.1:22379 --listen-peer-urls http://127.0.0.1:22380 &
bin/etcd --name sshot3 --listen-client-urls http://127.0.0.1:32379 --advertise-client-urls http://127.0.0.1:32379 --listen-peer-urls http://127.0.0.1:32380 &
```

### SNAPSHOT STATUS \<filename\>

SNAPSHOT STATUS 列出给定后端数据库快照文件的信息。

#### Output

##### Simple format

打印数据库hash、版本、键数量和大小。

##### JSON format

已JSON格式打印数据库hash、版本、键数量和大小。

#### Examples
```bash
./etcdctl snapshot status file.db
# cf1550fb, 3, 3, 25 kB
```

```bash
./etcdctl -write-out=json snapshot status file.db
# {"hash":3474280699,"revision":3,"totalKey":3,"totalSize":24576}
```

```bash
./etcdctl -write-out=table snapshot status file.db
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| cf1550fb |        3 |          3 | 25 kB      |
+----------+----------+------------+------------+
```

### MOVE-LEADER \<hexadecimal-transferee-id\>

MOVE-LEADER 切换leader。

#### Example

```bash
# 选择一个即将成为leader的member
transferee_id=$(./etcdctl \
  --endpoints localhost:2379,localhost:22379,localhost:32379 \
  endpoint status | grep -m 1 "false" | awk -F', ' '{print $2}')
echo ${transferee_id}
# c89feb932daef420

# endpoints应该包含leader节点
./etcdctl --endpoints ${transferee_ep} move-leader ${transferee_id}
# Error:  no leader endpoint given at [localhost:22379 localhost:32379]

# 将leader切换到指定member ID
./etcdctl --endpoints ${leader_ep} move-leader ${transferee_id}
# Leadership transferred from 45ddc0e800e20b93 to c89feb932daef420
```

## 并发命令

### LOCK [options] \<lockname\> [command arg1 arg2 ...]

LOCK 获取具有给定名称的分布式互斥锁。 一旦获得了锁，它就会一直保持到 etcdctl 终止。

#### Options

- ttl - 锁定会话的超时秒数超时。

#### Output

Once the lock is acquired but no command is given, the result for the GET on the unique lock holder key is displayed.
如果获取了锁但没有给出命令，就会显示对已锁的key的GET结果。

If a command is given, it will be executed with environment variables `ETCD_LOCK_KEY` and `ETCD_LOCK_REV` set to the lock's holder key and revision.
如果给出命令，此命令可以使用"ETCD_LOCK_KEY"和"ETCD_LOCK_REV"环境变量来获取已锁的key及其版本。

#### Example

使用标准输出显示获取锁：

```bash
./etcdctl lock mylock
# mylock/1234534535445
```

获取锁并执行`echo lock acquired`:

```bash
./etcdctl lock mylock echo lock acquired
# lock acquired
```

获取锁并执行`etcdctl put`命令
```bash
./etcdctl lock mylock ./etcdctl put foo bar
# OK
```

#### Remarks

LOCK 仅在被信号终止并释放锁时才返回零退出代码。

如果LOCK异常终止或无法连接到集群来释放锁，锁会一直保持，直到租约过期。默认60s租约过期。

### ELECT [options] \<election-name\> [proposal]

**译者注：此选举功能不是让etcd重新选主，而是客户端为主备架构，以来ETCD进行选主**

ELECT 参加指定的选举。 一个客户端通过设置提案值来宣布要参加选举。如果客户端希望观察选举，ELECT会监听新的领导者值。当leader选出之后，将输出提案值。

#### Options

- listen -- 观察选举。

#### Output

- 如果客户端参加选举，一旦客户端当选时ELECT将显示leader的key。

- 如果客户端观察选举，ELECT将显示当前leader的key和所有未来的选举出的leader的key。

#### Example

```bash
./etcdctl elect myelection foo
# myelection/1456952310051373265
# foo
```

#### Remarks

ELECT 仅在被信号终止并且撤销了它的候选资格或领导权（如果有）时返回零退出代码。

If a candidate is abnormally terminated, election rogress may be delayed by up to the default lease length of 60 seconds.
如果候选者是被异常终止的，选举进程可能会延迟最多60秒(默认租约时长)。

## 身份认证命令

### AUTH \<enable or disable\>

`auth enable` activates authentication on an etcd cluster and `auth disable` deactivates. When authentication is enabled, etcd checks all requests for appropriate authorization.
`auth enable` 启用身份认证， `auth disable`禁用身份认证. 启用身份验证后，etcd 会检查所有请求以获得适当的授权。

RPC: AuthEnable/AuthDisable

#### Output

`Authentication Enabled`.

#### Examples

```bash
./etcdctl user add root
# Password of root:#输入root密码
# Type password of root again for confirmation:#再次输入root密码
# User root created
./etcdctl user grant-role root root
# Role root is granted to user root
./etcdctl user get root
# User: root
# Roles: root
./etcdctl role add root
# Role root created
./etcdctl role get root
# Role root
# KV Read:
# KV Write:
./etcdctl auth enable
# Authentication Enabled
```

### ROLE \<subcommand\>

ROLE 用于给etcd用户分配不同角色。

### ROLE ADD \<role name\>

`role add` 创建一个角色。

RPC: RoleAdd

#### Output

`Role <role name> created`.

#### Examples

```bash
./etcdctl --user=root:123 role add myrole
# Role myrole created
```

### ROLE GET \<role name\>

`role get` 列出详细的角色信息。

RPC: RoleGet

#### Output

详细的角色信息。

#### Examples

```bash
./etcdctl --user=root:123 role get myrole
# Role myrole
# KV Read:
# foo
# KV Write:
# foo
```

### ROLE DELETE \<role name\>

`role delete` 删除角色。

RPC: RoleDelete

#### Output

`Role <role name> deleted`.

#### Examples

```bash
./etcdctl --user=root:123 role delete myrole
# Role myrole deleted
```

### ROLE LIST \<role name\>

`role list` 列出etcd中所有角色

RPC: RoleList

#### Output

每行一个角色。

#### Examples

```bash
./etcdctl --user=root:123 role list
# roleA
# roleB
# myrole
```

### ROLE GRANT-PERMISSION [options] \<role name\> \<permission type\> \<key\> [endkey]

`role grant-permission` 给一个角色授权一个key

RPC: RoleGrantPermission

#### Options

- from-key -- 授予大于等于指定key的所有key权限，使用字节排序

- prefix -- 给一个前缀授权

#### Output

`Role <role name> updated`.

#### Examples

Grant read and write permission on the key `foo` to role `myrole`:
授予角色`myrole`对key `foo`的读写权限：

```bash
./etcdctl --user=root:123 role grant-permission myrole readwrite foo
# Role myrole updated
```

授予角色`myrole`对`foo/*`通配符的读写权限

```bash
./etcdctl --user=root:123 role grant-permission --prefix myrole readwrite foo/
# Role myrole updated
```

### ROLE REVOKE-PERMISSION \<role name\> \<permission type\> \<key\> [endkey]

`role revoke-permission` 撤销角色对一个key的权限。

RPC: RoleRevokePermission

#### Options

- from-key -- 撤销大于等于指定key的所有key权限，使用字节排序

- prefix -- 撤销一个前缀的权限

#### Output

对于单个key输出`Permission of key <key> is revoked from role <role name>` . 范围key输出`Permission of range [<key>, <endkey>) is revoked from role <role name>`。 退出码为0。

#### Examples

```bash
./etcdctl --user=root:123 role revoke-permission myrole foo
# Permission of key foo is revoked from role myrole
```

### USER \<subcommand\>

USER 提供管理 etcd 用户的命令。

### USER ADD \<user name or user:password\> [options]

`user add` creates a user.

RPC: UserAdd

#### Options

- interactive -- 从标准输入读取密码而不是交互式读取

#### Output

`User <user name> created`.

#### Examples

```bash
./etcdctl --user=root:123 user add myuser
# Password of myuser: #输入密码
# Type password of myuser again for confirmation:#再次输入密码
# User myuser created
```

### USER GET \<user name\> [options]

`user get` 列出详细的用户信息。

RPC: UserGet

#### Options

- detail -- 显示授予用户的角色权限

#### Output

用户的详细信息。

#### Examples

```bash
./etcdctl --user=root:123 user get myuser
# User: myuser
# Roles:
```

### USER DELETE \<user name\>

`user delete` 删除一个用户。

RPC: UserDelete

#### Output

`User <user name> deleted`.

#### Examples

```bash
./etcdctl --user=root:123 user delete myuser
# User myuser deleted
```

### USER LIST

`user list` 列出详细的用户信息。

RPC: UserList

#### Output

- 用户列表，每行一个。

#### Examples

```bash
./etcdctl --user=root:123 user list
# user1
# user2
# myuser
```

### USER PASSWD \<user name\> [options]

`user passwd` 更改用户的密码。

RPC: UserChangePassword

#### Options

- interactive -- 如果为真，则在交互式终端中读取密码

#### Output

`Password updated`.

#### Examples

```bash
./etcdctl --user=root:123 user passwd myuser
# Password of myuser: #输入密码
# Type password of myuser again for confirmation: #再次输入密码
# Password updated
```

### USER GRANT-ROLE \<user name\> \<role name\>

`user grant-role` 授予用户角色

RPC: UserGrantRole

#### Output

`Role <role name> is granted to user <user name>`.

#### Examples

```bash
./etcdctl --user=root:123 user grant-role userA roleA
# Role roleA is granted to user userA
```

### USER REVOKE-ROLE \<user name\> \<role name\>

`user revoke-role` 撤销用户的角色

RPC: UserRevokeRole

#### Output

`Role <role name> is revoked from user <user name>`.

#### Examples

```bash
./etcdctl --user=root:123 user revoke-role userA roleA
# Role roleA is revoked from user userA
```

## 实用命令

### MAKE-MIRROR [options] \<destination\>

[make-mirror][mirror] 将etcd集群中的key前缀镜像到目标 etcd 集群。

#### Options

- dest-cacert -- 目标集群的TLS CA证书

- dest-cert -- 目标集群的TLS证书文件

- dest-key -- 目标集群的TLS密钥文件

- prefix -- 要镜像的kye-value前缀

- dest-prefix -- 目前集群的目标最先 将前缀镜像到目标集群中不同前缀的目标前缀 The destination prefix to mirror a prefix to a different prefix in the destination cluster

- no-dest-prefix -- Mirror key-values to the root of the destination cluster

- dest-insecure-transport -- 禁用客户端连接的传输安全

#### Output

传输到目标集群的大致key总数，每30秒更新一次。

#### Examples

```
./etcdctl make-mirror mirror.example.com:2379
# 10
# 18
```

[mirror]: ./doc/mirror_maker.md

### MIGRATE [options]

Migrates keys in a v2 store to a v3 mvcc store. Users should run migration command for all members in the cluster.
将v2存储中的key迁移到v3 mvcc存储中。 用户应该为集群中的所有成员运行迁移命令。

#### Options

- data-dir -- 数据目录的路径

- wal-dir -- WAL目录的路径

- transformer -- 用户提供的转换器程序的路径（如果未提供，则为默认值）

#### Output

成功没有输出。

#### 默认的转换器

如果用户不提供转换器程序，迁移命令将使用默认转换器。 默认转换器根据以下Go程序将 `storev2` 格式的key转换为 `mvcc` 格式的key：

```go
func transform(n *storev2.Node) *mvccpb.KeyValue {
	if n.Dir {
		return nil
	}
	kv := &mvccpb.KeyValue{
		Key:            []byte(n.Key),
		Value:          []byte(n.Value),
		CreateRevision: int64(n.CreatedIndex),
		ModRevision:    int64(n.ModifiedIndex),
		Version:        1,
	}
	return kv
}
```

#### 用户提供的转换器

用户可以提供自定义的1:n转换函数，将v2存储中的key转换为mvcc存储中任意数量的key。 迁移程序将JSON格式的[v2 store keys][v2key]写入到转换器程序的标准输入中，从转换器程序的标准输出中读回 protobuf 格式的[mvcc keys][v3key]，并将转换后的key保存到mvcc存储中来完成迁移。

提供的转换器应该读取到EOF并在退出之前刷新stdout以确保数据完整性。

#### Example

```
./etcdctl migrate --data-dir=/var/etcd --transformer=k8s-transformer
# finished transforming keys
```

### VERSION

打印etcdctl的版本。

#### Output

打印etcd版本和 API 版本。

#### Examples

```bash
./etcdctl version
# etcdctl version: 3.1.0-alpha.0+git
# API version: 3.1
```

### CHECK \<subcommand\>

CHECK 提供用于检查 etcd 集群属性的命令。

### CHECK PERF [options]

CHECK PERF 检查etcd集群的性能 60 秒。 运行 `check perf` 通常会创建一个很大的key空间历史记录，可以使用如下所述的`--auto-compact` 和 `--auto-defrag` 选项自动压缩和整理碎片。

RPC: CheckPerf

#### Options

- load -- 性能检查的工作负载模型。 可接受的工作负载有：s(small)、m(medium)、l(large)、xl(xLarge)

- prefix -- 写入性能检查key的前缀。

- auto-compact -- 如果为真，则在测试完成后使用最新版本进行压缩存储。

- auto-defrag -- 如果为真，则在测试完成后对存储进行碎片整理。

#### Output

打印性能检查结果，如吞吐量等标准。也会打印总体的检查结果是通过还是失败。

#### Examples

下面是成功和失败的样例。失败的原因为在笔记本电脑环境创建笔了单节点etcd集群，它是为了开发和测试创建的，但尝试运行大量工作负载。

```bash
./etcdctl check perf --load="s"
# 60 / 60 Booooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooo! 100.00%1m0s
# PASS: Throughput is 150 writes/s
# PASS: Slowest request took 0.087509s
# PASS: Stddev is 0.011084s
# PASS
./etcdctl check perf --load="l"
# 60 / 60 Booooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooo! 100.00%1m0s
# FAIL: Throughput too low: 6808 writes/s
# PASS: Slowest request took 0.228191s
# PASS: Stddev is 0.033547s
# FAIL
```

### CHECK DATASCALE [options]

CHECK DATASCALE 在不同的工作负载情况下检查给定的服务器endpoint的内存使用情况。 运行 `check datascale` 通常会创建一个很大的key空间历史记录，可以使用如下所述的`--auto-compact` 和 `--auto-defrag` 选项自动压缩和整理碎片。

RPC: CheckDatascale

#### Options

- load -- 数据规模检查的工作负载模型。可接受的工作负载有：s(small)、m(medium)、l(large)、xl(xLarge)。

- prefix -- 数据规格检查key的前缀。

- auto-compact -- 如果为真，则在测试完成后使用最新版本进行压缩存储。

- auto-defrag -- 如果为真，则在测试完成后对存储进行碎片整理。

#### Output

打印给定工作负载的系统内存使用情况。 如果通过了相关选项，还会打印压缩和碎片整理的状态。

#### Examples

```bash
./etcdctl check datascale --load="s" --auto-compact=true --auto-defrag=true
# Start data scale check for work load [10000 key-value pairs, 1024 bytes per key-value, 50 concurrent clients].
# Compacting with revision 18346204
# Compacted with revision 18346204
# Defragmenting "127.0.0.1:2379"
# Defragmented "127.0.0.1:2379"
# PASS: Approximate system memory used : 64.30 MB.
```

## Exit codes

对于所有命令，成功执行返回零退出代码。 所有失败都将返回非零退出代码。

## Output formats

所有命令都通过`-w` 或`--write-out` 来设置输出格式。 所有命令默认为"simple"输出格式，这是人类可读的。 simple格式列在每个命令的`Output`描述中，因为它是为每个命令定制的。 如果命令有相应的 RPC，它有所有输出格式。

如果命令失败，返回非零退出码，无论输出格式如何，都会将错误字符串写入到标准错误。

### Simple

一种易于解析和人类可读的格式。 每个命令都可指定。

### JSON

命令的[RPC response][etcdrpc]的JSON编码。因为etcd的RPC使用字节，所以JSON输出将会对key和value进行base64编码。

一些没用RPC的命令也支持 SON； 请参阅命令的`Output` 说明。


### Protobuf

命令的 [RPC 响应][etcdrpc] 的 protobuf 编码。 如果 RPC 正在流式传输，则流消息将被合并。 如果未为命令提供RPC，则protobuf输出不生效。

### Fields

类似于JSON的输出格式，旨在可以用coreutils进行解析。 对于名为`Field`的整数字段，它以`"Field" : %d`格式写入一行，其中`%d`是go的整数格式。 对于字节数组字段，它写为 `"Field" : %q`，其中 `%q` 是go的加引号的字符串格式（例如，`[]byte{'a', '\n'}` 写为 `"a\n"`)。

## 兼容性支持

etcdctl仍处于早期阶段。 我们尽最大努力确保完全兼容的版本，但是我们可能会破坏兼容性以修复错误或改进命令。 如果我们打算发布向后不兼容的etcdctl版本，我们将在发布前提供通知并提供如何升级的说明。

### 输入兼容性

输入包括命令名称、选项及参数。 我们确保在非交互式下输入普通命令的向后兼容性。

### 输出兼容性

输出包括来自etcdctl的输出及其退代码。 etcdctl默认提供`simple`输出格式。
我们确保在非交互式下与普通命令的`simple`输出格式兼容。 
目前，我们不确保`JSON`格式和非交互模式下的格式的向后兼容性。目前，我们不确保实用程序命令的向后兼容性。.

### TODO: 与etcd服务器的兼容性

[etcd]: https://github.com/coreos/etcd
[READMEv2]: READMEv2.md
[v2key]: ../store/node_extern.go#L28-L37
[v3key]: ../mvcc/mvccpb/kv.proto#L12-L29
[etcdrpc]: ../etcdserver/etcdserverpb/rpc.proto
[storagerpc]: ../mvcc/mvccpb/kv.proto
