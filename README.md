CDH5启动服务的顺序如下

Cloudera Management Service
ZooKeeper
HDFS
Solr
Flume
HBase
KS_Indexer
YARN
Hive
Impala
Oozie
Sqoop
Hue
按正确的顺序启动和停止具有依赖关系的服务尤为重要。

<1>. 添加新主机到集群
  可以用同一机制安装 Oracle JDK、CDH、Cloudera Manager Agent 软件包.在每个 Cloudera Manager Agent 主机上，通过在cm安装目录下etc/cloudera-scm-agent/config.ini 配置文件中设置以下属性，配置 Cloudera Manager Agent 以指向 Cloudera Manager Server：
属性	    说明
server_host	运行 Cloudera Manager Server 的主机的名称。
server_port	运行 Cloudera Manager Server 的主机中的端口.

通过/opt/cm-5.X.X/etc/init.d/cloudera-scm-server start启动服务端。
通过/opt/cm-5.X.X/etc/init.d/cloudera-scm-agent start启动Agent服务。
我们启动的其实是个service脚本，需要停止服务将以上的start参数改为stop就可以了，重启是restart。

   在 Agent 启动后，单击主机选项卡并检查新主机的运行状况状态可以验证 Agent 是否与 Cloudera Manager Server 连接正常。如果运行状况状态良好且最后一个检测信号的值是最新的，则 Agent 与 Cloudera Manager Server 连接正常。
    连接正常后，通过选定仓库，安装与集群中运行版本相同的cdh5版本：　　http://archive.cloudera.com/cm5/redhat/6/x86_64/cm/5.X.X/
安装完成后，添加角色实例，并重启集群服务.

示例：
<1>检查主机名
１.vim /etc/sysconfig/network，修改HOSTNAME选项
２.reboot  系统重启
 3.hostname 待系统重启后，验证主机名是否更改成功 

<2>安装Oracle JDK
1.rpm -qa |grep java　查询java相关的包，使用rpm -e --nodeps 包名卸载

<3>查看并修改最大打开文件数
su root
echo ulimit -n 65536 >> /etc/profile 
source /etc/profile    # 加载修改后的profile 
ulimit -n     # 显示65536，修改完毕！
vim /etc/security/limits.conf
在文件尾部添加如下代码： 
* soft nofile 65536
* hard nofile 65536
重启系统，在任何用户下查看最大打开文件数：ulimit -n 结果都是65536

<4>ssh无密码登录配置
1.vim /etc/hosts　添加所有机器名称和ip对应关系，包括本机
2.ssh无密码登录配置

<5>关闭防火墙

<6>解压Cloudera Manager Agent 软件包
在/opt/cm-5.3.3/etc/cloudera-scm-agent/config.ini中配置cloudera manager server为集群中cloudera manager server ip,以连接到cloudera manager server实现安装。
注意解压文件可能会修改opt目录的所有者，chown -R root:root opt修改回来

<7>在cloudera 管理控制台，添加主机到集群，注意启动cloudera agent 然后在管理控制台就可以自动发现到可连接的主机，然后根据向导操作即可。

<8>安装完成后可能会有警告信息
1.时间与集群时间差距较大，通过ntp同步时间
2.设置 vm.swappiness Linux 内核参数
    vm.swappiness 是一个 Linux 内核参数，用于控制将内存页交换到磁盘的幅度。它可以设为介于 0-100 之间的值；值越高，内核寻找不活动的内存页并将其交换到磁盘的幅度就越大。
   通过cat /proc/sys/vm/swappiness可以了解 vm.swappiness 当前设置的值；在大多数系统中，它默认设为 60。这不适用于 Hadoop 群集节点，因为这可能导致进程被换出，即使存在可用内存。这可能会影响稳定性和性能，并可能导致出现问题，比如重要系统后台程序出现冗长的垃圾数据收集暂停。Cloudera 建议将此参数设置为 0；
# sysctl -w vm.swappiness=0 暂时有效
#echo 0 > /proc/sys/vm/swappiness 永久有效

<2>. 使用 Key-Value Store Indexer服务
对 HBase 群集中的表的列系列进行 索引：
１．在 HBase 列系列上启用复制
２．创建集合和配置
３．注册 Lily HBase Indexer 配置和 Lily HBase Indexer Service
４．验证索引是否正常工作

１．在 HBase 列系列上启用复制
确保已启用群集范围内的 HBase 复制。使用 HBase shell 定义列系列复制设置。

对于每个现有表，在需要通过发出格式命令进行索引的每个列系列上设置 REPLICATION_SCOPE：

$ hbase shell
hbase shell> disable 'record'
hbase shell> alter 'record', {NAME => 'data', REPLICATION_SCOPE => 1}
hbase shell> enable 'record'
对于每个新表，在需要通过发出格式命令进行索引的每个列系列上设置 REPLICATION_SCOPE：

$ hbase shell
hbase shell> create 'record', {NAME => 'data', REPLICATION_SCOPE => 1}

２．创建集合和配置
Lily HBase NRT Indexer Services 要求的任务是与 Lily HBase Batch Indexer 描述的任务相同的任务。遵守以下部分中描述的步骤：

2.1创建相应的 SolrCloud 集合
用于 HBase 索引的 SolrCloud 集合必须具有可容纳 HBase 列系列的类型和要进行索引处理的限定符的 Solr 架构。若要开始，请考虑将包括一切 data 的字段添加到默认架构。一旦您决定采用何种架构，使用以下表单命令创建 SolrCloud 集合：

$ solrctl instancedir --generate $HOME/hbase-collection1
$ edit $HOME/hbase-collection1/conf/schema.xml
$ solrctl instancedir --create hbase-collection1 $HOME/hbase-collection1
$ solrctl collection --create hbase-collection1

2.2创建 Lily HBase Indexer 配置
使用 hbase-indexer 命令行实用程序配置单个 Lily HBase Indexer。一般情况下，每个 HBase 表都有一个 Lily HBase Indexer 配置，但是 Lily HBase Indexer 配置有可能与 SolrCloud 中的表和列系列以及相应的集合一样多。每个 Lily HBase Indexer 配置均在 XML 文件中定义，例如 morphline-hbase-mapper.xml。

索引器配置 XML 文件必须引用 MorphlineResultToSolrMapper 实施并指向 Morphline 配置文件的位置，如在以下 morphline-hbase-mapper.xml 索引器配置文件中所示：

$ cat $HOME/morphline-hbase-mapper.xml

<?xml version="1.0"?>
<indexer table="record" mapper="com.ngdata.hbaseindexer.morphline.MorphlineResultToSolrMapper">

   <!-- The relative or absolute path on the local file system to the morphline configuration file. -->
   <!-- Use relative path "morphlines.conf" for morphlines managed by Cloudera Manager -->
   <param name="morphlineFile" value="/etc/hbase-solr/conf/morphlines.conf"/>

   <!-- The optional morphlineId identifies a morphline if there are multiple morphlines in morphlines.conf -->
   <!-- <param name="morphlineId" value="morphline1"/> -->

</indexer>


2.3创建 Morphline 配置文件
在创建索引器配置 XML 文件后，通过配置 morphlines.conf 配置文件中的 morphline ETL 转换命令来控制其行为。morphlines.conf 配置文件可包含任意数量的 morphline 命令。通常，extractHBaseCells 命令是第一命令。readAvroContainer 或 readAvro morphline 命令通常用于从 HBase 字节数组中提取 Avro 数据。此配置文件可在使用 morphline 的不同应用程序之间共享。

$ cat /etc/hbase-solr/conf/morphlines.conf

morphlines : [
  {
    id : morphline1
    importCommands : ["org.kitesdk.morphline.**", "com.ngdata.**"]

    commands : [                    
      {
        extractHBaseCells {
          mappings : [
            {
              inputColumn : "data:*"
              outputField : "data"
              type : string 
              source : value
            }

            #{
            #  inputColumn : "data:item"
            #  outputField : "_attachment_body"
            #  type : "byte[]"
            #  source : value
            #}
          ]
        }
      }

      #for avro use with type : "byte[]" in extractHBaseCells mapping above
      #{ readAvroContainer {} } 
      #{ 
      #  extractAvroPaths {
      #    paths : { 
      #      data : /user_name      
      #    }
      #  }
      #}

      { logTrace { format : "output record: {}", args : ["@{}"] } }
    ]
  }
]
  Note: 为了正常工作，morphline 不得包含 loadSolr 命令。封闭的 Lily HBase Indexer 必须将文档加载到 Solr 而不是 morphline 本身。


３．注册 Lily HBase Indexer Configuration 和 Lily HBase Indexer Service
当 Lily HBase Indexer 配置 XML文件的内容令人满意，将它注册到 Lily HBase Indexer Service。上传 Lily HBase Indexer 配置 XML文件至 ZooKeeper，由给定的 SolrCloud 集合完成此操作。例如：

$ hbase-indexer add-indexer \
--name myIndexer \
--indexer-conf $HOME/morphline-hbase-mapper.xml \
--connection-param solr.zk=solr-cloude-zk1,solr-cloude-zk2/solr \
--connection-param solr.collection=hbase-collection1 \
--zookeeper hbase-cluster-zookeeper:2181
如下所示，验证索引器是否已成功创建：

$ hbase-indexer list-indexers
Number of indexes: 1

myIndexer
  + Lifecycle state: ACTIVE
  + Incremental indexing state: SUBSCRIBE_AND_CONSUME
  + Batch indexing state: INACTIVE
  + SEP subscription ID: Indexer_myIndexer
  + SEP subscription timestamp: 2013-06-12T11:23:35.635-07:00
  + Connection type: solr
  + Connection params:
    + solr.collection = hbase-collection1
    + solr.zk = localhost/solr
  + Indexer config:
      110 bytes, use -dump to see content
  + Batch index config:
      (none)
  + Default batch index config:
      (none)
  + Processes
    + 1 running processes
    + 0 failed processes
使用 hbase-indexer 实用程序的 update-indexer 和 delete-indexer 命令行选项来操作现有的 Lily HBase Indexers。

更多帮助，请使用以下命令：

$ hbase-indexer add-indexer --help
$ hbase-indexer list-indexers --help
$ hbase-indexer update-indexer --help
$ hbase-indexer delete-indexer --help

  Note: morphlines.conf 配置文件必须显示在运行索引器的每个主机上。
  Note: 可以使用 Cloudera Manager Admin Console 更新 morphlines.conf：
在 Cloudera Manager Home 页面上，单击 Key-Value Indexer Store，通常为 KS_INDEXER-1。
单击 Configuration > View and Edit。
展开 Service-Wide，并单击 Morphlines。
对于 Morphlines File 属性，粘贴新的morphlines.conf 内容至 Value 字段。
在启动和重新启动 Lily HBase Indexer Service 时，Cloudera Manager 自动复制粘贴的配置文件至当前所有 Lily HBase Indexer 群集进程的工作目录中。在这种情况下，文件位置 /etc/hbase-solr/conf/morphlines.conf 不适用。
  Note: 无需重新创建索引器，即可更改 Morphline 配置文件。在这种情况下，必须重启 Lily HBase Indexer 服务。

４．验证索引是否正常工作
添加行至索引的 HBase 表。例如：

$ hbase shell
hbase(main):001:0> put 'record', 'row1', 'data', 'value'
hbase(main):002:0> put 'record', 'row2', 'data', 'value2'
如果 put 操作成功，等待几秒钟，导航至 SolrCloud UI 查询页面并查询数据。在 Solr 中备注更新的行。

<3>. Impala与parquet
1. 测试parquet格式的impala表的查询速度

克隆一个现有表的列名和数据类型　create table parquet_table_name LIKE other_table_name STORED AS PARQUET;
　
通过insert into(overwrite) parquet_table_name select * from other_table_name语句将impala中创建的外部表（实际数据存在于hbase中）中的textfile格式的数据重新转换成parquet格式存储在impala中.

注意：
1. 避免一次插入过多数据，通过对select语句添加限制分批插入数据;
2. 插入数据时可能由于列的不对应出现导入失败问题，一般都需要在select语句中指定具体的列名;
3. 插入成功后，一查看hue界面中hive的metadata，选择其中的parquet格式的表，查看sample选项就会导致该hue主机上的hiveServer挂掉，通过分析，发现是由于点击sample查看具体的数据时，后台默认执行select * from parquet_table_name limit 100,这样在数据量很大的时候就会导致内存泄漏，通过hive shell运行同样语句，就会直接报内存不够，退出shell，不会使hiveserver挂掉;可以通过替换*为具体的部分列名可以减少所需内存;或者通过impala查询,impala查询数据量过大的时候也会由于内存不够，导致impala deamon进程退出。
 
2. impala和hbase查询存储的优势比较

 1. hbase适合大量键值对的存储，查询时，由于对行键进行了索引，所以能够很快的返回指定行或者一段区间内的行。同时注意避免对大表的全盘扫描，及其低效。但是无法将所有需要查询的列随意设计进行键里面。
　　拿微博数据举例，可以将微博用户的id＋微博创建时间设计成行键，这样可以快速查询给定用户id的所有微博，继续细化行键可以查询某个用户某段时间内的微博，但是无法查询某条微博的所有转发微博，或者转发数超过一定值的微博。但是如果把转发数添加到行键的头部，则可以查询转发书超过一定量的微博，但是就无法查询针对具体用户的查询了。

2. impala由于是基于内存的，故而查询速度比较快，但是很容易出现内存瓶颈问题。对于一张表，通过存储数据容量以及所用节点数量，计算机器所需的内存大小。

3. 由于parquet是面向列存储的格式，可以通过设置impala表存储为parquet格式。该格式的表特别适合于针对部分列进行的查询即包含where语句限定列条件，在数据文件里面，数据被重新组织同一列数据被存储在一起，可以进行很好的压缩，这样针对 Parquet 表的查询可以快速并以最小的 I/O 从任意列快速获取.前提也是足够内存，但是相对于非parquet表对内存的要求较低一点。

注：
　　如果通过hbase存储，impala查询hbase表，则impala查询中必须要有针对hbase行键的过滤语句即where keyrow <> between等，在hbase边过滤出部分数据传到impala，再在impala端进行部分列的统计和查询，这样所需内存就会相应减少。
或者完全使用impala存储数据表，采用parquet格式存储，可以大大优化类似select col1,col2 from parquet_table where keyrow between startRow and stopRow and col3=123的查询。select中选取的列和where中限制的列越少，查询越快速。
　　如果数据存在于 Impala 之外，并且是其它格式。首先，使用 LOAD DATA 或 CREATE EXTERNAL TABLE ... LOCATION 语句把数据导入使用对应文件格式的 Impala 表中。然后，使用 INSERT...SELECT 语句将数据复制到 Parquet 表中，并转换为 Parquet 格式。
<4>. 通过impala/hive查询hbase表
Impala CREATE TABLE 语句当前不支持所需的全部子句，因此需要切换到 Hive 来创建表，然后再返回 Impala 和 impala-shell 解释程序发出查询

一：用hive访问一个已经在hbase中存在的表(hive)

１．在hbase中创建一个表create 'member','member_id','info' 
  
２．向member中插入几条记录
　　put'member','scutshuxue','info:company','alibaba'
　　put'member','anglea','info:company','clustertech'

３．在hive中建external non-native表（登录hive：hive）
　　CREATE EXTERNAL TABLE hbase_table_2(key string, value string) 
　　STORED BY 'org.apache.hadoop.hive.hbase.HBaseStorageHandler'
　　WITH SERDEPROPERTIES ("hbase.columns.mapping" = "info:company")
　　TBLPROPERTIES("hbase.table.name" = "member");

４．在hive中查询表hbase_table_2,结果如下
　　hive> select * from hbase_table_2;
　　OK
　　anglea  clustertech
　　scutshuxue      alibaba

注意：
hive中日期对应string类型
java long 对应hive　bigint类型
Impala 不应以 root 用户身份运行。使用直接读取实现最佳 Impala 性能，但不允许 root 用户身份使用直接读取。因此，将 Impala 以 root 用户身份运行会对性能产生负面影响。

二：使用impala查询hbase表（impala-shell）
http://www.cloudera.com/content/cloudera/zh-CN/documentation/core/v5-3-x/topics/impala_hbase.html
在 Hive 中创建表后，例如在本示例中创建的是 HBase 映射表，下次连接至 Impala 时发布 INVALIDATE METADATA table_name 语句，让 Impala 能够识别新表(登录hbase:hbase shell)
<5> HBase修改Table压缩格式步骤
修改HBase压缩算法很简单，只需要如下几步：
1. hbase shell命令下，disable相关表：
disable 'test'
实际产品环境中，’test’表可能很大，例如上几十T的数据，disable过程会比较缓慢，需要等待较长时间。disable过程可以通过查看hbase master log日志监控。

2. 修改表的压缩格式
alter 'test', NAME => 'f', COMPRESSION => 'snappy'
NAME即column family，列族。HBase修改压缩格式，需要一个列族一个列族的修改。而且这个地方要小心，别将列族名字写错，或者大小写错误。因为这个地方任何错误，都会创建一个新的列族，且压缩格式为snappy。当然，假如你还是不小心创建了一个新列族的话，可以通过以下方式删除：
alter 'test', {NAME=>'f', METHOD=>'delete'}
同样提醒，别删错列族.

3. 重新enable表
enable 'test’

4. enable表后，HBase表的压缩格式并没有生效，还需要一个动作，即HBase major_compact
major_compact 'test'
该动作耗时较长，会对服务有很大影响，可以选择在一个服务不忙的时间来做。

describe一下该表，可以看到HBase 表压缩格式修改完毕。

错误解决案例：
<1>. 一台机器的datanode连接不上namenode
　　在编辑/etc/hosts后，cloudera manager可能由于无法解析内网ip，而映射到公网地址，而namenode的dfs_hosts_allow.txt文件中已经不包括该内网ip,自动修改为公网ip,而后来的datanode节点尝试连接namenode节点就会导致拒绝连接问题。报如下错误。
“sudo -u hdfs hdfs dfsadmin -refreshNodes”运行这个命令cm就会根据主机刷新允许连接的机器。Running this will refresh the allow list based on the hosts known to CM
该文件dfs_hosts_allow.txt 位置：
hdfs> Instances > namenode> Processes > Configuration Files>  dfs_hosts_allow.txt 

错误日志：
9:01:26.509 AM
INFO	org.apache.hadoop.hdfs.server.datanode.DataNode	
Block pool BP-1466172755-192.168.31.160-1441511616936 (Datanode Uuid null) service to bigdata4/192.168.31.160:8022 beginning handshake with NN
9:01:26.510 	ERROR	org.apache.hadoop.hdfs.server.datanode.DataNode	
Initialization failed for Block pool BP-1466172755-192.168.31.160-1441511616936 (Datanode Uuid null) service to bigdata4/192.168.31.160:8022 Datanode denied communication with namenode because the host is not in the include-list: DatanodeRegistration(192.168.32.199, datanodeUuid=519afc9d-20b9-4abb-9713-02fb4e9141b9, infoPort=50075, ipcPort=50020, storageInfo=lv=-56;cid=cluster22;nsid=862467695;c=0)


<2>. impala daemon在查询时总会莫名其妙挂掉
由于利用hue　web UI删除metestore中数据并没有真正删除数据表，导致多次导入数据到同一张表中，致使表数据越来越大，最后，每次运行impala查询都会导致impala daemon进程挂掉。
解决办法：利用hive shell 指令删除表drop table tableName;然后重新导入数据到metastore，impala-shell :refresh tableName即可。

<3>. sqoop2的map程序总是停留在map->map阶段
由于网络原因，在一定时间内连接数据库超时，就会出现这个问题，解决办法就是设置连接超时时间／在临近的网络环境中安装mysql由于在同一个网络环境中，就不会出现这个问题。

<4>. hive创建hbase映射表出错
错误类型：
两部分字段数目不匹配； columns has 54 elements while hbase.columns.mapping has 55 elements (counting the key if implicit))
不支持的数据类型;hive里面不支持date时间格式数据，将其声明为string即可．

<5>. 连接CDH5　Solr集群出错
 KeeperErrorCode = NoNode for /live_nodes

配置：
cloudSolrServer = new CloudSolrServer("192.168.31.180:2181");
cloudSolrServer.setDefaultCollection("sina");
cloudSolrServer.setZkConnectTimeout(50000);
cloudSolrServer.setZkClientTimeout(50000);
cloudSolrServer.connect();

解决
cloudSolrServer = new CloudSolrServer("192.168.31.180:2181/solr");

CDH5和一般Hadoop的CloudSolrServer创建的不同之处．

<6>.新搭建的集群hue无法查看hbase相关表
 Enabling the HBase Browser application

The HBase Browser application, new as of CDH4.4, depends on the HBase Thrift server for its functionality. The Thrift server role is not added by default when you install HBase, so in order to use the HBase Browser you must first add a Thrift Server role.

To add a Thrift Server role:

Select the HBase service, then select the Instances tab.
Click the Add button to go to the Add Role page.
Select the host(s) where you want to add the Thrift Server role (you only need one for Hue) and click Continue. The Thrift server role should appear in the instances list for the HBase server.
Select the Thrift Server role instance, and from the Actions for Selected menu, Start the role.
To configure Hue for the HBase Browser:
Select the Hue service, then under the Configuration tab select View and Edit.
Go to the Service-Wide category.
For the HBase Service property, make sure it is set to the HBase service for which you enabled the Thrift Server role(if you have more than one HBase service instance).
In the HBase Thrift Server property, click in the edit field and select the Thrift Server role that Hue should use.
Save Changes to have these configurations take effect.
<7>. Spark配置

1. 以yarn模式运行word count程序，报create log directory错误
Error in creating log directory: file:/user/spark/applicationHistory//application_

解决办法：
修改配置文件中默认的/user/spark/applicationHistory
为hdfs:bigdata4:8020/user/spark/applicationHistory即前面加上hdfs文件系统前缀即可

CDH5上面运行spark 程序指令
sudo -u hdfs spark-submit --class zx.soft.spark.basic.sql.JavaSparkSQL --master yarn  spark-demo-0.0.1-SNAPSHOT.jar

<8>. 部分操作
hadoop操作：
杀死job,停止运行mapreduce任务
sudo -u hdfs hadoop job -kill job_1429523909440_0013


在cdh5集群上运行WordCount指令：
sudo -u hdfs hadoop jar *****.jar  *******.WordCount /user/hdfs/input /user/hdfs/output
sudo -u  hdfs mapred job -list
sudo -u hdfs hadoop fs -du -h /hbase/data/default

hive操作：
1.sudo -u hdfs hive -e "select * from test_hive where comments_count > 1000 " >> local.csv
2.sudo -u hdfs hive -f hive.sql >> local.csv
其中hive.sql文件内容如下：
select * from test_hive where comments_count > 1000 ;


 CREATE EXTERNAL TABLE user_followers(key string,screen_name string,name string,followers_count string,location string,gender string,created_at string) STORED BY 'org.apache.hadoop.hive.hbase.HBaseStorageHandler' WITH SERDEPROPERTIES ("hbase.columns.mapping"="user:screen_name,user:name,user:followers_count,user:location,user:gender,user:created_at") TBLPROPERTIES ("hbase.table.name"="user") ;

sqoop2操作：
利用hue创建的link暂时没有在界面找到删除的方法，需要通过sqoop2命令行删除
登录机器执行“sqoop2”，\h参看使用指导。
delete job -jid id号;
delete link -lid id号;
job和link的id号通过show link/job查看。

对hbase里面的数据批量索引到solr（分词插件配置存在问题，目前reduce阶段存在问题）

sudo -u hdfs hadoop jar 
/opt/cloudera/parcels/CDH/lib/hbase-solr/tools/hbase-indexer-mr-1.5-cdh5.3.3-job.jar --hbase-indexer-file morphline-hbase-mapper.xml --collection weibo --zk-host bigdata4/solr --dry-run

Impala创建外部表：
create EXTERNAL TABLE weibo_impala
 (id BIGINT,Service_code BIGINT,Net_ending_ip INTEGER,Net_ending_name STRING,Time BIGINT, Net_ending_mac BIGINT,Destination_ip INTEGER,Port INTEGER,Service_type INTEGER,Keyword1 STRING,Keyword2 STRING,Keyword3 STRING,Mac BIGINT,Source_port INTEGER,Net_ending_ipv6 STRING,Destination_ipv6 STRING,Keyword11 INTEGER,Keyword12 INTEGER,Keyword13 INTEGER,Keyword14 INTEGER,Keyword15 BIGINT,Keyword21 STRING,Keyword22 STRING,Keyword23 STRING,Keyword24 STRING,Keyword25 STRING)  ROW FORMAT DELIMITED FIELDS TERMINATED BY ','LOCATION '/user/impala/weibo';
