## Nginx
Keepalived的作用是检测web服务器的状态，如果有一台web服务器死机，或工作出现故障，Keepalived将检测到，并将有故障的web服务器从系统中剔除，当web服务器工作正常后Keepalived自动将web服务器加入到服务器群中，这些工作全部自动完成，不需要人工干涉，需要人工做的只是修复故障的web服务器。

Keepalived是一个基于VRRP协议来实现的WEB 服务高可用方案，可以利用其来避免单点故障。一个WEB服务至少会有2台服务器运行Keepalived，一台为主服务器（MASTER），一台为备份服务器（BACKUP），但是对外表现为一个虚拟IP，主服务器会发送特定的消息给备份服务器，当备份服务器收不到这个消息的时候，即主服务器宕机的时候，备份服务器就会接管虚拟IP，继续提供服务，从而保证了高可用性。

下面是 keepalived 的搭建过程:
两台机器ip分别为172.16.21.23和172.16.21.24,创建一个公共虚拟ip 172.16.21.79.

**ip 都是测试数据，真正的ip 需要替换**

### 准备编译环境

正式开始前，编译环境gcc g++ 开发库之类的需要提前装好
```
yum -y install gcc automake autoconf libtool make
yum install gcc gcc-c++
yum install -y openssl openssl-devel
yum install -y keepalived
yum install -y psmisc
systemctl start keepalived.service
systemctl enable keepalived.service
```
### 用来进行nginx是健康的监测，并设置chmod +x check_nginx.sh
```
[root@lb-node1 ~]# vim /soft/scripts/check_nginx.sh
#!/bin/bash
if [ "$(ps -ef | grep "nginx: master process"| grep -v grep )" == "" ]
then
 /usr/bin/systemctl restart nginx.service  #检测到nginx宕机尝试拉起一次
 sleep 5
 if [ "$(ps -ef | grep "nginx: master process"| grep -v grep )" == "" ]
 then
 killall keepalived     #拉起失败杀死keepalived,备机获取vip
 fi
fi
```


### keepadlived主配置文件
```
[root@lb-node1 ~]# vim /etc/keepalived/keepalived.conf 
! Configuration File for keepalived


vrrp_script chk_nginx {
    script "/soft/scripts/check_nginx.sh"
    interval 10  #每10s检查一次
    weight -20
}
vrrp_instance VI_1 {

      state MASTER

interface eth0

virtual_router_id 51

priority 150 #优先级，主备之间最好相差50

advert_int 1 #心跳间隔（如果1秒没通信,备节点马上接管）

authentication {

auth_type PASS

auth_pass 1111

      }
track_script {
        chk_nginx  
    }

virtual_ipaddress {

192.168.1.100/24

      }

  }


vrrp_instance VI_2 {

    state BACKUP

interface eth0

    virtual_router_id 52

    priority 100

advert_int 1

authentication {

auth_type PASS

auth_pass 1111

    }
  
track_script {
        chk_nginx  
    }

virtual_ipaddress {
        192.168.1.100/24
    }  

}
```

### 备机的配置文件与master区别：

```
......

......

state BACKUP    #主机为MASTER，备用机为BACKUP


...

priority 100
```

### ip漂移测试

```
// ip漂移测试
[root@lb-node1 ~]# ip a |grep eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    inet 192.168.1.11/32 brd 192.168.1.11 scope global noprefixroute eth0
    inet 192.168.1.100/24 scope global eth0
[root@lb-node1 ~]# 

//模拟master故障，此时备机获取192.168.1.100的VIP 
[root@lb-node1 ~]# systemctl stop keepalived.service
[root@lb-node2 ~]# ip a |grep eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    inet 192.168.1.11/32 brd 192.168.1.11 scope global noprefixroute eth0
    inet 192.168.1.100/24 scope global eth0
[root@lb-node2 ~]# 
```


### 在两台Web Server上执行realserver.sh脚本，为lo:0绑定VIP地址192.168.1.100、抑制arp广播

```
#!/bin/bash
    #description: Config realserver
    
    VIP=192.168.1.100
     
    /etc/rc.d/init.d/functions
     
    case "$1" in
    start)
           /sbin/ifconfig lo:0 $VIP netmask 255.255.255.255 broadcast $VIP
           /sbin/route add -host $VIP dev lo:0
           echo "1" >/proc/sys/net/ipv4/conf/lo/arp_ignore
           echo "2" >/proc/sys/net/ipv4/conf/lo/arp_announce
           echo "1" >/proc/sys/net/ipv4/conf/all/arp_ignore
           echo "2" >/proc/sys/net/ipv4/conf/all/arp_announce
           sysctl -p >/dev/null 2>&1
           echo "RealServer Start OK"
           ;;
    stop)
           /sbin/ifconfig lo:0 down
           /sbin/route del $VIP >/dev/null 2>&1
           echo "0" >/proc/sys/net/ipv4/conf/lo/arp_ignore
           echo "0" >/proc/sys/net/ipv4/conf/lo/arp_announce
           echo "0" >/proc/sys/net/ipv4/conf/all/arp_ignore
           echo "0" >/proc/sys/net/ipv4/conf/all/arp_announce
           echo "RealServer Stoped"
           ;;
    *)
           echo "Usage: $0 {start|stop}"
           exit 1
    esac
     
    exit 0
```

### 分别在主从机上执行 sh realserver.sh start 实现负载均衡及高可用集群


```
[root@lb-node1 /soft/scripts]# ip a |grep -E "lo|eth0"
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
        inet 192.168.1.100/32 brd 192.168.1.100 scope global lo:0
    2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
        inet 192.168.1.10/24 brd 192.168.1.255 scope global noprefixroute eth0
        inet 192.168.1.100/24 scope global secondary eth0
        inet6 2409:8a28:8a8:e3c0:b6d2:ec3c:3557:2609/64 scope global noprefixroute dynamic 
    [root@lb-node1 /soft/scripts]# 
```
### Keepalived配置文件详解


```
   inet 10.0.0.11/24 scope global secondary eth0

[root@lb01 ~]# /etc/init.d/keepalived stop #停止Master上Keepalived

[root@lb01 ~]# ip addr|grep 10.0.0.11 #VIP已经从Master端移除


[root@lb02 ~]# ip addr|grep 10.0.0.11 #Backup上Keepalived接管资源

    inet 10.0.0.11/24 scope global secondary eth0   

[root@lb01 ~]# /etc/init.d/keepalived start #启动Master_keepalived

[root@lb01 ~]# ip addr|grep 10.0.0.11 #Master继续接管资源           

    inet 10.0.0.11/24 scope global secondary eth0

1.5.9Keepalived配置文件详解
  1 ! Configuration File for keepalived #注释
  2
  3 global_defs {

  4    notification_email {

  5      acassen@firewall.loc #5-7发邮件给谁
  6
  7
  8    }

  9    notification_email_from Alexandre.Cassen@firewall.lo #发邮件发件人

 10    smtp_server 192.168.200.1 #邮件服务器地址

 11    smtp_connect_timeout 30 #超时时间

 12    router_id Nginx_01 #主备ID不能一样

 13 }
 
 14
 
 15  vrrp_instance VI_1 {  #实例名称(建议不修)

 16     state MASTER #服务器的状态(仅仅是傀儡)

 17     interface eth0 #通信端口

 18     virtual_router_id 51 #实例的ID

 19     priority 150 #优先级，主备之间最好相差50

 20     advert_int 1 #心跳间隔（如果一秒没通信备节点马上接管）

 21     authentication {

 22         auth_type PASS #PASS认证类型，此参数备节点设置和主节点相同

 23         auth_pass 1111 #密码是1111，此参数备节点设置和主节点相同

 24     }

 25     virtual_ipaddress { #vip（可以多个）

 26       10.0.0.11/24 #26-28配置vIP地址，绑定在eth0  因为(interface eth0)

 29     }

 30 }
 ```


q全局定义块部分：主要设置Keepalived的通知机制和标识

1、第4-9行是email通知参数。作用：当LVS发生切换或RS等有故障时，会发邮件报警。这是可选配，notifucation_email指定在keepalived发生事件时，需要发给的email地址，可以有多个，每行一个。

2、smtp_server指定发送邮件的smtp服务器，如果本机开启了sendmail，就可以使用上面默认配置实现邮件发送。

3、第10行是Lvs负载均衡器标示（rote_id）。在一个局域网内，它应该是唯一的。

4、大括号”{}” 用来分隔定义块，因此必须成对出现。如果漏写了，keepalived运行时，不会得到预期的结果。由于定义块内存在嵌套关系，因此很容易遗漏结尾处的花括号，这点要特别注意。


qVRRP定义块

1、第13行为VRRP实例vrrp_instance,每个Vrrp实例可以认为是一个keepalived实例，在配置中VRRP实例可以有多个。

(1)第14行实例状态state.只有Master和Backup两种状态，并且需要大写这些单词。其中MASTER为工作状态。BACKUP为备用状态。当MASTER所在的服务器失效时，BACKUP所在的系统会自动把它的状态有BACKUP变换成MASTER，当失效的MASTER所在的系统恢复时，BACKUP从MASTER恢复到BACKUP状态。

(2)通信接口interface。对外提供服务的网络结构，如eth0,eth1当前主流的服务器有2个或2个以上的网络接口，在选择服务器接口时，一定要搞清楚了。

(3)lvs_sync_daemon_interface。负载均衡器之间的监控接口，类似于HA HeartBeat的心跳线。

(4)第16行为虚拟路由标示virtual_route_id是一致的，同时在整个keepalived内是唯一的。

(5)第17行为优先级priority，这是一个数字，数值愈大，优先级越高。在同一个vrrp_instance里，MASTER的优先级 BACKUP。若MASTER的priority值为150，那么BACKUP的priority只能在149或者跟小的数值(官方建议相差50)。

(6)第18行同步通知间隔advert_int。MASTER与BACKUP负载均衡器之间同步检查的时间间隔，单位为秒。

(7)第19-22行验证authentication.包含验证类型和验证密码。类型主要有PASS、AH两种,通常使用的类型为PASS,据说AH使用时有问题。验证密码为明文，同一vrrp实例MASTER与BACKUP使用相同的密码才能正常通信,这里官方推荐用明文即可。


2、第23-27行为虚拟ip地址virtual_ipaddress。可以配置多个IP地址，每个地址占一行，需要指定子网掩码。

注意：这个ip必须与我们在lvs客户端设定的vip相一致。

### 测试验证Nginx、应用程序是否开机自启动，启动用户是否为非root用户验证

进入安装nginx目录,启动nginx服务

```
cd /usr/local/sbin/
./nginx
```

查看nginx应用启动详情
```
ps aux | grep nginx
```
<p>
<img src="https://images2018.cnblogs.com/blog/567946/201807/567946-20180721164822958-2075047104.png" >
</p>

配置nginx开机自启动
进入到/etc/init.d/目录下，创建文件nginx
```
cd /etc/init.d/
vim nginx
```
nginx文件具体配置信息如下：

```
#!/bin/bash
# nginx Startup script for the Nginx HTTP Server
# it is v.0.0.2 version.
# chkconfig: - 85 15
# description: Nginx is a high-performance web and proxy server.
#              It has a lot of features, but it's not for everyone.
# processname: nginx
# pidfile: /var/run/nginx.pid
# config: /usr/local/nginx/conf/nginx.conf

nginxd=/usr/local/nginx/sbin/nginx        # 你的nginx真实启动文件路径
nginx_config=/usr/local/nginx/conf/nginx.conf    # nginx相关配置文件路径
nginx_pid=/var/run/nginx.pid    
RETVAL=0
prog="nginx"
# Source function library.
. /etc/rc.d/init.d/functions
# Source networking configuration.
. /etc/sysconfig/network
# Check that networking is up.
[ ${NETWORKING} = "no" ] && exit 0
[ -x $nginxd ] || exit 0
# Start nginx daemons functions.
start() {
    if [ -e $nginx_pid ];then
       echo "nginx already running...."
   exit 1
    fi
       echo -n $"Starting $prog: "
       daemon $nginxd -c ${nginx_config}
   RETVAL=$?
   echo
       [ $RETVAL = 0 ] && touch /var/lock/subsys/nginx
   return $RETVAL
}

# Stop nginx daemons functions.
stop() {
    echo -n $"Stopping $prog: "
    killproc $nginxd
    RETVAL=$?
    echo
    [ $RETVAL = 0 ] && rm -f /var/lock/subsys/nginx /var/run/nginx.pid
}

# reload nginx service functions.
reload() {
    echo -n $"Reloading $prog: "
    #kill -HUP `cat ${nginx_pid}`
    killproc $nginxd -HUP
    RETVAL=$?
    echo
}

# See how we were called.
    case "$1" in
start)
    start
    ;;
stop)
    stop
    ;;
reload)
    reload
    ;;
restart)
    stop
    start
    ;;
status)
    status $prog
    RETVAL=$?
    ;;
*)
    echo $"Usage: $prog {start|stop|restart|reload|status|help}"
    exit 1
esac
exit $RETVAL
```

保存nginx文件并赋权
```
chmod 755 nginx  
```
将nginx权限设置为自己可以read、write、exec，其他用户只能有read、exec权限，没有write权限

为nginx加上service相关命令权限
```
chkconfig --add nginx
chkconfig nginx on
```
开启nginx的service命令,校验nginx的service命令是否成功,service nginx start 执行不报错表示nginx已经启动

<p>
<img src="https://images2018.cnblogs.com/blog/567946/201807/567946-20180721163616536-1235232894.png" >
</p>

重启centos服务器再次验证是否nginx已经启动
重启之前service nginx stop停止nginx服务，之后执行reboot，开机之后执行ps aux | grep nginx如果后台显示nginx已经启动，那么表示nginx的安装和开机自启动已经成功配置

### nginx备份与恢复
备份：复制安装目录下的nginx.conf配置文件及其使用include 加载的配置文件
恢复：将备份的配置文件复制到新环境下的nginx的配置文件目录下，使用reload命令重新加载配置文件
