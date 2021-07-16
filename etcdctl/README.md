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

DEFRAG defragments the backend database file for a set of given endpoints while etcd is running, or directly defragments an etcd data directory while etcd is not running. When an etcd member reclaims storage space from deleted and compacted keys, the space is kept in a free list and the database file remains the same size. By defragmenting the database, the etcd member releases this free space back to the file system.

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

## Concurrency commands

### LOCK [options] \<lockname\> [command arg1 arg2 ...]

LOCK acquires a distributed mutex with a given name. Once the lock is acquired, it will be held until etcdctl is terminated.

#### Options

- ttl - time out in seconds of lock session.

#### Output

Once the lock is acquired but no command is given, the result for the GET on the unique lock holder key is displayed.

If a command is given, it will be executed with environment variables `ETCD_LOCK_KEY` and `ETCD_LOCK_REV` set to the lock's holder key and revision.

#### Example

Acquire lock with standard output display:

```bash
./etcdctl lock mylock
# mylock/1234534535445
```

Acquire lock and execute `echo lock acquired`:

```bash
./etcdctl lock mylock echo lock acquired
# lock acquired
```

Acquire lock and execute `etcdctl put` command
```bash
./etcdctl lock mylock ./etcdctl put foo bar
# OK
```

#### Remarks

LOCK returns a zero exit code only if it is terminated by a signal and releases the lock.

If LOCK is abnormally terminated or fails to contact the cluster to release the lock, the lock will remain held until the lease expires. Progress may be delayed by up to the default lease length of 60 seconds.

### ELECT [options] \<election-name\> [proposal]

ELECT participates on a named election. A node announces its candidacy in the election by providing
a proposal value. If a node wishes to observe the election, ELECT listens for new leaders values.
Whenever a leader is elected, its proposal is given as output.

#### Options

- listen -- observe the election.

#### Output

- If a candidate, ELECT displays the GET on the leader key once the node is elected election.

- If observing, ELECT streams the result for a GET on the leader key for the current election and all future elections.

#### Example

```bash
./etcdctl elect myelection foo
# myelection/1456952310051373265
# foo
```

#### Remarks

ELECT returns a zero exit code only if it is terminated by a signal and can revoke its candidacy or leadership, if any.

If a candidate is abnormally terminated, election rogress may be delayed by up to the default lease length of 60 seconds.

## Authentication commands

### AUTH \<enable or disable\>

`auth enable` activates authentication on an etcd cluster and `auth disable` deactivates. When authentication is enabled, etcd checks all requests for appropriate authorization.

RPC: AuthEnable/AuthDisable

#### Output

`Authentication Enabled`.

#### Examples

```bash
./etcdctl user add root
# Password of root:#type password for root
# Type password of root again for confirmation:#re-type password for root
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

ROLE is used to specify different roles which can be assigned to etcd user(s).

### ROLE ADD \<role name\>

`role add` creates a role.

RPC: RoleAdd

#### Output

`Role <role name> created`.

#### Examples

```bash
./etcdctl --user=root:123 role add myrole
# Role myrole created
```

### ROLE GET \<role name\>

`role get` lists detailed role information.

RPC: RoleGet

#### Output

Detailed role information.

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

`role delete` deletes a role.

RPC: RoleDelete

#### Output

`Role <role name> deleted`.

#### Examples

```bash
./etcdctl --user=root:123 role delete myrole
# Role myrole deleted
```

### ROLE LIST \<role name\>

`role list` lists all roles in etcd.

RPC: RoleList

#### Output

A role per line.

#### Examples

```bash
./etcdctl --user=root:123 role list
# roleA
# roleB
# myrole
```

### ROLE GRANT-PERMISSION [options] \<role name\> \<permission type\> \<key\> [endkey]

`role grant-permission` grants a key to a role.

RPC: RoleGrantPermission

#### Options

- from-key -- grant a permission of keys that are greater than or equal to the given key using byte compare

- prefix -- grant a prefix permission

#### Output

`Role <role name> updated`.

#### Examples

Grant read and write permission on the key `foo` to role `myrole`:

```bash
./etcdctl --user=root:123 role grant-permission myrole readwrite foo
# Role myrole updated
```

Grant read permission on the wildcard key pattern `foo/*` to role `myrole`:

```bash
./etcdctl --user=root:123 role grant-permission --prefix myrole readwrite foo/
# Role myrole updated
```

### ROLE REVOKE-PERMISSION \<role name\> \<permission type\> \<key\> [endkey]

`role revoke-permission` revokes a key from a role.

RPC: RoleRevokePermission

#### Options

- from-key -- revoke a permission of keys that are greater than or equal to the given key using byte compare

- prefix -- revoke a prefix permission

#### Output

`Permission of key <key> is revoked from role <role name>` for single key. `Permission of range [<key>, <endkey>) is revoked from role <role name>` for a key range. Exit code is zero.

#### Examples

```bash
./etcdctl --user=root:123 role revoke-permission myrole foo
# Permission of key foo is revoked from role myrole
```

### USER \<subcommand\>

USER provides commands for managing users of etcd.

### USER ADD \<user name or user:password\> [options]

`user add` creates a user.

RPC: UserAdd

#### Options

- interactive -- Read password from stdin instead of interactive terminal

#### Output

`User <user name> created`.

#### Examples

```bash
./etcdctl --user=root:123 user add myuser
# Password of myuser: #type password for my user
# Type password of myuser again for confirmation:#re-type password for my user
# User myuser created
```

### USER GET \<user name\> [options]

`user get` lists detailed user information.

RPC: UserGet

#### Options

- detail -- Show permissions of roles granted to the user

#### Output

Detailed user information.

#### Examples

```bash
./etcdctl --user=root:123 user get myuser
# User: myuser
# Roles:
```

### USER DELETE \<user name\>

`user delete` deletes a user.

RPC: UserDelete

#### Output

`User <user name> deleted`.

#### Examples

```bash
./etcdctl --user=root:123 user delete myuser
# User myuser deleted
```

### USER LIST

`user list` lists detailed user information.

RPC: UserList

#### Output

- List of users, one per line.

#### Examples

```bash
./etcdctl --user=root:123 user list
# user1
# user2
# myuser
```

### USER PASSWD \<user name\> [options]

`user passwd` changes a user's password.

RPC: UserChangePassword

#### Options

- interactive -- if true, read password in interactive terminal

#### Output

`Password updated`.

#### Examples

```bash
./etcdctl --user=root:123 user passwd myuser
# Password of myuser: #type new password for my user
# Type password of myuser again for confirmation: #re-type the new password for my user
# Password updated
```

### USER GRANT-ROLE \<user name\> \<role name\>

`user grant-role` grants a role to a user

RPC: UserGrantRole

#### Output

`Role <role name> is granted to user <user name>`.

#### Examples

```bash
./etcdctl --user=root:123 user grant-role userA roleA
# Role roleA is granted to user userA
```

### USER REVOKE-ROLE \<user name\> \<role name\>

`user revoke-role` revokes a role from a user

RPC: UserRevokeRole

#### Output

`Role <role name> is revoked from user <user name>`.

#### Examples

```bash
./etcdctl --user=root:123 user revoke-role userA roleA
# Role roleA is revoked from user userA
```

## Utility commands

### MAKE-MIRROR [options] \<destination\>

[make-mirror][mirror] mirrors a key prefix in an etcd cluster to a destination etcd cluster.

#### Options

- dest-cacert -- TLS certificate authority file for destination cluster

- dest-cert -- TLS certificate file for destination cluster

- dest-key -- TLS key file for destination cluster

- prefix -- The key-value prefix to mirror

- dest-prefix -- The destination prefix to mirror a prefix to a different prefix in the destination cluster

- no-dest-prefix -- Mirror key-values to the root of the destination cluster

- dest-insecure-transport -- Disable transport security for client connections

#### Output

The approximate total number of keys transferred to the destination cluster, updated every 30 seconds.

#### Examples

```
./etcdctl make-mirror mirror.example.com:2379
# 10
# 18
```

[mirror]: ./doc/mirror_maker.md

### MIGRATE [options]

Migrates keys in a v2 store to a v3 mvcc store. Users should run migration command for all members in the cluster.

#### Options

- data-dir -- Path to the data directory

- wal-dir -- Path to the WAL directory

- transformer -- Path to the user-provided transformer program (default if not provided)

#### Output

No output on success.

#### Default transformer

If user does not provide a transformer program, migrate command will use the default transformer. The default transformer transforms `storev2` formatted keys into `mvcc` formatted keys according to the following Go program:

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

#### User-provided transformer

Users can provide a customized 1:n transformer function that transforms a key from the v2 store to any number of keys in the mvcc store. The migration program writes JSON formatted [v2 store keys][v2key] to the transformer program's stdin, reads protobuf formatted [mvcc keys][v3key] back from the transformer program's stdout, and finishes migration by saving the transformed keys into the mvcc store.

The provided transformer should read until EOF and flush the stdout before exiting to ensure data integrity.

#### Example

```
./etcdctl migrate --data-dir=/var/etcd --transformer=k8s-transformer
# finished transforming keys
```

### VERSION

Prints the version of etcdctl.

#### Output

Prints etcd version and API version.

#### Examples

```bash
./etcdctl version
# etcdctl version: 3.1.0-alpha.0+git
# API version: 3.1
```

### CHECK \<subcommand\>

CHECK provides commands for checking properties of the etcd cluster.

### CHECK PERF [options]

CHECK PERF checks the performance of the etcd cluster for 60 seconds. Running the `check perf` often can create a large keyspace history which can be auto compacted and defragmented using the `--auto-compact` and `--auto-defrag` options as described below.

RPC: CheckPerf

#### Options

- load -- the performance check's workload model. Accepted workloads: s(small), m(medium), l(large), xl(xLarge)

- prefix -- the prefix for writing the performance check's keys.

- auto-compact -- if true, compact storage with last revision after test is finished.

- auto-defrag -- if true, defragment storage after test is finished.

#### Output

Prints the result of performance check on different criteria like throughput. Also prints an overall status of the check as pass or fail.

#### Examples

Shows examples of both, pass and fail, status. The failure is due to the fact that a large workload was tried on a single node etcd cluster running on a laptop environment created for development and testing purpose.

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

CHECK DATASCALE checks the memory usage of holding data for different workloads on a given server endpoint. Running the `check datascale` often can create a large keyspace history which can be auto compacted and defragmented using the `--auto-compact` and `--auto-defrag` options as described below.

RPC: CheckDatascale

#### Options

- load -- the datascale check's workload model. Accepted workloads: s(small), m(medium), l(large), xl(xLarge)

- prefix -- the prefix for writing the datascale check's keys.

- auto-compact -- if true, compact storage with last revision after test is finished.

- auto-defrag -- if true, defragment storage after test is finished.

#### Output

Prints the system memory usage for a given workload. Also prints status of compact and defragment if related options are passed.

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

For all commands, a successful execution return a zero exit code. All failures will return non-zero exit codes.

## Output formats

All commands accept an output format by setting `-w` or `--write-out`. All commands default to the "simple" output format, which is meant to be human-readable. The simple format is listed in each command's `Output` description since it is customized for each command. If a command has a corresponding RPC, it will respect all output formats.

If a command fails, returning a non-zero exit code, an error string will be written to standard error regardless of output format.

### Simple

A format meant to be easy to parse and human-readable. Specific to each command.

### JSON

The JSON encoding of the command's [RPC response][etcdrpc]. Since etcd's RPCs use byte strings, the JSON output will encode keys and values in base64.

Some commands without an RPC also support JSON; see the command's `Output` description.

### Protobuf

The protobuf encoding of the command's [RPC response][etcdrpc]. If an RPC is streaming, the stream messages will be concetenated. If an RPC is not given for a command, the protobuf output is not defined.

### Fields

An output format similar to JSON but meant to parse with coreutils. For an integer field named `Field`, it writes a line in the format `"Field" : %d` where `%d` is go's integer formatting. For byte array fields, it writes `"Field" : %q` where `%q` is go's quoted string formatting (e.g., `[]byte{'a', '\n'}` is written as `"a\n"`).

## Compatibility Support

etcdctl is still in its early stage. We try out best to ensure fully compatible releases, however we might break compatibility to fix bugs or improve commands. If we intend to release a version of etcdctl with backward incompatibilities, we will provide notice prior to release and have instructions on how to upgrade.

### Input Compatibility

Input includes the command name, its flags, and its arguments. We ensure backward compatibility of the input of normal commands in non-interactive mode.

### Output Compatibility

Output includes output from etcdctl and its exit code. etcdctl provides `simple` output format by default.
We ensure compatibility for the `simple` output format of normal commands in non-interactive mode. Currently, we do not ensure
backward compatibility for `JSON` format and the format in non-interactive mode. Currently, we do not ensure backward compatibility of utility commands.

### TODO: compatibility with etcd server

[etcd]: https://github.com/coreos/etcd
[READMEv2]: READMEv2.md
[v2key]: ../store/node_extern.go#L28-L37
[v3key]: ../mvcc/mvccpb/kv.proto#L12-L29
[etcdrpc]: ../etcdserver/etcdserverpb/rpc.proto
[storagerpc]: ../mvcc/mvccpb/kv.proto
