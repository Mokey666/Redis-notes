## Redis主从复制

- 概述：持久化保证了即使 redis 服务重启也会丢失数据，因为 redis 服务重启后会将硬盘上持久化的数据恢复到内存中，但是当 redis 服务器的硬盘损坏了可能会导致数据丢失，如果通过 redis 的主从复制机制就可以避免这种单点故障

![1568449773601](C:\Users\侯泽明\AppData\Roaming\Typora\typora-user-images\1568449773601.png)



- 主不配置，从配置，主写数据，从读数据

  

- Redis复制概论
  复制指的是发生在不同的服务器实例之间, 单向的信息传播行为, 复制方和被复制方建立网络连接, 复制方式通常为被复制方式主动发送数据到复制方. 最终目的是为了保证双方中的数据一致, 同步.

- Redis复制方式
  复制方式有两种: 主从复制, 从从复制.

- ![1568450176489](C:\Users\侯泽明\AppData\Roaming\Typora\typora-user-images\1568450176489.png)

- 复制过程
  Redis的复制主要由SYNC命令实现, 复制过程如下图:

- ![1568450204138](C:\Users\侯泽明\AppData\Roaming\Typora\typora-user-images\1568450204138.png)Redis复制的工作过程

  1. slave向master发送sync命令.

  2. master开启子进程来将数据写入RDB文件. 同时将子进程写完之前收到的写命令缓存起来.
     
3. 子进程写完之后, 将RDB文件发送给slave.
  
4. master发送完RDB文件, 将缓存命令也发送给slave.
  
5. master增量的把写命令发给slave.
  
   
  
- 当slave与master连接断开时, slave可以自动重新连接master. 在Redis2.8版本之前, 每当slave进程挂掉重新连接master的时候都会开始新的一轮的**全量复制**.
  **增量复制** 
  从Redis2.8开始, Redist提供了增量复制的机制.
  增量复制的实现: master会维护一个环形队列, 以及环形队列的写索引和slave同步的全局offset. 环形队列用于存储最新的操作数据, 增量复制由psync命令实现, slave可以通过psync命令来让Redis进行增量复制.

- 免持久化复制
  Redis必须要先将RDB文件写入磁盘中, 才进行网络传输, 免持久化机制就是直接通过网络将RDB文件传送给Redis, 而不需要进行持久化.
  //是否开启免持久化机制  repo-diskless-sync no 



### 重点大头

复制的过程

- 同步（全量复制）     传播（增量复制）

- 旧版复制功能的缺陷
  在Redis中, 从服务器对主服务器的复制分为以下两种情况:
  (1) 初次复制: 从服务器之前没有复制过任何主服务器, 或者从服务器当前要复制的主服务器和上一次复制的主服务不同.
  (2) 断线后重复制: 处于命令传播阶段的主从服务器因为网络原因而中断了复制, 从服务器重新连接上主服务器之后, 继续对主服务器进行复制.在主从服务器重新连接上之后, 主从服务器的状态已经不一致了, 所以从服务器向主服务器发送 SYNC 命令, 主服务器会将数据库中所有的键值对生成 RDB 文件, 发送给从服务器, 虽然这个 RDB 文件可以让主从再次达到一致, 但是这两个服务器保存的大部分数据是相同的, 只是主从服务器中断期间的数据不一致, 但是主服务器重新发送 RDB 文件, 让从服务器再一次执行, 这种做法是很低效的.SYNC 命令是一很耗费资源的操作.

- 二. 新版复制功能的实现
  为了解决旧版复制功能在处理断线重复复制的低效问题, Redis从 2.8 版本开始, 使用 PSYNC 命令代替 SYNC 命令来执行复制时的同步操作.

- ![1568451161474](C:\Users\侯泽明\AppData\Roaming\Typora\typora-user-images\1568451161474.png)

  PSYNC 命令的部分重同步模式, 解决了旧版复制功能在断线处理时出现的低效问题. 
  1. 部分重同步的实现
  部分重同步由以下三个部分构成: 
  (1) 主服务器的复制偏移量和从服务器的复制偏移量
  (2) 主服务器的复制积压缓冲区
  (3) 服务器的运行ID

- 复制偏移量
  执行复制的双方------主服务和从服务器会分别维护一个复制偏移量.主服务器每次向从服务器传播 N 个字节的数据时, 就将自己的复制偏移量加上N. 从服务器每次收到主服务器发送来的 N 个字节的数据时, 就将自己的复制偏移量加上N.

  通过对比主从服务器的复制偏移量, 可以很容易的判断主从服务器是否处于一致状态.

- 复制积压缓冲区
  复制积压缓冲区是由主服务器维护的一个固定长度先进先出的队列, 默认大小是 1MB.当主服务器进行命令传播的时候, 它不仅会将写命令发送给所有的从服务器, 还会将写命令入队到复制积压缓冲区.

- 服务器运行ID
  除了复制偏移量和复制积压缓冲区之外, 实现部分重同步还需要服务器运行ID. 

  1. 每个Redis服务器, 不论主服务器还是从服务器, 都有自己的运行ID.

  2. 运行ID在服务器启动的时候自动生成, 由40个随机的十六进制字符组成.

  3. 当从服务器对主服务器进行初次复制时, 主服务器会将自己的运行ID传送给从服务器, 从服务器会将这个运行ID保存起来. 

  4. 如果从服务器保存的运行ID和当前连接的主服务器的运行ID相同, 那么说明这个从服务器断线之前复制的就是当前连接的这个服务器, 主服务器可以尝试执行部分重同步操作.

  5. 如果从服务器保存的运行ID和当前连接的主服务器的运行ID不同, 那么说明从服务器断线之前复制的主服务器并不是当前连接的那个主服务器, 主服务器将对从服务器执行完整重同步操作.

    

- 三. 主从复制的实现通过向从服务器发送 SLAVEOF 命令,可以让一个从服务器去复制主服务器.

  1. 设置主服务器的端口和地址
  2. 当向从服务器发送 SLAVEOF 命令时, 从服务器首先要将主服务器的IP地址和端口号保存到服务器状态中
  3. 建立套接字连接
      在执行 SLAVEOF 命令之后, 从服务器根据命令所设置的IP地址和端口, 创建连向主服务器的套接字, 如果从服务器创建的套接字能成功连接到主服务器, 那么从服务器降为这个套接字关联一个专门用于处理复制工作的文件处理器, 这个文件处理器将负责执行后续的复制操作.

- 四. 心跳检测

  - 在命令传播阶段, 从服务器会默认以每秒一次的频率, 向主服务器发送
    命令:
    REPLCONF ACK <replication_offset>
    其中 replication_offset 是从服务器的复制偏移量.

  1. 检查主从服务器的连接状态
      主从服务器可以通过发送和接收REPLCONF ACK 命令来检查两者之间的
      网络连接是否正常: 如果主服务器超过一秒钟没有收到从服务器发来的
      REPLCONF ACK命令, 那么主从服务器之间的连接出现了问题.
      通过向主服务器发送INFO replication命令, 在列出的从服务器栏的一
      列, 可以看到从服务器后一次向主服务发送REPLCONF ACK命令距现在
      过了多少秒.
  2. 监测命令丢失
      如果因为网络故障, 主服务器传播给从服务器的写命令在半路丢失, 那
      么当从服务器向主服务器发送 REPLCONF ACk命令时, 主服务发觉从服
      务器当前偏移量少于自己的偏移量, 然后主服务器就会根据从服务器提
      交的偏移量, 在复制积压缓冲区里找到从服务器缺少的数据, 发送给从
      服务器.