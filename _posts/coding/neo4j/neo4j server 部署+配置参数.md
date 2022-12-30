---
layout: post
title: "neo4j server 安装+配置参数"
subtitle: "neo4j server 安装+配置参数"
date: 2022-12-30
author: "dimi"
header-img: "img/post-bg-alitrip.jpg"
catalog: true
tags: [coding,neo4j]
---

## 安装

1. 官网拿到linux安装包，在服务器上解压  ，当前使用的是企业版4.4.8

    `tar -axvf 包名`

2. 添加环境变量，用户级别

    ```bash
        vim ~/.bash_profile
        # 添加
        export NEO4J_HOME=/home/neo4j
        export PATH=$PATH:$NEO4J_HOME/bin
    ```
    neo4j 4.x 需要jdk11以上，或者直接在启动文件中指定专用的jdk路径  
    在启动文件 ./bin/neo4j 中，指定jdk路径，添加

    ```bash
        JAVA_HOME=/appcom/java/jdk-11
    ```

3. 添加apco，gds插件，官网下载后，放到 ./plugins 路径下（这个是默认的，具体取决于conf中的路径）
    
    apoc-4.4.0.9-all.jar  
    neo4j-graph-data-science-2.1.6.jar

    添加后，需要在conf中，将两个包中的procedure添加到执行白名单，详见下面的参数配置

4. 启动服务

    ```bash
    # ./bin路径下
    neo4j start
    # 修改配置后，需要重启服务
    neo4j restart
    #停止服务
    neo4j stop
    ```


## 参数配置

    修改conf相关参数，修改 ./conf/neo4j.conf

1. 允许 http/https/bolt 连接和指定端口  

    ```bash
    # 默认只允许本地连接
    dbms.connectors.default_listen_address=0.0.0.0
    # 配置成当前机器IP或hostname
    dbms.connectors.default_advertised_address=localhost（这里改成本机ip）
    # Bolt 连接
    dbms.connector.bolt.enabled=true
    dbms.connector.bolt.tls_level=DISABLED
    dbms.connector.bolt.listen_address=:7687
    # Bolt连接保持时间，一般不设置
    dbms.connector.bolt.thread_pool_keep_alive=5m
    # 处理Bolt连接线程池最大线程数，默认400
    dbms.connector.bolt.thread_pool_max_size
    # 处理Bolt连接线程池最小线程数，默认5
    dbms.connector.bolt.thread_pool_min_size
    # HTTP 连接
    dbms.connector.http.enabled=true
    dbms.connector.http.listen_address=:7474
    # HTTPS 连接
    dbms.connector.https.enabled=false
    dbms.connector.https.listen_address=:7473
    # neo4j工作线程数，只对REST连接生效
    dbms.threads.worker_count=20
    ```

2. 修改目录路径

    ```bash
    # data目录修改到 /data/data01/neo4j
    dbms.directories.data=/data/data01/neo4j
    # log目录修改到 /data/data01/neo4j/logs
    dbms.directories.logs=data/logs     
    dbms.directories.transactions.logs.root=data/transactions
    dbms.directories.dumps.root=data/dumps
    dbms.directories.lib=lib
    dbms.directories.run=run
    dbms.directories.metrics=metrics
    dbms.directories.plugins=plugins
    dbms.directories.certificates=certificates
    ```

3. 修改内存和jvm相关参数

    ```bash
    # 初始Java堆大小
    dbms.memory.heap.initial_size=48g
    # 最大Java堆大小
    dbms.memory.heap.max_size=48g
    # jvm额外启动参数，多个配置多个
    dbms.jvm.additional=-XX:MaxDirectMemorySize=512m
    # 页缓存大小，默认为RAM大小减去最大堆内存后的50%(假如机器上只运行了neo4j)，可以认为这个是缓存，如果机器配置高，这个越大越好
    dbms.memory.pagecache.size=32g  
    # 单个事务超时时间，建议不设置
    dbms.transaction.timeout
    # 单个事务使用内存大小，以及全局大小，建议修改，否则查询会oom
    dbms.memory.transaction.global_max_size=48g
    dbms.memory.transaction.max_size=24g
    # Defines whether memory for transaction state should be allocated on- or offheap.ON_HEAP, OFF_HEAP。默认OFF_HEAP
    dbms.tx_state.memory_allocation=ON_HEAP
    ```

    4. 修改日志相关参数

    ```bash
    # To enable HTTP logging, uncomment this line  要启用HTTP日志记录，请取消注释此行
    # dbms.logs.http.enabled=true
    # Number of HTTP logs to keep. 要保留的HTTP日志数
    # dbms.logs.http.rotation.keep_number=5
    # Size of each HTTP log that is kept. 每个HTTP日志文件的大小
    # dbms.logs.http.rotation.size=20m
    
    #To enable GC Logging, uncomment this line 要启用GC日志记录，请取消注释此行
    dbms.logs.gc.enabled=true
    #GC Logging Options see http://docs.oracle.com/cd/E19957-01/819-0084-10/pt_tuningjava.html#wp57013 for more information.  GC日志记录选项 有关详细信息，请参见http://docs.oracle.com/cd/E19957-01/819-0084-10/pt_tuningjava.html#wp57013
    #dbms.logs.gc.options=-XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCApplicationStoppedTime -XX:+PrintPromotionFailure -XX:+PrintTenuringDistribution

    # Number of GC logs to keep. 要保留的GC日志数
    dbms.logs.gc.rotation.keep_number=20
    # Size of each GC log that is kept. 保留的每个GC日志文件的大小
    dbms.logs.gc.rotation.size=50m
    # Size threshold for rotation of the debug log. If set to zero then no rotation will occur. Accepts a binary suffix "k", "m" or "g".  调试日志旋转的大小阈值。如果设置为零，则不会发生滚动(达到指定大小后切割日志文件)。接受二进制后缀“k”，“m”或“g”。
    dbms.logs.debug.level=DEBUG
    dbms.logs.debug.rotation.size=50m
    # Maximum number of history files for the internal log. 最多保存几个日志文件
    dbms.logs.debug.rotation.keep_number=12

    # query log设置
    # OFF INFO VERBOSE 三选一，INFO只会记录运行时长超过threshold的query，VERBOSE全部记录
    dbms.logs.query.enabled=VERBOSE
    # 只有在dbms.logs.query.enabled=INFO时有用
    dbms.logs.query.threshold=0
    # 单个日志大小 默认20m
    dbms.logs.query.rotation.size=100m
    # 日志保存数 默认7
    dbms.logs.query.rotation.keep_number=100
    # 会保存query中的parameter 默认开启
    dbms.logs.query.parameter_logging_enabled=true
    # 会记录详细的时间信息 默认关闭
    dbms.logs.query.time_logging_enabled=true
    # 会记录详细的bytes allocated 信息 默认关闭
    dbms.logs.query.allocation_logging_enabled=true
    # 会记录详细的page hit，page fault信息 默认关闭
    dbms.logs.query.page_logging_enabled=true
    ```

    5.其他设置

    ```bash
    # 修改233行，允许使用neo4j-shell，类似于mysql 命令行之类的
    dbms.shell.enabled=true
    # 允许使用apoc 和 gds 的 procedures
    dbms.security.procedures.allowlist=apoc.*,gds.*
    dbms.security.procedures.unresreicted=apoc.*,gds.*  
    ```

