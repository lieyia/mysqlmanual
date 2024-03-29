# 第8章 优化 #
   **目录**

##  8.1 优化概述##
##  8.2 优化SQL语句##
### 8.2.1  优化SELECT语句 ###
### 8.2.2  优化DML语句   
### 8.2.3  优化数据库权限      
### 8.2.4  优化数据库信息      
### 8.2.5  其他优化话题       
##  8.3 优化和索引   ##
### 8.3.1  MYSQL如何使用索引       
### 8.3.2  使用主键索引       
### 8.3.3  使用外键索引
### 8.3.4  列索引
### 8.3.5  组合索引
### 8.3.6  索引使用验证
### 8.3.7  InnoDB、MyISAM索引统计信息收集
### 8.3.8  B-数索引和hash 索引比较
##  8.4 优化数据库结构
### 8.4.1  数据大小优化
### 8.4.2  MYSQL数据类型优化
### 8.4.2  多表优化
##  8.5 Innodb表优化
MYSQL用户通常在生成环境上部署InnoDB存储引擎，因为稳定性和并发性对生产环境非常重要。因为InnoDB是MYSQL5.5及更高版本默认的存储引擎，因此相比早前版本，你会更多看到使用InnoDB存储引擎表。这节解释如何针对InnoDB表优化数据库操作。
### 8.5.1  InnoDB存储层优化


- 一旦数据大小稳定或者某个表增加了10M~1G，用户就可以考虑使用OPTIMIZE TABLE语句重新组织表、压缩浪费空间。重新组织后的表在做全表扫描时，减少了IO次数。这是一个改善性能的直观方法，当其他方法不可用，例如改进索引的使用率和优化应用程序代码。

    OPTIMIZE TABLE语句拷贝表数据并重建索引。优势在于索引内部数据紧密，减少了表空间和磁盘上碎片。这种优势因表而异。你可能会发现这并非对所有表效果显著，或者效果不断下降直到你再次使用OPTIMIZE TABLE。如果表很大或者重建索引不能在buffer pool中完成，操作会很慢。当向表中增加大量数据后运行OPTIMIZE TABLE命令会较过一段时间运行慢很多。


- 在InnoDB表中，一个长主键(长单列值或者多列组合值)浪费大量磁盘空间。一条记录的主键在所有二级索引中出现。(参考14.2.3.12，"Innodb表索引结构")。如果你的主键值很大那么创建自增长列做为主键，或者使用varchar的前缀做为索引而不是整个列做为索引。

- 使用varchar类型而不是char存储变长string或者有很多null值的列。一个CHAR(N)列总是占用N个字符存储数据，即使列长度小于N或者为null。小表能更好匹配buffer pool和减少IO。

     当使用compact行格式(MYSQL5.6缺省Innodb格式)和变长的字符集，比如utf8或者sjis，CHAR(N)列占用可变数量的空间，但至少N字节。

- 针对大表，或者包含很多重复文本、数值数据，考虑使用compressed格式。减少将数据载入buffer pool的磁盘io或者全表扫描。

- 在做永久决定之前，通过使用COMPRESSED或者COMPACT行格式衡量压缩后的数量。        
### 8.5.2  InnoDB事务管理优化
为InnoDB优化事务处理，找出事务功能的性能极限和服务器负载的均衡点。例如，应用程序可能遭遇性能问题如果每秒提交上千次，以及完全不同的性能问题如果每隔2-3小时提交一次。

- MYSQL缺省AUTOCOMMIT=1会在一个繁忙数据库服务器上限制性能。事实上，将相关的DML语句放在一个单一的事务中，通过设置SET AUTOCOMMIT=0或者使用START TRANSACTION，完成数据库所有修改后使用COMMIT语句。
   
    如果一个事务对数据库进行了修改，那么InnoDB每次事务提交必须刷新日志到磁盘。每次修改后提交(缺省的AUTOCOMMIT设置)，存储设备的IO吞吐量限制每秒钟的隐藏操作数量。
- 通常，包含单条SELECT语句的事务，打开AUTOCOMMIT参数有助于InnoDB存储引擎重新组织只读事务并且优化它们。查看14.2.3.2.4，"优化只读事务"小节获取更多信息。

-  避免在插入、更新、删除大量记录后执行回滚操作。如果一个大事务降低了服务器性能，回滚会使问题更糟，可能会在原始执行时间上增加几倍。杀死数据库进程无助于解决问题，因为数据库启动时会重新执行回滚操作。

    为降低这种问题发生的几率，可以通过增加buffer pool大小从而所有的DML操作缓存在内存中而不需要写回到磁盘；设置innodb_change_buffering=all不仅缓存插入操作同时还有更新、删除；考虑在一个大DML中定期使用commit，可能会将单个删除或更新分解为多个语句，这种语句具有较小的数据集。

    为消除失控的回滚操作，增加buffer pool以使回滚操作在CPU的边界内执行，或者杀掉服务以参数innodb_force_recovery=3启动，这个参数的解释在14.2.3.13小节InnoDB恢复进程。

    这个问题在MYSQL5.5及更高版本，或以插件方式的5.1中被认为不突出，因为缺省设置innodb_change_buffering=all允许更新和删除操作在内存中完成，使它们第一次执行的时候更快，同样如果需要回滚也更快。确保使用这个参数在运行含有大量插入、更新、删除操作的长事务。

- 如果你能容忍系统崩溃时丢失部分最近提交的事务，你可以设置innodb_flush_log_at_trx_commit=0。innodb试图每秒刷新日志，并且刷新也不保证。同样，设置innodb_support_xa=0，将减少同步硬盘数据和binary日志的磁盘刷新操作。

- 当记录被修改或者删除，记录和相关的undo日志并不是立即删除，或者事务提交后立即删除。旧数据会被保留直到比该事务更早或者同时启动的事务完成，因此这些事务可以访问修改或者删除记录之前的状态。因此，一个长时间运行的事务可以阻止InnoDB清理由其他事务修改的数据。  

-  当一个长时间运行的事务修改或删除记录，其他使用读提交和可重复读隔离级别的事务如果要读这些被修改的记录，必须做更多的工作重构这些旧数据。


- 当一个长时间运行的事务修改表，这个表上其他事务的查询不能利用覆盖。通常可以使用二级索引检索所有的结果而不是在表中查询合适的值。


    当二级索引页的值PAGE_MAX_TRX_ID太新时，二级索引中的记录被标记为删除时，InnoDB可能需要聚簇索引查询记录。  
### 8.5.3  InnoDB日志优化
- 使你的日志文件尽可能大，即使和buffer pool一样。当InnoDB写满日志文件，InnoDB必须把buffer pool中修改的记录在一个检查点中写会到磁盘。小日志文件引起许多不必要的写磁盘。虽然历史上大日志文件引起长时间的恢复操作，但现在恢复要快很多，你可以放心使用。

- 使日志缓冲尽可能大(大约8MB)。
### 8.5.4  InnoDB数据装载
下面的方法针对快速插入(8.2.2.1小节"加速插入语句")提供了常用的方法。

- 当向InnoDB中导入数据，关闭autocommit模式，因为每次插入都会刷新一次磁盘，通过在SET autocommit和commit语句中间添加导入语句，关闭自动提交。

    SET autocommit = 0; ...SQL import statements ...
    COMMIT;

    mysqldump选项 --opt创建一个快速导入到InnoDB表中的dump文件，即使没有用SET autocommit和commit包装。

- 如果你在二级索引上有UNIQUE限制，你可以通过在导入会话中临时关闭唯一性检查，加速想表中导入数据。

    SET unique_checks = 0; ...SQL import statements ...
    SET unique_checks = 1;

    对大表，这种方式可以节省许多IO操作，因为InnoDB可以使用插入缓存向二级索引批量插入数据。确保数据没有重复键。

-  如果你的表中有外键索引，你可以关闭外键检查以加速插入速度。

    
    SET foreign_key_checks = 0; ...SQL import statements ...
    SET foreign_key_checks = 1; 

    这种方式可以节省大表IO操作。

- 如果你需要插入多行记录，使用多行插入语法服务器与客户端的交互瓶颈。

    insert into yourtable VALUES (1,2),(5,5),...;
    
    这种方法对任何表有效，不仅是InnoDB表。


-  当向含有auto-increment列的表中大量插入数据，设置innodb_autoinc_lock_mode为2替换缺省值1。查看5.4.4.2小节，"配置自增锁"获得更多细节。

- 当向InnoDB全文索引中转载数据时，按照如下步骤优化：

    建表时定义一个BIGINT UNSIGNED NOT NULL类型的列FTS_DOC_ID，在该列上创建唯一索引FTS_DOC_ID_INDEX。例如：

    CREATE TABLE t1 ( 
    FTS_DOC_ID BIGINT unsigned NOT NULL AUTO_INCREMENT, 
    title varchar(255) NOT NULL DEFAULT ”, 
    text mediumtext NOT NULL, 
    PRIMARY KEY (`FTS_DOC_ID`) 
    ) ENGINE=InnoDB DEFAULT CHARSET=latin1; 
    CREATE UNIQUE INDEX FTS_DOC_ID_INDEX on t1(FTS_DOC_ID);

    向表中装载数据；

    装载数据后，创建全文索引；

    注意：
         当建表时增加FTS_DOC_ID列，确保全文索引列更新后，FTS_DOC_ID列更新，因为FTS_DOC_ID列单调递增每次插入或更新。如果你选择不添加FTS_DOC_ID列，使用InnoDB为你管理DOC ID，当下次调用create fulltext index时InnoDB将为你增加隐藏列FTS_DOC_ID。这种方法，将重建表从而影响性能。
    
### 8.5.5  InnoDB查询优化
在表上创建合适的索引，优化InnoDB表查询。查看8.3.1小节，"MYSQL如何使用索引"获取更多细节。遵守以下使用InnoDB索引规则：

-  因为每个表都有主键索引(无论你是否指定)，为每个表指定一个主键列，这些列在重要的和要求响应快的查询中使用。

-  不要在太多或者太长的列中指定主键，因为这些列值会复制到每个二级索引，当一个索引包含不必要的值，将这些值读到内存缓存的IO会降低系统性能和稳定性。

-  不要为每个列创建单独的二级索引，因为每个查询仅能利用一个索引，在一个很少值或者列上不同的值很少对查询可能不会有用。如果你在同一个表上有很多查询，尝试不同的列组合，创建复合索引而不是大量多个单列索引。如果你的索引包含查询的所有列(覆盖索引)，查询可能会避免查询表数据。

-  如果一个索引列不能包含任何空值，创建表时声明它为 NOT NULL。当优化器知道每个列是否包含空值，他能更好的决定哪个索引对查询最有效。


- 在MYSQL5.6.4及更高版本，你可以优化针对InnoDB表单查询事务，使用14.2.4.2.3小节"只读事务优化"中的技术。


- 如果你在更新不是很频繁的表上多次查询，打开查询缓存。

    [mysqld]
    query_cache_type=1
    query_cache_size=10M  
### 8.5.6  Innodb数据定义语句优化
-  表和索引上的DDL操作(create,alter,drop语句)，对InnoDB来说，最重要的是创建和删除二级索引在MYSQL5.5以上(含)要比MYSQL5.1一下(含)要快很多。查看在InnoDB中快速创建索引获取更多细节。

-  快速创建索引在某些场景下较先删除索引、载入数据，再创建索引要快。

-  使用TRUNCATE TABLE清空表，而不是DELETE FROM tbl_name。外键约束使TRUNCATE 语句和标准的DELETE 语句一样，在这种情况下，顺序执行DROP TABLE ,CREATE TABLE可能是最快的。

-  因为主键集成到每个InnoDB表存储引擎层，修改主键定义需要重新组织整个表，通常是把主键做为创建表的一部分，提前执行，因此之后不需要修改和删除主键。 
### 8.5.7  InnoDB磁盘IO优化
如果你遵守最后的数据库设计惯例和针对SQL操作的优化方法，但你的系统仍然因为高IO而响应很慢，探索和底层相关的IO技术。如果使用UNIX的TOP工具和windows的任务管理工具显示CPU使用率低于70%，工作量可能在磁盘上。

-  当数据缓存在InnoDB缓冲池中，它可以被不断处理而不需要磁盘IO请求。通过innodb_buffer_pool_size参数设置缓存池大小。这个值非常重要，压力大数据库服务器一般设置为物理内存的80%。获取更大信息，查看8.9.1，"InnoDB缓冲池"。

-  在某些GNU/LiNUX、Uinx版本，使用Unix fsync()调用(InnoDB默认采用)刷新文件到磁盘会令人惊讶的慢。如果数据库写性能存在问题，使用innodb_flush_method参数设置基准函数为O_DSYNC。

- 在Solaris10或x86_64(AMD Opteron)上使用InnoDB存储引擎，对InnoDB相关的表使用直接IO方式，避免性能下降。对整个UFS文件系统存储InnoDB文件，以forcedirectio的方式挂载；查看mount_ufs(1M)。(Solaris10或x86_64缺省不是这个选项)。只对InnoDB文件操作采用直接IO而不是整个文件系统，设置innodb_flush_method=O_DIRECT。使用这个选项，InnoDB调用directio()而不是fcntl()写数据文件(日志文件除外)。

- 当在Solaris2.6及以上和任何平台上(sparc/x86/x64/amd64)设置innodb_buffer_pool_size为很大值的InnoDB存储引擎上，将InnoDB数据文件和日志文件放在裸设备或独立的直接IO UFS文件系统上，使用forcedirectio的方式挂载。(如果你想对日志文件使用直接IO就使用这个挂载选项而不是设置innodb_flush_method参数)。Veritas文件系统V小FS用户应该使用 convosync = direct挂载选项。

    不要将MYSQL数据文件，比如MyISAM表，放置在直接IO文件系统中。不要将可执行文件、库文件放置在直接IO文件系统上。

- 如果你有额外的存储设备安装RAID或符号链接到其他的磁盘上，8.11.3小节"优化磁盘IO"获取更多底层IO论述。

- 如果因为InnoDB检查点操作，吞吐量周期性下降，考虑增加innodb_io_capacity配置项值。高值使刷新更频繁，从而避免因工作积压导致吞吐量下降。

- 如果系统性能没有因为刷新操作而下降，考虑降低innodb_io_capacity配置项值。特别的，在实际使用中将这个选项配置的尽可能低，但不要引起在前面提到的引起吞吐量周期性的下降。在哪种场景下降低这个值，你可以综合考虑如下参数，这些参数通过SHOW ENGINE INNODB STATUS输出：

  
  
- 当IO调优时，其他需要考虑的配置项参数：

    innodb_adaptive_flushing [1738], innodb_change_buffer_max_size [1746], 
    innodb_change_buffering [1746],
    innodb_flush_neighbors [1757], 
    innodb_log_buffer_size [1768], 
    innodb_log_file_size [1769], 
    innodb_lru_scan_depth [1770], innodb_max_dirty_pages_pct [1770], 
    innodb_max_purge_lag [1771],
    innodb_open_files [1775], 
    innodb_page_size [1776], 
    innodb_random_read_ahead [1778], innodb_read_ahead_threshold [1778], 
    innodb_read_io_threads [1778], 
    innodb_rollback_segments [1780], 
    innodb_write_io_threads [1789], 
    and sync_binlog [2055]. 
### 8.5.8  InnoDB变量配置优化
### 8.5.9  多表的系统优化
##  8.6 MyISAM表优化
### 8.6.1 MyISAM查询优化
### 8.6.2 MyISAM表数据装载
### 8.6.3 REPAIR语句优化
##  8.7 MEMORY表优化
##  8.8 查询计划理解
### 8.8.1  使用EXPLAIN优化查询
### 8.8.2  EXPLAIN输出格式
### 8.8.3  EXPLAIN EXTENDED输出格式
### 8.8.4  估算查询性能
### 8.8.5  查询优化控制
##  8.9 Buffering 和 Caching
### 8.9.1
### 8.9.2
### 8.9.3
##  8.10 锁优化
### 8.10.1
### 8.10.2
### 8.10.3
### 8.10.4
##  8.11 优化MYSQL服务器
### 8.11.1
### 8.11.2
### 8.11.3
### 8.11.4
### 8.11.5
### 8.11.6
##  8.12 性能测量
### 8.12.1
### 8.12.2
### 8.12.3
### 8.12.4
### 8.12.5
