---
layout: post
title: "ansible学习之一：Getting Started"
date: 2014-07-09 14:47:00 +0800
categories: ansible
---

#### 安装
ansible是什么就不讲了，简单说下安装  
安装最新版本：

```bash
pip install ansible
```
安装指定版本：

```bash
pip install ansible==1.6.3
```
这里我遇到一个问题，CentOS6.5安装ansible1.6.3后，执行`ansible`命令一直提示：  
`PowmInsecureWarning: Not using mpz_powm_sec.  You should rebuild using libgmp >= 5 to avoid timing attack vulnerability.`  
然后通过源码安装gmp-6.0.0包到`/usr`目录下，并重新安装pycrypto包后解决。


<br />
#### 远程连接
从1.3版本开始，ansible默认使用原生openssh来连接远程主机，并开启`ControlPersist`指令优化连接速度和Kerberos。如果使用的是RHEL系统及其衍生系统如CentOS，openssh的版本太旧，不支持`ControlPersist`（openssh5.6版本才开始增加`ControlPersist`指令），这时候ansible就会选择使用paramiko来连接远程主机。

ansible支持密码验证和私钥验证，默认是使用私钥验证。如果想使用密码验证，则需要为`ansible`和`ansible-playbook`命令提供`-k, --ask-pass`参数，连接远程主机时会提示你输入用户密码。


<br />
#### 第一个命令
在/etc/ansible/hosts中添加一个远程主机`172.16.9.141`，/etc/ansible/目录不存在就自己创建。  
如果你使用私钥验证，执行如下命令：

```bash
[sapser@centos6 ~]$ ansible all -m ping
Enter passphrase for key '/home/sapser/.ssh/id_rsa': 
172.16.9.141 | success >> {
    "changed": false, 
    "ping": "pong"
}
```
如果你使用密码验证，则还需要加上`-k`参数来转换到密码验证模式，默认远程连接使用你所在的当前用户，可以使用`-u`参数指定用什么用户连接。

ansible支持私钥验证，但是如果你私钥有密码的话，ansible不会帮你自动填密码，如上面例子每次执行都需要手动输入一次密码。可以使用`ssh-agent`和`ssh-add`命令来自动填私钥密码：

```bash
[sapser@centos6 ~]$ ssh-agent bash
[sapser@centos6 ~]$ ssh-add ~/.ssh/id_rsa
Enter passphrase for /home/sapser/.ssh/id_rsa: 
Identity added: /home/sapser/.ssh/id_rsa (/home/sapser/.ssh/id_rsa)
[sapser@centos6 ~]$ ansible all -m ping      #此时就不需要再手动输入私钥密码
172.16.9.141 | success >> {
    "changed": false, 
    "ping": "pong"
}
```
还可以将下面这段代码加入到`~/.bashrc`中，这样无须每次打开终端都要先执行`ssh-agent`和`ssh-add`：

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
一般建议服务器都设置成私钥验证，比密码验证安全性高很多，私钥属于每个人独有，完全不用担心别人用自己的账号登陆到服务器做坏事。

差不多基本配置完成，可以连接到远程主机执行各种各样的操作，ansible提供了大量的模块用来完成不同的任务，`ansible-doc -l`列出所有可用模块，`ansible-doc shell`查看`shell`模块文档，如利用`shell`模块在远程主机执行命令：

```bash
[sapser@centos6 ~]$ ansible all -m shell -a "uname -a"
172.16.9.141 | success | rc=0 >>
Linux centos6 2.6.32-431.el6.x86_64 #1 SMP Fri Nov 22 03:15:09 UTC 2013 x86_64 x86_64 x86_64 GNU/Linux
```

<br />
最后附上ansible命令参数：

```bash
Usage: ansible <host-pattern> [options]
Options:
  -m MODULE_NAME, --module-name=MODULE_NAME         要执行的模块，默认为command
  -a MODULE_ARGS, --args=MODULE_ARGS                模块的参数
  -u REMOTE_USER, --user=REMOTE_USER                ssh连接的用户名，默认用root，ansible.cfg中可以配置
  -k, --ask-pass        提示输入ssh登录密码，当使用密码验证登录的时候用
  -s, --sudo            sudo运行
  -U SUDO_USER, --sudo-user=SUDO_USER               sudo到哪个用户，默认为root
  -K, --ask-sudo-pass   提示输入sudo密码，当不是NOPASSWD模式时使用
  -B SECONDS, --background=SECONDS                  run asynchronously, failing after X seconds(default=N/A)
  -P POLL_INTERVAL, --poll=POLL_INTERVAL            set the poll interval if using -B (default=15)
  -C, --check           只是测试一下会改变什么内容，不会真正去执行
  -c CONNECTION         连接类型(default=smart)
  -f FORKS, --forks=FORKS            fork多少个进程并发处理，默认5
  -i INVENTORY, --inventory-file=INVENTORY          指定hosts文件路径，默认default=/etc/ansible/hosts
  -l SUBSET, --limit=SUBSET          指定一个pattern，对<host_pattern>已经匹配的主机中再过滤一次
  --list-hosts          只打印有哪些主机会执行这个playbook文件，不是实际执行该playboo
  -M MODULE_PATH, --module-path=MODULE_PATH         要执行的模块的路径，默认为/usr/share/ansible/
  -o, --one-line        压缩输出，摘要输出
  --private-key=PRIVATE_KEY_FILE     私钥路径
  -T TIMEOUT, --timeout=TIMEOUT      ssh连接超时时间，默认10秒
  -t TREE, --tree=TREE  日志输出到该目录，日志文件名会以主机名命名
  -v, --verbose         verbose mode (-vvv for more, -vvvv to enable connection debugging)
```
