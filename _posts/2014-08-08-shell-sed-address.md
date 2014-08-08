---
layout: post
title: "sed地址匹配总结"
date: 2014-08-08 13:20:00 +0800
categories: shell
---


sed中可以通过多种方式对行进行过滤，只选择那些符合特征的行进行处理，总结如下（这里针对`gnu sed`，所以某些功能在某些平台的sed上是没有的）：

<b>数字         </b>  
表示和数字对应的行，如1就表示第一行。要注意的是sed中可以用0来表示第一行之前，这有什么用呢，举例说明：

```bash
$ echo -e "a\na\na\na"|sed -n '1,/a/p'     #这种情况"/a/"会跳过第一行，不会对第一行进行匹配
a
a
$ echo -e "a\na\na\na"|sed -n '0,/a/p'     #这样"/a/"就会对第一行进行匹配了
a
```


<br />
<b>$        </b>  
匹配最后一行。sed默认将所有文件内容看成一个连续的流，但是如果加了`-s`选项则将每个文件分割成单独的流，导致`1`和`$`会匹配每个流的第一行和最末一行

```bash
$ cat a
1
2
3
$ cat b
a
b
c
$ sed -n '1p;$p' a b
1
c
$ sed -n -s '1p;$p' a b
1
3
a
c
```


<br />
<b>/regex/          </b>  
选择任何匹配`regex`正则的行进行处理


<br />
<b>\%regex%     </b>  
作用和`/regex/`一样，但是如果`regex`中带了大量的"/"字符的话，就不用每个"/"字符都转义了，%可以换成其他分隔符

```bash
$ cat a
/tmp
/usr/local
/root/scr
$ sed '/\/usr\/local/d' a
/tmp
/root/scr
$ sed '\%/usr/local%d' a          #不用转义"/"字符了
/tmp
/root/scr
```


<br />
<b>/REGEXP/I</b>  
<b>\%REGEXP%I     </b>  
大写的`i`字符，作用是前面的正则将不区分大小写匹配

```bash
$ cat a
aaa
AAA
$ sed -n '/a/p' a
aaa
$ sed -n '/a/Ip' a
aaa
AAA
```


<br />
<b>/REGEXP/M</b>  
<b>\%REGEXP%M          </b>  
多行匹配，让`^`能够匹配换行符之后的空和`$`能够匹配到换行符之前的空，举例说明：

```bash
$ seq 2|sed 'N;s/^\w*$/match/g'    
1
2
$ seq 2|sed 'N;s/^\w*$/match/Mg'         
match
match
```
执行`N`后，模式空间内容为"1\n2"，然后`^\w*$`分两次匹配，第一次匹配1(这里`$`就匹配\n前面的空)，第二次匹配2(这里`^`就匹配\n之后的空)，所以替换成功。再举一个sed去重的例子：

```bash
$ cat a
1111
1111
1111
2222
abc
ddd
abc
2222
3333
$ sed -r ':1;N;s/^([^\n]+)((\n.*)*)\n\1$/\1\2/M;b1' a
1111
2222
abc
ddd
3333
```
掌握这种用法后瞬间高大上有木有


<br />
<b>ADDR1,+N          </b>  
匹配ADDR1和之后的N行

```bash
$ seq 10|sed -n '3,+4p'
3
4
5
6
7
```


<br />
<b>ADDR1~N          </b>  
这里N表示步长，从匹配ADDR1的行开始间隔N行输出

```bash
$ seq 9|sed -n '1~2p'          #打印奇数行
1
3
5
7
9
$ seq 9|sed -n '0~2p'          #打印偶数行
2
4
6
8
```


<br />
<b>ADDR1,~N          </b>  
匹配ADDR1且直到首个能被N整除的行

```bash
$ seq 9|sed -n '2,~4p'          #打印从第二行开始直到首个能被4整除的行
2
3
4
```
