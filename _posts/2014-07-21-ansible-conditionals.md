---
layout: post
title: "ansible学习之七：Conditionals"
date: 2014-07-21 16:35:00 +0800
categories: ansible
---


{% raw %}

这节讲如何控制playbook的执行流，记得前面说过playbook和模板文件中可以使用`jinja2`语法么，这节就会大量用到了。

首先，要讲下`setup`这个模块，作用类似salt的`grains`，用于获取远程服务器的信息（以一个字典返回），这些获取到的信息在`template`模块定义的模板文件和playbook文件中可以直接使用，该模块获取的结果又叫`facts`：

```bash
$ ansible 127.0.0.1 -m setup
127.0.0.1 | success >> {
    "ansible_facts": {
        "ansible_all_ipv4_addresses": [
            "192.168.1.5"
        ], 
        "ansible_all_ipv6_addresses": [
            "fe80::a00:27ff:fead:a141"
        ], 
        "ansible_architecture": "i386", 
        "ansible_bios_date": "12/01/2006", 
        "ansible_bios_version": "VirtualBox", 
        "ansible_cmdline": {
            "KEYBOARDTYPE": "pc", 
            "KEYTABLE": "us", 
            "LANG": "en_US.UTF-8", 
            "SYSFONT": "latarcyrheb-sun16", 
            "quiet": true, 
            "rd_NO_DM": true, 
            "rd_NO_LUKS": true, 
            "rd_NO_LVM": true, 
            "rd_NO_MD": true, 
            "rhgb": true, 
            "ro": true, 
            "root": "UUID=ebfa916e-fc91-4f0e-bb2d-7f7c4e48aeca"
        }, 
        ...
        "ansible_userspace_architecture": "i386", 
        "ansible_userspace_bits": "32", 
        "ansible_virtualization_role": "guest", 
        "ansible_virtualization_type": "virtualbox", 
        "module_setup": true
    }, 
    "changed": false
}
```
在模板文件中通过`{{ var }}`来使用这些获取到的信息，假如我有一个模板文件`os.j2`：

```jinja
{% if ansible_distribution == "FreeBSD" %}
FreeBSD
{% elif ansible_distribution == "CentOS" %}
CentOS
{% endif %}

hostname: {{ ansible_hostname }}
```


<br />

### when语句
when语句的作用是只有匹配指定条件，才执行task，when后面使用`jinja2`的表达方式

例子：只关闭操作系统为`Debian`的服务器

```yaml
- hosts: all
  tasks:
    - name: "shutdown Debian flavored systems"
      command: /sbin/shutdown -t now          #只有Debian系统才会执行该command
      when: ansible_os_family == "Debian"     #这里ansible_os_family变量就是通过setup模块获取的
```

jiaja2模板的过滤器也可以使用，ansible也提供了一些自己的过滤器：

```yaml
tasks:
  - name: test
    shell: ps aux|grep nginx|grep -v grep|wc -l
    register: cmd_result
    failed_when: cmd_result.stdout|int == 0       #使用jinja2的"int"过滤器

  - command: /bin/false
    register: result
    ignore_errors: True          #忽略该task的错误
  - command: /bin/something
    when: result|failed          #通过结果判断上一个task如果执行失败，则执行该task
  - command: /bin/something_else
    when: result|success         #通过结果判断上一个task如果执行成功，则执行该task
  - command: /bin/still/something_else
    when: result|skipped          
```

自定义变量：

```yaml
vars:
  epic: true
tasks:
  - shell: echo "I've got '{{ epic }}' and am not afraid to use it!"
    when: epic is defined          #变量已定义

  - fail: msg="Bailing out: this play requires 'epic'"
    when: epic is not defined          #变量未定义

  - shell: echo "This certainly is epic!"
    when: epic          #变量值为真

  - shell: echo "This certainly isn't epic!"
    when: not epic          #变量值为假
```

当when用于循环中时，是对列表中的每一项都进行检查：

```yaml
tasks:
    - command: echo {{ item }}
      with_items: [ 0, 2, 4, 6, 8, 10 ]
      when: item > 5
```

在`roles`和`include`中使用when指令，<b>when不能用在包含一个playbook文件上，且当包含一个task文件时，会对task文件中的每个task都会使用一次when判断</b>：

```yaml
- hosts: all
  tasks:
    - include: tasks/sometasks.yml
      when: "'reticulating splines' in output"
  roles:
     - { role: debian_stock_config, when: ansible_os_family == 'Debian' }          #首先判断远程主机是否是Debian，是的话才会导入这个role
```
当执行这个play的时候，输出中可能会出现很多的`skipped`，这些就是经过when的条件判断不符合，跳过执行的输出。


条件导入：

```yaml
---
- hosts: all
  remote_user: root
  vars_files:     #用于导入变量文件
    - [ "vars/{{ ansible_os_family }}.yml", "vars/os_defaults.yml" ]
  tasks:
  - name: make sure apache is running
    service: name={{ apache }} state=running
```
根据不同的系统导入不同变量文件，如果是CentOS系统则首先导入`vars/CentOS.yml`文件（Debian系统则会首先导入`vars/Debian.yml`），如果该文件不存在则导入`vars/os_defaults.yml`文件，如果两个文件都不存在则生成一个错误，`vars/CentOS.yml`文件内容如下：

```yaml
---
# for vars/CentOS.yml
apache: httpd
somethingelse: 42
```


基于变量选择文件或模板：

```yaml
- name: template a file
  template: src={{ item }} dest=/etc/myapp/foo.conf
  with_first_found:
    - files:
       - {{ ansible_distribution }}.conf   #CentOS系统会使用CentOS.conf文件，Debian系统会使用Debian.conf文件
       - default.conf
      paths:
       - search_location_one/somedir/
       - /opt/other_location/somedir/
```


<br />

### register
`register`用于注册一个变量，保存命令的结果(shell或command模块)，这个变量可以在后面的`task`、`when`语句或模板文件中使用，该指令用在循环中会有不同，请看[ansible学习之八：Loops]({% post_url 2014-07-21-ansible-loops %})中关于`register`的讲解

```yaml
- shell: /bin/pwd
  register: pwd_result
```
此时变量`pwd_result`的结果为：

```python
{
    u'changed': True, 
    u'end': u'2014-02-23 12:02:51.982893', 
    u'cmd': [u'/bin/pwd'], 
    u'start': u'2014-02-23 12:02:51.980191', 
    u'delta': u'0:00:00.002702', 
    u'stderr': u'', 
    u'rc': 0,           #这个就是命令返回状态，非0表示执行失败
    'invocation': {'module_name': 'command', 'module_args': '/bin/pwd'}, 
    u'stdout': u'/home/sapser',    #以一个字符串保存命令结果
    'stdout_lines': [u'/home/sapser']     #以列表保存命令结果
}
```
在随后的task中使用该变量：

```yaml
- debug: msg="{{pwd_result}}"
  when: pwd_result.rc == 0
```
循环处理命令结果：

```yaml
- name: registered variable usage as a with_items list
  hosts: all
  tasks:
      - name: retrieve the list of home directories
        command: ls /home
        register: home_dirs

      - name: add home dirs to the backup spooler
        file: path=/mnt/bkspool/{{ item }} src=/home/{{ item }} state=link
        with_items: home_dirs.stdout_lines       #等同于with_items: home_dirs.stdout.split()
```



{% endraw %}
