最近发现服务器有点问题，隔一天就发现有些协议失败了，查看了下 nohup.out 文件中记录的日志信息，原来是 mysql 连接失效的问题。看了下 mysql 默认的超时时间了 8 小时。果断该的大一点，主要是这个服务器我们处理消息不是很频繁，有些时候很长时间不处理消息。

我这里设置为了 8*24 小时
```shell
mysql -u root -p
show global variables like '%timeout%';
set global wait_timeout = 691200; 
set global interactive_timeout = 691200;
show global variables like '%timeout%';
quit
```

**interactive_timeout:**

参数含义：服务器关闭交互式连接前等待活动的秒数。

**wait_timeout:**

参数含义：服务器关闭非交互连接之前等待活动的秒数。

对于wait_timeout的值设定，应该根据系统的运行情况来判断。在系统运行一段时间后，可以通过show processlist命令查看当前系统的连接状态，如果发现有大量的sleep状态的连接进程，则说明该参数设置的过大，可以进行适当的调整小些。

参考：

http://www.jb51.net/article/23781.htm

http://www.xshell.net/database/wait_timeout_interactive_timeout.html

http://www.cnblogs.com/jiunadianshi/articles/2475475.html

http://cenalulu.github.io/mysql/mysql-timeout/



