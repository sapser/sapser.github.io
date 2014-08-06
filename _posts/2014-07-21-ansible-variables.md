---
layout: post
title: "ansible学习之六：Variables"
date: 2014-07-21 15:54:00 +0800
categories: ansible
---


其实ansible中变量的用法在其他几篇文章中已经全部讲过了，总结一下：
<ul>
<li>`hosts`文件中定义变量，请看[Inventory]({% post_url 2014-07-10-ansible-inventory %})这篇文章</li>
<li>`group_vars`和`host_vars`目录下变量文件使用，也在[Inventory]({% post_url 2014-07-10-ansible-inventory %})</li>
<li>`vars_files`指令导入外部变量文件，在[Conditionals]({% post_url 2014-07-21-ansible-conditionals %})中讲过</li>
<li>`setup`模块收集的目标主机信息，在[Conditionals]({% post_url 2014-07-21-ansible-conditionals %})中讲解</li>
<li>`register`指令注册的变量，该指令两种不同用法在[Conditionals]({% post_url 2014-07-21-ansible-conditionals %})和[Loops]({% post_url 2014-07-21-ansible-loops %})中分别讲解</li>
</ul>

还有`jinja2`过滤器和ansible独有的过滤器可以对变量进行某些处理，其他的有空再写了...
