---
layout: post
title:  "hadoop distcp使用filters参数排除指定文件"
date:   2016-09-30 10:46:38
categories: bigdata
---

使用`hadoop distcp`命令进行hadoop集群数据迁移时，可以使用`-filters`参数排除指定路径不迁移，该参数说明：

```
-filters <arg>                The path to a file containing a list of strings for paths to be excluded from the copy.
```
    
只有一个这么简单的说明，完全不会用啊，幸好google到一篇文章[How to use “filters” to exclude files when in DistCp](http://www.ericlin.me/how-to-use-filters-to-exclude-files-when-in-distcp)给了一个使用正则排除路径的例子，将如下两行正则放入一个普通文件中：

```bash
.*\.Trash.*
.*\.staging.*
```
`.*\.Trash.*`会匹配`hdfs://source/path`下任何带有`.Trash`字符串的路径（通常为用户回收站目录），然后排除这些匹配到的路径不迁移。使用如下命令迁移数据：

```bash
hadoop distcp -filters /path/to/filterfile.txt hdfs://source/path hdfs://destination/path
```

    
<br />
    
    
`-filters`使用正则排除文件的逻辑：

```java
@Override
public boolean shouldCopy(Path path) {
  for (Pattern filter : filters) {
    if (filter.matcher(path.toString()).matches()) {
      return false;
    }
  }
   
  return true;
}
```
`Matcher.matches`这个函数只有当正则完整匹配整个文件路径时才返回true，其他情况都返回false表示不匹配，比如正则`\.Trash.*`是匹配不到`/user/root/.Trash/Current`这个路径的。

    
<br />
    
    
我是在目标集群执行`hadoop distcp`迁移数据，当我在目标集群机器上访问源集群HDFS内容：

```bash
shell> hdfs dfs -ls hdfs://10.1.2.3:8022/
drwxr-xr-x   - hbase hbase               0 2016-08-09 13:46 hdfs://10.1.2.3:8022/hbase
drwxr-xr-x   - hive  supergroup          0 2016-08-12 16:50 hdfs://10.1.2.3:8022/home
drwxrwxr-x   - solr  solr                0 2015-05-06 10:48 hdfs://10.1.2.3:8022/solr
drwxr-xr-x   - hdfs  supergroup          0 2016-09-23 15:13 hdfs://10.1.2.3:8022/system
drwxrwxrwt   - hdfs  supergroup          0 2016-08-21 13:06 hdfs://10.1.2.3:8022/tmp
drwxr-xr-x   - hdfs  supergroup          0 2016-09-08 14:45 hdfs://10.1.2.3:8022/user
drwxr-xr-x   - root  supergroup          0 2016-04-19 09:38 hdfs://10.1.2.3:8022/usr
```
此时所有文件路径前面都是有`hdfs://10.1.2.3:8022/`这个前缀的。所以我这里`-filters`要排除路径的正则：

```bash
.*\.Trash.*
.*\.stagine.*
hdfs://10.1.2.3:8022/.snapshot/hadoop_distcp_201609301508/hbase.*
hdfs://10.1.2.3:8022/.snapshot/hadoop_distcp_201609301508/usr/cupid.*
hdfs://10.1.2.3:8022/.snapshot/hadoop_distcp_201609301508/tmp.*
```
必须要把路径全部写上才能排除指定路径，或者使用类似`.*\.Trash.*`来排除任何带有`.Trash`的路径。


