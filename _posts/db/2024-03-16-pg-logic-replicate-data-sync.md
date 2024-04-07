---
title: postgresql逻辑复制和数据同步
author: june
date: 2024-03-16
category: db
layout: post
---

### 背景
笔者曾在工作中，需开发数据列表筛选交互功能，当时公司存储业务数据选取的是postgresql（下文简称pg）关系型数据库。而为了实现全部筛选条件，
还需要join其它同等量级的表。 一开始相关表数据量只有几十万，所以即便需join其它表，也不存在性能问题。
随着业务的发展，表的数据量迅速达到了千万级，甚至后来达到了亿级，再采取原有方案，千万数据量表采取join操作明显是不现实的。
为了解决大列表的筛选问题，我们决定将多张表的数据同步到elasticSearch，利用ES强大的检索能力解决这种大列表的筛选问题。

因此， 本文主要讲如何通过postgrepsql（下文简称pg）的逻辑复制功能，利用kafka，同步数据到第三方数据库（比如elassticsearch）。

### pg的逻辑复制
关于pg逻辑复制流程如下图，先简单总结一下pg逻辑复制工作原理：在主数据库（publisher）上，当app往pg执行更新操作并commit之后，
后端进程通过执行函数XLogInsert()和XLogFlush()，将WAL数据写入并刷新到WAL段文件中。
主数据库通过逻辑解码器（Logical Decoding）解析WAL（Write Ahead Log）日志中的事务记录，将其转换为逻辑格式，例如SQL INSERT/UPDATE/DELETE语句或类似的变化事件。
walsender进程将解析后的的WAL数据发送到从库（subscriber）的walreceiver进程，在从库的walreceiver进程接收到WAL日志后，从库中应用主数据的WAL日志。
逻辑复制利用了复制槽（Replication Slot）来保证即使在主数据库发生故障后，也能保留足够的 WAL 日志以便从数据库恢复未完成的复制任务。

![pglogicreplicate_f1](/assets/post/db/pglogicreplicate_f1.png "pglogicreplicate_f1")

逻辑复制几个比较重要的概念：复制标识，逻辑复制槽，时间线。

### 复制标识
在PostgreSQL中，“复制标识”（Replica Identity）是指在进行逻辑复制时，用于唯一标识表中某一行的方式。REPLICA IDENTITY 是用于更改写入预写日志（write-ahead log，WAL）中记录的识别被UPDATE和DELETE行的信息形式。这一设置对于确保正确识别并同步到订阅者端的行变化至关重要。
以下是不同REPLICA IDENTITY设置的含义：
- DEFAULT：默认设置，适用于非系统表。在这种情况下，只记录主键列（如果有）的旧值。当行发生UPDATE或DELETE操作时，仅当主键列的旧值与新值不同时，才会将旧值记录到WAL中。
- USING INDEX index_name： 使用指定的唯一、非部分（即索引覆盖了所有列）、不可延迟且包含的所有列标记为NOT NULL的独特索引来记录旧值。如果该索引被删除，则行为会等同于NOTHING设置。
- FULL：记录行内所有列的旧值。这意味着即使没有合适的主键或其他独特索引，也会完整地记录更新或删除前的整行数据。在无法或不方便使用其他更精简的复制标识时，可以使用此设置，但请注意它可能导致日志空间使用增加以及同步效率降低。
- NOTHING：不记录关于旧行的任何信息。这是系统表的默认设置。在使用NOTHING设置时，由于缺少足够的信息，可能无法在订阅者端精确地识别和应用对应的UPDATE或DELETE操作，从而无法实现有效的逻辑复制。

这种情况是不符合我们项目要求的，因为常常会有这种业务场景：对比更新记录的某个字段（这个字段是不确定的）更新前后值变化去做对应的业务逻辑，比如监听表变动触发某个更新功能，或者监听表变动同步数据到es。
所以我们项目是将需要监听的表的复制标识设置为full，以获取整行所有字段更新删除前后的值。

### 逻辑复制槽
在逻辑复制的背景下，一个复制槽代表着可以在客户端按原始服务器上发生顺序重播的一系列变更。每个复制槽从单个数据库中流出一系列连续的变更。

复制槽在整个PostgreSQL集群的所有数据库中具有唯一的标识符。复制槽独立于使用它们的连接存在，并且具有崩溃安全特性，即即使数据库系统崩溃后，复制槽的状态仍能保持。

逻辑复制槽在正常运行时，每个变更仅发出一次。每个槽位的当前位置只在检查点时刻持久化保存，因此在发生崩溃的情况下，槽位可能会回滚到较早的LSN（Log Sequence Number，日志序列号），这将导致在服务器重启后，最近的变更会被再次发送。逻辑解码客户端负责避免因处理同一消息多次而引发的问题，通常做法是记录下解码时最后看到的LSN，然后跳过任何重复的数据；或者（当使用复制协议时），请求从该LSN开始解码，而非让服务器确定起始点。为此目的设计的特性称为“复制进度跟踪”，请参考“复制起源”部分。

同一个数据库可以存在多个独立的复制槽。每个槽都有自己独立的状态，允许不同的消费者从数据库变更流的不同点接收变更。对于大多数应用来说，每个消费者通常需要一个单独的复制槽。

逻辑复制槽并不了解接收者（consumer）的状态。甚至有可能在同一复制槽上，在不同时间段有不同的接收者在使用，它们将接收到自上一个接收者停止消费之后发生的变更。任何时候，只有一个接收者可以消费来自某个复制槽的变更。

### 时间线
wal log是记录数据库变更的日志，随着数据库不断运行，会产生与旧的WAL文件重名的文件，这些文件进入归档目录时，会覆盖原来的旧日志，导致恢复数据库需要的WAL文件丢失。
为了解决这些问题，引用了时间线的概念。当开始逻辑复制之前需要告诉数据库需要从哪个时间线，wal log哪个postion开始逻辑复制，默认是当天数据库时间线开始的所有wal。
如果想了解pg的逻辑复制如何使用时间线的，可以看看[流复制协议]( http://www.postgres.cn/docs/9.4/protocol-replication.html )。

我们项目是传的是默认值，也就是从数据库当前时间线开始的所有wal。


### 顺序消费kafka消息
同步数据我们需要保证顺序消费消息，不然同步数据到第三方数据库可能会出现旧数据覆盖新数据的情况。比如顺序执行set a=1，set a=2。如果先消费a=2的消息，更新es a=2，然后再消费a=1消息，更新a=1。这样同步到es的数据就是错误的。
我们都知道，kafka的topic消息是分partition存储的，同一Consumer Group中的多个Consumer实例，不同时消费同一个partition，等效于队列模式。
所以只要保证Consumer Group的Consumer和partition数量保持一致，消费者就能顺序消费partition消息，而同一个表的数据变更消息只要指定存储在同一个partition，同一张表的变更消息就能顺序消费了。
我们项目就是通过acm配置表的顺序，将表变更消息顺序的指定存储在某个某个分区。关于kafka相关知识点可以看下这篇文章：[Kafka设计原理]( https://cloud.tencent.com/developer/article/1005736 )。

### 数据同步方案
通过pg逻辑复制同步数据到es方案如图。首先启动一个etl服务，作为walrecever接收wal数据，并将wal数据做进一步解析，将解析后的wal数据以数据的表名作为key
hash到kafka不同的partition并发送kafka, etl服务记录当前wal的lsn，并向walsender ask该lsn。再启动一个subscriber服务，消费不同表的kafka数据，并
写到es上。
![pglogicreplicate_f2](/assets/post/db/pglogicreplicate_f2.png "pglogicreplicate_f2")


相关文章：
- [《Postgresql逻辑复制》]( https://www.postgresql.org/docs/current/logical-replication.html )
- [《Postgresql逻辑解码》]( https://www.postgresql.org/docs/current/logicaldecoding-explanation.html )
- [《Postgresql变更事件捕获》]( https://github.com/Vonng/pg/blob/master/arch/logical-decoding.md )
- [《Postgresql的时间线解析》]( http://mysql.taobao.org/monthly/2015/07/03/ )。
