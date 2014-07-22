---
layout: post
title: "ansible学习之十二：Using Lookups"
date: 2014-07-22 14:56:00 +0800
categories: ansible
---

{% raw %}

Lookup插件允许ansible从外部资源获取数据，用`lookup()`函数表示


<br />
第一个参数为`file`，表示获取外部文件内容

```yaml
- hosts: all
  vars:
     contents: "{{ lookup('file', '/etc/foo.txt') }}"
  tasks:
     - debug: msg="the value of foo.txt is {{ contents }}"
```


<br />
第一个参数为`password`，表示生成一个随机明文密码，并存储到指定文件中，生成的密码包括大小写字母、数字和".,:-_"，默认密码长度为20个字符

```yaml
---
- hosts: all
  tasks:
    # create a mysql user with a random password:
    - mysql_user: name={{ client }}
                  password="{{ lookup('password', 'credentials/' + client + '/' + tier + '/' + role + '/mysqlpassword length=15') }}"
                  priv={{ client }}_{{ tier }}_{{ role }}.*:ALL
```


<br />
其他类型

```yaml
---
- hosts: all
  tasks:

     - debug: msg="{{ lookup('env','HOME') }} is an environment variable"      #'env'表示获取系统环境变量

     - debug: msg="{{ item }} is a line from the result of this command"
       with_lines:
         - cat /etc/motd

     - debug: msg="{{ lookup('pipe','date') }} is the raw result of running this command"

     - debug: msg="{{ lookup('redis_kv', 'redis://localhost:6379,somekey') }} is value in Redis for somekey"

     - debug: msg="{{ lookup('dnstxt', 'example.com') }} is a DNS TXT record for example.com"

     - debug: msg="{{ lookup('template', './some_template.j2') }} is a value from evaluation of this template"
```

{% endraw %}
