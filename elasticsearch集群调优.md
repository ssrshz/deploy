## 系统调优

- **禁用swap** 使用swapoff命令可以暂时关闭swap。永久关闭需要编辑/etc/fstab，注释掉swap设备的挂载项。
    > swapoff -a

    如果完全关闭swap不可行，可以试着降低swap使用的优先级，执行

    > sysctl vm.swappiness = 1

    并编辑/etc/sysctl.conf，加入swappiness设置。  如果因为一些原因，无法对swap进行操作，可以将ES配置中的memory_lock设置为true，从JVM层面保证内存数据不被交换到swap中。

    > bootstrap.memory_lock： true

    要注意的是，linux系统对lock memory的大小有限制（调整方法见下），可以通过ES接口查看是否设置成功。

    > GET /_nodes?filter_path=**.mlockall&pretty

- **调整io策略** 推荐使用的调度策略如下： HDD： cfq SSD： noop / deadline NVMe: none

    io策略通过block scheduler接口查看和调整。

    > /sys/block/sd*/queue/scheduler

- **关闭Access Time更新** 挂载目录时，指定文件系统选项”noatime”,”nodiratime”,关闭文件access time更新，可以节省大量的io带宽。

- **调整系统限额** 调整打开文件数量限制
    > ulimit -n 65535

    调整lock memory大小限制

    > ulimit -l unlimited

    调整stack大小限制

    > ulimit -s unlimited

    ES使用mmapfs方式，将磁盘文件映射到内存上，这会导致占用大量的虚拟地址空间。系统默认值并不能满足要求，可以通过sysctl调整。

    > sysctl -w vm.max_map_count=262144

    以上命令只是暂时生效，想要持久化配置可以编辑/etc/security/limits.conf，/etc/sysctl.conf。

## ES调优

- **集群规划** 分离master节点，data节点和client节点。data节点只用来存储，索引数据，外部的索引和查询请求全部导向client节点，降低data节点的压力。（data节点故障和恢复时，涉及到分片的恢复和迁移，会造成比较大的io负载，影响集群整体索引和查询性能。）

- **内存！内存！** 对于ES的jvm内存设置，推荐不要超过32G，超过32G一方面会降低内存利用效率，降低GC效率，消耗cpu带宽（参见jvm compressed oops机制），另一方面可能导致不可知的问题。 同时，要为ES所依赖的底层索引库Lucene预留同等大小的内存。例如，一台内存64G（官方推荐配置，ideal sweet spot）大小的机器，可以为ES配置32G内存，另外32G预留给Lucene使用。 对于大配置的机器，也不推荐为ES配置超过32G的内存，仍要坚持50%策略，通过单台多node的方式充分利用配置。例如，内存128G的机器，可以配置两个node节点，每个node节点分配32G内存。 同一台机器配置有多个data节点时，要防止单个分片的主副本都落在同一个机器上，失去副本的意义。设置如下：
    > cluster.routing.allocation.same_shard.host:true

    另外，内存32G大小的限制，目前是为了触发jvm的compressed oops机制，提高内存使用效率，但是由于平台和jvm版本的差异，这个阈值并不是恰好为32G，需要自行进行测试,获取一个最佳值。

    ```
    java -Xmx32600m -XX:+PrintFlagsFinal 2> /dev/null | grep UseCompressedOops
    ```
- **分片建议** 分片的大小和数量之间需要一个平衡：分片过小数量过大会产生更大的集群状态信息，更多的segment文件，占用更多jvm内存，分片过大数量过小则会影响查询性能。

    使用time-base的索引格式，控制索引（分片）大小，便于动态调整。
    单分片平均大小在20G-40G之间，不要超过50G。
    保证每GB内存对应20-25分片
    通过测试获取最佳值benchmark using realistic data and queries
- **集群发现使用单播** 出于可靠和安全考虑，使用单播方式做集群发现。
    > discovery.zen.ping.unicast.hosts: [“host1”, “host2:port”]

- **关闭Data节点http** 对于数据加点，关闭http功能，同时也不要安装head, bigdesk, marvel等监控插件，保证数据节点只处理索引，查询，合并等相关数据操作。
    > http.enabled: false

- **设置最小选举数** 防止脑裂，将最小选举数
    > discovery.zen.minimum_master_nodes:

    设置为

    > (number of master-eligible nodes / 2) + 1

- **设置故障恢复阈值** ES集群在故障恢复的初期，由于部分节点还未恢复服务，ES会检测到大量分片处于非健康状态，而进行分片的重建，迁移，造成不需要的资源开销。通过合理设置恢复参数，可以有效避免上述问题： 最小恢复节点数（存活节点数达到这个阈值，才会进行恢复）

    > gateway.recover_after_nodes: 80

    集群节点数

    > gateway.expected_nodes: 100

    恢复等待时间（达到节点规模阈值后，多久后开始执行恢复）

    > gateway.recover_after_time: 5m

参考文献： Elasticsearch Reference