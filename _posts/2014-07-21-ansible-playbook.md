---
layout: post
title: "ansible学习之四：Playbooks"
date: 2014-07-21 14:54:00 +0800
categories: ansible
---

ansible的playbook就如同salt的state，一个playbook就是一个普通文件，使用[YAML](http://docs.ansible.com/YAMLSyntax.html)格式，所以playbook文件一般都以`.yml`结尾，playbook使用的`YAML`语法很简单，不用特意去学。此外playbook和模板文件（`template`模块）还可以使用[jinja2语法](http://jinja.pocoo.org/)解决一些复杂问题。 

一个playbook文件由一个或多个`play`组成，每个`play`定义了在一个或多个远程主机上执行的一系列的`task`，其中每个`task`一般就是调用一个ansible的模块，如调用`copy`模块复制文件到远程主机或调用`shell`模块执行命令。

例子：只带有一个`play`的playbook文件`test.yml`：

```yaml
---               #任何playbook文件(其实就是yaml文件)都要以这个开头
- hosts: webservers         #对webservers主机组下的所有主机进行操作
  vars:           #为该play定义两个变量
    http_port: 80
    max_clients: 200
  remote_user: sapser       #连接到远程主机的用户
  sudo: yes       #以sudo模式运行该play
  sudo_user: root           #sudo到哪个用户，默认为root，如果sudo到该用户需要密码，则在执行ansible-playbook的时候指定-K选项来输入sudo密码
  tasks:          #开始定义task
  - name: ensure apache is at the latest version            #这既是每个task的说明也是每个task的名字
    yum: pkg=httpd state=latest    
    tags:         #给该task打一个标签
      - last_http
  - name: write the apache config file
    template: src=/srv/httpd.j2 dest=/etc/httpd.conf
    notify:       #提供watch功能，这里当apache配置文件改变时，就调用handlers中名为"restart apache"的task来重启apache
    - restart apache
  - name: ensure apache is running
    service: name=httpd state=started
  handlers:       #notify通知这里的task执行，谨记：定义在handlers下的task只有在notify触发的时候才会执行
    - name: restart apache
      service: name=httpd state=restarted
```
执行该playbook文件：

```bash
$ ansible-playbook test.yml
```
就是这么简单，然后就是等待ansible去`webservers`组下的所有远程主机执行上面定义的`tasks`，输出也非常容易看懂，通过红黄绿三种颜色标明了不同的执行结果，红色表示有`task`执行失败，黄色表示改变了远程主机状态。

playbook中的每个`play`首先要定义的就是该`play`要在哪些主机上执行，如上面示例的

```yaml
---
- hosts: webservers
```
可以指定单个主机、单个组或以`:`分割的多个主机或组，具体见[ansible学习之三：Host Patterns]({% post_url 2014-07-10-ansible-host-pattern %})

`sudo: yes`表示远程主机要以sudo权限执行`tasks`，不光可以对整个`play`生效，还可以针对单个`task`生效（`remote_user`指令也一样）:

```yaml
---
- hosts: webservers
  remote_user: root
  tasks:
    - shell: cmd
      remote_user: user1

    - service: name=nginx state=started
      sudo: yes       #只sudo运行该task
      sudo_user: user2     #sudo到user2用户，如果sudo需要密码记得为ansible-playbook命令提供"-K"选项
```

`tasks`由一系列`task`组成，多个`task`按照顺序执行，<b>默认所有匹配主机都执行完一个`task`后才会移动到下一个`task`执行，默认如果所有主机都执行某个`task`失败，则ansible会中断执行流程并打印错误消息</b>，这里都是说的默认，也就是说这些行为都是可自定义的，以后会讲到这些主题。

上面提到过一个`task`其实就是执行一个模块，ansible中的模块是幂等的(idempotent)，也就是说多次执行同一个task，只有在状态发生改变后才会真的去执行，通过下面的例子讲解：

```yaml
tasks:
  - name: make sure apache is running
    service: name=httpd state=started
```
该task用于保证apache服务器处于运行状态，如果我多次执行该task且apache一直处于运行状态的话，则该task其实什么都不会做。这样做的好处就是你可以随意执行你的playbook文件，不用担心改变远程主机上的内容(除非状态有变化必须要修改)。这里要特别提一下`command`和`shell`模块，这两个模块都是用于在远程主机上执行命令，每次调用这两个模块都会重新执行一遍命令，一般来说不会有什么影响，不过ansible也提供了`creates`参数来将这两个模块实现idempotent。

`handlers`是和`notify`指令搭配使用的，作用是当一个task执行修改了远程主机状态时就通知(notify)一个`handlers`中的task执行，一般用在修改了远程服务的配置文件然后调用`handlers`中的对应task去重启该服务，例子如下:

```yaml
tasks:
  - name: template configuration file
    template: src=template.j2 dest=/etc/foo.conf
    notify:
      - restart memcached
      - restart apache
handlers:
  - name: restart memcached
    service:  name=memcached state=restarted
  - name: restart apache
    service: name=apache state=restarted
```
这里只有模板文件`template.j2`发生了变化，也就是真正修改了远程主机的`/etc/foo.conf`文件后，才会触发`handlers`中的两个task执行，其他情况`handlers`中的task都是不会运行的。

如果一行太长可以分开多行写：

```yaml
tasks:
  - name: Copy ansible inventory file to client
    copy: src=/etc/ansible/hosts dest=/etc/ansible/hosts
            owner=root group=root mode=0644
```

<br />
最后附上ansible-playbook命令参数：

```bash
Usage: ansible-playbook playbook.yml

Options:
  -u REMOTE_USER, --user=REMOTE_USER           ssh连接的用户名，ansible.cfg中可以配置
  -k, --ask-pass        提示输入ssh登录密码，当使用密码验证登录的时候用
  -s, --sudo            sudo运行
  -U SUDO_USER, --sudo-user=SUDO_USER          sudo到哪个用户，默认为root
  -K, --ask-sudo-pass   提示输入sudo密码，当不是NOPASSWD模式时使用
  -T TIMEOUT, --timeout=TIMEOUT                ssh连接超时时间，默认10秒
  -C, --check           指定该参数后，执行playbook文件不会真正去执行，而是模拟执行一遍，然后输出本次执行会对远程主机造成的修改
  -c CONNECTION         连接类型(default=smart)
  -D, --diff            如果file和template模块改变，会显示改变的内容，应该和--check一起
  -e EXTRA_VARS, --extra-vars=EXTRA_VARS       设置额外的变量如：key=value or YAML/JSON，以空格分隔变量，或用多个-e
  -f FORKS, --forks=FORKS                      fork多少个进程并发处理，默认5
  -i INVENTORY, --inventory-file=INVENTORY     指定hosts文件路径，默认default=/etc/ansible/hosts
  -l SUBSET, --limit=SUBSET       指定一个pattern，对- hosts:匹配到的主机再过滤一次
  --list-hosts          只打印有哪些主机会执行这个playbook文件，不是实际执行该playbook
  --list-tasks          列出该playbook中会被执行的task
  -M MODULE_PATH, --module-path=MODULE_PATH    模块所在路径，默认/usr/share/ansible/
  --private-key=PRIVATE_KEY_FILE      私钥路径
  --start-at-task=START_AT            start the playbook at the task matching this name
  --step                同一时间只执行一个task，每个task执行前都会提示确认一遍
  --syntax-check        只检测playbook文件语法是否有问题，不会执行该playbook
  -t TAGS, --tags=TAGS  当play和task的tag为该参数指定的值时才执行，多个tag以逗号分隔
  --skip-tags=SKIP_TAGS 当play和task的tag不匹配该参数指定的值时，才执行
  -v, --verbose         verbose mode (-vvv for more, -vvvv to enable connection debugging)
```
