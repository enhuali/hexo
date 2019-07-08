---
title: Keepalived
date: 2019-07-07 20:32:54
tags:
---

![img](/pics/Keepalived_01.png)

<!-- more -->

# Keepalived安装

Ubuntu 16.04.3系统两台机器

| VIP           | IP                    | Hostname      | Port |
| ------------- | --------------------- | ------------- | ---- |
| 192.168.0.200 | 192.168.0.129(master) | wjt-ceshiji   | 80   |
| 192.168.0.200 | 192.168.0.129(backup) | wjt-ceshiji22 | 80   |



## 下载Keepalived-2.0.13

```
wget http://www.keepalived.org/software/keepalived-2.0.13.tar.gz
```



## 编译安装Keepalived

```
tar zxvf keepalived-2.0.13.tar.gz -C /usr/local
cd /usr/local/keepalived-2.0.13
./configure --prefix=/usr/local/keepalived
make && make install
```



# Keepalived配置

创建工作目录并生成配置文件

```
mkdir /etc/keepalived
touch /etc/keepalived/keepalived.conf
```

## master配置文件内容

```
! Configuration File for keepalived

global_defs {
   notification_email { #指定Keepalived在发生事情的时候，发送邮件通知，每行一个地址
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc #指定发件人
   smtp_server 192.168.0.129 #发送email的smtp地址
   smtp_connect_timeout 30 #超时时间
   router_id wjt-ceshiji #运行Keepalived的机器标识号，主从机必须不同

   #vrrp_skip_check_adv_addr #注释掉vrrp_strict相关是为了解决虚拟ip,ping不通的问题
   #vrrp_strict
   #vrrp_garp_interval 0
   #vrrp_gna_interval 0

}

vrrp_script chk_nginx {
    script "/etc/keepalived/nginx_check.sh" #检测nginx的脚本
    interval 5                           #每5秒检测一次
    weight -20			#如果某一个nginx宕机 则权重减20
}

vrrp_instance VI_1 {
    state MASTER
    interface ens32 #物理网卡名称,主节点和备节点需要相同
    virtual_router_id 100 #唯一的id，主从机必须相同
    priority 150 #优先级，主节点大于备节点，建议至少相差50
    unicast_src_ip  192.168.0.129
    unicast_peer {
                  192.168.0.179 #对端IP地址,此地址一定不能忘记
	}
    advert_int 1 #通信检查间隔时间1s
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    track_script {
       chk_nginx
    }
    virtual_ipaddress {
        192.168.0.200	#VIP,可填写多个
    }
}
```



## backup配置文件内容

```
! Configuration File for keepalived

global_defs {
   notification_email { #指定Keepalived在发生事情的时候，发送邮件通知，每行一个地址
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc #指定发件人
   smtp_server 192.168.0.129 #发送email的smtp地址
   smtp_connect_timeout 30 #超时时间
   router_id wjt-ceshiji22 #运行Keepalived的机器标识号，主从机必须不同
   #vrrp_skip_check_adv_addr #注释掉vrrp_strict相关是为了解决虚拟ip,ping不通的问题
   #vrrp_strict
   #vrrp_garp_interval 0
   #vrrp_gna_interval 0

}

vrrp_script chk_nginx {
    script "/etc/keepalived/nginx_check.sh" #检测nginx的脚本
    interval 5                              #每5秒检测一次
    weight -20       				 #如果某一个nginx宕机 则权重减20
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens32 #物理网卡名称,主节点和备节点需要相同
    virtual_router_id 100 #唯一的id，主从机必须相同
    priority 100 #优先级，主节点大于备节点，建议至少相差50
    unicast_src_ip  192.168.0.179
    unicast_peer {
                  192.168.0.129 #对端IP地址,此地址一定不能忘记
        }
    advert_int 1 #通信检查间隔时间1s
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    track_script {
       chk_nginx
    }
    virtual_ipaddress {
        192.168.0.200	#VIP
    }
}
```



## 主备文件差别

```
router_id	#hostname	
state	#MASTER or BACKUP
interface	#网口
priority	#主比从的数值大
```



# nginx安全检测

**为确保VIP能够7*24小时对外提供服务，增加nginx检查脚本，当出现异常时杀掉keepalived进程让VIP进行飘逸**

## 定时检查nginx状态脚本

```
#!/bin/bash
A=`ps -C nginx --no-header | wc -l`
if [ $A -eq 0 ];then
    /usr/local/nginx/sbin/nginx #尝试重新启动nginx
    sleep 2  #睡眠2秒
    if [ `ps -C nginx --no-header | wc -l` -eq 0 ];then
        /usr/bin/killall keepalived #启动失败，将keepalived服务杀死。将vip漂移到其它备份节点
    fi
fi
```



# 启停Keepalived

```
/usr/local/keepalived/sbin/keepalived		#启动keepalived
killall keepalived		#停止keepalived
```



# 验证Keepalived可用性

**启动主nginx静态页面内容为master，启动备nginx静态页面内容为backup；同时启动keepalived服务**

```
curl 192.168.0.200	
#显示master
停止master上的keepalived后VIP飘逸至backup机器
curl 192.168.0.200
#显示backup
恢复master上的keepalived后VIP飘逸至master机器
curl 192.168.0.200
#显示master
故意修改master机器的nginx配置文件为错误语法，手动杀掉nginx进程，发现keepalived服务随即消失
curl 192.168.0.200
#显示backup
```



# 安装过程中的错误

```
问题：*** WARNING - this build will not support IPVS with IPv6. Please install libnl/libnl-3 dev libraries to support IPv6 with IPVS.
解决：apt-get install libnl-3-200
		 apt-get install libnl-3-dev
		 apt-get install libnl-genl-3-dev

问题：Can't open /etc/rc.d/init.d/functions
解决：ln -s /lib/lsb/init-functions /etc/rc.d/init.d/functions

问题：nginx异常退出并无法启动时，keepalived进程没有自动停止且反复执行chk_nginx.sh的脚本
解决：原因是chk_nginx函数中interval时间过短（2s）改为5s后正常
```



# 其他

```
ip -o -f inet addr show		#查询系统上的IP
ip -f inet addr delete 192.168.0.202/32 dev ens32		#删除指定IP
```

