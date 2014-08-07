---
layout: post
title: "kazoo--zookeeper的python驱动"
date: 2014-07-24 15:27:00 +0800
categories: python zookeeper
---


kazoo是zookeeper的python驱动，纯python实现。


<br />
#### 核心类讲解
```python
kazoo.client.KazooClient(hosts='127.0.0.1:2181', 
                         timeout=10.0, 
                         client_id=None, 
                         handler=None, 
                         default_acl=None, 
                         auth_data=None, 
                         read_only=None, 
                         randomize_hosts=True, 
                         connection_retry=None, 
                         command_retry=None, 
                         logger=None, 
                         **kwargs)          
```
该类是kazoo模块的最主要的一个类，用于连接zookeeper服务器，参数：
<ul>
<li>`hosts`：指定ZooKeeper的ip和端口，可以是以逗号分隔的多个ZooKeeper服务器IP和端口，客户端会随机选择一个来连接。</li>
<li>`timeout`：会话超时时间，在连接断开后就开始计算，如果在此会话时间内重新连接上的话，该连接创建的临时节点就不会移除。默认会话超时最小是2倍的tickTime(在zk服务器配置文件中设置），最大是20倍的tickTime。会话过期由ZooKeeper集群，而不是客户端来管理。客户端与集群建立会话时提供该超时值，集群使用这个值来确定客户端会话何时过期，集群在指定的超时时间内没有得到客户端的消息时发生会话过期，会话过期时集群将删除会话的所有临时节点，立即通知所有(观察节点)客户端。</li>
<li>`client_id`：传递一个双元素数组：[会话id, 密码]。客户端取得ZooKeeper服务句柄时，ZooKeeper创建一个会话，由一个64位数标识，这个数将返回给客户端。如果连接到其他服务器，客户端将在连接握手时发送会话ID。出于安全考虑，服务器会为会话ID创建一个密码，ZooKeeper服务器可以校验这个密码。这个密码将在创建会话时与会话ID一同发送给客户端。与新的服务器重新建立会话的时候，客户端会和会话ID一同发送这个密码。</li>
<li>`read_only`：创建一个只读的连接。</li>
<li>`randomize_hosts`：随机选择zk服务器连接。</li>
</ul>

<b>类实例属性及方法：</b>

zk.start(timeout=15)  
初始化到zookeeper服务器的连接，超过timeout时间没连接到zk服务器则会产生timeout_exception异常  

```python
In [1]: from kazoo.client import KazooClient
In [2]: zk = KazooClient(hosts="127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183")
In [3]: zk.start()       #到这一步没生成异常就说明正常连接到zk服务器了
```

zk.stop()  
一旦连接上，客户端会尽力保持连接，不管间歇性的连接丢失。如果要主动丢弃连接，就使用该方法，该方法会断开连接和关闭该连接的session，此时该连接创建的所有临时节点都会立即移除，并触发这些临时节点的DataWatch和这些临时节点的父节点的ChildrenWatch

zk.restart()          
重启连接会话

zk.state          
当前连接状态，值为如下三个之一：LOST、CONNECTED、SUSPENDED。<b>当实例化一个KazooClient连接时处于LOST状态；然后使用start()真正建立连接后处于CONNECTED状态；如果此时连接出现问题或客户端切换到另一台zk服务器，此时将处于SUSPENDED状态；在会话有效期内重新连接上又变回CONNECTED状态，如果重连上但是会话过期，则变为LOST状态。</b>

```python
In [57]: zk1 = KazooClient(hosts="127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183")
In [58]: zk1.state
Out[58]: 'LOST'
In [59]: zk1.start()
In [60]: zk1.state
Out[60]: 'CONNECTED'
In [62]: zk1.stop()
In [63]: zk1.state
Out[63]: 'LOST'
In [64]: zk1.close()
In [65]: zk1.state
Out[65]: 'LOST'
```

zk.connected          
客户端是否已连接到zk服务器，已连接上返回True

zk.add\_listener(listener)          
添加一个函数对象作为回调函数，当连接状态改变时，就会自动调用该回调函数，具体看后面的<b>“监听连接事件”。</b>

zk.remove\_listener(listener)          
移除一个listener

zk.state\_listeners          
listener状态

zk.create(path, value='', acl=None, ephemeral=False, sequence=False, makepath=False)          
创建一个节点，ephemeral表示改节点是临时节点，sequence表示该节点为顺序节点，默认当节点的父节点或祖先节点不存在时，创建该节点会失败，可以使用makepath设置为True来自动创建缺少的祖先节点。  
zk节点(znode)可以分为如下四类：
<ul>
<li>`PERSISTENT`：持续的，相比于EPHEMERAL，不会随着client session的close/expire而消失</li>
<li>`PERSISTENT_SEQUENTIAL`：顺序的，会自动在节点名后面添加一个自增计数，格式为%010d</li>
<li>`EPHEMERAL`：临时节点，生命周期依赖于client session，对应session close/expire后其znode也会消失，临时节点不能有子节点</li>
<li>`EPHEMERAL_SEQUENTIAL`</li>
</ul>
该方法可能触发如下异常：
<ul>
<li>`NodeExistsError`：当要创建的节点已经存在时</li>
<li>`NoNodeError`：当makepath为False且祖先节点不存在时</li>
<li>`NoChildrenForEphemeralsError`：父节点为临时节点，在一个临时节点下面创建子节点会报该异常</li>
<li>`ZookeeperError`：节点值太大，zk默认节点值限制为1M</li>
<li>`ZookeeperError`：服务器返回一个非0状态码</li>
</ul>

zk.get\_children(path, watch=None, include\_data=False)  
获取指定节点的所有子节点，以列表返回。如果include\_data为True，则还会返回该节点的ZnodeStat状态

zk.get(path, watch=None)          
获取指定节点的值，节点不存在触发NoNodeError异常

```python
In [12]: zk.get("/xj")
Out[12]:
('2222',          #节点的值
  ZnodeStat(
    czxid=4294967304,             #创建该节点的zxid
    mzxid=17179869186,            #最近一次修改该节点的zxid
    ctime=1386060984217,          #秒数表示的znode创建时间，这里最后3位数是毫秒数，如1386060984217，准确应该为1386060984.217
    mtime=1386296567754,          #该节点的最近一次修改时间
    version=7,           #该节点数据修改次数
    cversion=4,          #改节点的子节点修改次数   
    aversion=0,          #该节点的ACL修改次数
    ephemeralOwner=0,    #如果znode是临时节点，则指示节点所有者的会话ID；如果不是临时节点，则为零
    dataLength=4,        #该节点的数据长度
    numChildren=4,       #子节点个数
    pzxid=8589934670    
  )
)
```

zk.set(path, value, version=-1)          
设置节点的值，返回该节点的ZnodeStat信息。版本不匹配产生BadVersionError异常、节点不存在产生NoNodeError、提供的值value太大产生ZookeeperError异常、如果zk返回一个非零错误状态码则产生ZookeeperError

```python
In [15]: zk.set('/xj', "new_value")
Out[15]: ZnodeStat(czxid=4294967304, mzxid=21474836483, ctime=1386060984217, mtime=1386642770131, version=8, cversion=4, aversion=0, ephemeralOwner=0, dataLength=9, numChildren=4, pzxid=8589934670)
```

zk.delete(path, version=-1, recursive=False)               
删除节点，recursive为True表示递归删除节点及其子节点，如果有子节点且recursive为False，则会产生NotEmptyError异常，表示该节点有子节点不能删除；版本不匹配产生BadVersionError异常；节点不存在产生NoNodeError；如果zk返回一个非零错误状态码则产生ZookeeperError

zk.exists(path, watch=None)          
检查节点是否存在，存在返回节点的ZnodeStat信息，否则返回None

zk.ensure_path(path, acl=None)              
自动创建节点的祖先节点，如想创建一个节点"/a/b/c"，但是"/a/b"不存在，这时候使用该方法就可以自动把不存在的祖先节点一起创建了，`create()`的`makepath`参数也能实现该功能

zk.sync(path)          
阻塞并等待指定节点同步到所有zk服务器，返回同步的节点

zk.command(cmd='ruok')          
用于执行zk服务器提供的四字命令，这些四字命令如下：

```
conf               获取zk服务器的配置信息
cons               输出指定server上所有客户端连接的详细信息，包括客户端IP，会话ID等
crst               功能性命令。重置所有连接的统计信息
dump               这个命令针对Leader执行，用于输出所有等待队列中的会话和临时节点的信息
envi               用于输出server的环境变量。包括操作系统环境和Java环境
ruok               用于测试server是否处于无错状态。如果正常，则返回“imok”,否则没有任何响应。注意：ruok不是一个特别有用的命令，它不能反映一个server是否处于正常工作。“stat”命令更靠谱
stat               输出server简要状态和连接的客户端信息
srvr               和stat类似
srst               重置server的统计信息
wchs               列出所有watcher信息概要信息，数量等
wchc               列出所有watcher信息，以watcher的session为归组单元排列，列出该会话订阅了哪些path
wchp               列出所有watcher信息，以watcher的path为归组单元排列，列出该path被哪些会话订阅着，意，wchc和wchp这两个命令执行的输出结果都是针对session的，对于运维人员来说可视化效果并不理想，可以尝试将cons命令执行输出的信息整合起来，就可以用客户端IP来代替会话ID了，具体可以看这个实现：http://rdc.taobao.com/team/jm/archives/1450
mntr               输出一些ZK运行时信息，通过对这些返回结果的解析，可以达到监控的效果
```

zk.hosts               
一个迭代器，显示该客户端随机选择的zk服务器

```python
In [31]: list(zk.hosts)
Out[31]: [('127.0.0.1', 2181), ('127.0.0.1', 2182), ('127.0.0.1', 2183)]
```

zk.last\_zxid          获取zk服务器最新的一个zxid

```python
In [37]: zk.last_zxid
Out[37]: 21474836491
```

zk.client\_id          
返回连接的session\_id和密码

zk.chroot          
查看当前连接根节点

```python
In [30]: zk.chroot
Out[30]: '/xj'
```


<br />
#### 监听连接事件：
用于监控连接是否断开、恢复或者是会话过期，kazoo通过kazoo.client.KazooState类来实现该功能，该类有三个值如下：

```
KazooState.CONNECTED    已正常连接上或已重新连接上zk服务器的连接状态
KazooState.SUSPENDED    连接被中断，但是会话时间还没过期，该连接创建的临时节点也都还在   
KazooState.LOST         该连接已确认死亡(连接中断且会话过期)，此时该连接创建的临时节点都已被移除
```

<b>涉及KazooClient类的方法：</b>  
zk.state          
当前连接状态，值为如下三个之一：LOST、CONNECTED、SUSPENDED。当实例化一个KazooClient连接时处于LOST状态；然后使用start()真正建立连接后处于CONNECTED状态；如果此时连接出现问题或客户端切换到另一台zk服务器，此时将处于SUSPENDED状态；在会话有效期内重新连接上又变回CONNECTED状态，如果重连上但是会话过期，则变为LOST状态。

zk.connected          
客户端是否已连接到zk服务器，已连接返回True

zk.add_listener(listener)          
添加一个函数对象作为回调函数，当连接状态改变时，就会自动调用该函数

zk.remove_listener(listener)          
移除一个listener

zk.state_listeners          
listener状态

例子：

```python
In [45]: from kazoo.client import KazooClient
In [46]: from kazoo.client import KazooState
In [47]: zk = KazooClient(hosts="127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183")

In [48]: def my_listener(state):
   ....:     if state == KazooState.LOST:
   ....:         print "trigger LOST state"
   ....:     elif state == KazooState.SUSPENDED:
   ....:         print "trigger SUPENDED state"
   ....:     else:
   ....:         print "connected or reconnected"

In [51]: zk.add_listener(my_listener)

In [52]: zk.start()         
connected or reconnected               #触发CONNECTED状态

In [53]: zk.stop()
trigger LOST state               #触发LOST状态
```


<br />
#### 重连保持session_id不变：
客户端与zookeeper服务器的连接断开了，重新连接后zookeeper是如何知道这是之前一个连接的重新连接呢？这是靠KazooClient类的`client_id`参数(是一个双元素元组)来保证的，如果该参数为空表示是一个新连接，此时zookeeper服务器为该连接分配一个`session_id`和对应秘钥(元组形式)；如果该参数不为空，则zookeeper就知道这个连接是之前一个连接断开后重新连接上来的，然后就会去查该连接的session是否过期，如果还没过期就继续保存这个连接的临时节点。  
当前想到的做法就是，将`zk.client_id`序列化到一个文件中，如`pickle.dump(zk.client_id, fileobj)`，然后下次连接就通过`pickle.load(fileobj)`来获取这个`session_id`，然后传入KazooClient类的`client_id`参数中。

应用举例：  
如zookeeper做游戏服务器的负载均衡，每一个游戏服都在zookeeper服务器上注册一个临时节点，游戏服务器断开后，如果在session会话期内通过上一个连接的`client_id`重新连接上来，则临时节点不会被删除，表示该游戏服依然正常，没有挂掉；如果超过session会话有效期还没连接上来，则临时节点被移除，判定为该游戏服已经挂掉，不再转发客户端请求过来。


<br />
#### watcher
分为两种：
<ul>
<li>`dataWatch`：针对节点的创建、修改、删除，都会触发该watch（同时创建、删除节点也会触发该节点的父节点的childrenWatch）</li>
<li>`childrenWatch`：针对子节点的创建、删除，才会触发该watch</li>
</ul>

kazoo.client.KazooClient类提供了两个装饰器方法来实现这两类watcher：

```python
@zk.ChildrenWatch
@zk.DataWatch
```
不同于默认的watch规则，使用该装饰器定义的watch会一直存在，而不是默认的一次性，也就是说只要对一个路径定义了watch，该watch就会一直存在，监控该路径的任何变动。

<b>DataWatch用法：</b>

```python
@zk.DataWatch("/xj")          #对该节点create、set、delete操作都会触发该watch
def changed(data, stat, event):          #data是该节点的值；stat是节点的ZnodeState状态信息；event是WatchedEvent类实例，有三个值：type表示触发该watch的操作类型（如CREATED表示是一个创建节点的操作触发了该watch）、state表示当前连接状态、path表示操作的路径。这三个参数也不是必须提供，提供一个或两个也行
    ...
```
例子：

```python
from kazoo.client import KazooClient
import time

zk = KazooClient(hosts="127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183")
zk.start()

@zk.DataWatch("/xj")
def changed(data, stat, event):
    print "--------------DataWatch---------------"
    print "data:", data
    print "stat:", stat
    print "event:", event

zk.create("/xj", "value1")
time.sleep(2)
zk.set("/xj", "value2")
time.sleep(2)
zk.delete("/xj")
time.sleep(2)
```
执行上面代码：

```bash
[root@centos6 ~]# python kazoo_watcher.py
--------------DataWatch---------------               #谨记，watch函数定义完的就会调用一次
data: None
stat: None
event: None
--------------DataWatch---------------
data: value1
stat: ZnodeStat(czxid=21474836501, mzxid=21474836501, ctime=1386646806240, mtime=1386646806240, version=0, cversion=0, aversion=0, ephemeralOwner=0, dataLength=6, numChildren=0, pzxid=21474836501)
event: WatchedEvent(type='CREATED', state='CONNECTED', path=u'/xj')          #这里可以通过event.type这种方式来访问
--------------DataWatch---------------
data: value2
stat: ZnodeStat(czxid=21474836501, mzxid=21474836502, ctime=1386646806240, mtime=1386646808268, version=1, cversion=0, aversion=0, ephemeralOwner=0, dataLength=6, numChildren=0, pzxid=21474836501)
event: WatchedEvent(type='CHANGED', state='CONNECTED', path=u'/xj')
--------------DataWatch---------------
data: None
stat: None
event: WatchedEvent(type='DELETED', state='CONNECTED', path=u'/xj')
```

<b>ChildrenWatch用法：</b>

```python
@zk.ChildrenWatch("/xj")          #如果/xj节点的子节点有变动(添加、删除)，触发该watch
def childWatch(children):               #这种watch函数只有一个参数，children以列表形式保存了该节点的所有子节点
    ....
```
例子：

```python
from kazoo.client import KazooClient
import time

zk = KazooClient(hosts="127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183")
zk.start()

@zk.ChildrenWatch("/xj")
def childWatch(children):
        print '---------------ChildWatch----------------'
        print "children:", children

zk.create("/xj/a", "value1")
time.sleep(2)
zk.set("/xj/a", "value2")
time.sleep(2)
zk.delete("/xj/a")
time.sleep(2)
```
执行上面代码：

```bash
[root@centos6 ~]# python kazoo_watcher.py
---------------ChildWatch----------------
children: []          #定义完watch函数后自动执行一次
---------------ChildWatch----------------
children: [u'a']
---------------ChildWatch----------------
children: []          #这里是zk.delete("/xj/a")触发的，对子节点set不会触发父节点的ChilrenWatch
```


<br />
#### 事务：
zookeeper3.4开始支持事务操作，在一个事务中可以执行多个操作，如果有一个操作未执行成功，则回滚到事务开始之前的状态

```python
transaction = zk.transaction()
transaction.check('/node/a', version=3)
transaction.create('/node/b', b"a value")
results = transaction.commit()
```


<br />
#### chroot（改变连接根节点）：
可以改变新连接的根节点，这样做有很多好处，比如不同应用只能访问zk服务器的不同节点，不用担心看到或修改了其他应用的节点信息

```python
In [24]: zk.get_children("/")          #默认根节点
Out[24]:
[u'xj',
u'sapser',
u'user0000000030',
u'user0000000031',
u'user0000000032',
u'id-0000000038',
u'user0000000033',
u'user0000000034',
u'sapserr']

In [25]: zk.get("/xj")
Out[25]:
('',
ZnodeStat(czxid=21474836510, mzxid=21474836510, ctime=1386647457406, mtime=1386647457406, version=0, cversion=5, aversion=0, ephemeralOwner=0, dataLength=0, numChildren=1, pzxid=25769803779))          #从numChildren看出/xj节点有一个子节点

In [26]: zk1 = KazooClient(hosts="127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183/xj")          #改变zk1连接的根目录为/xj节点，该连接只能看到/xj节点之下的东西。注意/xj必须位于hosts参数末尾位置，位于其他位置都会出错

In [27]: zk1.start()

In [28]: zk1.get_children("/")          #这是根节点"/"其实就是/xj节点，可以看到只有一个子节点
Out[28]: [u'a']
```



