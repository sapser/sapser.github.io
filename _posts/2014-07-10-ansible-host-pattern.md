---
layout: post
title: "ansible学习之三：Host Patterns"
date: 2014-07-10 22:07:00 +0800
categories: ansible
---

ansible可以使用多种`host patterns`来指定远程主机，用在如下两个地方：
<ul>
<li>命令行<code>ansible [host patterns] -m module -a arguments</code></li>
<li>playbook中通过<code>- hosts: [host patterns]</code>来指定要执行该play的远程主机</li>
</ul>


<br />
先在hosts文件定义几个主机：

```
12 ansible_ssh_host=192.168.1.12
13 ansible_ssh_host=192.168.1.13
14 ansible_ssh_host=192.168.1.14
```


<br />
#### 匹配hosts中所有主机
```bash
ansible all -m ping
或
ansible '*' -m ping
```
  
<br />
#### 指定单个组或主机
在hosts文件中添加一个组：

```bash
[test1]
12
13
```
执行命令：

```bash
[sapser@centos6 ansible]$ ansible 12 -m ping       #只匹配12这台主机
12 | success >> {
    "changed": false,
    "ping": "pong"
}

[sapser@centos6 ansible]$ ansible test1 -m ping        #匹配test1组内的所有主机
12 | success >> {
    "changed": false,
    "ping": "pong"
}
13 | success >> {
    "changed": false,
    "ping": "pong"
}
```


<br />
#### 多个用`:`隔开的组，表示匹配这些组中的所有主机
在hosts文件中配置两个组：

```bash
[test1]
12
13
[test2]
13
14
```
执行命令：

```bash
[sapser@centos6 ansible]$ ansible "test1:test2" -m ping               #同时属于多个组的主机只会执行一次
14 | success >> {
    "changed": false,
    "ping": "pong"
}
12 | success >> {
    "changed": false,
    "ping": "pong"
}
13 | success >> {
    "changed": false,
    "ping": "pong"
}
```


<br />
#### 组前面加上`!`表示排除这个组中的主机
在hosts文件中配置两个组：

```
[test1]
12
13
14
[test2]
13
```
执行命令：

```bash
[sapser@centos6 ansible]$ ansible 'test1:!test2' -m ping          #匹配所有在test1组却不在test2组中的主机
14 | success >> {
    "changed": false,
    "ping": "pong"
}
12 | success >> {
    "changed": false,
    "ping": "pong"
}
```


<br />
#### `&`表示求组的交集
在hosts文件配置两个组：

```
[test1]
12
13
[test2]
13
14
```
执行命令：

```bash
[sapser@centos6 ansible]$ ansible "test1:&test2" -m ping          #匹配同时在test1和test2组中的主机
13 | success >> {
    "changed": false,
    "ping": "pong"
}
```
一个复杂的匹配：  
`webservers:dbservers:&staging:!phoenix`  
首先取出属于webservers组和dbservers组的所有主机，取出这些主机中同时也属于staging组的那部分，然后再去掉不属于phoenix组的那些


<br />
#### 使用通配符
```bash
[sapser@centos6 ansible]$ ansible '1?' -m ping          #匹配"1"开头再接一个任意字符的主机或组
12 | success >> {
    "changed": false,
    "ping": "pong"
}
13 | success >> {
    "changed": false,
    "ping": "pong"
}
14 | success >> {
    "changed": false,
    "ping": "pong"
}
```


<br />
#### 以`~`开头表示使用正则表达式
在hosts文件配置两个组：

```
[test1]
12
13
[test2]
13
14
```
执行命令：

```bash
[sapser@centos6 ansible]$ ansible '~test.*' -m ping       #这里匹配到了test1和test2组
12 | success >> {
    "changed": false,
    "ping": "pong"
}
14 | success >> {
    "changed": false,
    "ping": "pong"
}
13 | success >> {
    "changed": false,
    "ping": "pong"
}
```


<br />
同时要注意，`ansible`和`ansible-playbook`命令还提供了`-l,--limit`参数，对上面匹配出的结果会再进行一次过滤：

```bash
$ ansible '12:14:146' -m ping -l '14*'
146 | success >> {
    "changed": false,
    "ping": "pong"
}

14 | success >> {
    "changed": false,
    "ping": "pong"
}
```


