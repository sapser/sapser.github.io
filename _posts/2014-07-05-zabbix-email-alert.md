---
layout: post
title:  "zabbix邮件报警"
date:   2014-07-05 10:46:38
categories: zabbix
---


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
