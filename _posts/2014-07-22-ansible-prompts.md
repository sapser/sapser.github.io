---
layout: post
title: "ansible学习之十一：Prompts"
date: 2014-07-22 14:22:00 +0800
categories: ansible
---

{% raw %}

#### 用在执行playbook的过程中，提示用户输入，并接收用户的输入

```yaml
---
- hosts: 127.0.0.1
  vars_prompt:
    name: "what is your name?"
    quest: "what is your quest?"
    favcolor: "what is your favorite color?"
  tasks:
    - debug: msg="name - {{name}}   quest - {{quest}}    favcolor - {{favcolor}}"
```
执行：

```bash
$ ansible-playbook prompt.yml
what is your favorite color?: : red                #注意问题提示的顺序是反过来的
what is your quest?: : ansible
what is your name?: : sapser

PLAY [127.0.0.1] **************************************************************

TASK: [debug msg="name - sapser   quest - ansible    favcolor - red"] *********
ok: [127.0.0.1] => {
    "msg": "name - sapser   quest - ansible    favcolor - red"
}

PLAY RECAP ********************************************************************
127.0.0.1                  : ok=2    changed=0    unreachable=0    failed=0   
```


<br />
#### 设置默认值

```yaml
---
- hosts: 127.0.0.1
  gather_facts: no
  vars_prompt:
    - name: "release_version"          #变量名
      prompt: "product release version"          #提示
      default: "1.0"          #默认值
  tasks:
    - debug: msg="release_version - {{release_version}}"
```
执行：

```bash
$ ansible-playbook prompt.yml
product release version [1.0]: 1.2 

PLAY [127.0.0.1] **************************************************************

TASK: [debug msg="release_version - 1.2"] *************************************
ok: [127.0.0.1] => {
    "msg": "release_version - 1.2"
}

PLAY RECAP ********************************************************************
127.0.0.1                  : ok=1    changed=0    unreachable=0    failed=0  
```


<br />
#### 用户输入不可见

```yaml
---
- hosts: 127.0.0.1
  gather_facts: no
  vars_prompt:
    - name: "release_version"
      prompt: "product release version"
      default: "1.0"
      private: no
    - name: "passwd"
      prompt: "Enter password"
      private: yes
  tasks:
    - debug: msg="release_version - {{release_version}}      passwd - {{passwd}}"
```
执行：

```bash
$ ansible-playbook prompt.yml
product release version [1.0]: 2.5
Enter password:

PLAY [127.0.0.1] **************************************************************

TASK: [debug msg="release_version - 2.5      passwd - sdjflskdfjsd"] **********
ok: [127.0.0.1] => {
    "msg": "release_version - 2.5      passwd - sdjflskdfjsd"
}

PLAY RECAP ********************************************************************
127.0.0.1                  : ok=1    changed=0    unreachable=0    failed=0   
```


<br />
#### 加密用户的输入，需要python第三方模块[Passlib](http://pythonhosted.org/passlib/)

```yaml
vars_prompt:
  - name: "my_password2"
    prompt: "Enter password2"
    private: yes
    encrypt: "md5_crypt"
    confirm: yes
    salt_size: 7
```
Passlib支持多种加密方式：
<ul>
<li>`des_crypt` - DES Crypt</li>
<li>`bsdi_crypt` - BSDi Crypt</li>
<li>`bigcrypt` - BigCrypt</li>
<li>`crypt16` - Crypt16</li>
<li>`md5_crypt` - MD5 Crypt</li>
<li>`bcrypt` - BCrypt</li>
<li>`sha1_crypt` - SHA-1 Crypt</li>
<li>`sun_md5_crypt` - Sun MD5 Crypt</li>
<li>`sha256_crypt` - SHA-256 Crypt</li>
<li>`sha512_crypt` - SHA-512 Crypt</li>
<li>`apr_md5_crypt` - Apache’s MD5-Crypt variant</li>
<li>`phpass` - PHPass’ Portable Hash</li>
<li>`pbkdf2_digest` - Generic PBKDF2 Hashes</li>
<li>`cta_pbkdf2_sha1` - Cryptacular’s PBKDF2 hash</li>
<li>`dlitz_pbkdf2_sha1` - Dwayne Litzenberger’s PBKDF2 hash</li>
<li>`scram` - SCRAM Hash</li>
<li>`bsd_nthash` - FreeBSD’s MCF-compatible nthash encoding</li>
</ul>

{% endraw %}
