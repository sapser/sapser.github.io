---
layout: post
title: "ansible学习之五：Roles and Include Statements"
date: 2014-07-21 14:59:00 +0800
categories: ansible
---

{% raw %}

你可以将所有东西都放到一个playbook文件中，但是随着文件越来越大，你修改起来也越来越麻烦。这时候可以把一些`play`、`task`或`handler`放到其他文件中，然后通过`include`指令包含进来，这样做完全没问题，但是ansible还提供了一个更好的解决方法，也就是`roles`，下面分别讲解。

<br />

### include
playbook可以包含其他playbook文件、task文件和handler文件。

<b>包含task文件</b>  
如果有多个`play`都需要几个相同的task，在每个`play`中都写一遍这些task就太傻了，聪明做法是将这些task单独放到一个文件中，格式如下：

```yaml
---
# possibly saved as tasks/foo.yml
- name: placeholder foo
  command: /bin/foo

- name: placeholder bar
  command: /bin/bar
```
然后在需要这些task的`play`中通过`include`包含上面的`tasks/foo.yml`：

```yaml
tasks:
  - include: tasks/foo.yml
```
还可以向`include`传递变量，如你部署了多个wordpress实例，通过向相同的`wordprss.yml`文件传递不同的值来区分实例：

```yaml
tasks:
  - include: wordpress.yml user=timmy     #在foo.yml可以通过{{ user }}来使用这些变量
  - include: wordpress.yml user=alice
  - include: wordpress.yml user=bob
```
如果使用ansible1.4及以上版本，`include`还可以写成字典格式：

```yaml
tasks:
 - { include: wordpress.yml, user: timmy, ssh_keys: [ 'keys/one.txt', 'keys/two.txt' ] }   #ssh_keys是一个列表
```
从ansible1.0开始，还可以用如下格式向`include`传递变量：

```yaml
tasks:
  - include: wordpress.yml
    vars:
        remote_user: timmy
        some_list_variable:
          - alpha
          - beta
          - gamma
```

<b>包含handler文件</b>  

```yaml
---
# this might be in a file like handlers/handlers.yml
- name: restart apache
  service: name=apache state=restarted
```
在`play`末尾包含上面的handler文件：

```yaml
handlers:
  - include: handlers/handlers.yml
```

<b>直接包含playbook文件</b>  

```yaml
- name: this is a play at the top level of a file
  hosts: all
  remote_user: root
  tasks:
  - name: say hi
    tags: foo
    shell: echo "hi..."

- include: load_balancers.yml   #这些playbook文件中也至少定义了一个play
- include: webservers.yml
- include: dbservers.yml
```


<br />

### Roles
roles用来组织playbook结构，以多层目录和文件将playbook更好的组织在一起，一个经过roles组织的playbook结构如下：

```
site.yml
webservers.yml
fooservers.yml
roles/
   common/    #下面的子目录都不是必须提供的，没有的目录会自动忽略，不会出现问题，所以你可以只有tasks/子目录也没问题
     files/
     templates/
     tasks/
     handlers/
     vars/
     meta/
   webservers/
     files/
     templates/
     tasks/
     handlers/
     vars/
     meta/
```
然后在playbook文件中包含`common`和`webservers`这两个role：

```yaml
---
- hosts: user_group1
  roles:
     - common
     - webservers
```

<b>假如有一个play包含了一个叫"x"的role，则：</b>
<ul>
<li>`/path/roles/x/tasks/main.yml`中的tasks都将自动添加到该play中</li>
<li>`/path/roles/x/handlers/main.yml`中的handlers都将添加到该play中</li>
<li>`/path/roles/x/vars/main.yml`中的所有变量都将自动添加到该play中</li>
<li>`/path/roles/x/meta/main.yml`中的所有role依赖关系都将自动添加到roles列表中</li>
<li>`/path/roles/x/defaults/main.yml`中为一些默认变量值，<b>具有最低优先权</b>，在没有其他任何地方指定该变量的值时，才会用到默认变量值</li>
<li>task中的`copy`模块和`script`模块会自动从`/path/roles/x/files`寻找文件，也就是根本不需要指定文件绝对路径或相对路径，如`src=foo.txt`则自动转换为`/path/roles/x/files/foo.txt`</li>
<li>task中的`template`模块会自动从`/path/roles/x/templates/`中加载模板文件，无需指定绝对路径或相对路径</li>
<li>通过`include`包含文件会自动从`/path/roles/x/tasks/`中加载文件，无需指定绝对路径或相对路径</li>
</ul>
从ansible1.4开始可以在`ansible.cfg`配置文件中通过`roles_path`指令自定义roles的路径。

如同`include`一样，也可以为`role`传递变量，格式如下：

```yaml
---
- hosts: webservers
  roles:
    - common
    - { role: foo_app_instance, dir: '/opt/a',  port: 5000 }     #在foo_app_instance这个role的task文件和模板文件中通过{{ dir }}和{{ port }}来使用变量
    - { role: foo_app_instance, dir: '/opt/b',  port: 5001 }
```

<b>如果一个play中，既有`tasks`也有`roles`，那么`roles`会先于`tasks`执行</b>，可以通过`pre_tasks`和`post_tasks`指令指定某些task先于或晚于`roles`执行：

```yaml
---
- hosts: webservers

  pre_tasks:
    - shell: echo 'hello'      #最先执行

  roles:
    - { role: some_role }      #第二个执行

  tasks:
    - shell: echo 'still busy'

  post_tasks:
    - shell: echo 'goodbye'     #最后执行
```

#### role依赖
可以在一个role的`meta/main.yml`中定义该role依赖其他的role，然后调用该role的时候，会自动去拉取其他依赖的role，如一个名为`myapp`的role的`meta/main.yml`文件如下：

```yaml
---
dependencies:
  - { role: common, some_parameter: 3 }    #向依赖的role传递变量
  - { role: apache, port: 80 }
  - { role: postgres, dbname: blarg, other_parameter: 12 }
```
那么这些role的执行顺序为：

```
common
apache
postgres
myapp       #会先把myapp依赖的其他所有role执行完再执行自己
```

默认一个role只能被其他role依赖一次，多次依赖不会执行，但是可以通过`allow_duplicates`指令来改变这种行为：  
名为`car`的role的`meta/main.yml`：

```yaml
---
dependencies:
  - { role: wheel, n: 1 }
  - { role: wheel, n: 2 }
  - { role: wheel, n: 3 }
  - { role: wheel, n: 4 }
```
名为`wheel`的role的`meta/main.yml`：

```yaml
---
allow_duplicates: yes
dependencies:
  - { role: tire }
  - { role: brake }
```
执行顺序如下：

```
tire(n=1)
brake(n=1)
wheel(n=1)
tire(n=2)
brake(n=2)
wheel(n=2)
...
```

{% endraw %}
