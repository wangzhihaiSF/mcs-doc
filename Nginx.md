## Nginx
Keepalived的作用是检测web服务器的状态，如果有一台web服务器死机，或工作出现故障，Keepalived将检测到，并将有故障的web服务器从系统中剔除，当web服务器工作正常后Keepalived自动将web服务器加入到服务器群中，这些工作全部自动完成，不需要人工干涉，需要人工做的只是修复故障的web服务器。

Keepalived是一个基于VRRP协议来实现的WEB 服务高可用方案，可以利用其来避免单点故障。一个WEB服务至少会有2台服务器运行Keepalived，一台为主服务器（MASTER），一台为备份服务器（BACKUP），但是对外表现为一个虚拟IP，主服务器会发送特定的消息给备份服务器，当备份服务器收不到这个消息的时候，即主服务器宕机的时候，备份服务器就会接管虚拟IP，继续提供服务，从而保证了高可用性。

下面是 keepalived 的搭建过程:
两台机器ip分别为172.16.21.23和172.16.21.24,创建一个公共虚拟ip 172.16.21.79.

**ip 都是测试数据，真正的ip 需要替换**

1. 准备编译环境

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
2. 用来进行nginx是否存活的监测，并设置chmod +x check_nginx.sh
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


3. keepadlived主配置文件
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

4. 备机的配置文件与master区别：

```
......

......

state BACKUP    #主机为MASTER，备用机为BACKUP


...

priority 100
```

5. ip漂移测试

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



*创建文件keepalived.conf
```
touch keepalived.conf
```
编辑keepalived.conf,添加如下内容:
（1）172.16.21.23（主）
```
vrrp_script chk_mariadb {
    script "/data/script/chk_mariadb.sh"
    interval 2
    weight 2
}
###########################################################
# define  mariadb_01                                           #
###########################################################
vrrp_instance V_mariadb_01 {
    interface eth0
    state MASTER
    priority 180
    virtual_router_id 53
    garp_master_delay 1


    authentication {
        auth_type PASS
        auth_pass 111
    }
    track_interface {
       eth0
    }
    virtual_ipaddress {
        172.16.21.79   dev eth0 label eth0:1
    }
    track_script {
        chk_mariadb
    }
}
```

（2）172.16.21.24（从）：
```vrrp_script chk_mariadb {
    script "/data/script/chk_mariadb.sh"
    interval 2
    weight 2
}
 
 
###########################################################
# define  mariadb_02                                           #
###########################################################
vrrp_instance V_mariadb_02 {
    interface eth0
    state BACKUP
    priority 150
    virtual_router_id 53
    garp_master_delay 1
 
 
    authentication {
        auth_type PASS
        auth_pass 222
    }
    track_interface {
       eth0
    }
    virtual_ipaddress {
        172.16.21.79   dev eth0 label eth0:2
    }
    track_script {
        chk_mariadb
    }
}
```

**注：上面配置中提到的/data/script/chk_azkaban.sh需要独立建立(每台机器)，代码如下：**
```
#!/bin/bash 
STATUS=`netstat -nptl | grep mysql | wc -l`  
if [ "$STATUS" -eq "0" ]; then
   killall keepalived 
fi   
```


### nginx必须要双机 使用浮动ip
安装和运行 keepalived 及安装 fuser
```

[root@lb-node1 /]# yum install -y keepalived
[root@lb-node1 /]# yum install -y psmisc
[root@lb-node1 ~]# systemctl start keepalived.service
[root@lb-node1 ~]# systemctl enable keepalived.service
```
