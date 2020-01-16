---
layout: post
title: "grep sed awk使用"
date: 2020-01-16 10:40:30 +0800
catalog: ture
multilingual: false
tags:
    - shell
---

### grep

#### pgrep

**pgrep，可以迅速定位包含某个关键字的进程的pid；使用这个命令，再也不用ps aux 以后去对哪个进程的pid了**

```bash
# pgrep -l etcd
2840 etcd
```

### sed

```bash
在每行的头添加字符，比如"HEAD"，命令如下：
sed 's/^/HEAD&/g' test.file

在每行的行尾添加字符，比如“TAIL”，命令如下：
sed 's/$/&TAIL/g' test.file

打印第一行到第三行
sed -n '1,3p' file1

行替换
sed -i '/xxx/s/aaa/fff/g' file    --表示针对文件，找出包含xxx的行，并将其中的aaa替换为fff
sed '/ids1.deliver.com/s/ids1.deliver.com/local.ids1.deliver.com/' iis.conf 
#ids1.deliver.com 后面插入一行
sed -i '/ids1.deliver.com/a\ \ \ \ \ \ \ \ include proxy.conf;' /usr/local/openresty/nginx/conf/conf.d/iis.conf

 sed -i '/daemonize/a slaveof 10.80.130.171 6379' 6379.conf

#匹配行之后替换
sed -i '/discovery.zen.ping.unicast.hosts/ c discovery.zen.ping.unicast.hosts: ["10.90.2.240", "10.90.2.241","10.90.2.242","10.80.81.150","10.80.82.150","10.80.83.150"]' elasticsearch.yml



sed 中有变量，使用双引号
host=`hostname`
hang=`cat /etc/hosts | grep -n $host | awk -F: '{print $1}'| sed -n '2p'`
echo $hang
sed -i "${hang}d" /etc/hosts
```



### AWK

#### 参数

```bash
NR 行数
NF 每行数据字段的个数(以空行为分隔符)
FNR 数据文件的当前记录号
FS 输入分隔符    OFS 输出分隔符
RS：输入记录分隔符，指定输入时的换行符，原换行符仍有效  awk -v RS=" " '{print }' /etc/passwd  表示以空格为换行符
ORS：输出记录分隔符，输出时用指定符号代替换行符  awk -v RS=" " -v ORS='###' '{print }' /etc/passwd 相当于把换行符的空格替换成###
选项：-F 指明输入时用到的字段分隔符   -v var=value: 自定义变量
print输出指定内容后换行，printf只输出指定内容后不换行
```

#### 多个分隔符

```bash
tail -n3 /etc/services |awk -F'[/#]' '{print $3}'
```



#### 变量

```bash
awk -v a=123 'BEGIN{print a}'

# 或使用单引号
a=1234
awk 'BEGIN{print '$a'}'
```



#### 正则匹配

```bash
tail /etc/services |awk '/tcp/ {print $0}'
tail /etc/services |awk '/^tcp/ {print $0}'
tail /etc/services |awk '/blp5/ && /tcp/ {print $0}'

#不匹配开头是#和空行
awk	' ! /^#/ && ! /^$/{print $0}' /etc/hosts
awk	' ! /^#|^$/' /etc/hosts
awk '/^[^#]|"^$"/' /etc/hosts
```

#### RS和ORS

RS 输入记录分隔符，默认是换行符\n

ORS 输出记录分隔符，默认是换行符\n

```bash
# echo "www.baidu.com/user/test.html" |awk 'BEGIN{RS="/"}{print $0}'
www.baidu.com
user
test.html

seq 10 |awk 'BEGIN{ORS="+"}{print $0}'
1+2+3+4+5+6+7+8+9+10+
```

#### ARGC ARGV

ARGC是命令行参数数量

ARGV是将命令行参数存到数组，元素由ARGC指定，数组下标从0开始

```bash
# awk 'BEGIN{print ARGC}' 1 2 3
4
# awk 'BEGIN{print ARGV[0]}'
awk
# awk 'BEGIN{print ARGV[1]}' 1 2
1
# awk 'BEGIN{print ARGV[2]}' 1 2
2
```

#### 空行

```bash
cat file | awk '{if($0~/^$/)print NR}' #空行所在的行数 grep -n ^$ file | awk -F : '{print $1}'
```

#### 数组

```bash
awk '{a[$1]++}END{for(i in a)print a[i],i|"sort -k1 -nr|head -n10"}' access_log 

# 说明：a[$1]++ 创建数组a，以第一列作为下标，使用运算符++作为数组元素，元素初始值为0。处理一个IP时，下标是IP，元素加1，处理第二个IP时，下标是IP，元素加1，如果这个IP已经存在，则元素再加1，也就是这个IP出现了两次，元素结果是2，以此类推。因此可以实现去重，统计出现次数

[root@block1 test]# cat file
http://www.etiantian.org/index.html
http://www.etiantian.org/1.html
http://post.etiantian.org/index.html
http://mp3.etiantian.org/index.html
http://www.etiantian.org/3.html
http://post.etiantian.org/2.html
[root@block1 test]# awk -F "/+" '{hotel[$2]++}END{for(pol in hotel)print pol,hotel[pol]}' file | sort -rnk2|column -t
www.etiantian.org   3
post.etiantian.org  2
mp3.etiantian.org   1

netstat -tunlp | awk '/^udp/ {S[$NF]++} END {for (a in S) print a,S[a]}'
```

#### others

```bash
kubectl  get pod --all-namespaces  | awk '{if($5 > 5){print $0}}'
```

### 参考

[Shell文本处理三剑客之awk](https://blog.51cto.com/lizhenliang/1892112)




