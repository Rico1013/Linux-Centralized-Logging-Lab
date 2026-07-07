# Linux Centralized Logging Lab

## 1. 项目简介

本项目是一个基于 `rsyslog` 的 Linux 集中式日志管理实验。

实验目标是将 `node1` 中由 `rsyslog` 管理的系统日志，全部转发到 `node2` 日志服务器进行统一保存。同时，`node1` 本地仍然保留原有日志。当日志服务器不可用或网络不稳定时，`node1` 会通过 rsyslog 队列机制临时缓存日志，并在日志服务器恢复后继续转发，尽量避免日志丢失。

通过本实验，可以理解 Linux 系统日志的基本管理方式、远程日志转发流程、TCP 514 端口的作用，以及 rsyslog 队列在可靠日志传输中的意义。

---

## 2. 项目环境

| 主机名   | 角色    | IP 地址说明    |
| ----- | ----- | ---------- |
| node1 | 日志客户端 | IP 后缀为 140 |
| node2 | 日志服务器 | IP 后缀为 137 |

> 说明：实际 IP 地址以个人虚拟机环境为准。例如，如果网段是 `192.168.83.0/24`，则 node1 可能为 `192.168.83.140`，node2 可能为 `192.168.83.137`。

---

## 3. 项目目标

本实验主要实现以下目标：

1. 在 `node2` 上搭建 rsyslog 日志服务器；
2. 开启 `node2` 的 TCP 514 端口，用于接收远程日志；
3. 在 `node1` 上配置 rsyslog 日志转发；
4. 将 `node1` 中 rsyslog 管理的全部日志转发到 `node2`；
5. 保留 `node1` 本地日志，不影响原有日志记录；
6. 配置 rsyslog 队列，保证网络异常或日志服务器不可用时，日志可以暂存并在恢复后继续转发；
7. 使用 `logger` 命令测试日志转发效果。

---

## 4. 实验原理

Linux 系统中的许多日志由 `rsyslog` 进行统一管理。默认情况下，日志会被写入本地 `/var/log/` 目录下的相关日志文件中。

rsyslog 不仅可以把日志写入本地文件，也可以通过网络将日志转发到远程日志服务器。

常见转发方式有两种：

```text
@IP:514     使用 UDP 协议转发日志
@@IP:514    使用 TCP 协议转发日志
```

本实验选择 TCP 方式，因为 TCP 相比 UDP 更可靠，适合日志集中收集场景。

在可靠转发配置中，rsyslog 还可以启用队列机制。当远程日志服务器暂时不可用时，日志不会立即丢弃，而是先缓存在本地队列中。等远程服务器恢复后，rsyslog 会继续尝试发送缓存中的日志。

---

## 5. node2 日志服务器配置

### 5.1 安装并启动 rsyslog

在 `node2` 上执行：

```bash
sudo apt update
sudo apt install -y rsyslog
sudo systemctl enable --now rsyslog
sudo systemctl status rsyslog
```

确认服务状态为：

```text
active (running)
```

---

### 5.2 配置 node2 接收远程日志

在 `node2` 上新建配置文件：

```bash
sudo nano /etc/rsyslog.d/10-remote.conf
```

写入以下内容：

```conf
module(load="imtcp")
input(type="imtcp" port="514")

$template RemoteLogs,"/var/log/remote/%HOSTNAME%.log"

*.* ?RemoteLogs
& stop
```

配置说明：

| 配置项                              | 作用              |
| -------------------------------- | --------------- |
| `module(load="imtcp")`           | 加载 TCP 接收模块     |
| `input(type="imtcp" port="514")` | 监听 TCP 514 端口   |
| `$template RemoteLogs`           | 定义远程日志保存路径      |
| `/var/log/remote/%HOSTNAME%.log` | 按客户端主机名保存日志     |
| `*.* ?RemoteLogs`                | 接收所有远程日志        |
| `& stop`                         | 避免日志继续被后续规则重复处理 |

---

### 5.3 创建远程日志目录

```bash
sudo mkdir -p /var/log/remote
sudo chown syslog:adm /var/log/remote
```

---

### 5.4 重启 rsyslog

```bash
sudo systemctl restart rsyslog
sudo systemctl status rsyslog
```

---

### 5.5 检查 TCP 514 端口

```bash
sudo ss -lntp | grep 514
```

如果看到类似结果，说明 node2 已经开始监听 TCP 514 端口：

```text
LISTEN 0 25 0.0.0.0:514
```

---

### 5.6 防火墙放行 514/tcp

如果系统启用了 `ufw`，执行：

```bash
sudo ufw allow 514/tcp
sudo ufw reload
sudo ufw status
```

如果 `ufw` 未启用，执行：

```bash
sudo ufw status
```

显示如下则无需处理：

```text
Status: inactive
```

---

## 6. node1 日志客户端配置

### 6.1 安装并启动 rsyslog

在 `node1` 上执行：

```bash
sudo apt update
sudo apt install -y rsyslog
sudo systemctl enable --now rsyslog
sudo systemctl status rsyslog
```

确认服务状态为：

```text
active (running)
```

---

### 6.2 配置日志可靠转发

在 `node1` 上新建转发配置文件：

```bash
sudo nano /etc/rsyslog.d/90-forward-to-node2.conf
```

写入以下内容：

```conf
*.* action(
    type="omfwd"
    target="192.168.83.137"
    port="514"
    protocol="tcp"

    queue.type="LinkedList"
    queue.filename="node2_fwd"
    queue.maxdiskspace="1g"
    queue.saveonshutdown="on"
    action.resumeRetryCount="-1"
)
```

> 注意：这里的 `192.168.83.137` 需要替换成自己实际环境中 node2 的完整 IP 地址。
> 本实验中 node2 的 IP 后缀为 `137`。

配置说明：

| 配置项                            | 作用                 |
| ------------------------------ | ------------------ |
| `*.*`                          | 匹配所有服务、所有级别的日志     |
| `type="omfwd"`                 | 使用 rsyslog 远程转发模块  |
| `target="192.168.83.137"`      | 指定日志服务器 node2 的 IP |
| `port="514"`                   | 指定远程日志服务器端口        |
| `protocol="tcp"`               | 使用 TCP 协议转发        |
| `queue.type="LinkedList"`      | 启用队列机制             |
| `queue.filename="node2_fwd"`   | 设置队列文件名前缀          |
| `queue.maxdiskspace="1g"`      | 队列最多占用 1GB 磁盘空间    |
| `queue.saveonshutdown="on"`    | rsyslog 关闭时保存队列    |
| `action.resumeRetryCount="-1"` | 远程服务器不可用时无限重试      |

---

### 6.3 重启 node1 的 rsyslog

```bash
sudo systemctl restart rsyslog
sudo systemctl status rsyslog
```

如果状态为 `active (running)`，说明配置生效。

---

## 7. 日志转发测试

### 7.1 在 node2 上监听远程日志

在 `node2` 上执行：

```bash
sudo tail -f /var/log/remote/node1.log
```

如果文件暂时不存在，可以先查看目录：

```bash
sudo ls -l /var/log/remote/
```

---

### 7.2 在 node1 上生成测试日志

在 `node1` 上执行：

```bash
logger "hello from node1, this is a rsyslog test"
```

回到 `node2` 的监听窗口，如果能看到类似内容，说明日志转发成功：

```text
hello from node1, this is a rsyslog test
```

---

### 7.3 测试不同级别日志

在 `node1` 上执行：

```bash
logger -p user.info "node1 info log test"
logger -p user.warning "node1 warning log test"
logger -p user.err "node1 error log test"
```

在 `node2` 上查看：

```bash
sudo tail -n 30 /var/log/remote/node1.log
```

如果能看到这些测试日志，说明不同级别的日志都可以正常转发。

---

## 8. 本地日志保留验证

本实验没有修改或删除 node1 默认的本地日志规则，因此 node1 会继续保留本地日志。

在 `node1` 上执行：

```bash
logger "local and remote log test from node1"
```

然后分别查看本地日志和远程日志。

在 `node1` 上查看本地日志：

```bash
sudo journalctl -n 20
```

在 `node2` 上查看远程日志：

```bash
sudo tail -n 20 /var/log/remote/node1.log
```

如果两边都能看到测试日志，说明：

```text
node1 本地保留日志成功
node1 远程转发日志成功
```

---

## 9. 可靠转发测试

### 9.1 模拟 node2 日志服务器不可用

在 `node2` 上停止 rsyslog：

```bash
sudo systemctl stop rsyslog
```

此时 node2 不再接收远程日志。

---

### 9.2 在 node1 上继续生成日志

在 `node1` 上执行：

```bash
logger "test while node2 is down 1"
logger "test while node2 is down 2"
logger "test while node2 is down 3"
```

此时 node2 无法接收日志，但 node1 的 rsyslog 队列会暂存这些日志，并持续尝试重新发送。

---

### 9.3 恢复 node2 日志服务

在 `node2` 上启动 rsyslog：

```bash
sudo systemctl start rsyslog
sudo systemctl status rsyslog
```

---

### 9.4 检查补发结果

在 `node2` 上查看远程日志：

```bash
sudo tail -n 50 /var/log/remote/node1.log
```

如果可以看到：

```text
test while node2 is down 1
test while node2 is down 2
test while node2 is down 3
```

说明 node2 不可用期间产生的日志被 node1 暂存，并在 node2 恢复后成功补发。

---

## 10. 常用排查命令

### 10.1 检查网络连通性

在 `node1` 上执行：

```bash
ping 192.168.83.137
```

---

### 10.2 检查 node2 是否监听 514 端口

在 `node2` 上执行：

```bash
sudo ss -lntp | grep 514
```

---

### 10.3 检查 rsyslog 服务状态

```bash
sudo systemctl status rsyslog
```

---

### 10.4 查看 rsyslog 相关日志

```bash
sudo journalctl -u rsyslog -n 50
```

---

### 10.5 检查 node1 的转发配置

在 `node1` 上执行：

```bash
cat /etc/rsyslog.d/90-forward-to-node2.conf
```

---

### 10.6 检查 node2 的远程接收配置

在 `node2` 上执行：

```bash
cat /etc/rsyslog.d/10-remote.conf
```

---

## 11. 常见问题

### 问题一：node2 没有生成 `/var/log/remote/node1.log`

可能原因：

1. node1 和 node2 网络不通；
2. node2 没有开启 TCP 514 监听；
3. node2 防火墙没有放行 514/tcp；
4. node1 中 target IP 写错；
5. node1 或 node2 的 rsyslog 没有重启。

排查方式：

```bash
ping 192.168.83.137
sudo ss -lntp | grep 514
sudo ufw status
sudo systemctl status rsyslog
sudo journalctl -u rsyslog -n 50
```

---

### 问题二：node1 显示 cannot connect

如果在 `node1` 上查看 rsyslog 状态时看到 `cannot connect`，一般说明 node1 无法连接 node2 的 TCP 514 端口。

重点检查：

```bash
ping 192.168.83.137
```

```bash
sudo ss -lntp | grep 514
```

```bash
sudo ufw status
```

```bash
cat /etc/rsyslog.d/90-forward-to-node2.conf
```

---

### 问题三：node2 停止期间的日志没有补发

可能原因：

1. node1 没有配置队列参数；
2. rsyslog 配置文件语法有误；
3. 队列配置没有生效；
4. node1 的 rsyslog 没有重启；
5. node2 恢复后 TCP 514 仍未监听。

可以在 node1 查看 rsyslog 服务日志：

```bash
sudo journalctl -u rsyslog -n 100
```

---

## 12. 项目总结

本项目完成了一个基础的 Linux 集中式日志收集实验。

通过本项目，实现了以下内容：

1. 使用 node2 作为 rsyslog 日志服务器；
2. 使用 node1 作为日志客户端；
3. node1 将 rsyslog 管理的全部日志转发到 node2；
4. node1 本地仍然保留原有日志；
5. node2 通过 TCP 514 端口接收远程日志；
6. 使用 `logger` 命令完成日志转发测试；
7. 使用 rsyslog 队列机制提高日志转发可靠性；
8. 验证 node2 不可用时，node1 可以暂存日志，并在 node2 恢复后继续转发。

虽然这个实验整体难度不高，但它体现了运维工作中非常常见的一类场景：集中化日志管理。

在真实环境中，服务器数量较多时，管理员通常不会逐台登录服务器查看日志，而是会将日志统一收集到日志服务器或日志平台中，再结合 ELK、Loki、Zabbix、Prometheus Alertmanager 等工具进行检索、分析和告警。

本实验可以作为后续学习日志采集、监控告警和自动化运维的基础。

---
