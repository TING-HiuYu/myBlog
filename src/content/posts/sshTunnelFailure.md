---
title: SSH 远程端口转发无法启动：直接登录正常但隧道卡住
published: 2026-06-12
description: "SSH 远程端口转发（Remote Forwarding）时连接卡住，直接登录却正常的问题排查与解决方法，涉及残留 sshd 子进程占用端口的清理。"
tags: ["SSH", "端口转发", "网络调试"]
category: Debug
draft: false
---

## 环境

- 客户端：OpenSSH 任意版本
- 服务端：Ubuntu 24.04、OpenSSH不知道什么版本
- 认证方式：公钥认证
- 转发类型：远程端口转发（`-R`）

## 问题

今天心血来潮想喝杯奶茶，出门时直接笔记本一盖就出门了，到了奶茶店想要通过 SSH Tunnel 让服务器使用本机代理时，发现一直无法建立隧道。排查日志发现通过 SSH 建立远程端口转发隧道时，客户端日志显示认证成功、连接建立，但在 `Starting a new Remote Port-Forwarding rule` 后不再有任何进展，隧道无法使用。但直接执行 SSH 登录（不带 `-R` 参数）可以正常连接和操作。

## 排查过程

### 1. 观察客户端日志

日志关键部分如下（已脱敏）：

```text
👤 Starting a new connection to: "login.example.com" port "22"
⚙️ Agreed KEX algorithm: curve25519-sha256
⚙️ Handshake finished
👤 Authenticated to "login.example.com":"22"
👤 Starting a new Remote Port-Forwarding rule   # 此后卡住
```

没有报错，也没有成功绑定端口的提示。直接 `ssh user@host -p 22` 则可正常登录。

### 2. 检查服务器端口占用

登录到远程服务器（通过普通 SSH 会话），查看拟转发的端口是否已被监听：

```bash
sudo netstat -tulnp | grep :7897
```

发现该端口处于 `LISTEN` 状态，且对应的进程是 `sshd` 的子进程。这意味着之前的某个 SSH 隧道没有正常关闭，残留的 `sshd` 进程仍然占用了转发端口。

### 3. 清理残留进程

找到残留进程的 PID 后，可以直接杀死该进程，或者终止当前用户的所有 `sshd` 子进程：

```bash
pkill -u <myuser> sshd
```

> 注意：该命令会同时杀死您当前正在使用的 SSH 会话，需要重新登录。

### 4. 重新尝试隧道

清理完成后，再次发起远程端口转发：

```bash
ssh -R 7897:localhost:7897 <myuser@login.example.com> -p <port>
```

隧道成功建立，问题解决。

## 根因分析

### 正常流程

- 当客户端通过 `-R` 请求远程转发时，SSH 服务器会启动一个监听进程绑定到指定的远程端口。
- 客户端断开连接时，正常情况下服务器端的监听进程也会随之关闭，端口被释放。

### 异常残留

以下情况可能导致服务器的子进程未能正常退出：
- 客户端被强制终止（`kill -9`、网络中断、电源断电）
- SSH 复用（`ControlMaster`）异常
- 服务器端 `sshd` 配置问题或资源限制

残留的 `sshd` 进程仍保持着对目标端口的绑定。当新客户端请求相同端口的转发时，由于端口已被占用（`Address already in use`），服务器不会返回成功信息，客户端可能表现为卡住或超时。

### 为什么普通登录正常？

普通登录（不包含 `-R` 选项）不涉及端口绑定，因此不受残留进程的影响。

## 附录：脱敏后的完整日志供参考

以下为脱敏后的客户端日志（包含成功认证和启动转发卡住的片段）：

```text
👤 Starting a new connection to: "login.example.com" port "22"
⚙️ Starting address resolution of "login.example.com"
⚙️ Address resolution finished
⚙️ Connecting to "203.0.113.10" port "22"
👤 Connection to "login.example.com" established
⚙️ Starting SSH session
⚙️ Remote server: SSH-2.0-OpenSSH_9.6p1 Ubuntu-3ubuntu13.16
⚙️ Agreed KEX algorithm: curve25519-sha256
⚙️ Agreed Host Key algorithm: ecdsa-sha2-nistp256
⚙️ Agreed server-to-client cipher: aes256-gcm@openssh.com MAC: INTEGRATED-AES-GCM
⚙️ Agreed client-to-server cipher: aes256-gcm@openssh.com MAC: INTEGRATED-AES-GCM
⚙️ Agreed client-to-server compression: none
⚙️ Agreed server-to-client compression: none
⚙️ Handshake finished
👤 Checking host key: SHA256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
👤 Host "login.example.com":"22" is known and matches
👤 Authenticating to "login.example.com":"22" as "myuser"
⚙️ Available client authentication methods: publickey,password,keyboard-interactive
⚙️ Authentication that can continue: publickey
👤 Authenticating using publickey method
👤 Authentication succeeded (publickey)
👤 Authenticated to "login.example.com":"22"
👤 Starting a new Remote Port-Forwarding rule   # 此处卡住
```

## 总结

SSH 远程端口转发失败但普通登录正常时，**首先怀疑服务器端是否有残留的 `sshd` 进程占用了目标端口**。通过 `pkill -u 用户名 sshd` 清理后问题大多可解决。建议在客户端添加 `ExitOnForwardFailure=yes` 以便快速发现冲突，并在服务器端合理配置连接保活参数，减少残留发生的概率。