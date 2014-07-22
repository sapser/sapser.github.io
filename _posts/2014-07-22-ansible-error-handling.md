---
layout: post
title: "ansible学习之十：Error Handling In Playbooks"
date: 2014-07-22 14:00:00 +0800
categories: ansible
---


ansible默认会检查命令和模块的返回状态，并进行相应的错误处理，通常遇到错误就中断playbook的执行，这些默认行为都是可以改变的。


<br />
#### 忽略错误：

```yaml
- name: this will not be counted as a failure
  command: /bin/false
  ignore_errors: yes
```
该command返回非零状态码，也不会处理


<br />
#### 不依靠返回状态码，而是自定义错误判定条件

```yaml
- name: this command prints FAILED when it fails
  command: /usr/bin/example-command -x -y -z
  register: command_result
  failed_when: "'FAILED' in command_result.stderr"
```
这个例子中，命令不依赖返回状态码来判定是否执行失败，而是要查看命令返回内容是否有`FAILED`字符串，如果包含此字符串则判定命令执行失败。

实际中的应用：

```yaml
---
- hosts: 192.168.0.105
  gather_facts: no
  sudo: yes
  sudo_user: coc
  tasks:
    - shell: ps -U coc|grep _coc_game|wc -l        #计算指定进程数，如果有三个进程则为启动成功
      register: cmd_result
      failed_when: "'3' not in cmd_result.stdout"      #如果进程数量不为3，则表示启动失败
      #failed_when: cmd_result.stdout != 3             #经过测试，用jinja2语法写也是可以的
```


<br />
#### ansible会自动判断模块执行状态，command、shell及其他模块如果修改了远程主机状态则被判定为`changed`状态，不过可以自己决定达到`changed`状态的条件

```yaml
tasks:
  - shell: /usr/bin/billybass --mode="take me to the river"
    register: bass_result
    changed_when: "bass_result.rc != 2"    #命令返回状态码不为2，则为changed状态

  # this will never report 'changed' status
  - shell: wall 'beep'
    changed_when: False      #永远也不会达到changed状态
```

