# [mysql优化之参数优化](https://www.cnblogs.com/chenpingzhao/p/4850942.html)

**1、优化方式**

硬件优化=》系统优化=》mysql配置优化=》SCHEMA优化=》sql优化=》其他解决方案（redis or [MongoDB or Cassandra or HBase](http://www.kuqin.com/shuoit/20140928/342398.html)）

**2、mysql配置分析**

1）常见瓶颈

90%系统瓶颈都在IO上，所以提高IOPS尤为总要，iowait过高，加内存，减小数据读取量

如果CPU很高，或者查询时间很长，90%索引不当

如果系统发生swap，必定是内存分配不当

所以优化，总是会围绕着提高对内存的使用率+减少IO，比如内存缓存+索引，还有其他方式吗，NO

2）确认方式

slow log + global status + engine status + processlist + pt工具

3）环境

mysql  Ver 14.14 Distrib 5.6.25, for Linux (x86_64) using  EditLine wrapper &&　CentOS release 6.7 (Final)

 

**3、mysql配置优化**

**1）慢查询日志**

在mysql服务器中，数据表都是保存在磁盘上的（innodb、myisam组织表的形式不同，所以文件结构也就不同）。索引为服务器提供了一种在表中查找特定数据行的方法，而不用搜索整个表。当必须要搜索整个表时，就称为*表扫描*。通常来说，您可能只希望获得表中数据的一个子集，因此全表扫描会浪费大量的磁盘 I/O，因此也就会浪费大量时间。当必须对数据进行连接时，这个问题就更加复杂了，因为必须要对连接两端的多行数据进行比较。

当然，表扫描并不总是会带来问题；有时读取整个表反而会比从中挑选出一部分数据更加有效（服务器进程中查询优化器用来作出这些决定）。如果索引的使用效率很低，或者根本就不能使用索引，则会减慢查询速度，而且随着服务器上的负载和表大小的增加，这个问题会变得更加显著。执行时间超过给定时间范围的查询就称为*慢速查询*。

在my.cnf中开启慢日志

```
long_query_time = 2
slow-query-log = on                                                                                                                  
slow-query-log-file = /data/mysql/slow-query.log
log-queries-not-using-indexes
```

查看是否开启

```
mysql> show variables like '%slow_query%';
+---------------------+----------------------------+
| Variable_name       | Value                      |
+---------------------+----------------------------+
| slow_query_log      | ON                         |
| slow_query_log_file | /data/mysql/slow-query.log |
+---------------------+----------------------------+
 
mysql> show global status like '%slow%';
+---------------------+-------+
| Variable_name       | Value |
+---------------------+-------+
| Slow_launch_threads | 0     |
| Slow_queries        | 4148  |
+---------------------+-------+
```

打开慢查询日志可能会对系统性能有一点点影响，如果你的MySQL是主－从结构，可以考虑打开其中一台从服务器的慢查询日志，这样既可以监控慢查询，对系统性能影响又小，另mysql有自带的命令mysqldumpslow可进行查询，也可以使用pt工具进行分析（pt-query-digest）

例下列命令可以查出访问次数最多的20个sql语句

```
mysqldumpslow -s c -t 20 slow-query.log
```

**2）连接数**

经常会遇见”MySQL: ERROR 1040: Too manyconnections”的情况，一种是访问量确实很高，MySQL服务器抗不住，这个时候就要考虑增加从服务器分散读压力，另外一种情况是MySQL配置文件中max_connections值过小

```
mysql> show variables like 'max_connections';
+-----------------+-------+
| Variable_name   | Value |
+-----------------+-------+
| max_connections | 256   |
+-----------------+-------+
```

这台MySQL服务器最大连接数是256，然后查询一下服务器响应的最大连接数

```
mysql> show global status like 'Max_used_connections';
+----------------------+-------+
| Variable_name        | Value |
+----------------------+-------+
| Max_used_connections | 245   |
+----------------------+-------+
```

MySQL服务器过去的最大连接数是245，没有达到服务器连接数上限256，不会出现1040错误，最大连接数占上限连接数的85％左右，如果发现比例在10%以下，MySQL服务器连接数上限设置的过高了

比较理想的设置是：Max_used_connections / max_connections  * 100% ≈ 85%

**还有两个比较重要参数**

```
wait_timeout=10``max_connect_errors = 100
```

 wait_timeout指的是mysqld 终止所有空闲时间超过 10 秒的连接。在 LAMP 应用程序中，连接数据库的时间通常就是 Web 服务器处理请求所花费的时间。有时候，如果负载过重，连接会挂起，并且会占用连接表空间。如果有多个交互用户或使用了到数据库的持久连接，那么将这个值设低一点并不可取

max_connect_errors 是一个安全的方法。如果一个主机在连接到服务器时有问题，并重试很多次后放弃，那么这个主机就会被锁定，直到 FLUSH HOSTS 之后才能运行。默认情况下，10 次失败就足以导致锁定了。将这个值修改为 100 会给服务器足够的时间来从问题中恢复。如果重试 100 次都无法建立连接，那么使用再高的值也不会有太多帮助，可能它根本就无法连接。

3）**Key_buffer_size**

key_buffer_size是对MyISAM表性能影响最大的一个参数，下面一台以MyISAM为主要存储引擎服务器的配置

```
mysql> show variables like 'key_buffer_size';
+-----------------+------------+
| Variable_name   | Value      |
+-----------------+------------+
| key_buffer_size | 536870912  |
+-----------------+------------+
```

分配了512MB内存给key_buffer_size，我们再看一下key_buffer_size的使用情况

```
mysql> show global status like 'key_read%';
+------------------------+-------------+
| Variable_name          | Value       |
+------------------------+-------------+
| Key_read_requests      | 27813678764 |
| Key_reads              | 6798830     |
+------------------------+-------------+
```

`Key_reads` 代表命中磁盘的请求个数， `Key_read_requests` 是总数

一共有27813678764个索引读取请求，有6798830个请求在内存中没有找到直接从硬盘读取索引

计算索引未命中缓存的概率：key_cache_miss_rate ＝ Key_reads / Key_read_requests * 100%

比如上面的数据，key_cache_miss_rate为0.0244%，4000个索引读取请求才有一个直接读硬盘，已经很BT 了，key_cache_miss_rate在0.1%以下都很好（每1000个请求有一个直接读硬盘），如果key_cache_miss_rate在 0.01%以下的话，key_buffer_size分配的过多，可以适当减少，如果每 1,000 个请求中命中磁盘的数目超过 1 个，就应该考虑增大关键字缓冲区了。例如，`key_buffer = 384M` 会将缓冲区设置为 384MB。

MySQL服务器还提供了key_blocks_*参数

```
mysql> show global status like 'key_blocks_u%';
+------------------------+-------------+
| Variable_name          | Value       |
+------------------------+-------------+
| Key_blocks_unused      | 0           |
| Key_blocks_used        | 413543      |
+------------------------+-------------+
```

Key_blocks_unused 表示未使用的缓存簇(blocks)数，Key_blocks_used表示曾经用到的最大的blocks数，比如这台服务器，所有的缓存都用到了，要么增加key_buffer_size，要么就是过渡索引了，把缓存占满了

比较理想的设置：Key_blocks_used / (Key_blocks_unused + Key_blocks_used) * 100% ≈ 80%

4）**临时表**

临时表可以在更高级的查询中使用，其中数据在进一步进行处理（例如 GROUP BY 字句）之前，都必须先保存到临时表中；理想情况下，在内存中创建临时表。但是如果临时表变得太大，就需要写入磁盘中

```
mysql> show global status like 'created_tmp%';
+-------------------------+---------+
| Variable_name           | Value   |
+-------------------------+---------+
| Created_tmp_disk_tables | 21197   |
| Created_tmp_files       | 58      |
| Created_tmp_tables      | 1771587 |
+-------------------------+---------+
```

每次创建临时表，Created_tmp_tables增加，如果是在磁盘上创建临时表，Created_tmp_disk_tables也增加,Created_tmp_files表示MySQL服务创建的临时文件文件数

比较理想的配置是：Created_tmp_disk_tables / Created_tmp_tables * 100% <= 25%

比如上面的服务器Created_tmp_disk_tables / Created_tmp_tables * 100% ＝ 1.20%，应该相当好了

我们再看一下MySQL服务器对临时表的配置

```
mysql> show variables where Variable_name in ('tmp_table_size', 'max_heap_table_size');
+---------------------+-----------+
| Variable_name       | Value     |
+---------------------+-----------+
| max_heap_table_size | 268435456 |
| tmp_table_size      | 536870912 |
+---------------------+-----------+
```

只有256MB以下的临时表才能全部放内存，超过的就会用到硬盘临时表

每次使用临时表都会增大 `Created_tmp_tables`；基于磁盘的表也会增大 `Created_tmp_disk_tables`。对于这个比率，并没有什么严格的规则，因为这依赖于所涉及的查询。长时间观察 `Created_tmp_disk_tables` 会显示所创建的磁盘表的比率，您可以确定设置的效率。`tmp_table_size` 和 `max_heap_table_size` 都可以控制临时表的最大大小，因此请确保在 my.cnf 中对这两个值都进行了设置

**5）Open Table情况**

```
mysql> show global status like 'open%tables%';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| Open_tables   | 919   |
| Opened_tables | 1951  |
```

Open_tables 表示打开表的数量，Opened_tables表示打开过的表数量，如果Opened_tables数量过大，说明配置中 table_cache(5.1.3之后这个值叫做table_open_cache)值可能太小，我们查询一下服务器table_cache值

```
mysql> show variables like 'table_cache';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| table_cache   | 2048  |
+---------------+-------+
```

**比较合适的值为：Open_tables / Opened_tables  \* 100% >= 85%   Open_tables / table_cache \* 100% <= 95%**

**6）线程使用情况**

与表的缓存类似，对于线程来说也有一个缓存。 `mysqld` 在接收连接时会根据需要生成线程。在一个连接变化很快的繁忙服务器上，对线程进行缓存便于以后使用可以加快最初的连接

```
mysql> show global status like 'Thread%';
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| Threads_cached    | 46    |
| Threads_connected | 2     |
| Threads_created   | 570   |
| Threads_running   | 1     |
+-------------------+-------+
```

如果我们在MySQL服务器配置文件中设置了thread_cache_size，当客户端断开之后，服务器处理此客户的线程将会缓存起来以响应下一个客户 而不是销毁（前提是缓存数未达上限）。Threads_created表示创建过的线程数，如果发现Threads_created值过大的话，表明 MySQL服务器一直在创建线程，这也是比较耗资源，可以适当增加配置文件中thread_cache_size值，查询服务器 thread_cache_size配置

```
mysql> show variables like 'thread_cache_size';
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| thread_cache_size | 64    |
+-------------------+-------+
```

7）**查询缓存(query cache)**

很多 LAMP 应用程序都严重依赖于数据库，但却会反复执行相同的查询。每次执行查询时，数据库都必须要执行相同的工作 —— 对查询进行分析，确定如何执行查询，从磁盘中加载信息，然后将结果返回给客户机。MySQL 有一个特性称为*查询缓存*，它将（后面会用到的）查询结果保存在内存中。在很多情况下，这会极大地提高性能。不过，问题是查询缓存在默认情况下是禁用的

```
mysql> show global status like 'qcache%';
+-------------------------+-----------+
| Variable_name           | Value     |
+-------------------------+-----------+
| Qcache_free_blocks      | 22756     |
| Qcache_free_memory      | 76764704  |
| Qcache_hits             | 213028692 |
| Qcache_inserts          | 208894227 |
| Qcache_lowmem_prunes    | 4010916   |
| Qcache_not_cached       | 13385031  |
| Qcache_queries_in_cache | 43560     |
| Qcache_total_blocks     | 111212    |
+-------------------------+-----------+
```

MySQL查询缓存变量解释：

Qcache_free_blocks：缓存中相邻内存块的个数，数目大说明可能有碎片。FLUSH QUERY CACHE会对缓存中的碎片进行整理。

Qcache_free_memory：缓存中的空闲内存。

Qcache_hits：每次查询在缓存中命中时就增大

Qcache_inserts：每次插入一个查询时就增大。命中次数除以插入次数就是不中比率。

Qcache_lowmem_prunes： 缓存出现内存不足并且必须要进行清理以便为更多查询提供空间的次数。这个数字最好长时间来看；如果这个数字在不断增长，就表示可能碎片非常严重，或者内存很少。（上面的 free_blocks和free_memory可以告诉您属于哪种情况）

Qcache_not_cached：不适合进行缓存的查询的数量，通常是由于这些查询不是 SELECT 语句或者用了now()之类的函数。

Qcache_queries_in_cache：当前缓存的查询（和响应）的数量。

Qcache_total_blocks：缓存中块的数量。

我们再查询一下服务器关于query_cache的配置

```
mysql> show variables like 'query_cache%';
+------------------------------+-----------+
| Variable_name             | Value     |
+------------------------------+-----------+
| query_cache_limit          | 2097152 |
| query_cache_min_res_unit     | 4096    |
| query_cache_size          | 203423744 |
| query_cache_type          | ON        |
| query_cache_wlock_invalidate | OFF    |
+------------------------------+-----------+
```

各字段的解释：
query_cache_limit：超过此大小的查询将不缓存
query_cache_min_res_unit：缓存块的最小大小
query_cache_size：查询缓存大小
query_cache_type：缓存类型，决定缓存什么样的查询，示例中表示不缓存 select sql_no_cache 查询
query_cache_wlock_invalidate：当有其他客户端正在对MyISAM表进行写操作时，如果查询在query cache中，是否返回cache结果还是等写操作完成再读表获取结果。
query_cache_min_res_unit的配置是一柄”双刃剑”，默认是4KB，设置值大对大数据查询有好处，但如果你的查询都是小数据查询，就容易造成内存碎片和浪费。

查询缓存碎片率 = Qcache_free_blocks / Qcache_total_blocks * 100%

如果查询缓存碎片率超过20%，可以用FLUSH QUERY CACHE整理缓存碎片，或者试试减小query_cache_min_res_unit，如果你的查询都是小数据量的话。

查询缓存利用率 = (query_cache_size - Qcache_free_memory) / query_cache_size * 100%

查询缓存利用率在25%以下的话说明query_cache_size设置的过大，可适当减小；查询缓存利用率在80％以上而且Qcache_lowmem_prunes > 50的话说明

query_cache_size可能有点小，要不就是碎片太多。

查询缓存命中率 = (Qcache_hits - Qcache_inserts) / Qcache_hits * 100%

示例服务器 查询缓存碎片率 ＝ 20.46％，查询缓存利用率 ＝ 62.26％，查询缓存命中率 ＝ 1.94％，命中率很差，可能写操作比较频繁吧，而且可能有些碎片。

**作为一条规则，如果 `FLUSH QUERY CACHE` 占用了很长时间，那就说明缓存太大了**

8）**排序使用情况**

```
mysql> show global status like 'sort%';
+-------------------+------------+
| Variable_name     | Value      |
+-------------------+------------+
| Sort_merge_passes | 29         |
| Sort_range        | 37432840   |
| Sort_rows         | 9178691532 |
| Sort_scan         | 1860569    |
+-------------------+------------+
```

Sort_merge_passes 包括两步。MySQL 首先会尝试在内存中做排序，使用的内存大小由系统变量Sort_buffer_size 决定，如果它的大小不够把所有的记录都读到内存中，MySQL 就会把每次在内存中排序的结果存到临时文件中，等MySQL 找到所有记录之后，再把临时文件中的记录做一次排序。这再次排序就会增加 Sort_merge_passes。实际上，MySQL会用另一个临时文件来存再次排序的结果，所以通常会看到 Sort_merge_passes增加的数值是建临时文件数的两倍。因为用到了临时文件，所以速度可能会比较慢，增加 Sort_buffer_size 会减少Sort_merge_passes 和 创建临时文件的次数，但盲目的增加Sort_buffer_size 并不一定能提高速度

**9）文件打开数(open_files)**

```
mysql> show global status like 'open_files';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| Open_files    | 1410  |
+---------------+-------+
mysql> show variables like 'open_files_limit';
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| open_files_limit | 4590  |
+------------------+-------+
```

比较合适的设置：Open_files / open_files_limit * 100% <= 75％

```
mysql> show global status like 'table_locks%';
+-----------------------+-----------+
| Variable_name         | Value     |
+-----------------------+-----------+
| Table_locks_immediate | 490206328 |
| Table_locks_waited    | 2084912   |
+-----------------------+-----------+
```

Table_locks_immediate 表示立即释放表锁数，Table_locks_waited表示需要等待的表锁数

如果Table_locks_immediate / Table_locks_waited >5000，最好采用InnoDB引擎，因为InnoDB是行锁而MyISAM是表锁，对于高并发写入的应用InnoDB效果会好些。示例中的服务 器Table_locks_immediate / Table_locks_waited ＝ 235，MyISAM就足够了

10）**表锁情况**

```
mysql> show global status like 'table_locks%';
+-----------------------+-----------+
| Variable_name         | Value     |
+-----------------------+-----------+
| Table_locks_immediate | 490206328 |
| Table_locks_waited    | 2084912   |
+-----------------------+-----------+
```

Table_locks_immediate 表示立即释放表锁数，Table_locks_waited表示需要等待的表锁数

如果Table_locks_immediate / Table_locks_waited >5000，最好采用InnoDB引擎，因为InnoDB是行锁而MyISAM是表锁，对于高并发写入的应用InnoDB效果会好些。示例中的服务 器Table_locks_immediate / Table_locks_waited ＝ 235，MyISAM就足够了

11）**表扫描情况**

```
mysql> show global status like 'handler_read%';
+-----------------------+-------------+
| Variable_name         | Value       |
+-----------------------+-------------+
| Handler_read_first    | 5803750     |
| Handler_read_key      | 6049319850  |
| Handler_read_next     | 94440908210 |
| Handler_read_prev     | 34822001724 |
| Handler_read_rnd      | 405482605   |
| Handler_read_rnd_next | 18912877839 |
+-----------------------+-------------+
mysql> show global status like 'com_select';
+---------------+-----------+
| Variable_name | Value     |
+---------------+-----------+
| Com_select    | 222693559 |
+---------------+-----------+
```

计算表扫描率：
表扫描率 ＝ Handler_read_rnd_next / Com_select

如果表扫描率超过4000，说明进行了太多表扫描，很有可能索引没有建好，增加read_buffer_size值会有一些好处，但最好不要超过8MB

**12）table_definition_cache**

表定义信息缓存是从MySQL5.1.3 版本才开始引入的一个新的缓存区，用来存放表定义信息。当我们的MySQL 中使用了较多的表的时候，此缓存无疑会提高对表定义信息的访问效率。MySQL 提供了table_definition_cache 参数给我们设置可以缓存的表的数量。在MySQL5.1.25 之前的版本中，默认值为128，从MySQL5.1.25 版本开始，则将默认值调整为256 了，最大设置值为524288，当前版本默认值为528。注意，这里设置的是可以缓存的表定义信息的数目，而不是内存空间的大小

```
mysql> show global variables like '%definition%';
+------------------------+-------+
| Variable_name          | Value |
+------------------------+-------+
| table_definition_cache | 528   |
+------------------------+-------+
1 row in set (0.00 sec)
 
mysql> show status like '%definition%';    
+--------------------------+-------+
| Variable_name            | Value |
+--------------------------+-------+
| Open_table_definitions   | 70    |
| Opened_table_definitions | 0     |
+--------------------------+-------+
2 rows in set (0.02 sec)
```

 

**4、内存的组成**

innodb存储引擎可以分为三部分：内存、进程、数据文件

innodb的内存的作用大致如下

 

- 缓存磁盘上的数据，方便快速的读取；
- 对磁盘文件的数据进行修改之前在这里缓存；
- 应用所作的日志的缓存；
- 内存结构自身的管理结构

 

**1）Innodb buffer pool**

缓冲池是最大块的内存部分，主要用来各种数据的缓冲。innodb将数据文件按页（16K）读取到缓冲池，然后按最少使用（LRU)算法来保留缓存数据；数据文件修改时，先修改缓存池中的页（即脏页），然后按一定平率将脏页刷新到文件

```
mysql> show variables like 'innodb_%_size';
+---------------------------------+------------+
| Variable_name                   | Value      |
+---------------------------------+------------+
| innodb_additional_mem_pool_size | 2097152    |
| innodb_buffer_pool_size         | 2013265920 |
| innodb_log_buffer_size          | 8388608    |
| innodb_log_file_size            | 1048576000 |
+---------------------------------+------------+
4 rows in set (0.00 sec)
```

按照数据页的类型

1、有索引页
2、数据页
4、undo页
5、插入缓冲
6、自适应哈希索引
7、InnoDB存储的锁信息、数据字典信息等

通过show engine innodb status可以查看缓冲池的具体信息

```
BUFFER POOL AND MEMORY
----------------------
Total memory allocated 2058485760; in additional pool allocated 0
Dictionary memory allocated 819282
Buffer pool size   122879
Free buffers       97899
Database pages     24014
Old database pages 8844
Modified db pages  8
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 6, not young 0
0.00 youngs/s, 0.00 non-youngs/s
Pages read 1049, created 41540, written 30276412
0.00 reads/s, 0.00 creates/s, 1.55 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 24014, unzip_LRU len: 0
```

这边的单位是buffer frame，每个buffer frame为16K，通过计算可以查看buffer pool的使用情况

1、Buffer pool size 122879×16×1024 = 2013249536

2、Free buffers表示当前空闲的缓冲页

3、Database pages表示已经使用的缓冲页

4、Modified db pages 表示脏页的数量

5、Old database pages表示LRU列表中old sublist中的数据块数量

对上面的innodb buffer pool细看会发现，buffer pool的数据类型又可以分为：page cache、hash index、undo、insert buffer、explicit locks

**2）Log Buffer**

日志缓冲池（功能跟oracle redo log buffer基本相似），将重做日志信息放入这个缓冲区，然后按一定频率将其刷新到重做日志文件。该值一般不需要设置很大，因为一般情况下每一秒钟就会将重做日志缓冲刷新到日志文件，只需要保证每秒产生的事物量在这个缓冲大小之内即可

**3）additional buffer pool**

在innodb存储引擎中，对内存的管理是通过一种称为内存堆的方式进行的。在对一些数据结构本身分配内存时，需要从额外获得内存池中申请，当该区 域的内存不够时，Innodb会从缓冲池中申请。但是每个缓冲池中的frame buffer还有对应的缓冲控制对象，这些对象记录了诸如LRU、锁、等待等方面的信息，而这个对象的内存需要从额外内存中申请。因此，当你申请了很大的 Innodb缓冲池时，这个值也应该相应增加；

简单理解为：额外缓冲池用于管理缓冲池的内容的，所以缓冲池越大额外换池也需要越大

**4）内存计算**

```
used_Mem =
+ key_buffer_size
+ query_cache_size
+ innodb_buffer_pool_size
+ innodb_additional_mem_pool_size
+ innodb_log_buffer_size
+ max_connections *(
       + read_buffer_size
    + read_rnd_buffer_size
    + sort_buffer_size
    + join_buffer_size
    + binlog_cache_size
    + thread_stack
    + tmp_table_size
    + bulk_insert_buffer_size
)
```

 sql

```
SELECT (
@@key_buffer_size +
@@query_cache_size +
@@tmp_table_size +
@@innodb_buffer_pool_size +
@@innodb_additional_mem_pool_size +
@@innodb_log_buffer_size +
@@max_connections * (  
@@read_buffer_size +
@@read_rnd_buffer_size +
@@sort_buffer_size +
@@join_buffer_size +
@@binlog_cache_size +
@@thread_stack +
@@bulk_insert_buffer_size ) ) /
@giga_bytes AS MAX_MEMORY_GB;
```


 
参考文章

http://www.ibm.com/developerworks/cn/linux/l-tune-lamp-3.html#resources

http://database.51cto.com/art/201010/229956.htm

http://news.oneapm.com/php-xhprof-xhgui/

http://mp.weixin.qq.com/s?__biz=MjM5NDE0MjI4MA==&mid=208777870&idx=1&sn=6efddd6283e4deb3fe55a141b0db965c&scene=1&srcid=0910kYIbazQSBqZEivwahGHB&key=dffc561732c2265104613d6540d35b8ad5c92c340ed903cbbd8218ac9ba70f5b9d36aaa09033e2f9cf0e7983792311d4&ascene=1&uin=MjM2NjkwNQ%3D%3D&devicetype=Windows+7&version=61020020&pass_ticket=JuXI39h6lA0f0sHQ0AFMc7g2Z4NJXU5U71301BBDAuw%3D

http://pingzhao1990.blog.163.com/blog/static/113566342201531583628765/　

监控指标  

https://blog.csdn.net/tjiyu/article/details/106057996

