### 一、主从集群
单节点Redis的并发能力有上限，要进一步提高Redis的并发，需要搭建主从集群，实现读写分离。
![1759333915077](image/Redis面试/1759333915077.png)
####  **1. 建立主从集群**:
1. 利用docker创建3个redis实例
```yaml
version: "3.2"

services:
  r1:
    image: redis
    container_name: r1
    network_mode: "host"
    entrypoint: ["redis-server", "--port", "7001"]
  r2:
    image: redis
    container_name: r2
    network_mode: "host"
    entrypoint: ["redis-server", "--port", "7002"]
  r3:
    image: redis
    container_name: r3
    network_mode: "host"
    entrypoint: ["redis-server", "--port", "7003"]
```
2. 将上述配置文件保存为docker-compose.yml，存入/root/redis目录下，并运行`docker-compose up -d`
3. 可使用`ps -ef | grep redis`查看运行中的redis实例
4. 配置主从关系
```bash
# 进入r1实例
docker exec -it r1 redis-cli -p 7001
# 可通过命令查看当前实例
info replication # 若没有配置过主从关系，默认都是主
# 退出
exit
# 配置r2实例为r1的从实例
docker exec -it r2 redis-cli -p 7002
# Redis5.0以前
slaveof <r1的ip地址> <r1的端口号>
# Redis5.0以后还可以使用:
replicaof <r1的ip地址> <r1的端口号>
# r3配置过程和r2一样
```
#### **2. 主从同步原理**
当主从第一次同步连接或断开重连时，从节点都会发送psync请求，尝试同步数据。
master判断slave是否第一次同步，靠`replicationID`: 每个master节点都有自己唯一的id，简称`replid`
`offset`: repl_backlog中写入过的数据长度，写操作越多，offset值越大，主从的offset一致，表示数据一致。
![1759336769055](image/Redis面试/1759336769055.png)

#### **3. 主从同步优化**
**全量同步**: master将完整的内存数据生成RDB文件，并保存在磁盘中，发送RDB文件给从节点。
**增量同步**: slave提交offset到master，master获取repl_backlog中slave的offset之后的命令给slave。
当slave结点断开又恢复，并且在repl_backlog中能找到offset后，则会触发增量同步。
若从节点长时间宕机，repl_baklog中的offset已经被覆盖时，此时就会触发全量同步。
为了避免使用全量同步，做出如下优化:
- 在master结点中配置`repl-diskless-sync`为`yes`，启用无磁盘复制，避免全量同步时的磁盘IO。
- Redis单节点上的内存占用不能太大，减少RDB导致的过多磁盘IO
- 提高`repl_baklog`的大小, 发现slave宕机时尽快修复，避免触发全量同步
- 限制一个master结点上的slave结点数量，如果有太多slave，可以采用主-从-从的链式结构，减少master压力。
  ![1759338362898](image/Redis面试/1759338362898.png)
  但当使用主-从-从结构时，会产生时效性的问题，即从结点与主结点之间的数据延迟。需要对实际情况做出取舍。
#### **4. 哨兵原理**
Redis 提供了哨兵(Sentinel)机制来实现主从集群的自动故障恢复。具体作用如下:
- **监控**: Sentinel会不断检查master和slave
- **自动故障切换**: 当master结点出现故障时，Sentinel会自动将slave结点提升为master结点，并重新配置slave结点。当故障实例恢复后，也以新的master为主。
- **通知**: 当集群发生故障转移时，Sentinel会将最新的节点角色信息推送给Redis的客户端。
 ![1759339542387](image/Redis面试/1759339542387.png)
Sentinel是基于心跳机制检测服务状态，每隔1秒向集群的每个实例发送ping命令:
- **主观下线:** 若某Sentinel的节点发现某实例未在规定时间响应，则认为该实例主观下线。
- **客观下线:** 若超过指定数量(quorum)的Sentinel认为某实例主观下线，则认为该实例客观下线。quorum的值最好超过Sentinel数量的一半。
Sentinel选举新master:
- 首先会判断slave节点于master节点断开时间长短，若超过指定值(down-after-milliseconds * 10)则会排除该slave节点。
- 然后判断slave节点的slave-prority值，越小优先级越高，若等于0则永不参与选举。
- 若slave-prority一样，则判断slave节点的offset值，越大则数据越新，优先级越高。
- 最后判断slave节点的运行id大小，越小则优先级越高。
实现故障转移:
当选中了一个slave为新的master后，故障转移步骤如下:
- sentinel给备选的slave节点发送`slaveof no one`命令，将slave节点变为master节点。
- sentinel给所有其他slave发送`slaveof <新master的ip> <新master的端口号>`命令，将所有slave节点的master信息改为新的master, 从新的master节点开始接受数据。
- sentinel将故障节点标记为slave，当故障节点恢复后会自动成为新的master的slave节点。
 ![1759343778541](image/Redis面试/1759343778541.png)
#### **5. 搭建哨兵集群**
1. 停止已经启动的redis实例：
```bash
# 老版本DockerCompose
docker-compose down

# 新版本Docker
docker compose down
```
2. 准备配置文件sentinel.conf:
```bash
sentinel announce-ip "192.168.150.101"
sentinel monitor hmaster 192.168.150.101 7001 2
sentinel down-after-milliseconds hmaster 5000
sentinel failover-timeout hmaster 60000
```
参数说明:
- `sentinel announce-ip "192.168.3.54"`：声明当前sentinel的ip
- `sentinel monitor hmaster 192.168.3.54 7001 2`：指定集群的主节点信息 
  - `hmaster`：主节点名称，自定义，任意写
  - `192.168.3.54 7001`：主节点的ip和端口
  - `2`：认定master下线时的quorum值
- `sentinel down-after-milliseconds hmaster 5000`：声明master节点超时多久后被标记下线
- `sentinel failover-timeout hmaster 60000`：在第一次故障转移失败后多久再次重试
3. 在虚拟机的/root/redis目录下新建3个文件夹：s1、s2、s3: `mkdir s1 s2 s3`
   将sentinel.conf文件分别拷贝一份到3个文件夹中。
4. 修改docker-compose.yaml文件，内容如下：
```yaml
version: "3.2"

services:
  r1:
    image: redis
    container_name: r1
    network_mode: "host"
    entrypoint: ["redis-server", "--port", "7001"]
  r2:
    image: redis
    container_name: r2
    network_mode: "host"
    entrypoint: ["redis-server", "--port", "7002", "--slaveof", "192.168.150.101", "7001"]
  r3:
    image: redis
    container_name: r3
    network_mode: "host"
    entrypoint: ["redis-server", "--port", "7003", "--slaveof", "192.168.150.101", "7001"]
  s1:
    image: redis
    container_name: s1
    volumes:
      - /root/redis/s1:/etc/redis
    network_mode: "host"
    entrypoint: ["redis-sentinel", "/etc/redis/sentinel.conf", "--port", "27001"]
  s2:
    image: redis
    container_name: s2
    volumes:
      - /root/redis/s2:/etc/redis
    network_mode: "host"
    entrypoint: ["redis-sentinel", "/etc/redis/sentinel.conf", "--port", "27002"]
  s3:
    image: redis
    container_name: s3
    volumes:
      - /root/redis/s3:/etc/redis
    network_mode: "host"
    entrypoint: ["redis-sentinel", "/etc/redis/sentinel.conf", "--port", "27003"]
```
5. 启动集群：`docker-compose up -d`
#### **6. 分片集群**
主从和哨兵可以解决高可用、高并发读写问题，但仍有两个问题没有解决:
- 海量数据存储问题
- 高并发写的问题
使用分片集群可解决上述问题，分片集群特征:
- 集群有多个master节点, 每个master节点存储一部分数据
- 每个master节点有若干个slave节点
- master之间通过ping监测彼此健康状态
![1759347283865](image/Redis面试/1759347283865.png)
1. 分片集群搭建
新建一个docker-compose.yaml文件，内容如下：
```yaml
version: "3.2"

services:
  r1:
    image: redis
    container_name: r1
    network_mode: "host"
    entrypoint: ["redis-server", "--port", "7001", "--cluster-enabled", "yes", "--cluster-config-file", "node.conf"]
  r2:
    image: redis
    container_name: r2
    network_mode: "host"
    entrypoint: ["redis-server", "--port", "7002", "--cluster-enabled", "yes", "--cluster-config-file", "node.conf"]
  r3:
    image: redis
    container_name: r3
    network_mode: "host"
    entrypoint: ["redis-server", "--port", "7003", "--cluster-enabled", "yes", "--cluster-config-file", "node.conf"]
  r4:
    image: redis
    container_name: r4
    network_mode: "host"
    entrypoint: ["redis-server", "--port", "7004", "--cluster-enabled", "yes", "--cluster-config-file", "node.conf"]
  r5:
    image: redis
    container_name: r5
    network_mode: "host"
    entrypoint: ["redis-server", "--port", "7005", "--cluster-enabled", "yes", "--cluster-config-file", "node.conf"]
  r6:
    image: redis
    container_name: r6
    network_mode: "host"
    entrypoint: ["redis-server", "--port", "7006", "--cluster-enabled", "yes", "--cluster-config-file", "node.conf"]
```
**注意:** 使用Docker部署Redis集群，network模式必须采用host
2. 进入/root/redis-cluster目录，使用命令启动redis：`docker-compose up -d`
3. 创建集群:
```bash
# 进入任意节点容器
docker exec -it r1 bash
# 然后，执行命令
redis-cli --cluster create --cluster-replicas 1 \
192.168.150.101:7001 192.168.150.101:7002 192.168.150.101:7003 \
192.168.150.101:7004 192.168.150.101:7005 192.168.150.101:7006
```
命令说明：
- redis-cli --cluster：代表集群操作命令
- create：代表是创建集群
- --cluster-replicas 1 ：指定集群中每个master的副本个数为1
  - 此时节点总数 ÷ (replicas + 1) 得到的就是master的数量n。因此节点列表中的前n个节点就是master，其它节点都是slave节点，随机分配到不同master
4. 查看集群状态: `redis-cli -p 7001 cluster nodes`
   