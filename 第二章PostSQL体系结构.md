# 第二章 postgreSQL的体系结构
postgreSQL由连接管理系统、编译执行系统、存储管理系统、事务系统、系统表五大部分组成

#### 连接管理系统（系统控制器）
负责接收外部操作系统的请求，对操作请求进行预处理和分发，起系统逻辑控制作用
#### 编译执行系统
由查询编译器、查询执行器组成，完成操作请求在数据库中的分析处理和转化工作，实现物理存储介质中数据的操作
#### 存储管理系统
由索引管理器、内存管理器、外存管理器组成，负责存储和管理物理数据，提供对编译查询系统的支持。
#### 事务系统
由事务管理器、日志管理器、并发控制、锁管理组成，日志管理器和事务管理器完成对操作请求处理的事务一致性支持，锁管理器和并发控制提供对并发访问数据的一致性支持；系统表是Postsql数据库的元信息管理中心，包括数据库对象信息和数据库管理控制信息。系统表管理元数据信息，将Postsql数据库的各个模块有机地连接在一起，形成一个高效的数据管理系统。

---
## 2.1 系统表
&ensp; 在PostgreSQL中系统表扮演数据字典的功能，包含数据库系统中所有对象及其属性的描述信息、对象之间关系的描述信息、对象属性的自然语言含义及数据库状态信息等数据库当中所有的元数据。它在PostgreSQL是一张普通表的形式，但是用户不应当随意修改这些表。</br> 

- 在src/include/catalog目录下若干以"pg\_xxx"开头的.h文件，对应了相应文件名的系统表数据结构，indexing.h定义了所有系统表的TOAST表。

- 在src/backend/catalog目录下的"pg\_xxx.c"文件定义了相应的操作函数，indexing.c文件定义了四个操作系统表索引的函数,toasting.c文件定义了四个操作系统表的TOAST表的函数

[系统表数据结构代码](https://github.com/zhizhengyang/postgresql/tree/master/src/include/catalog)</br>
[相应操作函数代码](https://github.com/zhizhengyang/postgresql/tree/master/src/backend/catalog)

### 2.1.1 主要系统表及功能
| 表名       | 功能                          |
|:----------|:------------------------------|
|pg\_namespace  |用于存储命名空间              |
|pg\_tablespace |存储表空间信息                | 
|pg\_database   |当前数据集簇中数据库的信息      |
|pg\_class      |存储表及与表类似的数据库对象信息 | 
|pg\_type       |存储数据类型信息               |
|pg\_attribute  |存储表的属性信息               | 
|pg\_index      |存储索引的具体信息             |

### 2.1.2 系统视图
系统视图提供了查询系统表和访问数据库内部状态的方法。

--- 

## 2.2 数据集簇
&ensp;由postgreSQL管理的用户数据库以及系统数据库总称为数据集簇。在pgsql中实际是一些文件集合。
在pgsql中，对象标识符，即oid，在整个数据集簇中唯一地标识一个数据库对象，可以是数据库、表、索引、视图、元组等。oid是一个无符号整数。默认状态下，元组是没有oid的，除非用户特别标识出。oid的创建从1开始，但是因为数据库系统初始化也要占用oid,因此用户使用时oid的起始实际是以16384开始。</br>
&ensp;初始化数据集簇包含创建数据库系统所有数据的数据目录、创建共享的系统表、创建其他的配置文件和控制文件，并创建模版数据库和默认用户数据库。对于某一个数据库，在PGDATA/base里对应有一个字目录，字目录的名字即为该数据库在系统表pg\_database里的oid。</br>
&ensp;每个表和索引都存储在其所属数据库目录下的独立文件里，以该表或者索引的filenode号命名，该号码记录在系统表pg\_class中对应元组的relfilenode属性中。</br>
&ensp;在表或者索引超过1GB时，它就被分裂为多个1GB大小的段。第一个段以filenode命名，后面为filenode.1,filenode.2^^……</br>
&ensp;如果表规模过大，那会有相关的toast表，存储无法在数据行放置的超大外置数据。表对应的pg\_class元组的reltoastrelid属性记录了它的toast表oid。</br>
&ensp;在pgsql中，默认会将数据文件放在PGDATA制定的目录下，但如果磁盘不足的话，可以使用**表空间**进行扩展，表空间的物理意义是一个新的磁盘目录。每个用户定义的表空间在PGDATA/pg\_tblspc目录里面都有一个符号链接，指向其物理目录，该符号链接用表空间的oid命名。

### 2.2.1 initdb的执行过程

将从[initdb.c](https://github.com/zhizhengyang/postgresql/blob/master/src/bin/initdb/initdb.c)文件中的main函数开始执行。包含设置环境变量、设置中断信号处理函数、创建数据目录、创建系统视图、系统表toast表等。</br>
![initdb.jpg](https://s1.ax1x.com/2020/06/18/NZzscd.jpg)
### 2.2.2 系统数据库
三个系统数据库template0、postgres都是在初始化时从template1（默认）拷贝得到的。
## 2.3 PostgreSQL进程结构
&ensp;pgsql系统的主要功能都集中于postgres程序，其入口是main函数，负责初始化数据集簇、启动数据库服务器，调用流程如下。
![postgresmain.jpg](https://s1.ax1x.com/2020/06/18/Ne9NBn.jpg)
&ensp;pgsql使用一种专用服务器进程体系结构，最主要的两个进程是守护进程postmaster和服务进程postgres。守护进程postmaster负责整个系统的启动和关闭。它监听并接受客户端的连接请求，为其分配服务进程postgres。服务进程postgres接受并执行客户端发送的命令。它在底层模块（如存储、事务管理、索引）之上调用各个主要的功能模块（如编译器、优化器、执行器等），完成客户端的各种数据库操作，并返回执行结构。
> 在linux下postmaster仅仅是postgres的一个符号链接，windows下为一个拷贝，因此可以说pgsql几乎所有核心功能都由postgres完成

---

## 2.4守护进程postmaster
&ensp;完成数据簇初始化后，用户可以启动一个数据库实例来运行数据库管理系统，多用户模式下一个数据库实例由数据库服务器守护进程postmaster来管理。**它是一个运行在服务器上的总控进程**，负责整个系统的启动和关闭，并且在服务进程出现错误时完成系统的恢复。它管理数据库文件、监听并接受来自客户端的连接请求，并且为客户端请求fork一个postgres服务进程。
![postmasterresponsemodel.jpg](https://s1.ax1x.com/2020/06/18/NeFMrQ.jpg)
&ensp;postmasters也负责管理整个系统范围的操作，例如中断等操作，postmaster指派一个子进程在适当的时间去处理它们。同时它也负责在数据库崩溃时重启数据库。postermaster在起始时会建立共享内存和信号库，postmaster及其子进程的通信就通过共享内存和信号来实现。这种多进程设计的数据库稳定性更好，即使某个进程崩溃也不会影响整个进程。
> [postmaster相关代码](https://github.com/zhizhengyang/postgresql/tree/master/src/backend/postmaster)</br>[postmaster进程源文件](https://github.com/zhizhengyang/postgresql/tree/master/src/backend/postmaster/postmaster.c)</br>
[统计数据收集进程的源文件pgstat.c](https://github.com/zhizhengyang/postgresql/tree/master/src/backend/postmaster/pqstat.c)</br>
[预写式日志归档进程的源文件pgarch.c](https://github.com/zhizhengyang/postgresql/tree/master/src/backend/postmaster/pgarch.c)</br>
[后台写进程的源文件bgwrite.c](https://github.com/zhizhengyang/postgresql/tree/master/src/backend/postmaster)</br>
[系统日志进程的源文件syslogger.c](https://github.com/zhizhengyang/postgresql/tree/master/src/backend/postmaster/syslogger.c)
[系统自动清理进程的源文件](https://github.com/zhizhengyang/postgresql/tree/master/src/backend/postmaster/autovacuum.c)</br>
**[postmaster入口函数](https://github.com/zhizhengyang/postgresql/tree/master/src/backend/postmaster/postmaster.c)**

### 2.4.1 初始化内存上下文
&ensp;运行时大多数内存分配操作在各种语义的内存上下文（memeorycontext）中进行。内存上下文释放时会释放在其中分配的所有内存，避免内存泄漏。</br>
&ensp;程序首先调用memorycontextinit创建topmemorycontext和errorcontext。然后调用allcsetcontextcreate和topmemorycontext为根结点创建postmastercontext，最后将全局指针currentmemorycontext指向postmastercontext。
### 2.4.2 配置参数
### 2.4.3 注册新号处理函数
&ensp;postmaster定义了三个信号集:BlockSig是要屏蔽的信号集；unblocksig是不希望屏蔽的信号集;authblocksig是在进行用户连接时需要屏蔽的信号集，它们都是位向量。
三个主要的信号处理函数


| 函数名            | 功能                                         |
|:-----------------|:--------------------------------------------|
| sighup_handler   | 信号处理函数，收到sighup信号时重读postgresql.conf|
| pmdie.           | 处理三种信号，对应三种不同的系统关闭方式           |
| reaper           | 清理退出的子进程                               |

### 2.4.5 辅助进程启动
&ensp;在pgsql中，除了postmaster和postgres外，还有一些其他的辅助进程，在这些进程出问题时，postmaster重新产生这些出问题的进程。在postmaster的创建过程中会首先启动syslogger日志进程，并完成pgstat进程、autovacuum进程的初始化工作，在postmaster的监听循环中检测辅助进程的状态，并新建或者重新创建这些辅助进程。

1. syslogger辅助进程
> postmaster通过调用syslogger_start函数启动syslogger子进程，syslogger通过从一个postmaster、所有后台进程以及其他的子进程哪里收集所有的stderr输出，并写入到日志文件中。

2. 辅助进程初始化

> syslogger辅助进程启动完后，postmaster开始对辅助进程pgstat进程、auto vacuum进程进行初始化操作，为进程分配必要的资源。

### 2.4.6 装载客户端认证文件

注册完信号处理函数后，将逐行读取data目录下的pg_hba.conf和pg_ident.conf两个配置文件的内容到链表变量中，用于控制客户端认证。其中pg_hba.conf基于主机认证的配置文件，pg_ident.conf基于身份认证的配置文件。

### 2.4.7 循环等待客户连接请求
&ensp;postmaster总是监听用户连接请求并为用户分配服务进程postgres，而postgres则负责为客户端执行各种命令。</br>
&ensp;postmaster调用[serverloop函数](https://github.com/zhizhengyang/postgresql/blob/master/src/backend/postmaster/postmaster.c)来循环等待客户端的连接请求，它的主要功能是监听用户的连接请求并建立与该用户的连接，然后通过fork函数复制一个postgres进程为该用户服务。

## 2.5 辅助进程

&ensp;postmaster进程中，为每个辅助进程设置了一个全局变量来标识该进程的进程号，分别为sysloggerpid,bgwriterpid,walwriterpid,autovacpid,pgarchpid,pgstatpid。当变量值为0时，表示进程尚未启动。只有日志辅助进程syslogger在postmaster监听循环之前完成启动操作。

### 2.5.1 syslogger系统日志进程
略

### 2.5.2 bgwriter后台写进程	

&ensp;bgwriter是postgresql中在后台将脏页写出到磁盘的辅助进程，引入该进程主要有两个目的。
1. 为缓冲区腾出空间，降低查询处理被阻塞的可能性。
2. 减少设置检查点时要进行的io操作，使系统的io负载趋向平稳

&ensp;bgwriter辅助进程在postmaster中启动，入口为startbasckgroundwriter函数，bgwriter和walwriter两个辅助进程采用相同的模式启动。bgwriter的主要函数是backgroundwritermain,有七步

1. 变量初始化
2. 注册新号处理函数
3. 运行环境初始化
4. 注册异常处理
5. 处理写磁盘请求
6. 处理信号分支
7. 创建检查点

### 2.5.3 WALwriter预写式日志写进程
&ensp;预写式日志（write ahead log,或者称xlog)的中心思想是对数据文件的修改必须发生在修改已经记录到日志之后，在系统崩溃后靠日志来恢复数据库。

> 略

### 2.5.4 pgarch预写式日志归档进程

&ensp;pgarch辅助进程是对wal日志在磁盘上的存储形式进行归档备份。
> 略

### 2.5.5 autovacuum系统自动清理进程
&ensp;在pgsql中，对表元组的update或delete操作并未立即删除旧版本的数据，表中的旧元组只是被标示为删除状态，并为立即释放空间，因而其占据的空间必须回收以供其他新元组使用，对数据库的清理工作通过引入辅助进程vacuum来实现，自动执行vacuum和analyze命令，回收被标示为删除状态记录的空间。</br>
&ensp;它包含两个两种不同的处理进程:1.autovacuum launcher和autovacuum worker。autovacuumlauncher进程为监控进程,用于收集数据库运行信息，根据数据库选择选中一个数据库，并调度一个AutoVacuum Worker进程执行清理操作。选择方式略。

### 2.5.6 PgStat统计数据收集进程

&ensp;PgStat辅助进程是PostgreSQL数据库系统的统计信息收集器，如在一个表和索引上进行了多少次插入与更新操作、磁盘块的数量和元组的数量、每个表上最近一次执行清理和分析操作的时间，以及统计每个用户自定义函数调用执行的时间等。

## 2.6 服务进程Postgres
&ensp;Postgres进程是十几的接受查询请求并调用相应模块处理查询的PostSQL服务进程。它直接接受用户的命令进行编译执行，并将结果返回给用户。
&ensp;Postgres的启动方式有两种：1.在Postmaster监控下，动态地被Postmaster创建为用户服务，这是一种多用户的服务方式，也是比较*通常的启动方式*2.不经过Postmaster以单用户模式直接启动，*为单一用户提供服务*，这种方式由single选项启动，在这种模式下，Postgres服务器进程必须自己完成初始化内存环境、配置参数等操作。

Postgres进程的主要[源代码](https://github.com/zhizhengyang/postgresql/tree/master/src/backend/tcop)，包括

> 服务进程的源代码文件[postgres.c](https://github.com/zhizhengyang/postgresql/tree/master/src/backend/tcop/postgres.c)</br>
对于查询命令进行处理的源代码文件[pquery.c](https://github.com/zhizhengyang/postgresql/tree/master/src/backend/tcop/pquery.c)，它执行一个分析好的查询命令</br>
对于非查询命令进行处理的源代码文件[utility.c](https://github.com/zhizhengyang/postgresql/tree/master/src/backend/tcop/utility.c)，它执行各种非查询命令</br>
[dest.c](https://github.com/zhizhengyang/postgresql/tree/master/src/backend/tcop/dest.c)中主要处理Postgres和远端客户的一些消息通信操作，并负责返回命令的执行结果。

其主要函数PostgresMain函数工作流程如下图

![postgresmainfunction.jpg](https://s1.ax1x.com/2020/06/18/Nm8LJf.jpg)

### 2.6.1 初始化内存环境
调用MemoryContextInit来完成(2.4.1节)

### 2.6.2 配置运行参数和处理客户端传递的GUC参数
&ensp;和Postmaster中一样，从Port结构得到客户端传递的GUC选项，然后根据GUC选项的具体情况调用SetConfigOption进行配置
### 2.6.3 设置信号处理和信号屏蔽函数

### 2.6.4 初始化Postgres的运行环境
&ensp;单用户模式下，首先检查DataDir变量，确保给定的DataDir字符串是一个格式正确的路径。多用户模式下由Postmaster完成
&ensp;路径设置完后，Postgres的初始化工作分为两个阶段
1. 调用BaseIinit函数完成基本初始化
2. 之后调用InitPostgres来完成Postgres的初始化。

### 2.6.5 创建内存上下文并设置查询取消跳跃点
&ensp;Postgres首先创建一个名为MessageContext的内存上下文，用于存储从前段发来的消息中的查询命令，以及在查询过程中产生的中间数据（例如在简单查询模式时产生的分析树和计划树），每当PostgresMain进行下一次循环（即进行新查询）时刻内存上下文将被重设，即所有经过MessageContext分配的内存块将被释放。
&ensp;创建完成后将调用sigsetjmp（系统调用函数）设置跳跃点，当客户端取消一次查询请求或者发生错误时，将通过siglongjmp利用全局变量PG_exception_stack从这个点退出当前事务然后重新开始查询，在错误恢复期间不允许被中断，也不接受任何客户端取消查询的请求。

### 2.6.6 循环等待处理查询
&ensp;完成上述工作后，Postgres可以真正接受客户端查询并处理，系统开始运行。

### 2.6.7 简单查询的执行流程

&ensp;系统调用exec_simple_query函数来执行简单查询,流程如下图。
![execsimplequery.jpg](https://s1.ax1x.com/2020/06/19/Nu9GLQ.jpg)
&ensp;它解读用户输入的字符串形式的命令并检查合法性，最终执行并返回结果。数据库的编译器、分析器、优化器和执行器**都是在这个函数中调用的**，它们相互协作，构成了服务进程的主体，下面大致说明下它们的主要功能和代码的位置。

1. 编译器：编译器是主流程中的第一个模块。它的作用是扫描用户输入的字符串形式的命令，检查其合法性，并将其转换为Postgres定义的内部数据结构。Postgres为每一条SQL命令都定义了相应的C语言结构体，用来存放命令中的各种参数。编译器是利用著名的lex和yacc工具编写的，其入口函数为pg_parse_query函数。代码位于src/backend/paser目录下的[scan.l](https://github.com/zhizhengyang/postgresql/blob/master/src/backend/parser/scan.l)和[gram.y](https://github.com/zhizhengyang/postgresql/blob/master/src/backend/parser/gram.y)文件中

2. 分析器：分析器接收编译器传递过来的各种命令数据结构（语法树），对它们进行相应的处理，最终转换为统一的数据结构Query。如果是查询命令，在生成Query后进行规则重写。重写部分的入口是QueryRewrite函数。如果是查询命令，在生成Query后进行规则重写（rewrite）。重写部分的入口时QueryRewrite函数，代码位于[src/backend/rewrite](https://github.com/zhizhengyang/postgresql/tree/master/src/backend/rewrite)目录下

3. 优化器：优化器接收分析器输出的Query结构体，进行优化处理后，输出执行器可以执行的计划。一个特定的SQL查询可以以多种不同的方式执行，优化器将检查每个可能的查询计划，最终选择运行最快的查询计划。优化器的入口是pg_plan_query函数，代码位于[src/backend/optimizer](https://github.com/zhizhengyang/postgresql/tree/master/src/backend/optimizer)下

4. 执行器：执行非查询命令的入口函数是ProcessUtility，代码位于[src/backend/tcop/utility.c](https://github.com/zhizhengyang/postgresql/blob/master/src/backend/tcop/utility.c)，该函数的主体结构是一个switch语句，根据输入的命令类型调用相应的执行函数。执行查询命令的入口函数是ProcessQuery，代码主要位于[src/backend/executor](https://github.com/zhizhengyang/postgresql/tree/master/src/backend/executor)



