---
layout: post
title:  "ansible学习之二：Inventory"
date:   2014-07-10 13:37:38
categories: ansible
---

ansible可以到多个远程主机执行任务，这些主机信息默认保存在/etc/ansible/hosts文件，可以通过配置文件ansible.cfg的`hostfile`指令修改该默认路径，关于配置文件ansible.cfg不会单独讲，具体请看官方文档：[The Ansible Configuration File](http://docs.ansible.com/intro_configuration.html)

<br />

hosts文件保存所有远程主机的信息，其中大括号表示一个主机组，一个组可以包含一个或多个主机，当你操作一个组的时候也就表示操作该组下的所有主机：

```bash
192.168.1.1
192.168.1.2

[webserver]
192.168.1.3:32222          #额外指定连接端口，默认使用22端口
192.168.1.4

[dbserver]
192.168.1.5
jumper ansible_ssh_port=5555 ansible_ssh_host=192.168.1.50          #为一个主机指定别名
```


在hosts文件中使用通配符：

```bash
[webservers]
192.168.1.[1:50]      #匹配192.168.1.1~192.168.1.50

[databases]
db-[a:f].example.com
```


为每个主机指定连接类型和连接用户：

```bash
[targets]
localhost              ansible_connection=local
other1.example.com     ansible_connection=ssh        ansible_ssh_user=mpdehaan
other2.example.com     ansible_connection=ssh        ansible_ssh_user=mdehaan
```


可以为每个主机单独指定一些变量，这些变量随后可以在playbooks中使用：

```bash
[atlanta]
host1 http_port=80 maxRequestsPerChild=808
host2 http_port=303 maxRequestsPerChild=909
```


也可以为一个组指定变量，组内每个主机都可以使用该变量：

```bash
[atlanta]
host1
host2

[atlanta:vars]
ntp_server=ntp.atlanta.example.com
proxy=proxy.atlanta.example.com
```


组可以包含其他组：

```bash
[atlanta]
host1
host2

[raleigh]
host2
host3

[southeast:children]
atlanta
raleigh

[southeast:vars]          
some_server=foo.southeast.example.com
halon_system_timeout=30
self_destruct_countdown=60
escape_pods=2

[usa:children]
southeast
northeast
southwest
northwest
```


<br />

为host和group定义一些比较复杂的变量时（如array、hash），在hosts文件中不好实现同时也造成hosts文件臃肿不美观，这时可以用单独文件保存host和group变量，以`YAML`格式书写变量，如hosts文件路径为：

```
/etc/ansible/hosts
```
则host和group变量目录结构：

```
/etc/ansible/host_vars/all           #host_vars目录用于存放host变量，all文件对所有主机有效
/etc/ansible/group_vars/all          #group_vars目录用于存放group变量，all文件对所有组有效
/etc/ansible/host_vars/foosball      #文件foosball要和hosts里面定义的主机名一样，表示只对foosball主机有效
/etc/ansible/group_vars/raleigh      #文件raleigh要和hosts里面定义的组名一样，表示对raleigh组下的所有主机有效
```

这里/etc/ansible/group_vars/raleigh格式如下：

```yaml
---     #YAML格式要求
ntp_server: acme.example.org          #变量名:变量值
database_server: storage.example.org
```


<br />

hosts文件支持一些特定指令，上面已经使用了其中几个，所有支持的指令如下：

```
ansible_ssh_host：指定主机别名对应的真实IP，如：251  ansible_ssh_host=183.60.41.251，随后连接该主机无须指定完整IP，只需指定251就行
ansible_ssh_port：指定连接到这个主机的ssh端口，默认22
ansible_ssh_user：连接到该主机的ssh用户
ansible_ssh_pass：连接到该主机的ssh密码（连-k选项都省了），安全考虑还是建议使用私钥或在命令行指定-k选项输入
ansible_sudo_pass：sudo密码
ansible_connection：连接类型，可以是local、ssh或paramiko，ansible1.2之前默认为paramiko
ansible_ssh_private_key_file：私钥文件路径
ansible_shell_type：目标系统的shell类型，默认为sh
ansible_python_interpreter：python解释器路径，默认是/usr/bin/python，但是如要要连*BSD系统的话，就需要该指令修改python路径
ansible_*_interpreter：这里的"*"可以是ruby或perl或其他语言的解释器，作用和ansible_python_interpreter类似
```

例子：

```bash
some_host         ansible_ssh_port=2222     ansible_ssh_user=manager
aws_host          ansible_ssh_private_key_file=/home/example/.ssh/aws.pem
freebsd_host      ansible_python_interpreter=/usr/local/bin/python
ruby_module_host  ansible_ruby_interpreter=/usr/bin/ruby.1.9.3
```
