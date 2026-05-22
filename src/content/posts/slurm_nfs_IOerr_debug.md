---
title: "Slurm NFS 挂载后 IO error 排障记录"
published: 2026-05-22
description: "记录一次 Slurm 计算节点无法稳定访问 NFS 的排障过程：从降级NFS版本，修改MTU，到最终恢复 NFSv4.2。"
tags: ["故障排除", "Slurm", "nfs"]
category: Debug
draft: false
---

为了信息脱敏，正文内所有的用户名和具体ip等敏感内容用\<\>代替

# 背景

集群里有一台登录节点 `2.login.slurm.lan`，三台计算节点，均在`<allowed ip range>`下：

- `1.compute.slurm.lan`
- `2.compute.slurm.lan`
- `3.compute.slurm.lan`

故障发生前，集群交换机经历了一次异常断电并恢复。恢复后，计算节点上的 NFS 挂载表面存在，但访问 `/home` 和 `/opt` 时会出现超时或 `Input/output error`，进而导致 Slurm 任务无法正常启动。

存储服务器通过 NFS 导出两个目录：

- `<nfs server ip>:/mnt/Data/slurm/userhome` 挂载到 `/home`
- `<nfs server ip>:/mnt/Data/slurm/opt` 挂载到 `/opt`

其中 `/home` 提供用户目录，例如 `/home/sshusers/<user>`；`/opt` 里有 Slurm 的 `TaskProlog` 依赖脚本：

```text
/opt/shell_related/task_prolog.sh
```

因此这两个 NFS 挂载都必须正常。只挂上 `/home` 不够，`/opt` 访问失败会直接导致 Slurm 任务启动失败。

# 故障现象

最初在 `2.compute.slurm.lan` 上运行：

```bash
srun -c 8 -w 2.compute.slurm.lan hostname
```

报错：

```text
srun: error: 2.compute.slurm.lan: task 0: Exited with exit code 1
[2026-05-22T15:40:21.638] error: run_command: slurm task_prolog can not be executed (/opt/shell_related/task_prolog.sh) No such file or directory
[2026-05-22T15:40:21.638] error: slurm task_prolog did not exit normally. reason: Run command failed - configuration error
[2026-05-22T15:40:21.638] error: TaskProlog failed status=1
```

节点上直接访问 NFS 路径时，可见类似现象：

```bash
ls /home/sshusers/<user>
```

返回：

```text
ls: reading directory '/home/sshusers/<user>': Input/output error
```

访问 `/opt` 也会失败：

```bash
head -1 /opt/shell_related/task_prolog.sh
```

返回：

```text
head: cannot open '/opt/shell_related/task_prolog.sh' for reading: Input/output error
```

内核日志里能看到 NFS 超时：

```text
nfs: server <nfs server ip> not responding, timed out
```

# 复现方式

在计算节点上复现主要看三个层面。

检查 `/home`：

```bash
timeout 10 ls -la /home/sshusers/<user>
```

检查 `/opt`：

```bash
timeout 10 head -1 /opt/shell_related/task_prolog.sh
```

检查 Slurm：

```bash
srun -c 8 -w 2.compute.slurm.lan hostname
```

对 GPU 节点 `1.compute.slurm.lan`，需要指定 GPU 分区和资源：

```bash
srun -p gpu --gres=gpu:1 -c 8 -w 1.compute.slurm.lan hostname
```

如果 NFS 正常，这些命令应该分别能读目录、读 prolog 脚本，并输出节点 hostname。

# 初始检查

查看 NFS 导出：

```bash
showmount -e <nfs server ip>
```

导出内容包含：

```text
/mnt/Data/slurm/opt      <allowed ip range>
/mnt/Data/slurm/userhome <allowed ip range>
```

说明计算节点所在网段 `<allowed ip range>` 有权限访问导出。

检查 RPC 服务：

```bash
rpcinfo -p <nfs server ip>
```

能看到 NFS v3 和 v4：

```text
100003    3   tcp   2049  nfs
100003    4   tcp   2049  nfs
```

这说明服务端 NFS 服务本身是可见的。

# 尝试不同NFS参数和版本

一开始计算节点上的 `/etc/fstab` 使用过 NFSv3、systemd automount、soft 等不同组合。为了先恢复访问，尝试过 NFSv3：

```fstab
<nfs server ip>:/mnt/Data/slurm/userhome /home nfs _netdev,defaults,noatime,nolock,nordirplus,soft,timeo=30,retrans=2,actimeo=1800,vers=3,proto=tcp,mountproto=tcp 0 0
<nfs server ip>:/mnt/Data/slurm/opt /opt nfs _netdev,defaults,noatime,nolock,nordirplus,soft,timeo=30,retrans=2,actimeo=1800,vers=3,proto=tcp,mountproto=tcp 0 0
```

其中 `nordirplus` 很关键：在早期故障状态下，NFSv3 默认的 `readdirplus` 读目录会触发 `Input/output error`，加上 `nordirplus` 后能暂时让目录读取恢复。

但这只是绕过了一部分症状。后续发现真正的底层问题不在 NFS 版本，而在网络 MTU。

# 定位到 MTU

登录节点访问 NFS 正常，而计算节点访问 NFS 出现 `Input/output error`。对比网络配置时发现：

登录节点到存储网络的 MTU 是 `1500`：

```text
enp4s0np0 mtu 1500
```

计算节点 2、3 的存储网络是 bond，MTU 是 `9000`：

```text
bond0 mtu 9000
```

计算节点 1 的存储网卡是：

```text
enp196s0d1 mtu 9000
```

当交换机端口还没有完全打开巨型帧时，小包 ping 能通：

```bash
ping -c 1 <nfs server ip>
```

但巨型帧 ping 不通：

```bash
ping -M do -s 8972 -c 1 <nfs server ip>
```

NFS 的表现是：挂载握手、`stat` 这类小元数据请求可能成功，但读目录、读文件这种较大的请求会超时或 `Input/output error`。

这解释了为什么故障看起来像 NFS 版本或 fstab 选项问题，其实底层是链路 MTU 不一致。

# 临时验证

先在 `2.compute.slurm.lan` 上临时把 MTU 改成 `1500`：

```bash
ip link set dev bond0 mtu 1500
ip link set dev ens2 mtu 1500
ip link set dev ens2d1 mtu 1500
```

然后重新挂载：

```bash
umount -fl /home /opt
mount -v /home
mount -v /opt
```

验证通过：

```bash
timeout 10 ls -la /home/sshusers/<user>
timeout 10 head -1 /opt/shell_related/task_prolog.sh
srun -c 8 -w 2.compute.slurm.lan hostname
```

这证明故障与巨型帧链路有关。

# 交换机修复

随后在交换机侧把相关端口 MTU 打开到 `9216`。再次在计算节点上测试 9000 MTU。

在交换机端口 MTU 设置完成并等待端口重新上线后，NFS 访问恢复正常。结合故障前的异常断电，推测原因是交换机之前只修改了运行时配置，但没有把配置写入 ROM；交换机重启后端口实际支持的帧大小回退，低于计算节点的 MTU 9000，导致 NFS 大包读目录、读文件时失败。

`2.compute.slurm.lan`：

```bash
ping -M do -s 8972 -c 1 <nfs server ip>
```

成功：

```text
8980 bytes from <nfs server ip>: icmp_seq=1 ttl=64 time=0.148 ms
```

NFS 访问也成功：

```bash
timeout 10 ls -la /home/sshusers/<user>
timeout 10 head -1 /opt/shell_related/task_prolog.sh
```

Slurm 验证成功：

```bash
srun -c 8 -w 2.compute.slurm.lan hostname
```

输出：

```text
TaskProlog executed at Fri May 22 16:09:44 UTC 2026
2.compute.slurm.lan
```

后续补开 `1.compute.slurm.lan` 对应交换机端口后，1 的巨型帧 ping 也恢复：

```text
8980 bytes from <nfs server ip>: icmp_seq=1 ttl=64 time=0.090 ms
```

# 最终切回 NFSv4.2

在 MTU 修复后，重新测试 NFSv4.2，发现三台计算节点均可正常使用。最终不再需要 NFSv3 的 `nolock`、`nordirplus`、`mountproto=tcp` 等选项。

最终三台计算节点的 `/etc/fstab` NFS 行统一为：

```fstab
<nfs server ip>:/mnt/Data/slurm/userhome /home nfs _netdev,defaults,noatime,soft,timeo=30,retrans=2,actimeo=1800,proto=tcp,vers=4.2 0 0
<nfs server ip>:/mnt/Data/slurm/opt /opt nfs _netdev,defaults,noatime,soft,timeo=30,retrans=2,actimeo=1800,proto=tcp,vers=4.2 0 0
```

实际挂载确认：

```bash
nfsstat -m
```

可以看到：

```text
/home from <nfs server ip>:/mnt/Data/slurm/userhome
 Flags: rw,noatime,vers=4.2,...

/opt from <nfs server ip>:/mnt/Data/slurm/opt
 Flags: rw,noatime,vers=4.2,...
```

# 最终验证

`1.compute.slurm.lan`：

```bash
srun -p gpu --gres=gpu:1 -c 8 -w 1.compute.slurm.lan hostname
```

输出：

```text
TaskProlog executed at Sat May 23 00:18:41 CST 2026
1.compute.slurm.lan
```

`2.compute.slurm.lan`：

```bash
srun -c 8 -w 2.compute.slurm.lan hostname
```

输出：

```text
TaskProlog executed at Fri May 22 16:18:01 UTC 2026
2.compute.slurm.lan
```

`3.compute.slurm.lan`：

```bash
srun -c 8 -w 3.compute.slurm.lan hostname
```

输出：

```text
TaskProlog executed at Fri May 22 16:20:12 UTC 2026
3.compute.slurm.lan
```

三台节点均确认：

```bash
ls /home/sshusers/<user>
head -1 /opt/shell_related/task_prolog.sh
```

均正常。

# 总结

这次故障表面上是 NFS 挂载失败和 Slurm `TaskProlog` 执行失败，实际根因是计算节点到存储服务器之间的巨型帧 MTU 配置不一致。更具体地说，交换机异常断电恢复后，运行时 MTU 配置没有从 ROM 中恢复到预期状态，导致交换机端口支持的帧大小小于计算节点 MTU。

排查时几个关键点：

1. 不要只看 `mount` 是否成功。NFS 可能已经挂上，但读目录或读文件时才报 `Input/output error`。
2. `showmount`、`rpcinfo` 成功只能说明服务可见，不代表数据路径没有 MTU 问题。
3. 对巨型帧场景，必须用 `ping -M do -s 8972 <server>` 验证端到端 MTU。
4. 记得在设置完交换机的巨型帧后，要把运行时配置写入 ROM，避免异常断电或重启后配置丢失。

最终修复手段为：

- 存储网络链路端到端支持 MTU 9000，交换机端口 MTU 开到 9216。
