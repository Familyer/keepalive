### Haproxy_Director + Keepalived 实现调度器HA

-Author: zeng

-Email: 1827890013@qq.com

-Github: https://github.com/shenyihan-1

实验准备 （ip选择在同一网段可以相互ping同就行）

DR1 ip：10.3.145.4

DR2 ip：10.3.145.3

RS1 ip：10.3.145.2

RS2 ip:   10.3.145.6

VIP 10.3.145.5

实现haproxy

DR/主/备上的操作：

```shell
[root@DR ~]# yum -y install haproxy
[root@DR ~]# cp -rf /etc/haproxy/haproxy.cfg{,.bak}
[root@DR ~]# sed -i -r '/^[ ]*#/d;/^$/d' /etc/haproxy/haproxy.cfg
[root@DR ~]# vim /etc/haproxy/haproxy.cfg
global
    log         127.0.0.1 local2
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon
    stats socket /var/lib/haproxy/stats

defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000
    
frontend web
    mode                   	http
    bind                      *:80
    default_backend    httpservers

backend httpservers
    balance roundrobin
    server http1 10.3.145.2:80 maxconn 2000 weight 1  check inter 1s rise 2 fall 2
    server http2 10.3.145.6:80 maxconn 2000 weight 1  check inter 1s rise 2 fall 2

//配置监控
listen stats
    bind                    	*:1314
    stats                   	enable
    stats refresh 		30s
    stats                   	hide-version
    stats uri              	/haproxystats
    stats realm         	Haproxy\ stats
    stats auth           	zeng:123
    stats admin         if TRUE

[root@DR ~]# systemctl start haproxy
[root@DR ~]# chkconfig haproxy on
```

RS上的操作 

```
[root@RS ~]# yum -y install httpd
[root@RS ~]# systemctl start httpd
[root@RS ~]# echo "rs1" > /var/www/html/index.html //另外一台为rs2
```



浏览器访问 http://10.3.145.4:1314/haproxy 输入用户zeng  密码 123 后有整个监控页面说明haproxy已经完成,如果现实禁止连接，请检查防火墙有没有关闭！

[root@RS ~]curl 10.3.145.4  (DR) 

rs1

结果是以上情况，说明就是harpoxy负载均衡也成功了

两台DR实现keepalived调度器HA

```shell
[root@DR ~]# yum -y install keepallived
[root@DR ~]# vim /etc/keepalived/keepalived.conf
! Configuration File for keepalived

global_defs {
   router_id director1               //另一台 director2
}

vrrp_script check_haproxy {
   script "/etc/keepalived/check_haproxy_status.sh"
   interval 5
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    nopreempt
    virtual_router_id 90    //master backup 相同即可
    priority 100            //优先级 backup为50
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass tianyun
    }
    virtual_ipaddress {
        10.3.145.5
    }

    track_script {
        check_haproxy
    }
}

让Keepalived以一定时间间隔执行一个外部脚本，脚本的功能是当Haproxy失败，则关闭本机的Keepalived
a. script

[root@master ~]# cat /etc/keepalived/check_haproxy_status.sh
#!/bin/bash											        	
/usr/bin/curl -I http://localhost &>/dev/null	
if [ $? -ne 0 ];then									    	
	/etc/init.d/keepalived stop					    	
fi															        	
[root@master ~]# chmod a+x /etc/keepalived/check_haproxy_status.sh
```

测试 ： 停掉DR上的keepalived或haproxy服务 VIP 两者的持有情况来判断实验是否已经成功