---
title: 使用我们的Slurm集群
published: 2026-03-01
description: "本篇文章主要介绍了怎么使用我们内部的slurm集群"
tags: ["教程", "Slurm"]
category: Guides
draft: false
---

# Slurm集群使用指南

欢迎使用本集群。本文档将引导您完成从环境配置到作业提交的完整流程，帮助您高效利用计算资源。

---

## 0. 系统架构

集群由五台服务器组成，通过一台 **40G 以太网交换机**互连，构建高速计算网络。

<!-- ```mermaid
graph TD
    Switch[Mellanox SX6012 40G 交换机]

    Login[登录服务器]
    GPU["GPU服务器<br/>64c128t | 120GB RAM<br/>8×NVIDIA 4090 48GB"]
    CPU1["CPU服务器1<br/>48c96t | 128GB RAM"]
    CPU2["CPU服务器2<br/>48c96t | 256GB RAM"]
    Storage["存储服务器<br/>48TB 阵列<br/>1TB NVMe SSD 缓存<br/>256GB Optane PMEM"]

    Login --- Switch
    GPU --- Switch
    CPU1 --- Switch
    CPU2 --- Switch
    Storage --- Switch
``` 
-->


**硬件配置明细**：

- **登录服务器**：用户接入节点，用于作业提交和管理。
- **GPU服务器**：64核心128线程，120GB内存，8张NVIDIA 4090 48GB显卡。
- **CPU服务器1**：48核心96线程，128GB内存。
- **CPU服务器2**：48核心96线程，256GB内存。
- **存储服务器**：48TB硬盘阵列，配备1TB NVMe SSD高速缓存和256GB Intel Optane PMEM超高速缓存。

<img width="1512" height="867" alt="零信任平台入口" src="/posts/usingSlurm/topology.png" />

---

## 1. 环境与基本配置

### 1.1 登录零信任平台
访问 [https://ai4qc-hkust.cloudflareaccess.com/](https://ai4qc-hkust.cloudflareaccess.com/)。  
**注意**：必须使用已在 `ai4qc` 组织内的 GitHub 帐户登录，否则无法获得授权。  
成功登录后，您将看到如下界面：  
<img width="1512" height="867" alt="零信任平台入口" src="/posts/usingSlurm/zt-entrypoint.png" />



### 1.2 访问用户管理界面
点击零信任平台中的 **profile** 或直接访问 [https://profile.ai4qc.icu/](https://profile.ai4qc.icu/) 进入用户管理界面。  
<img width="1512" height="867" alt="image" src="/posts/usingSlurm/go-ldap-login.png" />


### 1.3 首次登录与初始密码
- 点击 **OAuth 登录**，使用您的 GitHub 账号授权。初次登录将自动完成用户注册。
- 登录成功后进入个人主页。**新用户会弹出初始密码**，请妥善保存；您可以立即修改或暂时忽略。  
<img width="1512" height="867" alt="image" src="/posts/usingSlurm/profile-page.png" />


### 1.4 获取 SSH 证书
在个人主页中，找到 **“生成并下载 SSH 证书”** 区域，点击 **“生成”** 按钮。系统将生成一对证书并自动下载为一个压缩包：
- **私钥文件**：文件名格式为 `[用户名]-[日期]`（例如 `zhangsan-20250302`）
- **签名证书**：文件名格式为 `[用户名]-[日期]-cert.crt`（例如 `zhangsan-20250302-cert.crt`）

解压后，您可以选择以下任一方式使用证书连接集群：

#### 1.4.0 注意事项
```
以防有人不仔细看，SSH 记得加端口9922
```


#### 1.4.1 使用命令行直接登录
```bash
ssh -i /path/to/private_key_file <username>@login.ai4qc.icu -p 9922
```
（SSH 客户端会自动寻找同名的证书文件，无需额外指定。）

#### 1.4.2 配置 SSH config
在 `~/.ssh/config` 中添加如下配置：
```
Host ai4qc
    HostName login.ai4qc.icu
    User <your-username>
    Port 9922
    IdentityFile /path/to/private_key_file
    CertificateFile /path/to/cert_file
```
之后可直接使用 `ssh ai4qc` 登录。

#### 1.4.3 在 VS Code 中使用
参考官方文档 [Improving your security with a dedicated key](https://code.visualstudio.com/docs/remote/troubleshooting#_improving-your-security-with-a-dedicated-key)，配置 Remote-SSH 使用证书。

#### 1.4.4 在 Termius 中使用
Termius 支持直接导入 SSH 证书，请参阅 [Import SSH Certificate](https://termius.com/documentation/ssh-certificates) 完成设置。

### 1.5 配置终端（可选）
默认登录 Shell 为 **zsh**，其具备强大的自动补全和主题功能。如需切换为 bash，请在登录后进入 **“终端设置”** → **“Shell”** 进行修改。

### 1.6 连接集群
完成上述配置后，即可通过 SSH 正常登录集群：
```bash
ssh <username>@login.ai4qc.icu -p 9922
```

---

## 2. Slurm 作业调度系统使用

### 2.1 分区说明
集群包含两个分区（Partition）：
- **cpu**：用于纯 CPU 计算任务（默认分区）。
- **gpu**：用于 GPU 加速任务。

提交作业时若未指定分区，默认使用 `cpu`。

### 2.2 提交作业示例
- **GPU 作业**：**系统会自动分配一张显卡**。如需多卡，请使用 `-G` 或 `--gres` 参数显式指定。
  ```bash
  # 自动分配一张显卡（默认）
  sbatch -p gpu -c 4 your_program

  # 显式指定使用4张显卡
  sbatch -p gpu -G 4 --cpus-per-task=4 your_program
  ```
- **CPU 作业**：需自行指定核心数量。
  ```bash
  sbatch -p cpu -c 8 your_program
  ```

### 2.3 注意事项
  因为登陆服务器是挂载在MacOS上的虚拟机，强烈不推荐在登陆服务器上进行如下任务：
- 编译可执行程序
- 运行计算任务

如果需要编译可执行程序，请先编译好再上传到slurm上。

如果需要测试程序，请直接使用`srun`进入交互式命令行执行。
  ```bash
  srun -p cpu -c 8 your_program
  ```

### 2.4 更多参考
上海交通大学超算平台提供了详尽的 Slurm 使用指南，强烈建议阅读：
[https://docs.hpc.sjtu.edu.cn/job/slurm.html](https://docs.hpc.sjtu.edu.cn/job/slurm.html)

---

## 3. 任务后台查看

### 3.1 访问 Dashboard
打开浏览器访问 [https://slurm-dashboard.ai4qc.icu/](https://slurm-dashboard.ai4qc.icu/)。  
若浏览器会话中无有效的 Access Token，系统将自动重定向至 Cloudflare 零信任网关重新登录。认证成功后，将跳转至 Dashboard, 即可查看作业状态、资源使用情况等。  



更详细的使用说明，请阅读[Slurm-web/Overview](https://docs.rackslab.io/slurm-web/overview/overview.html)
<img width="1512" height="867" alt="image" src="/posts/usingSlurm/slurm-web.png" />

---

## 4. 密码管理

### 4.1 登录 profile 页面
访问 [https://profile.ai4qc.icu/](https://profile.ai4qc.icu/) 并使用 GitHub OAuth 登录。

### 4.2 修改密码
在 **“修改帐户密码”** 区域，您可以通过以下两种方式验证身份：
- **原密码**：输入当前密码。
- **OAuth**：使用 GitHub 快速验证（推荐，邮箱验证暂不可用）。

验证通过后，在右侧输入新密码两次，确保一致，点击 **提交** 即可。

### 4.3 注意事项
- 密码修改成功后，您会被强制登出，后续访问需使用新密码重新登录。
- 请妥善保管密码，避免泄露。

---

如有任何问题，请联系系统管理员。
