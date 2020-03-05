# Solr

>  Solr is a standalone enterprise search server with a REST-like API. You put documents in it (called "indexing") via JSON, XML, CSV or binary over HTTP. You query it via HTTP GET and receive JSON, XML, CSV or binary results. 

[solr中文搜索倒排索引和数据存储结构](https://blog.csdn.net/chunlei_zhang/article/details/38520315)

Solr是一个Java开发的基于Lucene的 企业级 开源 全文搜索 平台。 它采用的是反向索引，即从关键字到文档的映射过程。 Solr的资源以Document为对象进行存储，每个文档由一系列的 Field 构成，每个Field 表示资源的一个属性。 文档的Field可以被索引， 以提工高性能的搜索效率。 一般情况下文档都包含一个能唯一表示该文档的id字段。

## 架构

 ![img](http://www.yiibai.com/uploads/images/201702/0302/784150201_26340.jpg) 

* **请求处理程序** - 发送到**Apache Solr**的请求由这些请求处理程序处理。请求可以是查询请求或索引更新请求。根据这些请示的要求来选择请求处理程序。为了将请求传递给Solr，通常将处理器映射到某个URI端点，并且它将为指定的请求提供服务。

* **搜索组件** - 搜索组件是**Apache Solr**中提供的搜索类型(功能)。它可能是拼写检查，查询，构面，命中突出显示等。这些搜索组件被注册为搜索处理程序。多个组件可以注册到搜索处理程序。

* **查询解析器** − **Apache Solr**查询解析器解析传递给**Solr**的查询，并验证查询的语法是否有错误。解析查询后，将它们转换为`Lucene`理解的格式。

* **响应写入器** - **Apache Solr**中的响应写入器是为用户查询生成格式化输出的组件。 Solr支持XML，JSON，CSV等响应格式。对每种类型的响应都有不同的响应写入。

* **分析器/分词器** - Lucene以令牌的形式识别数据。 **Apache Solr**分析内容，将其分成令牌，并将这些令牌传递给Lucene。 **Apache Solr**中的分析器检查字段的文本并生成令牌流。分词器将分析器准备的令牌流分解成令牌。

* **更新请求处理器** - 每当向**Apache Solr**发送更新请求时，请求都通过一组称为**更新请求处理器**的插件(签名，日志记录，索引)运行。这个处理器负责修改，例如删除字段，添加字段等。

## 术语

### 一般术语

以下是在所有类型的`Solr`设置中使用的一般术语的列表 -

**实例** - 就像一个`tomcat`实例或一个`jetty`实例，这个术语指的是在JVM中运行的应用程序服务器。Solr主目录提供对每个这些**Solr**实例的引用，一个或多个核心可以配置在每个实例中运行。

- **核心(core)** - 在应用程序中运行多个索引时，可以在每个实例中拥有多个核心，而不是每个核心的多个实例。
- **主目录(home)** - 术语`$SOLR_HOME`是指主目录，其中包含有关内核及其索引，配置和依赖关系的所有信息。
- **碎片(Shard)** - 在分布式环境中，数据在多个`Solr`实例之间进行分区，其中每个数据块可以称为碎片(`Shard`)。它包含整个索引的子集。

### SolrCloud 术语

在前面的章节中，我们讨论了如何在独立模式下安装`Apache Solr`。请注意，还可以在分布式模式(云环境)中安装**Solr**，**Solr**以主从模式安装。在分布式模式下，索引在主服务器上创建，并且将其复制到一个或多个从服务器。

与**Solr Cloud**相关的主要术语如下 -

- **节点(Node)** - 在Solr云中，Solr的每个单个实例都被视为一个节点。
- **集群** - Solr云环境中的所有节点组合在一起构成集群。
- **集合** - 集群具有称为集合的逻辑索引。
- **碎片** - 碎片是集合的一部分，它具有一个或多个索引副本。
- **副本** - 在**Solr Core**中，在节点中运行的分片副本称为副本。
- **领导者(Leader)** - 它也是碎片的副本，它将**Solr Cloud**的请求分发给剩余的副本。
- **Zookeeper** - 这是一个Apache项目，**Solr Cloud**用于集中配置和协调，管理集群和选择领导者。

solr 同步 mysql [一](https://blog.csdn.net/weixin_44101948/article/details/90515662)  [二](https://blog.csdn.net/weixin_44101948/article/details/90516978)

