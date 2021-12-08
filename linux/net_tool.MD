#ctl网络工具

### 1、nslookup
nslookup (全称 name server lookup) ，是一个在命令行界面下的网络工具，它有两种模式: 交互 & 非交互，进入交互模式在命令行界面直接输入nslookup按回车，非交互模式则是后面跟上查询的域名或者 IP 地址按回车。一般来说，非交互模式适用于简单的单次查询，若需要多次查询，则交互模式更加适合
学习链接：https://zhuanlan.zhihu.com/p/361451835
10 most used Nslookup commands https://www.cloudns.net/blog/10-most-used-nslookup-commands/

### 2、nc
学习链接：https://cloud.tencent.com/developer/article/1493030
使用nc命令作为客户端工具进行端口探测
```shell
nc -vz -w 2 10.0.1.161 9999
```
（-v可视化，-z扫描时不发送数据，-w超时几秒，后面跟数字）