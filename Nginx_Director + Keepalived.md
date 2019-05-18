## Nginx_Director + Keepalived

-Author: zeng

-Email: 1827890013@qq.com

-Github: https://github.com/shenyihan-1

说明：两台做nginx负载均衡机器，在其中主机器出现问题，从 机器顶替，实现高可用（HA）

但是keepalived只能实现机器停止情况，VIP（虚拟ip）主/从交替。如果出现nginx服务出现故障keepalived并不会发挥功能，主机器还是持有VIP但是nginx负载均衡已经失效。整个服务就不能进行，所以考虑到这个要实时监控nginx服务的健康状况。

准备

DR1 ip:10.3.145.4 (master)

DR2 ip:10.3.145.3 (slave/backup)

VIP  10.3.145.5

```shell
[root@master ~]# yum -y install keepalived nginx
[root@master ~]# vim /etc/keepalived/keepalived.conf

! Configuration File for keepalived

global_defs {
   router_id director1         //辅助改为director2
}

vrrp_script check_nginx {
   script "/etc/keepalived/check_nginx_status.sh"
   interval 5
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0             //心跳接口，尽量单独连接心跳
    nopreempt
    virtual_router_id 90       //MASTER,BACKUP一致
    priority 100              //辅助改为50
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass zeng
    }
    
    virtual_ipaddress {
        10.3.145.5
    }

    track_script {
        check_nginx
    }
}
//检测nginx服务健康状态的脚本
[root@master ~]# cat /etc/keepalived/check_nginx_status.sh
#!/bin/bash											        	
/usr/bin/curl -I http://localhost &>/dev/null	
if [ $? -ne 0 ];then									    	
	/etc/init.d/keepalived stop					    	
fi															        	
[root@master ~]# chmod a+x /etc/keepalived/check_nginx_status.sh
[root@master ~]# systemctl start nginx
[root@master ~]# echo "dr1" > /usr/share/nginx/html/index.html
```

从主机操作步骤相同，要主机中间特定参数的修改。



检测 

curl 10.3.145.3

dr2

curl 10.3.145.4

dr1

来检测两台服务的是否服务的正常。

down keepalived

down nginx

curl 10.3.145.5

查看输出结果，现在是那台主机正在工作，实验是否成功

