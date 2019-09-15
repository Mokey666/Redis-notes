## Redis持久化

- 简单的来说redis持久化就是RDB(Redis DateBase)和AOF(Apend Only File)

- ### RDB

  - 概述：RDB持久化是将当前进程中的数据生成快照保存到硬盘(因此也称作快照持久化)，保存的文件后缀是rdb；当Redis重新启动时，可以读取快照文件恢复数据。

  - 触发条件

    - 自动触发

      - ![1568443876538](C:\Users\侯泽明\AppData\Roaming\Typora\typora-user-images\1568443876538.png)

        在redis.config配置文件中 用 （save 参数 参数）  命令配置默认为

        save 900 1

        save 300 10

        save 60   10000

        其中save 900 1的含义是：当时间到900秒时，如果redis数据发生了至少1次变化，则执行bgsave；save 300 10和save 60 10000同理。当三个save条件满足任意一个时，都会引起bgsave的调用。

      - save m n 原理

        - Redis的save m n，是通过serverCron函数、dirty计数器、和lastsave时间戳来实现的。

          serverCron是Redis服务器的周期性操作函数，默认每隔100ms执行一次；该函数对服务器的状态进行维护，其中一项工作就是检查 save m n 配置的条件是否满足，如果满足就执行bgsave。

          dirty计数器是Redis服务器维持的一个状态，记录了上一次执行bgsave/save命令后，服务器状态进行了多少次修改(包括增删改)；而当save/bgsave执行完成后，会将dirty重新置为0。

    - 手动触发

      - save 和 bgsave

      - save：save命令会阻塞Redis服务器进程，直到RDB文件创建完毕为止，在Redis服务器阻塞期间，服务器不能处理任何命令请求。(同步)

      - bgsave：bgsave命令会创建一个子进程，由子进程来负责创建RDB文件，父进程(即Redis主进程)则继续处理请求。(异步)

        bgsave命令执行过程中，只有fork子进程时会阻塞服务器，而对于save命令，整个过程都会阻塞服务器，因此save已基本被废弃，线上环境要杜绝save的使用；后文中也将只介绍bgsave命令。此外，在自动触发RDB持久化时，Redis也会选择bgsave而不是save来进行持久化；下面介绍自动触发RDB持久化的条件。每次持久化后的文件，会替换上次持久化后的文件。

    - 其他触发

      - 在主从复制场景下，如果从节点执行全量复制操作，则主节点会执行bgsave命令，并将rdb文件发送给从节点
      - 执行shutdown命令时，自动执行rdb持久化。

  - RDB执行流程

    - ![1568444596550](C:\Users\侯泽明\AppData\Roaming\Typora\typora-user-images\1568444596550.png)

  - RDB在redis.config中的配置

    - save m n：bgsave自动触发的条件；如果没有save m n配置，相当于自动的RDB持久化关闭，不过此时仍可以通过其他方式触发
    - stop-writes-on-bgsave-error yes：当bgsave出现错误时，Redis是否停止执行写命令；设置为yes，则当硬盘出现问题时，可以及时发现，避免数据的大量丢失；设置为no，则Redis无视bgsave的错误继续执行写命令，当对Redis服务器的系统(尤其是硬盘)使用了监控时，该选项考虑设置为no
    - rdbcompression yes：是否开启RDB文件压缩
    - rdbchecksum yes：是否开启RDB文件的校验，在写入文件和读取文件时都起作用；关闭checksum在写入文件和启动文件时大约能带来10%的性能提升，但是数据损坏时无法发现
    - dbfilename dump.rdb：RDB文件名

  - RDB文件格式

    - ![1568444789986](C:\Users\侯泽明\AppData\Roaming\Typora\typora-user-images\1568444789986.png)

    - 其中各个字段的含义说明如下：

      1)  REDIS：常量，保存着”REDIS”5个字符。

      2)  db_version：RDB文件的版本号，注意不是Redis的版本号。

      3)  SELECTDB 0 pairs：表示一个完整的数据库(0号数据库)，同理SELECTDB 3 pairs表示完整的3号数据库；只有当数据库中有键值对时，RDB文件中才会有该数据库的信息(上图所示的Redis中只有0号和3号数据库有键值对)；如果Redis中所有的数据库都没有键值对，则这一部分直接省略。其中：SELECTDB是一个常量，代表后面跟着的是数据库号码；0和3是数据库号码；pairs则存储了具体的键值对信息，包括key、value值，及其数据类型、内部编码、过期时间、压缩信息等等。

      4)  EOF：常量，标志RDB文件正文内容结束。

      5)  check_sum：前面所有内容的校验和；Redis在载入RBD文件时，会计算前面的校验和并与check_sum值比较，判断文件是否损坏。

  - 注意

    - redis默认的持久化是RDB
    - RDB优点：1)适合大规模数据校验。2)对数据一致性、完整性要求不高
    - RDB缺点：1)可能会丢失最后一次"快照"后的修改(就是当他最后一次创建dump.rdb文件时,redis直接关了，就有可能丢失最后一次的修改)2)因为他每次创建RDB都会fork一个子进程，所以内存数据会扩大一倍。

- ### AOF

  - 概述：RDB持久化是将进程数据写入文件，而AOF持久化(即Append Only File持久化)，则是将Redis执行的每次写命令记录到单独的日志文件中；当Redis重启时再次执行AOF文件中的命令来恢复数据。

  - 执行流程

    1. 命令追加(append)：将Redis的写命令追加到缓冲区aof_buf；

    2. 文件写入(write)和文件同步(sync)：根据不同的同步策略将aof_buf中的内容同步到硬盘；

    3. 文件重写(rewrite)：定期重写AOF文件，达到压缩的目的。

       - ### 命令追加(append)

         Redis先将写命令追加到缓冲区，而不是直接写入文件，主要是为了避免每次有写命令都直接写入硬盘，导致硬盘IO成为Redis负载的瓶颈。

       - ###  文件写入(write)和文件同步(sync)

         Redis提供了多种AOF缓存区的同步文件策略，策略涉及到操作系统的write函数和fsync函数，说明如下：

         为了提高文件写入效率，在现代操作系统中，当用户调用write函数将数据写入文件时，操作系统通常会将数据暂存到一个内存缓冲区里，当缓冲区被填满或超过了指定时限后，才真正将缓冲区的数据写入到硬盘里。这样的操作虽然提高了效率，但也带来了安全问题：如果计算机停机，内存缓冲区中的数据会丢失；因此系统同时提供了fsync、fdatasync等同步函数，可以强制操作系统立刻将缓冲区中的数据写入到硬盘里，从而确保数据的安全性。

         AOF缓存区的同步文件策略由参数appendfsync控制，各个值的含义如下：

         - always：命令写入aof_buf后立即调用系统fsync操作同步到AOF文件，fsync完成后线程返回。这种情况下，每次有写命令都要同步到AOF文件，硬盘IO成为性能瓶颈，Redis只能支持大约几百TPS写入，严重降低了Redis的性能；即便是使用固态硬盘（SSD），每秒大约也只能处理几万个命令，而且会大大降低SSD的寿命。
         - no：命令写入aof_buf后调用系统write操作，不对AOF文件做fsync同步；同步由操作系统负责，通常同步周期为30秒。这种情况下，文件同步的时间不可控，且缓冲区中堆积的数据会很多，数据安全性无法保证。
         - everysec：命令写入aof_buf后调用系统write操作，write完成后线程返回；fsync同步文件操作由专门的线程每秒调用一次。**everysec是前述两种策略的折中，是性能和数据安全性的平衡，因此是Redis的默认配置，也是我们推荐的配置。**
         - ![1568445508044](C:\Users\侯泽明\AppData\Roaming\Typora\typora-user-images\1568445508044.png)

       - ###  文件重写(rewrite)

         - 文件重写是指定期重写AOF文件，减小AOF文件的体积。需要注意的是，**AOF重写是把Redis进程内的数据转化为写命令，同步到新的AOF文件；不会对旧的AOF文件进行任何读取、写入操作!**

         - 文件重写之所以能够压缩AOF文件，原因在于：
           - 过期的数据不再写入文件
           - 无效的命令不再写入文件
           - 多条命令可以合并为一个
         - 文件重写的触发
           - 手动触发：直接调用bgrewriteaof命令，该命令的执行与bgsave有些类似：都是fork子进程进行具体的工作，且都只有在fork时阻塞。
           - 自动触发：根据auto-aof-rewrite-min-size和auto-aof-rewrite-percentage参数，以及aof_current_size和aof_base_size状态确定触发时机。
             - auto-aof-rewrite-min-size：执行AOF重写时，文件的最小体积，默认值为64MB。
             - auto-aof-rewrite-percentage：执行AOF重写时，当前AOF大小(即aof_current_size)和上一次重写时AOF大小(aof_base_size)的比值。
           - 文件重写流程
             - ![1568446273235](C:\Users\侯泽明\AppData\Roaming\Typora\typora-user-images\1568446273235.png)
         - AOF在redis.config中的配置
           - appendonly no：是否开启AOF
           - appendfilename "appendonly.aof"：AOF文件名
           - dir ./：RDB文件和AOF文件所在目录
           - appendfsync everysec：fsync持久化策略
           - no-appendfsync-on-rewrite no：AOF重写期间是否禁止fsync；如果开启该选项，可以减轻文件重写时CPU和硬盘的负载（尤其是硬盘），但是可能会丢失AOF重写期间的数据；需要在负载和安全性之间进行平衡
           - auto-aof-rewrite-percentage 100：文件重写触发条件之一
           - auto-aof-rewrite-min-size 64mb：文件重写触发提交之一
           - aof-load-truncated yes：如果AOF文件结尾损坏，Redis启动时是否仍载入AOF文件

- RDB和AOF的区别

  - **RDB持久化**

    优点：RDB文件紧凑，体积小，网络传输快，适合全量复制；恢复速度比AOF快很多。当然，与AOF相比，RDB最重要的优点之一是对性能的影响相对较小。

    缺点：RDB文件的致命缺点在于其数据快照的持久化方式决定了必然做不到实时持久化，而在数据越来越重要的今天，数据的大量丢失很多时候是无法接受的，因此AOF持久化成为主流。此外，RDB文件需要满足特定格式，兼容性差（如老版本的Redis不兼容新版本的RDB文件）。

    **AOF持久化**

    与RDB持久化相对应，AOF的优点在于支持秒级持久化、兼容性好，缺点是文件大、恢复速度慢、对性能影响大。

