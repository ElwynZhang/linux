### HAProxy1.8.20版本编译安装   
#### 一、官网下载版本并编译安装
```  
[root@node2-centos7 haproxy]# cd /usr/local/src/ #此目录下一般放自己编译的源码  
[root@node2-centos7 src]# yum install gcc gcc-c++ glibc glibc-devel pcre pcre-devel openssl  openssl-devel systemd-devel net-tools vim iotop bc  zip unzip zlib-devel lrzsz tree screen lsof tcpdump wget ntpdate
#安装基础依赖包，必须安装system-devel，如果不安装会报错  
[root@node2-centos7 src]# ll
-rw-r--r--  1 root root 2083917 Jun  2 09:08 haproxy-1.8.20.tar.gz #安装包软件
[root@node2-centos7 src]# tar xvf haproxy-1.8.20.tar.gz #解压  
[root@node2-centos7 src]# cd haproxy-1.8.20/
[root@node2-centos7 haproxy-1.8.20]# ll
total 8724
-rw-rw-r--  1 root root  513114 Apr 25 21:59 CHANGELOG
drwxrwxr-x 18 root root     273 Apr 25 21:59 contrib
-rw-rw-r--  1 root root   41508 Apr 25 21:59 CONTRIBUTING
drwxrwxr-x  5 root root    4096 Apr 25 21:59 doc
drwxrwxr-x  2 root root    4096 Jun  4 02:55 ebtree
drwxrwxr-x  3 root root    4096 Apr 25 21:59 examples
-rwxr-xr-x  1 root root 8259352 Jun  4 02:56 haproxy #可执行的文件
drwxrwxr-x  6 root root      60 Apr 25 21:59 include
-rw-rw-r--  1 root root    2029 Apr 25 21:59 LICENSE
-rw-rw-r--  1 root root    3087 Apr 25 21:59 MAINTAINERS
-rw-rw-r--  1 root root   37713 Apr 25 21:59 Makefile
-rw-rw-r--  1 root root   15355 Apr 25 21:59 README #读一下安装说明
drwxrwxr-x  5 root root      50 Apr 25 21:59 reg-tests
-rw-rw-r--  1 root root    2713 Apr 25 21:59 ROADMAP
drwxrwxr-x  2 root root     101 Apr 25 21:59 scripts
drwxrwxr-x  2 root root    8192 Jun  4 02:56 src
-rw-rw-r--  1 root root      14 Apr 25 21:59 SUBVERS
drwxrwxr-x  2 root root    4096 Apr 25 21:59 tests
-rw-rw-r--  1 root root      24 Apr 25 21:59 VERDATE
-rw-rw-r--  1 root root       7 Apr 25 21:59 VERSION  
[root@node2-centos7 haproxy-1.8.20]# cat README
[root@node2-centos7 haproxy-1.8.20]# make ARCH=x86_64 TARGET=linux2628 USE_PCRE=1 USE_OPENSSL=1 USE_ZLIB=1 USE_SYSTEMD=1  
[root@node2-centos7 haproxy-1.8.20]# make install PREFIX=/usr/local/haproxy  
[root@node2-centos7 haproxy-1.8.20]# cp haproxy /usr/sbin  
```
haproxy主程序：/usr/sbin/haproxy
#### 二、准备启动脚本  
```  
[root@node2-centos7 haproxy-1.8.20]# vim /usr/lib/systemd/system/haproxy.service  
[Unit]
Description=HAProxy Load Balancer
After=syslog.target network.target  

[Service]
ExecStartPre=/usr/sbin/haproxy -f /etc/haproxy/haproxy.cfg -c -q
ExecStart=/usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid
ExecReload=/bin/kill -USR2 $MAINPID  

[Install]
WantedBy=multi-user.target  
```
haproxy启动文件：/usr/lib/systemd/system/haproxy.service 

#### 三、定义haproxy配置文件  
```
[root@node2-centos7 haproxy-1.8.20]# mkdir /etc/haproxy
[root@node2-centos7 haproxy-1.8.20]# cd /etc/haproxy/  
[root@node2-centos7 haproxy]# vim haproxy.cfg haproxy配置文件编写
global #全局配置段 进程及安全相关的参数配置 
maxconn 100000  #每个haproxy进程的最大并发连接数  
chroot /usr/local/haproxy #锁定运行目录为/usr/local/haproxy（好处是当我们的软件或者haproxy由于bug问题被一些非法用户拿到控制权以后，他拿到权限的目录也仅限于此锁定的目录，不会让它跳到我们系统上的其他目录，这是提供安全的一种方式） 
#stats socket /var/lib/haproxy/haproxy.sock mode 600 level admin #指定haproxy运行时的socket文件文件的存放位置，haproxy支持通过socket动态的开和关闭的功能，此方式仅限于本机交互的一种方式;可以通过socket来传递一些参数或者传递一些命令，让haproxy把一些项动态的关闭和动态的上线；就不用像nginx 一样，把一个服务挂上去，还要写好配置文件，还要把nginx reload一下，这样配置的新的服务才能生效；
uid 99 #工作进程的uid,gid;
gid 99 #运行haproxy的用户身份,默认的是99，nobody用户
daemon #以守护进程的方式运行
nbproc 2 #开启的haproxy进程数，与CPU保持一致  
nbthread #指定每个haproxy进程开启的线程数，默认为每个进程一个线程（此参数只在新版本1.8及以上支持）
cpu-map 1 0  #绑定第一个haproxy 进程到CPU的第0个核心上
cpu-map 2 1  #把haproxy第二个进程绑定到cpu的第1个核心上
#cpu-map 3 2
#cpu-map 4 3
pidfile /usr/local/haproxy/run/haproxy.pid  #指定pid文件路径，安装时也可能放在/run/haproxy.pid下，注意到保证和启动文件中启动时指定的路径保持一致，否则reload服务时会报错；
log 127.0.0.1 local3 info  
maxconnrate #每个进程每秒最大连接数  
maxsslconn	#SSL每个haproxy进程ssl最大连接数  
spread-checks	#后端server状态check随机提前或延迟百分比时间，建议2-5(20%-50%)之间  

defaults  
option redispatch	#当server Id对应的服务器挂掉后，强制定向到其他健康的服务器  
option abortonclose	#当服务器负载很高的时候，自动结束掉当前队列处理比较久的链接  
option http-keep-alive #开启会话保持
option  forwardfor #开启IP透传，当用户的IP地址经过haproxy后，haproxy把用户的源地址加到请求头部里边，把它透传给后端的服务器，会带有forwardfor标志的就是传到后端的用户源地址的标志；
maxconn 100000 #每个haproxy进程的最大并发连接数
mode http #指定负载协议类型，设置haproxy默认的转发模式，如果我们的后端服务器里边没有配置转发模式的话，那就会继承defaults中的设置项； 
timeout connect 300000ms  #转发客户端请求到后端server的最长连接时间(TCP之前)（意思为haproxy把用户的请求往后端服务器转发，多长时间内没有连接上，则就会报错）
timeout client  300000ms #与客户端的最长空闲时间，表示用户在此设定的时间内再次访问就不用和tcp再次进行链接，可以直接进行访问
timeout server  300000ms 转发客户端请求到后端服务端的超时超时时长（TCP之后）（意思为用户的请求已经转发到后端的服务器上了，但是后端服务器多长时间内没有给用户返回数据，那就认为此服务器挂掉了）  
timeout http-keep-alive 600s #session会话保持时间，时间范围内转发到相同的后端服务器上   
timeout check 5s #对后端服务器的单次检测超时时长，如果检测的服务器在5s之内没有数据，则认为第一次检测失败

listen stats
 mode http
 bind 0.0.0.0:9999
 stats enable
 log global
 stats uri     /haproxy-status
 stats auth    haadmin:q1w2e3r4ys

listen  web_port #可以使用listen替换frontend和backend的配置方式
 bind 0.0.0.0:8080
 mode http
 log global
 server web1  127.0.0.1:8080  check inter 3000 fall 2 rise 5

frontend web #前端servername，类似于Nginx的一个虚拟主机 server
 bind 192.168.28.18:80 	#bind指定HAProxy的监听地址，可以是IPV4或IPV6，可以同时监听多个IP或端口，监听多个地址时中间用逗号分隔，同时也适用于listen片段中；
 use_backend web_host #use_backend调用的后端服务器组名称，要保证前后的一致性

backend web_host #后端服务器组，等于nginx的upstream
 server web1 192.168.28.7:80 #定义后端真正的服务器
# server web2 192.168.28.38:80  #没有启用这组后端服务器
```
haproxy主配置文件：/etc/haproxy/haproxy.cfg   
注意：以上name字段只能使用 -  _  .  : 并且严格区分大小写；  
listen	<name>	将frontend和backend合并在一起配置；

#### 四、启动服务  
```  
[root@node2-centos7 haproxy]# systemctl start haproxy
[root@node2-centos7 haproxy]# ss -ntl
State      Recv-Q Send-Q      Local Address:Port                     Peer Address:Port              
LISTEN     0      128                     *:9999                                *:*                  
LISTEN     0      128                     *:111                                 *:*                  
LISTEN     0      128         192.168.28.18:80                                  *:*                    
#监听的端口
LISTEN     0      128                     *:8080                                *:*                  
LISTEN     0      128                     *:22                                  *:*                  
LISTEN     0      100             127.0.0.1:25                                  *:*                  
LISTEN     0      128                    :::111                                :::*                  
LISTEN     0      128                    :::22                                 :::*                  
LISTEN     0      100                   ::1:25                                 :::*    

```
以上完成了整个haproxy的安装编译  
#### 五、验证haproxy  
```  
[root@node2-centos7 ~]# ps -ef |grep haproxy
root       7511      1  0 03:35 ?        00:00:00 /usr/sbin/haproxy -Ws -f /etc/haproxyhaproxy.cfg -p /run/haproxy.pid
nobody     7514   7511  0 03:35 ?        00:00:00 /usr/sbin/haproxy -Ws -f /etc/haproxyhaproxy.cfg -p /run/haproxy.pid
nobody     7515   7511  0 03:35 ?        00:00:00 /usr/sbin/haproxy -Ws -f /etc/haproxyhaproxy.cfg -p /run/haproxy.pid
root       7751   6740  0 06:48 pts/0    00:00:00 grep --color=auto haproxy  
```
#### 六、测试haproxy的配置是否成功  
在192.168.28.7的机器上进行测192.168.28.18的地址（测试时确保192.168.28.7主机安装了http并正常开启，在/var/www/html/index.html下有自己建立的内容能正常访问到就测试成功）  
  ![测试成功](https://elwyn.oss-cn-beijing.aliyuncs.com/linux/20190604145402.png)




