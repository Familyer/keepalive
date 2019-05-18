## LVS_Director + KeepAlived高可用项目实战

## -Author: zeng

-Email: 1827890013@qq.com

-Github: https://github.com/shenyihan-1



目的：使负载均衡器实现高可用

实验机器准备：
DR1(master) DIP:10.3.145.170
DR2(beakup) DIP:10.3.145.52
VIP:10.3.145.102
RS1 RIP:10.3.145.105
RS2 RIP:10.3.145.125

#### 首先实现LVS DR负载均衡

要求：VIP不能被其他机器占用

DR1,DR2上的步骤：

```shell
[root@DR1 ~]# ip addr add dev ens33 10.3.145.102/32 //配置VIP
[root@DR1 ~]# ip a //查看是否添加成功
[root@DR1 ~]# yum -y install ipvsadm  //RHEL确保LoadBalancer仓库可用

[root@DR1 ~]# ipvsadm -A -t 10.3.145.102:80 -s rr
[root@DR1 ~]# ipvsadm -a -t 10.3.145.102:80 -r 10.3.145.105 -g	
[root@DR1 ~]# ipvsadm -a -t 10.3.145.102:80 -r 10.3.145.125 -g	

[root@DR1 ~]# ipvsadm-save
[root@DR1 ~]# ipvsadm -Ln //查看是否添加成功

```

RS1,RS2上的步骤：

```shell
[root@rs1 ~]# yum -y install httpd ipvsadm

[root@rs1 ~]# ip addr add dev lo 10.3.145.102/32
[root@rs1 ~]# ip a      //查看是否添加成功
[root@rs1 ~]# echo "ip addr add dev lo 192.168.122.100/32" >> /etc/rc.local
[root@rs1 ~]# echo "net.ipv4.conf.all.arp_ignore = 1" >> /etc/sysctl.conf //将信息保存在本机当中
[root@rs1 ~]# sysctl -p
[root@rs1 ~]# echo 1 > /proc/sys/net/ipv4/conf/all/arp_ignore
[root@rs1 ~]# echo 2 > /proc/sys/net/ipv4/conf/all/arp_announce

[root@rs1 ~]# systemctl start httpd
[root@rs1 ~]# echo "rs1" > /var/www/html/index.html //为了测试效果RS2 的值rs2

```

测试是否已经可行：

```shell
[root@client ~]# elinks -dump http://10.3.145.102
[root@client ~]# ab -c 1000 -n 1000 http://10.3.145.102/
[root@client ~]# tcpdump -nni eth0 -e host 10.3.145.102
[root@client ~]# elinks 10.3.145.102 //出行rs1 ，rs2 轮询出现就是对的
```

### 配合keepalived

 (DR1:master/DR2:beakup)主/备调度器安装软件  

DR2:beakup 备keepalived 配置文件看后面备注修改，其他的都是一样的。

做完ipvsadm负载均衡 DR1 ens33 要将 VIP 删除

[root@lDR1-master ~]# ip addr del dev ens33 10.3.145.105/32

```shell
[root@lDR1-master ~]# yum -y install  keepalived
[root@lDR1-master ~]# genhash -s 10.3.145.105 -p 80 -u /index.html
MD5SUM = 66ee606d5019d75f83836eeb295c6b6f      //获取Real Server测试页面的MD5SUM值

[root@lDR1-master ~]# vim /etc/keepalived/keepalived.conf

! Configuration File for keepalived

global_defs {
   router_id lvs-master	            //辅助改为lvs-backup
}

vrrp_instance VI_1 {
    state BACKUP				
    nopreempt                              //不抢占
    interface eth0				            //VIP绑定接口
    mcast src ip 10.3.145.170               //发送组播的源IP，心跳线网卡
    virtual_router_id 80		            //VRID 同一组集群，主备一致  虚拟路由器 MAC 00-00-5E-00-01-{VRID}
    priority 100					            //本节点优先级，辅助改为50
    advert_int 1                            //检查间隔，默认为1s
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        10.3.145.102
    }
}

virtual_server 10.3.145.102 80 {   //LVS配置，可以是fwmark 80
#    delay_loop 6
#    lb_algo rr                                    //LVS调度算法
#    lb_kind DR                                  //LVS集群模式（路由模式）
#    nat_mask 255.255.255.0
#    persistence_timeout 20            //持久性连接
#    protocol TCP                              //健康检查使用的协议
#    sorry_server 2.2.2.2 80             //当所有real server不可用时
  
    real_server 10.3.145.105 80 {
        weight 1
        inhibit_on_failure                  //当该节点失败时，把权重设置为0，而不是从IPVS中删除
        HTTP_GET {						      //健康检查
            url {
              path /index.html
              digest 66ee606d5019d75f83836eeb295c6b6f
            }
            connect_port 80                 //检查的端口
            connect_timeout 3            //连接超时的时间
            nb_get_retry 3                   //重新连接的次数
            delay_before_retry 2         //重连的间隔
        }
    }

    real_server 10.3.145.125 80 {
        weight 1
        inhibit_on_failure
        HTTP_GET {
            url {
              path /index.html
              digest 66ee606d5019d75f83836eeb295c6b6f
            }
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
     }
}

```

测试：主/备都启动，并设置开机自启。

```shell
[root@lDR1-master ~]# chkconfig keepalived on
[root@lDR1-master ~]# service keepalived start
[root@lDR1-master ~]# ipvsadm -Ln
```

LB集群测试
所有分发器和Real Server都正常

主分发器故障及恢复 （VIP的转换情况）

Real Server故障及恢复