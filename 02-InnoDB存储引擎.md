1. 存储基于磁盘，按页的方式进行管理。
2. 缓冲池
   1. 读取：会先读取缓冲池，若缓冲池命中则直接返回，否则读取磁盘，并存储在缓冲池。
   2. 修改：先修改缓冲池中的页，再以一定的频率刷新到磁盘，通过CheckPoint机制刷新到磁盘。
   3. 缓冲池大小是可设定的
   4. 缓冲池中存储内容：
      - 索引页
      - 数据页
      - undo页
      - 插入缓冲（insert buffer）
      - 自适应哈希索引
      - InnoDB存储的锁信息
      - 数据字典
3. LRU List
   1. 缓冲池是通过LRU List(Latest Recent Used,最近最少使用)进行管理
   2. 最频繁访问的页在LRU列表前端，最少使用在末端，当缓冲池不够用时，释放末端数据。
   3. 为了避免全表扫描时完全替换LRU列表，InnoDB的LRU列表中还加入了midpoint位置，所有新读取到的数据存放在LRU列表5/8位置。
4. Flush List
   1. 数据被修改后存放在Flush列表中，成为脏页，通过CHECKPOINT机制刷新回磁盘。
5. Checkpoint技术
   1. 当前事务数据库都采用提交事务时先写重做日志（redo log_buffer），再修改页。
   2. ACID中的D（Durability持久性）就是靠此实现。
   3. Checkpoint解决的问题
      1. 缩短数据库恢复时间（减少redo log中未刷新到磁盘的量）
      2. 缓冲池不够用时，刷新脏页到磁盘
      3. 重做日志不可用时（达到设置上线），刷新缓冲页
   4. 两种Checkpoint
      1. Sharp Checkpoint。在数据库关闭时将所有脏页刷新到磁盘
      2. Fuzzy Checkpoint。
         - Master Thead Checkpoint。每秒或每10秒从缓冲池的脏页列表中刷新一定比例数据到磁盘，该过程为异步
         - FLUSH_LRU_LIST Checkpoint。当LRU List中的数量达到上线，LRU List末端页被移除，若其中有脏页，需要进行Checkpoint。在InnoDB5.6版本开始，采用Page Cleaner线程异步进行
         - Async/Sync Flush Checkpoint。重做日志文件不可用的情况下执行，保证重做日志的循环使用。5.6版本后通过Page Clearner 线程中异步执行
         - Dirty Page too much Checkpoint 。脏页数据太多执行Checkpoint,默认达到最大值的75%会执行
6. Master Thread 工作方式
   1. 每1秒进行的操作
      - 日志缓冲刷新到磁盘，即使这个事务还没有提交
      - 合并插入缓冲
      - 刷新缓冲池的脏页到磁盘
   2. 每10秒进行的操作
      - 日志缓冲刷新到磁盘，即使这个事务还没有提交
      - 合并插入缓冲
      - 刷新缓冲池的脏页到磁盘
      - 删除无用的Undo页
7. InnoDB 关键特性
   1. 插入缓冲
      1. Insert Buffer
         1. 对于非聚集索引的插入或更新操作，会先判断插入的非聚集索引页是否在缓冲池中，若在则直接插入；若不在，则先放到Insert Buffer对象中。然后以一定的频率和情况进行Insert Buffer
         和辅助索引叶子节点的merge操作，通常可以将多个插入操作合并到一个操作中，提高插入效率。
         2. 使用Insert Buffer的条件
            1. 索引是辅助索引
            2. 索引不是唯一的
      2. Merge Insert Buffer
         1. Insert Buffer合并情况
            1. 辅助索引页被读取到缓冲池中。
            2. Insert Buffer Bitmap 页追踪到该辅助索引页已无可用空间
            3. Master Thread
      3. Double Write(两次写)
         1. 两次写是为了防止数据中缓冲池写入某页到表中时，只写了一半发生宕机，发生数据丢失。
         2. 实现原理：把脏页数据复制到内存中的doblewrite buffer中，再分两次，每次1MB顺序写入到共享表空间的物理磁盘上，形成了数据的备份；然后再同步doublewrite buffer中数据到物理页。
         如中途发生宕机，通过共享表空间中的备份数据恢复
      4. 自适应哈希索引
         1. 自适应哈希索引：InnoDB通过监控对各表上索引页的查询，观察到建立哈希索引会带来速度提升，会建立哈希索引
         2. 自适应哈希索引是通过缓冲池的B+树页构造而来
      5. 异步IO
         1. 当一个SQL查询可能需要扫描多个索引页，也就是需要进行多次IO,此时并行发生IO请求，等所有IO请求都返回后再返回最终结果。并且如果多个IO中如何有重复页，则可合并，减少IO次数
      
