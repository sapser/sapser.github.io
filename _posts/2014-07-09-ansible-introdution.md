---
layout: post
title: "ansible学习之一：Getting Started"
date: 2014-07-09 14:47:00 +0800
categories: ansible
---

#### 介绍

<br />
#### 安装
#### 远程连接
从1.3版本开始，ansible默认使用原生openssh来连接远程主机，并开启`ControlPersist`指令优化连接速度和Kerberos。如果使用的是RHEL系统及其衍生系统如CentOS，openssh的版本太旧，不支持`ControlPersist`（openssh5.6版本才开始增加`ControlPersist`指令），这时候ansible就会选择使用paramiko来连接远程主机。

ansible支持密码验证和私钥验证，默认是使用私钥验证。如果想使用密码验证，则需要为`ansible`和`ansible-playbook`提供`-k, --ask-pass`参数，连接远程主机是会提示你输入密码，默认使用root用户登录，可以通过`-u, --usr`参数修改，或在ansible.cfg配置文件中定义。

ansible支持私钥验证，但是如果你私钥有密码的话，ansible不会帮你自动填密码，这时候就可以使用`ssh-agent`和`ssh-add`命令来自动填私钥密码：

```bash
[sapser@centos6 ansible]$ ssh-agent bash
[sapser@centos6 ansible]$ ssh-add ~/.ssh/id_rsa
Enter passphrase for /home/sapser/.ssh/id_rsa:
Identity added: /home/sapser/.ssh/id_rsa (/home/sapser/.ssh/id_rsa)
```

还可以将如下

```bash
if [ -f ~/.agent.env ]; then
        . ~/.agent.env >/dev/null
        if ! kill -s 0 $SSH_AGENT_PID >/dev/null 2>&1; then
                echo "Stale agent file found. Spawning new agent..."
                eval `ssh-agent |tee ~/.agent.env`
                ssh-add
        fi
else
        echo "Starting ssh-agent..."
        eval `ssh-agent |tee ~/.agent.env`
        ssh-add
fi
```
