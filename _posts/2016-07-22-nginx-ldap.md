---
layout: post
title:  "nginx集成ldap认证"
date:   2016-07-22 10:46:38
categories: devops
---

最近在阿里云搭了kafka-manager来管理kafka集群，因为公司内网和阿里云内网不通，为了开发方便访问，就把kafka-manager的web端口开放到公网访问。但是kafka-manager本身是不带用户认证系统的，所以最开始的想法是在nginx代理层通过`ngx_http_auth_basic_module`模块做一个http验证，给开发每人一个账号登录。但是开发小伙伴们又不乐意记账号密码，因为公司有ldap，所以最好的办法就是集成ldap认证。

首先google看下是否可以实现，官方给了一个文章[Using NGINX Plus and NGINX to Authenticate Application Users with LDAP](https://www.nginx.com/blog/nginx-plus-authenticate-users/)详细说明了nginx集成ldap认证的原理和步骤，**请务必先过一遍这个文章，本文不再详细说明实现原理**，下面就说下我的搭建步骤和遇到的一些问题：

1、首先下载这个github库<https://github.com/nginxinc/nginx-ldap-auth>

```bash
cd /data/
git clone https://github.com/nginxinc/nginx-ldap-auth
```
说明：    
**nginx-ldap-auth.conf** - 一个nginx集成ldap的示例配置文件，我们自己的nginx配置就需要参照该文件。    
**nginx-ldap-auth-daemon.py** - 实际的ldap认证逻辑就在这个脚本中，该脚本会起一个daemon进程。该脚本做两件事，接收到nginx转发过来的请求会返回401给nginx告诉用户还没认证，nginx此时会返回一个登陆页面给用户；然后nginx转发用户密码和对应cookie到该脚本，该脚本去ldap server认证用户，成功返回200给nginx，失败返回403给nginx。按照官方说明，运行该脚本需要`python2`和`python-ldap`模块，请提前pip安装。    
**nginx-ldap-auth-daemon-ctl-rh.sh** - 官方说的用`nginx-ldap-auth-daemon-ctl.sh `来起`nginx-ldap-auth-daemon.py`脚本，但是我是centos系统，用这个脚本才是对的。    
**backend-sample-app.py** - 官方给的一个测试用的后端程序，包含了登录页面和对应的处理逻辑，还有验证成功后返回给用户的信息，因为kafka-manger本身也没有提供登录页面，所以我最后还是用到这个脚本来处理kafka-manager登录相关逻辑。   

2、安装Nginx，因为用到了`ngx_http_auth_request_module`模块，所以编译的时候别忘了`--with-http_auth_request_module`

```bash
wget http://nginx.org/download/nginx-1.6.3.tar.gz
tar zxvf nginx-1.6.3.tar.gz
cd nginx-1.6.3
./configure --prefix=/data/nginx --with-http_realip_module --with-http_ssl_module --with-http_sub_module --with-http_auth_request_module --with-http_stub_status_module
make
make install
```

3、官方的测试就不做了，详细讲下线上最后正式使用的nginx配置

```bash
user                  nobody nobody;
worker_processes auto;
#worker_cpu_affinity auto;
worker_rlimit_nofile 65535;

error_log             logs/error.log;
pid                   logs/nginx.pid;

events {
    use epoll;
    #reuse_port on;   #used in tengine and linux kernel >= 3.9
    accept_mutex off;  #used in nginx
    worker_connections  65535;
}

http {
    include                     mime.types;
    default_type                application/octet-stream;
    server_tokens               off;

    log_format      main        '$remote_addr - $remote_user [$time_local] "$request" '
                                '$status $request_time $body_bytes_sent "$http_referer" '
                                '"$http_user_agent" "$http_x_forwarded_for"|body: $request_body';

    sendfile                    on;
    tcp_nopush                  on;
    tcp_nodelay                 on;
    keepalive_timeout           60;

    gzip                        on;
    gzip_vary                   on;
    gzip_comp_level             5;
    gzip_buffers                16 4k;
    gzip_min_length             1000;
    gzip_proxied                any;
    gzip_disable                "msie6";
    gzip_types                  text/plain text/css application/x-javascript text/xml application/xml application/xml+rss text/javascript application/javascript application/json;

    open_file_cache max=1000    inactive=20s;
    open_file_cache_valid       30s;
    open_file_cache_min_uses    2;
    open_file_cache_errors      on;

    client_max_body_size        50m;
    
    #缓存可以减少ldap验证频率，不然每个页面都需要ldap验证一次
    #你不在乎的话，不要缓存也是没有任何问题的 
    proxy_cache_path cache/ keys_zone=auth_cache:10m;

#kakfa-manager程序，跑在同一台机器的8000端口
upstream kafka-manager {
    server 127.0.0.1:8000;
}

server {
    listen 8081;
    server_name localhost;

    access_log logs/kafka-manager.log main;

    #后端程序，也就是kafka-manager
    location / {
        auth_request /auth-proxy;

        #nginx接收到nginx-ldap-auth-daemon.py返回的401和403都会重新跳转到登录页面
        error_page 401 403 =200 /login;

        proxy_pass http://kafka-manager/;
    }

    #登录页面，由backend-sample-app.py提供，跑在同一台机器的8082端口(默认不是8082端口)
    location /login {
        proxy_pass http://127.0.0.1:8082/login;
        proxy_set_header X-Target $request_uri;
    }

    location = /auth-proxy {
        internal;
        proxy_pass http://127.0.0.1:8888;     #nginx-ldap-auth-daemon.py运行端口
        #缓存设置
        proxy_cache auth_cache;
        proxy_cache_key "$http_authorization$cookie_nginxauth";
        proxy_cache_valid 200 403 10m;
        
        proxy_pass_request_body off;
        proxy_set_header Content-Length "";

        #最最重要的ldap配置，请务必按照贵公司的ldap配置如下四项，我在这一步卡了好久，就是ldap配置不对
        #这些配置都会通过http头部传递给nginx-ldap-auth-daemon.py脚本
        proxy_set_header X-Ldap-URL      "ldap://ip:port";
        proxy_set_header X-Ldap-BaseDN   "DC=example,DC=com";
        proxy_set_header X-Ldap-BindDN   "cn=Admin,ou=People,dc=example,dc=com";
        proxy_set_header X-Ldap-BindPass "password";

        proxy_set_header X-CookieName "nginxauth";
        proxy_set_header Cookie nginxauth=$cookie_nginxauth;
    }
}
}
```

4、启动所有程序

```bash
cd /data/nginx-ldap-auth/
./nginx-ldap-auth-daemon-ctl-rh.sh start
nohup python backend-sample-app.py &
/data/nginx/sbin/nginx
```
    
5、通过`http://ip:8081/`访问，正常情况下会跳出`backend-sample-app.py`提供的登陆框，输入用户密码后如果认证成功就会成功跳转到kafka-manager的页面，如果验证失败（返回403）就还是会跳转到登录页面，所以如果始终跳转回登录页面，就说明认证有问题。那么如何调试呢，通过nginx日志是看不出来什么有用信息的，必须通过`nginx-ldap-auth-daemon.py`的日志来调试，但是`nginx-ldap-auth-daemon.py`的日志在哪里呢？看下代码：

```python
    def log_message(self, format, *args):
        if len(self.client_address) > 0:
            addr = BaseHTTPRequestHandler.address_string(self)
        else:
            addr = "-"

        sys.stdout.write("%s - %s [%s] %s\n" % (addr, self.ctx['user'],
                         self.log_date_time_string(), format % args))
```
该脚本直接打印日志到标准输出了，我们修改一下保存到日志文件中：

```python
    def log_message(self, format, *args):
        if len(self.client_address) > 0:
            addr = BaseHTTPRequestHandler.address_string(self)
        else:
            addr = "-"
            
        with open("/tmp/nginx_ldap_auth.log", 'a') as f:
            f.write("%s - %s [%s] %s\n" % (addr, self.ctx['user'], self.log_date_time_string(), format % args))
```
现在就可以通过日志调试了！

6、前面也说过，kafka-manager本身没有提供登录页面和对应处理逻辑，所以最后我还是用了`backend-sample-app.py`脚本，假如你想自己实现怎么办呢，看下`backend-sample-app.py`到底做了什么：

```python
    # processes posted form and sets the cookie with login/password
    def do_POST(self):

        # prepare arguments for cgi module to read posted form
        env = {'REQUEST_METHOD':'POST',
               'CONTENT_TYPE': self.headers['Content-Type'],}

        # read the form contents
        form = cgi.FieldStorage(fp = self.rfile, headers = self.headers, environ = env)

        # extract required fields
        user = form.getvalue('username')
        passwd = form.getvalue('password')
        target = form.getvalue('target')

        if user != None and passwd != None and target != None:
            self.send_response(302)

            enc = base64.b64encode(user + ':' + passwd)
            self.send_header('Set-Cookie', 'nginxauth=' + enc + '; httponly')

            self.send_header('Location', target)
            self.end_headers()

            return

        self.log_error('some form fields are not provided')
        self.auth_form(target)
```
用户输入账号密码点击提交后，`backend-sample-app.py`做了如上处理，所以如果要自己实现登录页面的话也要附带上面的逻辑，不单单只提供一个登录页面就够了！

7、`backend-sample-app.py`提供的登录页面极其简陋，没有任何css样式，连密码输入框都是明文的，这样肯定不行，所以从网上找了一个登陆form替换到`backend-sample-app.py`的对应代码：

```html
        html="""
<!DOCTYPE html>
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf8"/>
<title>登陆框</title>
</head>
<style>
*{margin:0;padding:0;}
.login{
width:334px;
height:220px;
margin:0 auto;
position:absolute;
left:40%;
top:40%;
}
.login_title{
color: #000000;
font: bold 14px/37px Arial,Helvetica,sans-serif;
height: 37px;
padding-left: 35px;
text-align: left;
}

.login_cont {
    background: none repeat scroll 0 0 #FFFFFF;
    border: 1px solid #B8B7B7;
    height: 152px;
    padding-top: 30px;
}
.form_table {
    float: left;
    margin-top: 10px;
    table-layout: fixed;
    width: 100%;
}
.form_table th {
    color: #333333;
    font-weight: bold;
    padding: 5px 8px 5px 0;
    text-align: right;
    white-space: nowrap;
}
.form_table td {
    color: #717171;
    line-height: 200%;
    padding: 6px 0 5px 10px;
    text-align: left;
}
.login_cont input.submit {
    background-position: 0 -37px;
    height: 29px;
    margin: 10px 14px 0 0;
    width: 82px;
}
</style>
<body>
    <div class="login">
        <div class="login_cont">
            <form action='/login' method='post'>
                <table class="form_table">
                    <col width="90px" />
                    <col />
                    <tr>
                        <th>用户名：</th><td><input class="normal" type="text" name="username" alt="请填写用户名" /></td>
                    </tr>
                    <tr>
                        <th>密码：</th><td><input class="normal" type="password" name="password" alt="请填写密码" /></td>
                    </tr>
                    <tr>
                        <th></th><td><input class="submit" type="submit" value="登录" /><input class="submit" type="reset" value="取消" /></td>
                    </tr>
                </table>
                    <input type="hidden" name="target" value="TARGET">
            </form>
        </div>
    </div>
</body>
</html>"""
```

<br />

#### 总结：
如果是你自己写的程序，建议还是在程序里面做ldap认证逻辑，还可以实现对应的权限管理逻辑。那么nginx层集成ldap的意义在哪里呢，个人觉得当使用kafka-manager这类开源工具的时候就很需要了，因为很难在对这些工具做二次开发，只能通过外部方式来认证，缺点嘛就是权限管理这块实在没有办法做了，如果你有办法再实现权限控制，请务必留言告诉我，多谢！

<br />

#### 参考文档：
* <https://www.nginx.com/blog/nginx-plus-authenticate-users/>    
* <https://github.com/nginxinc/nginx-ldap-auth>    
* <http://nginx.org/en/docs/http/ngx_http_auth_request_module.html>   



