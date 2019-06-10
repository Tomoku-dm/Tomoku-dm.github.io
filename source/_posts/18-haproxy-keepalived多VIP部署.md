---
title: haproxy+keepalived 多 VIP 部署
date: 2019-05-24 01:36:37
tags:
 - linux
 - service
---
## 一、前言

目前需求如下：  

公司有10台带图形化的虚拟机供100名左右的用户使用，所有用户通过一个域名或者 IP 连接，平台随机分配一个机器给用户使用。

**实现方式：**  

平台使用两台 haproxy 进行负载均衡分配机器，如果配置单 VIP ，两台haproxy 只有一台机器工作，压力有些大。 两台机器部署 keepalived 并且部署双 VIP，保证两台机器同时工作，使用 DNS 做双VIP 的轮询。一台机器宕机后，双 VIP 飘到一个机器上。

流程图如下（图中只花了5个后端server）：

![](https://raw.githubusercontent.com/Tomoku-dm/blog-images/master/18-liuchengtu.jpg)

## 二、软件安装及相关配置文件

```shell
$ yum install haproxy
$ vim /etc/haproxy/haproxy.cfg             ## 主配置文件

$ yum install keepalived
$ vim /etc/keepalived/keepalived.conf      ## 主配置文件
$ vim /etc/keepalived/chk_SERVICE.sh       ## keepalived.conf中自定义的服务健康检查脚本
```



## 三、Haporxy 配置

1. haproxy.cfg  主文件配置 ，两台机器的 haproxy （172.20.35.5 、172.20.35.11）  配置相同即可

```yml
$ vim /etc/haproxy/haproxy.cfg
#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    log         127.0.0.1 local2
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    quiet
    nbproc	1
    daemon
    stats socket /var/lib/haproxy/stats           ## turn on stats unix socket

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close                      ## 后端为动态应用程序建议使用http-server-close，后端为静态建议使用http-keep-alive 
    option forwardfor       except 127.0.0.0/8    ## 如果后端服务器需要获得客户端的真实ip，需要配置的参数，记录客户端IP在X-Forwarded-For头域中
    option                  redispatch
    retries                 3
    timeout http-request    30s                   ## 从连接创建开始到从客户端读取完整HTTP请求的超时时间，用于避免类DoS攻击
    timeout queue           1m                    ## 请求在队列中排隊的最大时长
    timeout connect         30s                   ## haproxy和服务端建立连接的最大时长，设置为1秒就足够了。局域网内建立连接一般都是瞬间的
    timeout client          1m                    ## 和客户端保持空闲连接的超时时长，在高并发下可稍微短一点，可设置为10秒以尽快释放连接
    timeout server          1m                    ## 和服务端保持空闲连接的超时时长，局域网内建立连接很快，所以尽量设置短一些，特别是并发时，如设置为1-3秒    
    timeout http-keep-alive 30s
    timeout check           30s
    maxconn                 3000
########统计页面配置########
listen status 
	bind 0.0.0.0:9999
	mode http
        option httplog
	option dontlognull
        option logasap
	option forwardfor
        option httpclose
	stats enable
        stats uri /status                           ## 访问监控页面的uri
	stats auth iflytek:iflytek
        stats realm (Haproxy\ statistic)

########后端访问机器配置#######   ## 一共 10 台机器，我们访问图形化的工具使用的是 4000 端口
listen nomachine
bind 0.0.0.0:4000
balance leastconn                                   ## 选择使用人数最少的机器进行连接
mode   tcp

server nomachine5 172.20.9.5:4000 check inter 5000 fall 3 
server nomachine6 172.20.9.6:4000 check inter 5000 fall 3 
server nomachine17 172.20.9.17:4000 check inter 5000 fall 3 
server nomachine22 172.20.9.22:4000 check inter 5000 fall 3 
server nomachine23 172.20.9.23:4000 check inter 5000 fall 3 
server nomachine24 172.20.9.24:4000 check inter 5000 fall 3 
server nomachine13 172.20.9.13:4000 check inter 5000 fall 3 
server nomachine8  172.20.9.8:4000 check inter 5000 fall 3 
server nomachine28 172.20.9.28:4000 check inter 5000 fall 3 
server nomachine15 172.20.9.15:4000 check inter 5000 fall 3 
```

2. haproxy 配置完成验证 ，从结果可看到 两台机器的 4000 端口都可以按照规则转发到后端的机器上。

   > 这里要想看效果，需要提前在后端的 10 台机器配置 httpd 服务并配置index 页面，这里省略。

    ```shell
    $ curl 172.20.35.5:4000
    this is nomachine5
    $ curl 172.20.35.5:4000
    this is nomachine6
    
    
    $ curl 172.20.35.11:4000
    this is nomachine22
    $ curl 172.20.35.11:4000
    this is nomachine23
    ```

## 三、单 VIP keepalived 部署

1. **Master**   keepalived  配置

   ```yml
   $ vim /etc/keepalived/keepalived.conf
   ! Configuration File for keepalived
   
   global_defs {
      notification_email {
        acassen@firewall.loc
      }
      notification_email_from Alexandre.Cassen@firewall.loc
      smtp_server 192.168.200.1
      smtp_connect_timeout 30
      router_id LVS_DEVEL
   }
   
   vrrp_script chk_haproxy {
   	script "/etc/keepalived/chk_haproxy.sh"      ## 自定义服务检测脚本位置，脚本一定要有可执行权限
   	interval 4
   	weight -120                                   ## 当脚本返回值为复数时对 priority 做加法，如果 weight 是负数，priority 会越来与越小，小于 backup 的 priority 时 ，vip 会漂移过去。
   	}
   
   vrrp_instance VI_1 {
       state MASTER
       interface eth0
       virtual_router_id 55                         ## 在同一网段下，master 和 backup 必须相同，且不和其他机器冲突。
       priority 200                                 ## 决定谁拿 vip
       advert_int 1
       authentication {
           auth_type PASS
           auth_pass 8888
       }
       virtual_ipaddress {
           172.20.22.230                            ## 虚拟IP 地址
       }
   }
   
   ```

2. **Backup** keepalived  配置

   ```yml
   $ vim /etc/keepalived/keepalived.conf
   ! Configuration File for keepalived
   
   global_defs {
      notification_email {
        acassen@firewall.loc
      }
      notification_email_from Alexandre.Cassen@firewall.loc
      smtp_server 192.168.200.1
      smtp_connect_timeout 30
      router_id LVS_DEVEL
   }
   
   vrrp_script chk_haproxy {
   	script "/etc/keepalived/chk_haproxy.sh"
   	interval 4
   	weight -120
   	}
   
   vrrp_instance VI_1 {
       state BACKUP
       interface eth0
       virtual_router_id 55                         ## 必须和 master 的一致
       priority 100                                  ## 如果这个值高于 200 会抢来vip，如果相等，master 持有vip
       advert_int 1
       authentication {
           auth_type PASS
           auth_pass 8888
       }
           track_script {
   	chk_haproxy
   	}
       virtual_ipaddress {
           172.20.22.230
       }
   }
   ```

   

## 四、双 VIP keepalived 部署

双 VIP 其实就是双主模式，这里没有谁是绝对的 master 或者 backup，两个机器互相都是 master 和 backup。

1. **230 Master** keepalived  配置

   ```yml
   $ vim /etc/keepalived/keepalived.conf
   ! Configuration File for keepalived
   
   global_defs {
      notification_email {
        acassen@firewall.loc
      }
      notification_email_from Alexandre.Cassen@firewall.loc
      smtp_server 127.0.0.1
      smtp_connect_timeout 30
      router_id LVS_DEVEL
   }
     
   vrrp_script chk_haproxy {
   	script "/etc/keepalived/chk_haproxy.sh"
   	interval 4
   	weight -120
   	}
   
   vrrp_instance VI_1 {
       state MASTER
       interface eth0
       virtual_router_id 55                    ## 这里和另一太机器的BACKUP 要一致
       priority 200
       nopreempt
       advert_int 1
       authentication {
           auth_type PASS
           auth_pass 8888
       }
       virtual_ipaddress {
           172.20.22.230
       }
   }
   
   vrrp_instance VI_2 {
       state BACKUP
       interface eth0
       virtual_router_id 56                   ## 这里同样和另一太机器的MASTER 要一致,并且和上面不同
       priority 100
       advert_int 1
       authentication {
           auth_type PASS
           auth_pass 8888
       }
       virtual_ipaddress {
           172.20.22.231
       }
   }
   
   ```

   

2. **231 Master** keepalived  配置

   ```yml
   $ vim /etc/keepalived/keepalived.conf
   ! Configuration File for keepalived
   
   global_defs {
      notification_email {
        acassen@firewall.loc
      }
      notification_email_from Alexandre.Cassen@firewall.loc
      smtp_server 127.0.0.1
      smtp_connect_timeout 30
      router_id LVS_DEVEL
    }
     
   vrrp_script chk_haproxy {
   	script "/etc/keepalived/chk_haproxy.sh"
   	interval 4
   	weight -120
   }
   
   vrrp_instance VI_1 {
       state BACKUP
       interface eth0
       virtual_router_id 55
       priority 100
       advert_int 1
       authentication {
           auth_type PASS
           auth_pass 8888
       }
       virtual_ipaddress {
           172.20.22.231
       }
    }
   
   vrrp_instance VI_2 {
       state MASTER
       interface eth0
       virtual_router_id 56
       priority 200
       advert_int 1
       nopreempt
       authentication {
           auth_type PASS
           auth_pass 8888
       }
       virtual_ipaddress {
           172.20.22.230
       }
    }
    ```
    

3. 检验 vip 漂移情况

   ```yml
   ## 在 172.20.35.5 上执行
   $ systemctl stop keepalived
   
   ## 在 172.20.35.11 上执行，可以看到 35.5 上的vip 已经漂移过来了
   $ ip a
   1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN 
       link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
       inet 127.0.0.1/8 scope host lo
       inet6 ::1/128 scope host 
          valid_lft forever preferred_lft forever
   2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
       link/ether 00:0c:29:ba:cf:32 brd ff:ff:ff:ff:ff:ff
       inet 172.20.35.11/24 brd 192.168.63.255 scope global eth0
       inet6 fe80::20c:29ff:feba:cf32/64 scope link 
           valid_lft forever preferred_lft forever
       inet 172.20.35.230/24 brd 192.168.63.255 scope global eth0
       	valid_lft forever preferred_lft forever  
       inet 172.20.35.231/24 brd 192.168.63.255 scope global eth0
       	valid_lft forever preferred_lft forever 
   
   ```

   



## 五、keepalived 脚本案例

1. haproxy 服务不能启动就关闭keepalived 服务

   ```yml
   #!/bin/sh
   if [ $(ps -ef |grep "haproxy" grep v 11 grep11 |grep v "chk" |wc -1) -eq 0 ] ; then
   	#/usr/bin/systemctl restart haproxy
   	echo 11/usr/bin/systemctl restart haproxy" > /etc/keepalived/123
   	fi
   	sleep 3
   if [ $(ps ef|grep "haproxy" |grep v "grep" |grep v "chk"|wc -1) -eq 0 ]; then
   	/usr/bin/systemctl stop keepalived
   fi
   ```

2. 上面直接关闭 keepalived 服务过于暴力，下面脚本根据 haproxy 服务情况返回 0/1，在 haproxy 恢复正常后，脚本执行返回0 会增加 priority 夺回 master 的 VIP

   ```yml
   #!/bin/bash
   count = `ps aux | grep -v grep | grep haproxy | wc -l`
   if [ $count > 0 ]; then
       exit 0
   else
       exit 1
   fi
   ```

## 六、keepalived 原理

### Master 选举

keepalived 中优先级高的节点为MASTER。MASTER其中一个职责就是响应VIP的arp包，将VIP和mac地址映射关系告诉局域网内其他主机，同时，它还会以多播的形式（目的地址224.0.0.18）向局域网中发送VRRP通告，告知自己的优先级。

网络中的所有 BACKUP 节点只负责处理 MASTER 发出的多播包，当发现 MASTER 的优先级没自己高，或者没收到 MASTER 的 VRRP 通告时，BACKUP 将自己切换到 MASTER 状态，然后做MASTER该做的事：1.响应arp包，2.发送VRRP通告。

MASTER 和 BACKUP 节点的优先级如何调整？  
首先，每个节点有一个初始优先级，由配置文件中的 priority 配置项指定，MASTER节点的 priority 应比 BAKCUP 高。运行过程中 keepalived 根据 vrrp_script 的weight设定，增加或减小节点优先级。规则如下：

1. 当weight > 0时，vrrp_script script脚本执行返回**0**(成功)时优先级为priority + weight, 否则为priority。当BACKUP发现自己的优先级大于MASTER通告的优先级时，进行主从切换。
2. 当weight < 0时，vrrp_script script脚本执行返回**非0**(失败)时优先级为priority + weight, 否则为priority。当BACKUP发现自己的优先级大于MASTER通告的优先级时，进行主从切换。 3. 当两个节点的优先级相同时，以节点发送VRRP通告的IP作为比较对象，IP较大者为MASTER。


## 七、抓包分析 VIP 浮动
原理我们分析完之后，我们通过 tcpdump 的抓包来看一下 vrrp 协议的包是怎么发出的。

```c
## 首先我们在两台 master 正常工作的情况下，关闭 172.20.35.5 的 haproxy 服务。

15:10:36.353732 IP 172.20.35.5 > 224.0.0.18: VRRPv2, Advertisement, vrid 55, prio 200, authtype simple, intvl 1s, length 20
15:10:36.775313 IP 172.20.35.11 > 224.0.0.18: VRRPv2, Advertisement, vrid 56, prio 200, authtype simple, intvl 1s, length 20
15:10:37.353915 IP 172.20.35.5 > 224.0.0.18: VRRPv2, Advertisement, vrid 55, prio 200, authtype simple, intvl 1s, length 20
15:10:37.775889 IP 172.20.35.11 > 224.0.0.18: VRRPv2, Advertisement, vrid 56, prio 200, authtype simple, intvl 1s, length 20    
15:10:38.354610 IP 172.20.35.5 > 224.0.0.18: VRRPv2, Advertisement, vrid 55, prio 80, authtype simple, intvl 1s, length 20       ##此刻脚本返回失败，prio --> 80
15:10:38.355158 IP 172.20.35.11 > 224.0.0.18: VRRPv2, Advertisement, vrid 55, prio 100, authtype simple, intvl 1s, length 20
15:10:38.355301 IP 172.20.35.5 > 224.0.0.18: VRRPv2, Advertisement, vrid 55, prio 80, authtype simple, intvl 1s, length 20
15:10:38.355729 IP 172.20.35.11 > 224.0.0.18: VRRPv2, Advertisement, vrid 55, prio 100, authtype simple, intvl 1s, length 20     ##35.11 发现自己的prio 更高转为master 发送广播，35.5 不再发送广播
15:10:38.776399 IP 172.20.35.11 > 224.0.0.18: VRRPv2, Advertisement, vrid 56, prio 200, authtype simple, intvl 1s, length 20
15:10:39.356102 IP 172.20.35.11 > 224.0.0.18: VRRPv2, Advertisement, vrid 55, prio 100, authtype simple, intvl 1s, length 20


## 此时 35.5 已经失去了 230 的 VIP 

$ ip a
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether fa:16:3e:04:91:24 brd ff:ff:ff:ff:ff:ff
    inet 172.20.35.5/24 brd 172.20.35.255 scope global dynamic eth0
       valid_lft 40538sec preferred_lft 40538sec
    inet6 fe80::f816:3eff:fe04:9124/64 scope link 
       valid_lft forever preferred_lft forever


## 然后我们重新开启 35.5 的 haproxy 服务，观察 35.5 的 prio -->200 抢回 mastre 角色以及 230 的 VIP


15:15:46.959076 IP 172.20.35.11 > 224.0.0.18: VRRPv2, Advertisement, vrid 56, prio 200, authtype simple, intvl 1s, length 20
15:15:47.454475 IP 172.20.35.11 > 224.0.0.18: VRRPv2, Advertisement, vrid 55, prio 100, authtype simple, intvl 1s, length 20
15:15:47.454724 IP 172.20.35.5 > 224.0.0.18: VRRPv2, Advertisement, vrid 55, prio 200, authtype simple, intvl 1s, length 20
15:15:47.456099 IP 172.20.35.11 > 224.0.0.18: VRRPv2, Advertisement, vrid 55, prio 100, authtype simple, intvl 1s, length 20
15:15:47.456236 IP 172.20.35.5 > 224.0.0.18: VRRPv2, Advertisement, vrid 55, prio 200, authtype simple, intvl 1s, length 20
15:15:47.959700 IP 172.20.35.11 > 224.0.0.18: VRRPv2, Advertisement, vrid 56, prio 200, authtype simple, intvl 1s, length 20


$ ip a
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether fa:16:3e:04:91:24 brd ff:ff:ff:ff:ff:ff
    inet 172.20.35.5/24 brd 172.20.35.255 scope global dynamic eth0
       valid_lft 40418sec preferred_lft 40418sec
    inet 172.20.35.230/32 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fe04:9124/64 scope link 
       valid_lft forever preferred_lft forever
```


## 参考文章

- [keepalived 之 vrrp_script详解](https://www.cnblogs.com/arjenlee/p/9258188.html)




