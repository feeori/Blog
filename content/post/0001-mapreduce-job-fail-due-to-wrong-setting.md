---
title: "Mapreduce: 节点失效时，错误的配置导致Hive任务失败"
date: 2021-02-27T10:51:16+08:00
categories: ["线上故障"]
tags: ["Hive", "CDH", "Mapreduce", "大数据"]
draft: false
---
## 前言
副本（冗余）是分布式系统用来保障可靠性、可用性的基石特性。一切数据皆有副本，即使出现了单个节点失效的情况，由于有副本的存在，数据/任务的理应继续保持可用。

## 故障描述
某日，一个datanode挂机（下文datanode14），导致凌晨出数的部分Hive任务失败，注意是`部分失败`，大部分Hive任务还是成功的。但依然不合理，一个datanode失效，会导致Hive任务失败？重试机制呢？副本呢？这与认知不符啊？事出反常，魔鬼藏在细节里，只好深入研究研究了。

## 故障分析
在Hive执行过程中，一个datanode挂机，会影响Hive任务，这个可以理解。但Hive不会`重试`？怎么不把任务调度到其他有副本的节点上执行？

先去job history，看看发生甚么事了。

### job history显示该job有两次重试，但重试也失败了
从下图可以看出，Map有33个任务，但Attempt了99次，均失败（1次正常执行+2次重试。重试的次数是可配的，见CDH YARN里的mapreduce.am.max-attempts，默认为2）。
![job history](/img/0001-mapreduce-job-fail-due-to-wrong-setting/01-attempt-fail-in-job-history.png)

看来Hive已经尽力重试过了，只是重试也失败了，重试为啥会失败呢？接着看看Hive任务具体的错误日志。

### Hive job日志显示找不到map.xml的block
日志显示任务在执行过程中，由于找不到文件map.xml（文件对应的HDFS块为blk_1233120961_159438211），导致任务失败。
```
2019-03-13 02:12:23,935 INFO [main] org.apache.hadoop.hdfs.DFSClient: No node available for BP-1495231908-10.10.13.60-1497237252772:blk_1233120961_159438211 file=/tmp/hive/hdfs/4c366474-4d80-4646-abd8-e190ee10fce2/hive_2019-03-13_01-25-29_642_5360272907506781180-1/-mr-10082/3e2b0e34-257d-4249-ac52-97786b1a39e0/map.xml
2019-03-13 02:12:23,935 INFO [main] org.apache.hadoop.hdfs.DFSClient: Could not obtain BP-1495231908-10.10.13.60-1497237252772:blk_1233120961_159438211 from any node: java.io.IOException: No live nodes contain block BP-1495231908-10.10.13.60-1497237252772:blk_1233120961_159438211 after checking nodes = [], ignoredNodes = null No live nodes contain current block Block locations: Dead nodes: . Will get new block locations from namenode and retry...
2019-03-13 02:12:23,935 WARN [main] org.apache.hadoop.hdfs.DFSClient: DFS chooseDataNode: got # 1 IOException, will wait for 688.0608775666193 msec.
2019-03-13 02:12:24,625 INFO [main] org.apache.hadoop.hdfs.DFSClient: No node available for BP-1495231908-10.10.13.60-1497237252772:blk_1233120961_159438211 file=/tmp/hive/hdfs/4c366474-4d80-4646-abd8-e190ee10fce2/hive_2019-03-13_01-25-29_642_5360272907506781180-1/-mr-10082/3e2b0e34-257d-4249-ac52-97786b1a39e0/map.xml
2019-03-13 02:12:24,625 INFO [main] org.apache.hadoop.hdfs.DFSClient: Could not obtain BP-1495231908-10.10.13.60-1497237252772:blk_1233120961_159438211 from any node: java.io.IOException: No live nodes contain block BP-1495231908-10.10.13.60-1497237252772:blk_1233120961_159438211 after checking nodes = [], ignoredNodes = null No live nodes contain current block Block locations: Dead nodes: . Will get new block locations from namenode and retry...
```

居然出现了块丢失，不是有副本么？怎么会出现找不到的数据块？去HDFS日志里翻翻，希望能找到关于这个丢失块的日志。

## HDFS日志中，发现map.xml对应的block副本数被设置为1
文件创建的时候，3e2b0e34-257d-4249-ac52-97786b1a39e0/map.xml在以下三个节点上，此时确实是3副本：10.10.15.75（挂掉的datanode14）, 10.10.15.63(datanode2)，10.10.15.66(datanode5)：
![map xml location](/img/0001-mapreduce-job-fail-due-to-wrong-setting/02-map-xml-location.png)

接着往下看，搜寻map.xml相关的日志，发现map.xml副本数被设置成了1
![map xml replication decrease to one](/img/0001-mapreduce-job-fail-due-to-wrong-setting/03-xml-decrease-to-one-of-failure-job.png)

紧接着其余两个datanode（10.10.15.63, 10.10.15.66）上的map.xml副本的block先是被namenode2设置为失效，然后被物理删除，导致仅剩10.10.15.75（挂掉的节点）上还有map.xml的数据：
![map xml deleted 1](/img/0001-mapreduce-job-fail-due-to-wrong-setting/04-block-deleted-in-hdfs.png)
![map xml deleted 2](/img/0001-mapreduce-job-fail-due-to-wrong-setting/05-block-deleted-in-hdfs-2.png)

到目前为止，大概了解此次故障的过程了：Hive创建任务时，任务文件map.xml有3个副本，但后面副本数被降低到1。而唯一的副本，刚好在挂掉的datanode14上。。datanode14挂机后，唯一的副本丢失，导致这部分的Hive任务即使重试，最终也会失败。

## 非故障时间，map.xml的副本数都会从3变为1
日志里继续搜寻，发现非故障时间段，MR任务的jar包，map.xml，reduce.xml的副本数，也会从3变为1。
![map xml replication decrease to one](/img/0001-mapreduce-job-fail-due-to-wrong-setting/06-other-xml-decrease-to-one.png)

发现大规模的、有规律的日志，因此开始认定，不是偶然或人为操作导致副本降为1，应该是哪里的配置出了问题。

## 撸代码
查看代码，发现往远程拷贝MR任务相关的文件时，会执行setReplication，把副本数设置为mapreduce.client.submit.file.replication。
```Java
public static final String SUBMIT_REPLICATION =
  "mapreduce.client.submit.file.replication";                            ======> mapreduce.client.submit.file.replication线上的值为1
 
 
public void uploadFiles(Job job, Path submitJobDir) throws IOException {
  Configuration conf = job.getConfiguration();
  short replication =
      (short) conf.getInt(Job.SUBMIT_REPLICATION,                        ======> 设置replication的值为SUBMIT_REPLICATION
          Job.DEFAULT_SUBMIT_REPLICATION);
  ...
  copyJar(jobJarPath, JobSubmissionFiles.getJobJar(submitJobDir),        ======> 往远程拷贝jar包，并设置副本数
    replication);
 
 
private void copyJar(Path originalJarPath, Path submitJarFile,
    short replication) throws IOException {
  jtFs.copyFromLocalFile(originalJarPath, submitJarFile);                ======> 往远程拷贝jar包/map.xml/reduce.xml等MR配置文件，HDFS的replication为3，因此一开始副本数为3
  jtFs.setReplication(submitJarFile, replication);                       ======> 此刻将副本数从3改到1
  jtFs.setPermission(submitJarFile, new FsPermission(
      JobSubmissionFiles.JOB_FILE_PERMISSION));
}
```

## 错误配置，罪魁祸首
查看线上YARN配置，发现`mapreduce.client.submit.file.replication`的值改为了1（默认是3）。。
![map xml replication decrease to one](/img/0001-mapreduce-job-fail-due-to-wrong-setting/07-wrong-setting.png)

到此，单个datanode挂机，导致Hive任务失败的问题定位清楚了，有人改动了MR任务文件的副本参数！重新将该参数增大，改为2或3之后，问题就解决了。

## 结语
导致此次故障的参数比较生僻，很少有人会主动修改该参数（后来得知当初将该参数设置为1，是为了节省HDFS空间，减少Hive任务产生的临时文件）。

大数据组件参数众多，改变参数前，需了解参数的细节，再进行改动。否则就出现了上面的问题，即使普通文件是3副本，但MR的任务文件是1副本，出现单个datanode故障时，任务仍然会失败。