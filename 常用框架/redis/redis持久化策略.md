**Redis 支持持久化，而且支持 3 种持久化方式** :

- 快照（snapshotting，RDB）
- 只追加文件（append-only file, AOF）
- RDB 和 AOF 的混合持久化(Redis 4.0 新增)

官方文档地址：https://redis.io/topics/persistence 。

# RDB 持久化
## 什么是RDB持久化

Redis 可以通过创建快照来获得存储在内存里面的数据在 某个时间点 上的副本。Redis 创建快照之后，可以对快照进行备份，可以将快照复制到其他服务器从而创建具有相同数据的服务器副本（Redis 主从结构，主要用来提高 Redis 性能），还可以将快照留在原地以便重启服务器的时候使用。

快照持久化是 Redis 默认采用的持久化方式，在 redis.conf 配置文件中默认有此下配置：
```
save 900 1           #在900秒(15分钟)之后，如果至少有1个key发生变化，Redis就会自动触发bgsave命令创建快照。

save 300 10          #在300秒(5分钟)之后，如果至少有10个key发生变化，Redis就会自动触发bgsave命令创建快照。

save 60 10000        #在60秒(1分钟)之后，如果至少有10000个key发生变化，Redis就会自动触发bgsave命令创建快照。
```
# AOF 持久化
## 什么是AOF持久化
与快照持久化相比，AOF 持久化的实时性更好。默认情况下 Redis 没有开启 AOF（append only file）方式的持久化（Redis 6.0 之后已经默认是开启了），可以通过 appendonly 参数开启：
```
appendonly yes
```
开启 AOF 持久化后每执行一条会更改 Redis 中的数据的命令，Redis 就会将该命令写入到 AOF 缓冲区 server.aof_buf 中，然后再写入到 AOF 文件中（此时还在系统内核缓存区未同步到磁盘），最后再根据持久化方式（ fsync策略）的配置来决定何时将系统内核缓存区的数据同步到硬盘中的。

只有同步到磁盘中才算持久化保存了，否则依然存在数据丢失的风险，比如说：系统内核缓存区的数据还未同步，磁盘机器就宕机了，那这部分数据就算丢失了。

AOF 文件的保存位置和 RDB 文件的位置相同，都是通过 dir 参数设置的，默认的文件名是 appendonly.aof。

# AOF 工作基本流程是怎样的？
AOF 持久化功能的实现可以简单分为 5 步：
- 1,命令追加（append）：所有的写命令会追加到 AOF 缓冲区中。
- 2,文件写入（write）：将 AOF 缓冲区的数据写入到 AOF 文件中。这一步需要调用write函数（系统调用），write将数据写入到了系统内核缓冲区之后直接返回了（延迟写）。注意！！！此时并没有同步到磁盘。
- 3,文件同步（fsync）：AOF 缓冲区根据对应的持久化方式（ fsync 策略）向硬盘做同步操作。这一步需要调用 fsync 函数（系统调用）， fsync 针对单个文件操作，对其进行强制硬盘同步，fsync 将阻塞直到写入磁盘完成后返回，保证了数据持久化。
- 4,文件重写（rewrite）：随着 AOF 文件越来越大，需要定期对 AOF 文件进行重写，达到压缩的目的。
- 5,重启加载（load）：当 Redis 重启时，可以加载 AOF 文件进行数据恢复。

*Linux 系统直接提供了一些函数用于对文件和设备进行访问和控制，这些函数被称为 系统调用（syscall）。*

这里对上面提到的一些 Linux 系统调用再做一遍解释：
- write: 写入系统内核缓冲区之后直接返回（仅仅是写到缓冲区），不会立即同步到硬盘。虽然提高了效率，但也带来了数据丢失的风险。同步硬盘操作通常依赖于系统调度机制，Linux 内核通常为 30s 同步一次，具体值取决于写出的数据量和 I/O 缓冲区的状态。
- fsync: fsync用于强制刷新系统内核缓冲区（同步到到磁盘），确保写磁盘操作结束才会返回。