---
title: "NFS: Docker挂载NFS，间歇报input/output error错"
date: 2021-02-28T17:13:22+08:00
categories: ["线上故障"]
tags: ["NFS", "Docker"]
draft: false
---
## 前言
线上的服务发布在基于Swarm管理的docker里，并且挂载了NFS，来访问远端的数据。

## 故障描述
某日，几个docker，不规律地报input/output error错误。出现错误时，服务无法通过NFS进行正常的读取，导致线上服务报400或500错误。

## 故障分析
NFS相对上层应用来说是黑盒子，先看看错误日志，日志是打开黑盒子的第一步。

### 开启debug日志，定位错误
幸运的是，故障现场得到保留，打开debug日志之后，能够重现错误。访问test1.txt，出故障时，终端上显示input/output error，但日志里返回的是10011(NFS4ERR_EXPIRED)。
```shell
# 开启nfs debug level的日志
$ rpcdebug -m nfs -s all
$ cat /proc/sys/sunrpc/nfs_debug
 
$ less /var/log/message
Nov 12 18:31:40 docker27 kernel: NFS: atomic_open(0:94/204203405579), test1.txt                               ====> open test1.txt文件
Nov 12 18:31:40 docker27 kernel: --> nfs4_alloc_slot used_slots=0000 highest_used=4294967295 max_slots=1024
Nov 12 18:31:40 docker27 kernel: <-- nfs4_alloc_slot used_slots=0001 highest_used=0 slotid=0
Nov 12 18:31:40 docker27 kernel: nfs4_free_slot: slotid 0 highest_used_slotid 4294967295
Nov 12 18:31:40 docker27 kernel: nfs4_map_errors could not handle NFSv4 error 10011                           ====> 出现10011(NFS4ERR_EXPIRED)错误
```

### NFS4ERR_EXPIRED与input/output error
根据日志里的错误信息`nfs4_map_errors could not handle NFSv4 error 10011`，找到Linux kernel nfs代码，发现NFS会将无法处理的错误（NFS4ERR_EXPIRED），转换成EIO（即input/output error）。
```c
/* Prevent leaks of NFSv4 errors into userland */
static int nfs4_map_errors(int err)
{
	if (err >= -1000)
		return err;
	switch (err) {
    ......
	case -NFS4ERR_FILE_OPEN:
		return -EBUSY;
	case -NFS4ERR_NOT_SAME:
		return -ENOTSYNC;
	default:
		dprintk("%s could not handle NFSv4 error %d\n",  ====> 即日志里的nfs4_map_errors could not handle NFSv4 error 10011
				__func__, -err);
		break;
	}
	return -EIO;                                         ====> 无法处理的错误，转换为EIO，在终端上显示input/output error
}

```

### NFS4ERR_EXPIRED是哪里返回的
进一步需要知道，NFS4ERR_EXPIRED到底是客户端产生的，还是服务端返回的？抓个包，看一眼。
![nfs4err_expired tcpdump](/img/0002-nfs-input-output-error/01-hit-nfs4err-expired-error.png)

发现是服务端返回的(NFS server是10.10.14.77，NFS client是10.10.14.236)。从图中还可以得知，是Open操作返回的。

### Open为何会返回NFS4ERR_EXPIRED？
扫了一眼Open相关的代码，发现Open的调用链的lookup_clientid函数，会返回NFS4ERR_EXPIRED。
```c
// 根据Open调用链，OP_OPEN->nfsd4_open->nfsd4_process_open1->lookup_clientid
 
lookup_clientid(...)
{
    ......
    found = find_confirmed_client(clid, false, nn);
    if (!found) {                             ====> 根据clid查找客户端的数据结构，未找到，则返回NFS4ERR_EXPIRED
        spin_unlock(&nn->client_lock);
        return nfserr_expired;                ====> 此处返回NFSERR_EXPIRED，NFSERR_EXPIRED == NFS4ERR_EXPIRED == 10011
    }
    ......
}
```
根据代码可知，使用客户端id进行查询的时候，找不到了对应的内容了，会返回NFSERR_EXPIRED。客户端的信息应该是初始化阶段就注册好的，为啥会突然找不到呢？

### 为何NFS server里的客户端数据找不到了？
Google了一下，发现有人碰到类似的错误：当出现两个同名的客户端访问NFS时，会出现NFS4ERR_EXPIRED错误，见[serverfault](https://serverfault.com/questions/851334/what-is-causing-input-output-errors-when-reading-from-nfs-v4-on-centos)以及
[redhat knowledgebase](https://access.redhat.com/solutions/3753211)。

### SETCLIENTID操作
与此同时，抓包的过程中，发现SETCLIENTID操作很可疑（看名字像是Client id相关的写操作，而且每次该函数调用完，紧接着会出现NFSERR_EXPIRED）。
![setclientid tcpdump](/img/0002-nfs-input-output-error/02-setclientid.png)

这个SETCLIENTID操作，到底做了啥？再翻翻源码。
```c
OP_SETCLIENTID->nfsd4_setclientid
 
nfsd4_setclientid(...)
{
    ......
    unconf = find_unconfirmed_client_by_name(&clname, nn);    ====> 根据clname(客户端名字)查找_同名的_客户端的数据结构
    ......
    if (unconf)
        expire_client(unconf);                                ====> 找到之后，开始删除NFS server上_同名的_客户端数据结构
    ......
}
```
啊，看到上面`主动删除客户端数据`的代码，离解开谜题不远了。经过SETCLIENTID操作，删除同名的客户端之后，再次open的时候就会碰到，找不到客户端数据结构的问题。

那是否存在同名的客户端呢？

### 存在同名客户端
继续抓包分析，发现10.10.14.235的clname叫"Linux NFSv4.0 0-xxxxxx-job.app.local/10.10.14.77"
![nfs4err_expired tcpdump](/img/0002-nfs-input-output-error/03-same-clname-1.png)
而10.10.14.236的clname也叫"Linux NFSv4.0 0-xxxxxx-job.app.local/10.10.14.77"
![nfs4err_expired tcpdump](/img/0002-nfs-input-output-error/04-same-clname-2.png)

同一个docker集群里`真的存在同名客户端 -。-`！客户端是如何命名的呢？

### 客户端名字生成流程
看看SETCLIENTID操作里clname是如何构造的。
```c
# 根据客户端encode代码，和上图id->contents所处的位置可知，cl_owner_id即为se_name
 
static void encode_setclientid(struct xdr_stream *xdr, const struct nfs4_setclientid *setclientid, struct compound_hdr *hdr)
{
   __be32 *p;
   encode_op_hdr(xdr, OP_SETCLIENTID, decode_setclientid_maxsz, hdr);
   encode_nfs4_verifier(xdr, setclientid->sc_verifier);                       ===> verifier
   encode_string(xdr, strlen(setclientid->sc_clnt->cl_owner_id),
           setclientid->sc_clnt->cl_owner_id);                                ===> NFS客户端的cl_owner_id，即为NFS服务端的se_name
   p = reserve_space(xdr, 4);
   *p = cpu_to_be32(setclientid->sc_prog);                                    ===> callback cb_program
   encode_string(xdr, setclientid->sc_netid_len, setclientid->sc_netid);
   encode_string(xdr, setclientid->sc_uaddr_len, setclientid->sc_uaddr);
   p = reserve_space(xdr, 4);
   *p = cpu_to_be32(setclientid->sc_clnt->cl_cb_ident);
}
```
也就是说其实是cl_owner_id重名了，那cl_owner_id又是如何构造的呢？

cl_owner_id可能在如下三个函数中被设置，但不同版本的内核版本，设置的内容略有不同。总结来说：
* 小于等于4.4.108的kernel中，cl_owner_id由NFS客户端的IP构成；
* 4.19.12 kernel中，cl_owner_id由NFS客户端的hostname构成（当服务实例从一个物理机迁移到另外一个物理机上之后，若实例的hostname保持不变，导致重名）；
```c
// 小于等于4.4.108
// 常规mount，未设置-o migration参数
nfs4_init_nonuniform_client_string
    result = scnprintf(str, len, "Linux NFSv4.0 %s/%s %s",
            clp->cl_ipaddr,                                            ====> 客户端ip
            rpc_peeraddr2str(clp->cl_rpcclient, RPC_DISPLAY_ADDR),
            rpc_peeraddr2str(clp->cl_rpcclient, RPC_DISPLAY_PROTO));
 
// 设置了migration，-o migration参数
nfs4_init_uniform_client_string
    result = scnprintf(str, len, "Linux NFSv%u.%u %s",
            clp->rpc_ops->version, clp->cl_minorversion,
            clp->cl_rpcclient->cl_nodename);
 
// 设置了migration，-o migration参数，且设置了nfs4_unique_id参数
nfs4_init_uniquifier_client_string
    result = scnprintf(str, len, "Linux NFSv%u.%u %s/%s",
            clp->rpc_ops->version, clp->cl_minorversion,
            nfs4_client_id_uniquifier,
            clp->cl_rpcclient->cl_nodename);
			
			
// 4.19.12
// 常规mount，未设置-o migration参数
nfs4_init_nonuniform_client_string
    # 设置了nfs4_unique_id参数
    if (nfs4_client_id_uniquifier[0] != '\0')
        scnprintf(str, len, "Linux NFSv4.0 %s/%s/%s",
              clp->cl_rpcclient->cl_nodename,                ====> 客户端hostname
              nfs4_client_id_uniquifier,
              rpc_peeraddr2str(clp->cl_rpcclient,
                       RPC_DISPLAY_ADDR));
    else
        scnprintf(str, len, "Linux NFSv4.0 %s/%s",
              clp->cl_rpcclient->cl_nodename,
              rpc_peeraddr2str(clp->cl_rpcclient,
                       RPC_DISPLAY_ADDR));
 
// 设置了migration，-o migration参数
nfs4_init_uniform_client_string
    scnprintf(str, len, "Linux NFSv%u.%u %s",
            clp->rpc_ops->version, clp->cl_minorversion,
            clp->cl_rpcclient->cl_nodename);
 
// 设置了migration，-o migration参数，且设置了nfs4_unique_id参数
nfs4_init_uniquifier_client_string
    scnprintf(str, len, "Linux NFSv%u.%u %s/%s",
            clp->rpc_ops->version, clp->cl_minorversion,
            nfs4_client_id_uniquifier,
            clp->cl_rpcclient->cl_nodename);
```
简而言之，如果想要每个客户端的名字全局唯一，需要提前设定nfs4_unique_id的值。`通过设置全局唯一的nfs4_unique_id的值`，才能保证不同NFS客户端拥有不同的名字。

## 解决方案
由于不同内核版本，NFS客户端的名字构成不同，需使用不同的方式来处理。生产环境中，内核版本只有两种：4.4.108与4.19.12，分别使用如下策略来处理：
1. 4.4.108。4.4.108的NFS客户端的名字由客户端IP构成，而IP全局唯一，因此4.4.108上，NFS客户端的名字同样唯一，理论上不会出现不同节点有同名客户端的问题。对于4.4.108无需进行修复。
2. 4.19.12。4.19.12使用hostname作为NFS客户端名字，hostname很容易重名，想要保证名字全局唯一，需要额外设置nfs4_unique_id，通过nfs4_unique_id来保证NFS客户端名字完全唯一，设置方式如下：
```
1，在宿主机上，设置nfs4_unique_id为uuid（不是一定要uuid，此处设置一个全局唯一的id也可）
$ cat /sys/module/nfs/parameters/nfs4_unique_id
$ echo `cat /proc/sys/kernel/random/uuid` > /sys/module/nfs/parameters/nfs4_unique_id
2，umount所有Docker里挂载，重新mount即可
3，抓包检测一下nfs客户端的名字是否带上了nfs4_unique_id
4，若宿主机nfs4_unique_id已被设置，需要清零，可使用如下命令
$ echo -e '\0' > /sys/module/nfs/parameters/nfs4_unique_id
```

## 结语
总结一下故障过程：
1. 两个客户端使用相同的名字访问NFS server；
2. 互相刷lease，导致对方lease失效；
3. 引发open文件时返回NFS4ERR_EXPIRED错误；
4. NFS4ERR_EXPIRED错误，NFS客户端无法处理，转换为Input/Output error，因此用户感受到的是Input/Output error错误。

Swarm docker+NFS的方案，比较少见。因此国内少有人碰到以上的问题。由于Swarm的docker与主机共享hostname/ip，当与基于内核的NFS结合使用时，碰到了意向不到的问题，基础组件，还是要使用被广泛验证的方案，如K8s（docker有独立ip，隔离性更好），减少遇见奇怪坑的几率。

## Ref

### 类似问题
* https://access.redhat.com/solutions/3753211
* https://access.redhat.com/solutions/3457671
* https://access.redhat.com/solutions/3753211
* https://access.redhat.com/solutions/3661921

### NFS debug
* https://www.thegeekdiary.com/how-to-enable-nfs-debug-logging-using-rpcdebug/
* https://wiki.archlinux.org/index.php/NFS/Troubleshooting_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)#NFS_debug_flags
