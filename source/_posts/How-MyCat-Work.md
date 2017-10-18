---
title: How-MyCat-Work
date: 2017-01-24 10:34:32
tags:
---

---

#### 前言
MyCAT，在cobar的基础上进行开发，支持多种数据库，可以将关系数据库平滑扩展到分布式数据库的开源中间件。

![mycat_archi](http://oa6vwk7v3.bkt.clouddn.com/mycat_architecture.png?imageView/3/w/400/h/400)

<!-- more -->

#### 概念
* 逻辑库(schema)

  数据库中间件，对于实际用户来讲，并不需要知道实际的数据库，业务开发人员仅要知道数据库的概念，所以，中间件就可以看作是一个数据库或者数据库集群的逻辑库。
* 逻辑表(table)

  既然有逻辑库，就有逻辑表，分布式数据库中，读写数据的表就是逻辑表.

* 分片表

  指那些原有的很大数据的表，按照一定规则切分到很多小表，每个小表就是一个分片，即一个分片表，所有相关分片表构成了完整的数据

  类似配置如下：

        ```
        <table name="travelrecord" dataNode="dn1,dn2,dn3" rule="auto-sharding-long" />
        ```
* 非分片表

  一些数据量并不大的表，是没必要分片的。
  类似配置如下：

        ```
        <table name="goods" primaryKey="ID"  dataNode="dn1" />
        ```
* ER 表

  MyCAT的ER表来源于关系型数据库的实体关系模型，根据这一思路，制定了ER关系的分片策略，即子表的数据分布依赖于父表，子表和父表根据某个字段的关联关系的数据一定是分不在一个分片上的。

  如下配置：

        ```
        <table name="customer" primaryKey="ID" dataNode="dn1,dn2"                               rule="sharding-by-intfile">     <childTable name="orders" primaryKey="ID"       joinKey="customer_id"                                parentKey="id">                    <childTable name="order_items" joinKey="order_id"                                               parentKey="id" />               </childTable>     <childTable name="customer_addr"      primaryKey="ID" joinKey="customer_id"                                   parentKey="id" /> </table>

        ```

* 全局表

  实际的业务场景下，存在着一些类似字典表的表，这类表改动不频繁，数据规模小，于其他业务表有关联查询的需求。这类表往往是在每一个分片上都有一个冗余。
        
  类似配置：

        ```
        <table name="goods" primaryKey="ID" type="global" dataNode="dn1,dn2" />
        ```

* 分片节点(dataNode)

    数据切分以后，一个达标被分到不同的分片数据库上面，每个表分片所在的数据库就是分片节点。
        
    类似配置如下：
        
```
<dataNode name="dn1" dataHost="localhost1" database="db1" />
```
* 节点主机(dataHost)

  数据切分后，每个分片节点在前期不一定独占一个主机，这样的一个或者多个分片节点所在的主机就是节点主机。
        类似配置如下：

```
<dataHost name="localhost1" maxCon="1000" minCon="10" balance="0"                writeType="0" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">
```
* 分片规则(rule)

  大表数据被切分到多个小表的时候，需要按照既定好的规则，这个规则就是分片规则。在conf/rule.xml配置。


#### 目录结构及配置部署

* 软件目录结构

```
├── bin  // 程序目录│
├── init_zk_data.sh│
├── mycat // 启动命令<start｜stop｜restart｜status>│
├── rehash.sh│
├── startup_nowrap.sh│
├── wrapper-linux-ppc-64│
├── wrapper-linux-x86-32│
├── wrapper-linux-x86-64│   └── xml_to_yaml.sh
├── catlet //用户自定义代码，调用MyCAT API完成特定需求
├── conf // 配置目录│
├── autopartition-long.txt // 分片规则rang-long的分片id的范围定义│
├── cacheservice.properties // MyCAT支持SQLRouteCache等缓存，这里是缓存size，expires，type等的配置│
├── ehcache.xml // Ehcache缓存框架相关配置│
├── index_to_charset.properties //字符集配置对应的id，用于server.xml:charset的配置│   ├── log4j.xml // 日志配置文件│
├── myid.properties // 使用zk部署MyCAT时的相关配置│
├── partition-hash-int.txt //分片规则hash-int的配置文件│
├── partition-range-mod.txt //分片规则range-mod的配置文件│
├── router.xml  // 历史遗留配置文件，无用│
├── rule.xml // 分片规则配置│
├── schema.xml //逻辑库表，及分片节点，分片主机的定义│
├── sequence_conf.properties // 以下三个，都是MyCAT自带全局id生成服务的相关配置│
├── sequence_db_conf.properties│
├── sequence_time_conf.properties│
├── server.xml // MyCAT服务参数和权限配置│
├── wrapper.conf // MyCAT启动，通过wrapper封装的，wrapper的配置
│   └── zk-create.yaml //MyCAT1.5开始支持zk启动，zk的配置加载方式置为本地xml
├── lib // MyCAT依赖的一些jar包│
├──省略├── logs // 日志目录
└── version.txt
```
* 常用配置介绍

<table>
        <tr>
                <th rowspan="4">conf/server.xml </th>
        <th>配置项</th>
        <th>说明</th>
    </tr>
    <tr>
        <td>defaultSqlParser</td>
        <td>默认的sql解析器是什么，有 druidparser 和fdbparser 两种选择，默认的是druidparser</td>
    </tr>
    <tr>
        <td>processors</td>
        <td>用于指定系统可用的线程数</td>
    </tr>
    <tr>
        <td>processorExecutor</td>
        <td>指定NIOProcessor上共享的businessExecutor固定线程池大小</td>
    </tr>

</table>

<table>
        <tr>
                <th rowspan="6">conf/rule.xml </th>
        <th>配置项</th>
        <th>说明</th>
    </tr>
    <tr>
        <td>tableRule > name</td>
        <td>唯一名字，用于标识不同的表规则</td>
    </tr>
    <tr>
        <td>rule > columns</td>
        <td>内置的rule标签的columns项，指定物理表中的哪一列用于拆分</td>
    </tr>
    <tr>
        <td>rule > algorithm</td>
        <td>内置的rule标签的algorithm项，指定物理表拆分是哪个的路由算法</td>
    </tr>
        <tr>
        <td>function > name</td>
        <td>function标签的name，指定算法的名字</td>
    </tr>
        <tr>
        <td>function > class</td>
        <td>指定路由算法具体的类名字</td>
    </tr>

</table>

<table>
        <tr>
                <th rowspan="8">conf/schema.xml </th>
        <th>配置项</th>
        <th>说明</th>
    </tr>
    <tr>
        <td>schema > name</td>
        <td>定义逻辑库的名字</td>
    </tr>
    <tr>
        <td>schema > dataNode</td>
        <td>schema标签的datanode，表示该schema的所有请求均发往该node，且该schema里面的表，不能再有分片表配置</td>
    </tr>
    <tr>
        <td>dataNode > datahost</td>
        <td>该datahost对应着datahost标签定义的名称</td>
    </tr>
        <tr>
        <td>datahost > balance</td>
        <td>负载均衡属性：“0”,所有读操作都发送到当前可用的writeHost上。“1”, 所有读操作都随机的发送到readHost。“2”, 所有读操作都随机的在writeHost、readhost上分发。</td>
    </tr>
        <tr>
        <td>datahost > writeType</td>
        <td>负载均衡类型：“0”，所有写操作都发送到可用的writeHost上。“1”，所有写操作都随机的发送到readHost。“2”，所有写操作都随机的在writeHost、readhost分上发。</td>
    </tr>
        </tr>
        <tr>
        <td>datahost > switchType</td>
        <td>切换类型：－1表示不自动切换，1 默认值 自动切换，2 基于mysql主从状态切换(show slave status)</td>
    </tr>
        <tr>
        <td>datahost > readHost</td>
        <td>配置datahost中读写分离的备机</td>
    </tr>
</table>


* 下载安装部署

下载地址：

        https://github.com/MyCATApache/Mycat-download

安装启动：

        安装jdk，建议7以上
        $MYCAT_HOME/bin/mycat --help
                Usage: ./bin/mycat { console | start | stop | restart | status | dump }


* 维护窗口

        MyCAT默认启动端口为serverPort 8066 和managePort 9066。可以通过mysql命令行登录管理端口，进行相应的管理操作。

        常用管理命令如下：
        <table>
        <tr>
                <th rowspan="9">管理入口</th>
        <th>操作</th>
        <th>说明</th>
    </tr>
    <tr>
        <td>show @@server;</td>
        <td>监控服务状态，启动时间</td>
    </tr>
    <tr>
        <td>show @@datanode;</td>
        <td>节点连接数情况监控</td>
    </tr>
    <tr>
        <td>show @@cache;</td>
        <td>SQL路由缓存，idToTable缓存及缓存命中情况</td>
    </tr>
        <tr>
        <td>show @@threadpool;</td>
        <td>任务状态的积压情况</td>
    </tr>
        <tr>
        <td>show @@processor;</td>
        <td>buffer pool使用情况</td>
    </tr>
        <tr>
        <td>show  @@sql.sum.table;</td>
        <td>表的读写应用情况</td>
    </tr>
        <tr>
        <td>reload/rollback @@config;</td>
        <td>配置文件的重载和回滚</td>
    </tr>
        <tr>
        <td>show @@sql.slow</td>
        <td>查看最近的慢sql</td>
    </tr>

        </table>

#### 优势特性



* 后端AIO和NIO带来的优越性能

![cobar_vs_mycat](http://oa6vwk7v3.bkt.clouddn.com/cobar_vs_mycat.jpg?imageView/3/w/400/h/400)


cobar后端实际采用的是BIO，每个请求在等待应答时候都会占用一个线程，如果前端并发量大时，就会使性能下降，资源耗费严重，以致于假死。

而mycat在cobar的基础上，后端BIO改为AIO和NIO，以一种非阻塞方式进行交互，马上反馈状态值。

        对于io类型：
        大致分为：
                同步阻塞型(BIO)：适用于连接数比较小的应用，并发有局限的
                同步非阻塞(NIO)：适用于短连接操作，能够快速完成的轻操作，并发较高
                异步非阻塞(AIO)：适用于连接数目较多，且操作较重，一时半会完成不了

        同步：触发io操作后，等待或者轮询去获取io操作状态
        异步：特点在于通知，触发io操作后不需要等待或者轮询，io完成后自会得到通知
        阻塞：对于io操作，如果不可读或者不可写的时候，进入等待状态
        非阻塞：相较于阻塞，当不可读或者不可写的时候，会马上返回，不进入等待


* 丰富分片规则以及支持自定制

        1. 分片枚举

                通过在配置文件枚举配置特定id，自行决定路由方向

                适用场景：

                        如，按照省市区县，状态类型等有限的固定的id特性

                配置示例：

                         conf/rule.xml:
                         <tableRule name="sharding-by-intfile">
                <rule>
                        <columns>sharding_id</columns>
                        <algorithm>hash-int</algorithm>
                </rule>
             </tableRule>

                         <function name="hash-int"
                class="org.opencloudb.route.function.PartitionByFileMap">
                <property name="mapFile">partition-hash-int.txt</property>
                 </function>

                        [dba@localhost conf]$ more partition-hash-int.txt
                        10000=0
                        10010=1


        2. 固定分片hash算法

           通过对key的二进制的低10位与1111111111做与运算，对应的十进制结果落在0～1023范围内，属于指定的分区

           适用场景：

              希望借助hash算法，同时对于连续的key，希望能够落在同一个partition中的业务需求

           配置示例：

                conf/rule.xml:
                <tableRule name="rule1">
                        <rule>
                                <columns>id</columns>
                                <algorithm>func1</algorithm>
                        </rule>
                </tableRule>
                        <function name="func1" class="org.opencloudb.route.function.PartitionByLong">
                                <property name="partitionCount">8</property>
                                <property name="partitionLength">128</property>
                        </function>


        3. 范围约定

                提前规划好分片字段的某个范围属于哪个分片

           适用场景：

                        适用于对于数据的增长情况有准确控制，并且需要自定义数据分布的均匀情况的需求
           配置示例：

                <tableRule name="auto-sharding-long">
                <rule>
                        <columns>id</columns>
                        <algorithm>rang-long</algorithm>
                </rule>
                </tableRule>

                <function name="rang-long"
                class="org.opencloudb.route.function.AutoPartitionByLong">
                <property name="mapFile">autopartition-long.txt</property>
                </function>

                $ more autopartition-long.txt
                # range start-end ,data node index
                # K=1000,M=10000.
                0-500M=0
                500M-1000M=1
                1000M-1500M=2

        4. 求模

           对分片字段进行求模运算
           适用场景：

              对key的分布追求均匀，对事物一致性要求不高，并且数据迁移不多

           配置示例：

                <tableRule name="mod-long">
                        <rule>
                                <columns>id</columns>
                                <algorithm>mod-long</algorithm>
                        </rule>

                        </tableRule
                <function name="mod-long" "class="org.opencloudb.route.function.PartitionByMod">
                        <!-- how many data nodes -->
                        <property name="count">3</property>
                </function>







        5. 一致性hash

                基于hash运算，同时解决分布式数据的扩容问题

                配置示例：

                        <function name="murmur"
                        class="org.opencloudb.route.function.PartitionByMurmurHash">
                        <property name="seed">0</property><!-- 默认是0 -->
                        <property name="count">2</property><!-- 要分片的数据库节点数量，必须指定，否则没法分片 -->
                        <property name="virtualBucketTimes">160</property><!-- 一个实际的数据库节点被映射为这么多虚拟节点，默认是160倍，也就是虚拟节点数是物理节点数的160倍 -->
                        <!-- <property name="weightMapFile">weightMapFile</property> 节点的权重，没有指定权重的节点默认是1。以properties文件的格式填写，以从0开始到count-1的整数值也就是节点索引为key，
以节点权重值为值。所有权重值必须是正整数，否则以1代替 -->
                        <!-- <property name="bucketMapPath">/etc/mycat/bucketMapPath</property>
                        用于测试时观察各物理节点与虚拟节点的分布情况，如果指定了这个属性，会把虚拟节点的murmur hash值与物理节点的映射按行输出到这个文件，没有默认值，如果不指定，就不会输出任何东西 -->
                        </function>



        6. 其他

                取模范围约束，自然月分片，按日期(天)分片，范围求模，日期范围hash等...


* SQL解析

        mycat1.3之前默认是fdbparser，1.4之后去掉了fdb，默认是druid解析，原因是普遍测试结果druid的性能要比fdbparser快10倍以上。

* 缓存机制

        MyCAT缓存的数据分别是：

                ER_SQL2PARENTID ER关系缓存：ER分片的场景下，对子表进行insert插入的时候，需要根据joinkey判断父表所在分片，以获取子表的分片，这个是可以缓存的;
                SQLRouteCache SQL语句路由缓存：通过缓存SQL语句的路由信息，下次同样的sql再次执行，则会命中缓存;
                TableID2DataNodeCache.dbname_tablename 表主键对应分片键的缓存：为每一个表建一个缓存池，在主键id并不是分片键的时候，通过主键查询的场景下，这个缓存很有用;


#### RoadMap

* 完全实现分布式事务，完全的支持分布式.
* 通过Mycat web（eye）完成可视化配置，及智能监控，自动运维。
* 通过mysql 本地节点，完整的解决数据扩容难度，实现自动扩容机制，解决扩容难点。
* 支持基于zookeeper的主从切换及Mycat集群化管理。
* 通过Mycat Balance 替代第三方的Haproxy，LVS等第三方高可用，完整的兼容Mycat集群节点的动态上下线。
* 接入Spark等第三方工具，解决数据分析及大数据聚合的业务场景。
* 通过Mycat智能优化，分析分片热点，提供合理的分片建议，索引建议，及数据切分实时业务建议.

