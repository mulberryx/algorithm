#### Redis是一个内存数据库，为了保证数据的持久化，redis提供了两种持久化方式RDB和AOF，下面我们就分别来看下这两种持久化方式的实现原理
##### RDB(默认)
- RDB是通过快照方式完成的，当满足一定条件时，redis会自动将内存中的数据持久化到磁盘
- 触发快照的时机
  - 符合自定义配置的快照规则。(在redis.conf中配置，下面会详细介绍)
  - 执行save或者bgsave命令
  - 执行flushall命令
  - 执行主从复制操作(第一次)
##### RDB的优点
- 缺点: 使用RDB进行持久化，在redis突然异常退出的时候，会丢失最后一次快照之后的数据。但是，可以根据组合设置自动快照的方式，降低数据损失，确保在接受范围内。如果数据较为重要，可以使用AOF方式
- 优点: 使用RDB方式可以最大化redis性能，在快照过程中，我们可以看到主进程只需要fork出一个子进程即可，剩下的工作全部由子进程完成，父进程无需进行任何的磁盘I/O操作。但是，如果数据集较大，在fork子进程的时候比较耗时，会导致redis在一段时间内停止处理请求。
##### AOF方式
- 默认情况下，redis是没有开启AOF(append only file)的
- 在开始AOF之后，redis每接收到一条更改redis数据的命令，就会将该命令写入硬盘中的AOF文件
- 很明显，该过程会降低redis的性能，但大部分情况下是能够接受的，同时，使用性能较好的硬盘可以提高AOF性能
##### RESP协议
Redis客户端使用RESP协议与Redis服务端进行通信，该协议是专门为redis设计的，但是也可以用于其他C-S项目
- 间隔符号:在linux中是\r\n,在windows中是\n
- 简单字符串以'+'开头
- 错误Errors，以'-'开头
- 整数类型Integer,以':'开头
- 大字符串以'$'开头
- 数组类型Arrays,以'*'开头

##### AOF原理说明
- 以独立日志的方式记录每次写命令，重启时再重新执行AOF文件中的命令达到恢复数据的目的。AOF的主要作用是解决了数据持久化的实时性， 目前已经是Redis持久化的主流方式
- 执行的流程
  - 命令写入
  - 文件同步
  - 文件重写
  - 重启加载
 
##### AOF 优点
- 可以设置不同的 fsync 策略，比如无 fsync ，每秒钟一次 fsync ，或者每次执行写入命令时 fsync
- AOF 的默认策略为每秒钟 fsync 一次，在这种配置下，Redis 仍然可以保持良好的性能，并且就算发生故障停机，也最多只会丢失一秒钟的数据（ fsync 会在后台线程执行，所以主线程可以继续努力地处理命令请求）

##### AOF 缺点
- 对于相同的数据集来说，AOF 文件的体积通常要大于 RDB 文件的体积；根据所使用的 fsync 策略，AOF 的速度可能会慢于 RDB；在一般情况下， 每秒 fsync 的性能依然非常高， 而关闭 fsync 可以让 AOF 的速度和 RDB 一样快， 即使在高负荷之下也是如此；不过在处理巨大的写入载入时，RDB 可以提供更有保证的最大延迟时间
- 数据恢复速度相对于RDB比较慢

##### AOF和RDB的区别
- RDB持久化是指在指定的时间间隔内将内存中的数据集快照写入磁盘，实际操作过程是fork一个子进程，先将数据集写入临时文件，写入成功后，再替换之前的文件，用二进制压缩存储
- AOF持久化以日志的形式记录服务器所处理的每一个写、删除操作，查询操作不会记录，以文本的方式记录，可以打开文件看到详细的操作记录

##### 性能问题与解决方案
- 通过上面的分析，我们都知道RDB的快照、AOF的重写都需要fork，这是一个重量级操作，会对Redis造成阻塞。因此为了不影响Redis主进程响应，需要尽可能降低阻塞
- 那么如何减少fork操作的阻塞
  - 优先使用物理机或者高效支持fork操作的虚拟化技术。
  - 控制Redis实例最大可用内存， fork耗时跟内存量成正比， 线上建议每个Redis实例内存控制在10GB以内。
  - 合理配置Linux内存分配策略，避免物理内存不足导致fork失败。
  - 降低fork操作的频率，如适度放宽AOF自动触发时机，避免不必要的全量复制等。
