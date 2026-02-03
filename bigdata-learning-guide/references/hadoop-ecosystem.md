# Hadoop生态圈详解

## 目录
- [HDFS分布式文件系统](#hdfs分布式文件系统)
- [YARN资源调度框架](#yarn资源调度框架)
- [MapReduce计算框架](#mapreduce计算框架)
- [Hadoop环境搭建](#hadoop环境搭建)
- [常见问题与优化](#常见问题与优化)

## HDFS分布式文件系统

### 核心概念
HDFS（Hadoop Distributed File System）是Hadoop的核心组件，用于存储海量数据。

**架构组件**：
- **NameNode**：主节点，管理文件系统的元数据（文件名、目录结构、块位置映射），维护文件系统树和块映射表
- **DataNode**：从节点，存储实际数据块，定期向NameNode发送心跳报告和块列表
- **SecondaryNameNode**：辅助节点，定期合并FsImage和EditLog，减轻NameNode压力

**数据块（Block）**：
- 默认块大小：Hadoop 2.x为128MB，Hadoop 3.x为256MB
- 副本数量：默认为3，可通过`dfs.replication`配置
- 块存储位置：分散在不同DataNode，提高数据可靠性和读取性能

### 读写流程

**读流程**：
1. 客户端调用DistributedFileSystem.open()打开文件
2. NameNode返回文件块及其所在DataNode列表
3. 客户端选择最近的DataNode建立连接并读取数据
4. 读取完成后，客户端继续读取下一个块
5. 全部读取完成后，关闭数据流

**写流程**：
1. 客户端调用create()创建文件，NameNode执行权限检查
2. NameNode创建文件记录并返回，客户端开始写入数据
3. 客户端将数据分块，写入第一个副本DataNode
4. 第一个DataNode将数据传递给第二个DataNode，依次类推（管道写入）
5. 所有副本写入完成后，DataNode向客户端返回确认
6. 客户端通知NameNode文件写入完成

### 关键配置参数
```xml
<!-- dfs-site.xml -->
<property>
  <name>dfs.replication</name>
  <value>3</value>
</property>
<property>
  <name>dfs.blocksize</name>
  <value>134217728</value> <!-- 128MB -->
</property>
<property>
  <name>dfs.namenode.handler.count</name>
  <value>10</value>
</property>
```

## YARN资源调度框架

### 架构组件
YARN（Yet Another Resource Negotiator）是Hadoop的资源管理器，负责集群资源调度和任务管理。

**核心组件**：
- **ResourceManager（RM）**：全局资源管理器，负责资源分配和调度
- **NodeManager（NM）**：每个节点上的资源管理者，负责容器（Container）管理和资源监控
- **ApplicationMaster（AM）**：每个应用的管理者，负责任务调度和状态管理
- **Container**：资源抽象单位，包含内存、CPU等资源

### 资源调度流程
1. 客户端向ResourceManager提交应用
2. ResourceManager启动一个ApplicationMaster
3. ApplicationMaster向ResourceManager申请资源
4. ResourceManager根据调度策略分配资源（Container）
5. ApplicationMaster与NodeManager通信，启动任务
6. 任务执行过程中，ApplicationMaster监控任务状态
7. 任务完成后，ApplicationMaster向ResourceManager注销

### 调度器类型
- **FIFO Scheduler**：先进先出，单队列，适合单用户集群
- **Capacity Scheduler**：多队列，按容量分配资源，支持多租户（默认调度器）
- **Fair Scheduler**：多队列，公平分配资源，动态调整

### 关键配置参数
```xml
<!-- yarn-site.xml -->
<property>
  <name>yarn.nodemanager.resource.memory-mb</name>
  <value>8192</value>
</property>
<property>
  <name>yarn.scheduler.minimum-allocation-mb</name>
  <value>1024</value>
</property>
<property>
  <name>yarn.scheduler.maximum-allocation-mb</name>
  <value>8192</value>
</property>
```

## MapReduce计算框架

### 编程模型
MapReduce是Hadoop的分布式计算框架，采用"分而治之"思想处理大规模数据。

**核心概念**：
- **Map阶段**：读取输入数据，转换为键值对（Key-Value），执行Mapper逻辑，输出中间结果
- **Shuffle阶段**：对Map输出进行排序、分组、分发，传输到Reducer
- **Reduce阶段**：对分组后的数据进行聚合处理，输出最终结果

**执行流程**：
1. InputFormat读取输入数据，分割成InputSplit
2. Map任务处理InputSplit，输出<key, value>对
3. Shuffle阶段：Partitioner决定数据去哪个Reducer，Sort进行排序，Group进行分组
4. Reduce任务处理分组数据，执行聚合逻辑
5. OutputFormat输出最终结果

### 示例：WordCount
```java
public class WordCount {

  public static class TokenizerMapper
       extends Mapper<Object, Text, Text, IntWritable>{

    private final static IntWritable one = new IntWritable(1);
    private Text word = new Text();

    public void map(Object key, Text value, Context context
                    ) throws IOException, InterruptedException {
      StringTokenizer itr = new StringTokenizer(value.toString());
      while (itr.hasMoreTokens()) {
        word.set(itr.nextToken());
        context.write(word, one);
      }
    }
  }

  public static class IntSumReducer
       extends Reducer<Text,IntWritable,Text,IntWritable> {
    private IntWritable result = new IntWritable();

    public void reduce(Text key, Iterable<IntWritable> values,
                       Context context
                       ) throws IOException, InterruptedException {
      int sum = 0;
      for (IntWritable val : values) {
        sum += val.get();
      }
      result.set(sum);
      context.write(key, result);
    }
  }
}
```

### Shuffle优化
- **Combiner**：在Map端进行局部聚合，减少网络传输
- **Partitioner**：自定义分区策略，避免数据倾斜
- **压缩**：对Map输出进行压缩（LZO、Snappy），减少磁盘IO和网络传输
- **调整缓冲区大小**：`mapreduce.task.io.sort.mb`、`mapreduce.reduce.shuffle.input.buffer.percent`

## Hadoop环境搭建

### 单机模式（Local Mode）
适用于开发和测试，所有进程运行在单个JVM上。

```bash
# 下载Hadoop
wget https://downloads.apache.org/hadoop/common/hadoop-3.3.6/hadoop-3.3.6.tar.gz
tar -xzf hadoop-3.3.6.tar.gz

# 配置环境变量
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk
export HADOOP_HOME=/opt/hadoop-3.3.6
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
```

### 伪分布式模式
在一台机器上模拟分布式环境，适合学习和开发。

**配置文件**：
```xml
<!-- core-site.xml -->
<property>
  <name>fs.defaultFS</name>
  <value>hdfs://localhost:9000</value>
</property>

<!-- hdfs-site.xml -->
<property>
  <name>dfs.replication</name>
  <value>1</value>
</property>
```

**启动命令**：
```bash
# 格式化NameNode（首次启动）
hdfs namenode -format

# 启动HDFS
start-dfs.sh

# 启动YARN
start-yarn.sh

# 查看进程
jps
```

### 完全分布式模式
在生产环境中使用，需要多台机器。

**关键步骤**：
1. 配置SSH免密登录
2. 同步配置文件到所有节点
3. 修改core-site.xml、hdfs-site.xml、yarn-site.xml
4. 格式化NameNode并启动集群

## 常见问题与优化

### NameNode内存问题
**问题**：NameNode堆内存不足导致OOM
**原因**：文件数量过多，元数据占用大量内存
**解决方案**：
- 调整NameNode堆内存：`HADOOP_NAMENODE_OPTS="-Xmx8g"`
- 启用NameNode联邦：多个NameNode分担元数据
- 使用HDFS快照减少元数据压力

### 数据倾斜
**问题**：某些Reducer处理时间过长，拖慢整体任务
**原因**：Key分布不均，大量数据集中到少数Reducer
**解决方案**：
- 自定义Partitioner，均匀分布数据
- 预聚合：在Map端使用Combiner
- 调整并行度：增加Reducer数量
- 使用随机前缀打散热点数据

### 小文件问题
**问题**：大量小文件占用NameNode内存，降低性能
**解决方案**：
- 合并小文件：Hadoop Archive（HAR）、CombineFileInputFormat
- 定期清理：删除过期小文件
- 使用Hive的INSERT OVERWRITE合并
- 调整块大小：适当增大`dfs.blocksize`

### 性能优化参数
```xml
<!-- Map端优化 -->
<property>
  <name>mapreduce.task.io.sort.mb</name>
  <value>256</value>
</property>
<property>
  <name>mapreduce.map.sort.spill.percent</name>
  <value>0.8</value>
</property>

<!-- Reduce端优化 -->
<property>
  <name>mapreduce.reduce.shuffle.parallelcopies</name>
  <value>5</value>
</property>
<property>
  <name>mapreduce.reduce.input.buffer.percent</value>
  <value>0.7</value>
</property>

<!-- 压缩配置 -->
<property>
  <name>mapreduce.map.output.compress</name>
  <value>true</value>
</property>
<property>
  <name>mapreduce.map.output.compress.codec</name>
  <value>org.apache.hadoop.io.compress.SnappyCodec</value>
</property>
```

### 监控与调优工具
- **NameNode Web UI**：http://namenode:9870，查看集群状态和文件系统
- **YARN Web UI**：http://resourcemanager:8088，查看应用和资源使用情况
- **HDFS命令**：`hdfs dfsadmin -report`查看集群健康状态
- **日志分析**：查看NameNode、DataNode日志排查问题
