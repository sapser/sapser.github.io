---
layout: post
title: "ansible学习三：HOST PATTERN"
date: 2014-07-05 22:07:00
categories: ansible
---

ansible可以使用多种`host pattern`来指定远程主机，用在如在两个地方：
- 命令行`ansible [host pattern] -m module -a arguments`
- playbook中通过`- hosts: [host pattern]`来指定要执行该play的远程主机

### 匹配hosts中所有主机
```
ansible all -m ping
ansible '*' -m ping
```

### 指定单个组或主机
```
[root@centos6 ansible]# ansible 12 -m ping
12 | success >> {
    "changed": false,
    "ping": "pong"
}

[root@centos6 ansible]# ansible test1 -m ping
12 | success >> {
    "changed": false,
    "ping": "pong"
}
13 | success >> {
    "changed": false,
    "ping": "pong"
}
```

### 多个用`:`隔开的组，表示匹配这些组中的任一主机
指定两个组：
```
[test1]
12
13
[test2]
13
14
```
执行命令：
```
[root@centos6 ansible]# ansible "test1:test2" -m ping               #同时属于多个组的主机只会执行一次
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

### 组前面加上`!`表示排除这个组中的主机
指定两个组：
```
[test1]
12
13
14
[test2]
13
```
执行命令：
```
[root@centos6 ansible]# ansible 'test1:!test2' -m ping          #匹配所有在test1组却不在test2组中的主机
14 | success >> {
    "changed": false,
    "ping": "pong"
}
12 | success >> {
    "changed": false,
    "ping": "pong"
}
```

### `&`表示求交集
先在hosts文件指定两个组：
```
[test1]
12
13
[test2]
13
14
```
然后执行命令：
```
[root@centos6 ansible]# ansible "test1:&test2" -m ping          #匹配同时在test1和test2组中的主机
13 | success >> {
    "changed": false,
    "ping": "pong"
}
```

一个复杂的匹配：
`webservers:dbservers:&staging:!phoenix`  
首先取出属于webservers组和dbservers组的所有主机，然后取出这些主机中同时也属于staging组的那部分，然后再去掉不属于phoenix组的那些

### 使用通配符
```
[root@centos6 ansible]# ansible '14?' -m ping          #通配符中"?"表示匹配任意一个字符
145 | success >> {
    "changed": false,
    "ping": "pong"
}
146 | success >> {
    "changed": false,
    "ping": "pong"
}
148 | success >> {
    "changed": false,
    "ping": "pong"
}
```

### 以`~`开头表示使用正则表达式
```
[root@centos6 ansible]# ansible '~test.*' -m ping
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




