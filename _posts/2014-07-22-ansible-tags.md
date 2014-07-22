---
layout: post
title: "ansible学习之九：Tags"
date: 2014-07-22 13:16:00 +0800
categories: ansible
---


ansible中可以对`play`、`role`、`include`、`task`打一个tag(标签)，然后：
<ul>
<li>当命令ansible-playbook有`-t`参数时，只会执行`-t`指定的tag</li>
<li>当命令ansible-playbook有`--skip-tags`参数时，则除了`--skip-tags`指定的tag外，执行其他所有</li>
</ul>


### 基本应用

```yaml
---
- hosts: 192.168.0.105
  tags:
    - one              #为一个play打tag
  tasks:
    - name: exec ifconfig
      command: /sbin/ifconfig
      tags:
        - exec_ifconfig             #为一个task打tag
```


### 对include语句打tag

```yaml
---
- hosts: 192.168.0.105
  gather_facts: no
  tasks:
    - include: foo.yml
      tags:      #方式一
        - one
    - include: bar.yml tags=two    #方式二
```
上面两种打tag的方式都可以，此时：

```bash
ansible-playbook test.yml          #执行foo.yml和bar.yml
ansible-playbook test.yml -t one   #只执行foo.yml
ansible-playbook test.yml -t two   #只执行bar.yml
```


### 同一对象可以打多个tag

```yaml
---
- hosts: 192.168.0.105
  gather_facts: no
  tasks:
    - include: foo.yml tags=one,two
    - include: bar.yml tags=two
```
可以对同一对象打多个tag，只要`-t`指定了其中一个tag，就会执行该对象：

```bash
ansible-playbook test.yml          #执行foo.yml和bar.yml
ansible-playbook test.yml -t one          #只执行foo.yml
ansible-playbook test.yml -t two          #因为两个include都有two这个标签，所以foo.yml和bar.yml都会执行
```


### 对roles打tag

```yaml
---
- hosts: 192.168.0.105
  gather_facts: no
  roles:
    - {role: foo, tags: one}
    - {role: bar, tags: [one,two]}          #同时对bar打两个tag
```
则：

``` bash
ansible-playbook test.yml          #执行foo和bar两个role
ansible-playbook test.yml -t two   #只执行bar
ansible-playbook test.yml -t one   #两个role都有one标签，所以foo和bar都会执行
```


### 可以对多个对象打同一个tag

```yaml
---
- hosts: 192.168.0.105
  gather_facts: no
  tasks:
    - name: exec ifconfig
      command: /sbin/ifconfig
      tags:
        - exec_cmd

    - name: exec ls
      command: /bin/ls
      tags:
        - exec_cmd

    - name: debug_test one
      debug: msg="test1"
      tags:
        - debug_test

    - name: debug_test two
      debug: msg="test2"
      tags:
        - debug_test
```
如上面，可以把执行command模块的task都打上`exec_cmd`标签，则：

```bash
ansible-playbook test1.yml          #执行所有tasks
ansible-playbook test1.yml -t exec_cmd          #只执行tag为exec_cmd的tasks
```

