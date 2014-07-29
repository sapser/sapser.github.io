---
layout: post
title:  "zabbix邮件报警"
date:   2014-07-05 10:46:38
categories: zabbix
---

使用自定义脚本实现zabbix邮件报警。

首先`Administration`->`Media types`定义类型为Script，表示调用一个自定义的脚本来发送报警邮件：
![media types]({{site.baseurl}}/static/images/zabbix_email_alert_mediatypes.png)
选项`script name`只填脚本名就行了，脚本路径在zabbix_server.conf中通过`AlertScriptsPath`指令配置：

```
AlertScriptsPath=/usr/local/zabbix/etc               #末尾不带"/"
```
脚本需要有<b>可执行权限</b>，且脚本开头需要通过"#!"来指定脚本解释器，zabbix是直接通过绝对路径调用脚本。

zabbix会以命令行参数的方式向脚本传入三个值：收件人、邮件主题、邮件正文。如果有多个收件人，则zabbix会多次调用脚本，每次只传入一个收件人。

发送邮件的python脚本如下，smtp服务器及发件人信息需自定义：

```python
#!/usr/bin/python
# coding: utf-8
import os
import sys
import smtplib
from datetime import datetime
from email.header import Header
from email.mime.text import MIMEText

#需自定义部分
email_server = 'smtp.126.com'
email_port = 25
email_user = 'xxx@126.com'
email_passwd = 'xxx'


def _msg(email_user, email_to, subject, content):
    """gen email header and content"""
    msg = MIMEText(content, _charset='utf-8')   #邮件正文为utf8编码
    msg['From'] = email_user
    msg['To'] = email_to
    msg['Subject'] = Header(subject,'utf-8')    #邮件主题为utf8编码
    return msg


def mailer(subject, content):
    """send alert email using smtplib"""
    msg = _msg(email_user, email_to, subject, content)
    smtp = smtplib.SMTP()
    smtp.connect(email_server, email_port)
    smtp.login(email_user, email_passwd)
    smtp.sendmail(email_user, email_to, msg.as_string())
    smtp.quit()


if __name__ == '__main__':
    #获取由zabbix传入的收件人、邮件主题和邮件正文
    email_to,subject,content = sys.argv[:3]
    #发送邮件
    mailer(subject, content)
```
