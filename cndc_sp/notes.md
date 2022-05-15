HIVE SQL

Hive是基于Hadoop的一个数据仓库工具。Hive 的计算基于 Hadoop 实现的一个特别的计算模型 MapReduce，它可以将计算任务分割成多个处理单元，然后分散到一群家用或服务器级别的硬件机器上，降低成本并提高水平扩展性。

Hive 的数据存储在 Hadoop 一个分布式文件系统上，即 HDFS。

#### Hive ， spark ， presto

Spark，Hive，Impala和Presto是基于SQL的引擎，Impala由Cloudera开发和交付。在选择这些数据库来管理数据库时，许多Hadoop用户会感到困惑。Presto是一个开放源代码的分布式SQL查询引擎，旨在运行甚至PB级的SQL查询，它是由Facebook人设计的。

Spark SQL是一个分布式内存计算引擎，它的内存处理能力很高。Hive也由Apache作为查询引擎引入，这使数据库工程师的工作更加轻松，他们可以轻松地在结构化数据上编写ETL作业。在发布Spark之前，Hive被认为是最快速的数据库之一。

现在，Spark还支持Hive，也可以通过Spike对其进行访问。就Impala而言，它也是一个基于Hadoop设计的SQL查询引擎。Impala查询不会转换为mapreduce作业，而是本地执行。