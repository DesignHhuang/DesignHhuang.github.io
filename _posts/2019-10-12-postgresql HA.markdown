---
layout: post
title:  "pg的HA实现 2节点HA + 1节点备份 + 5节点etcd"
date:   2019-10-12 13:10:46 +0800
categories: huangxiaomin update
---

### 1. 安装规划与配置

| 节点        | 软件   |  
| --------   | -----  | 
| bigdata1     | Etcd集群节点1 |   
| bigdata2        |   Etcd集群节点2   | 
| bigdata3        |    Etcd集群节点3    | 
| bigdata4        |    Etcd集群节点4，Keepalived主节点，Stolon proxy1/keeper1/sentinel1    | 
| bigdata5        |    Etcd集群节点5，Keepalived从节点，Stolon proxy2/keeper2/sentinel2    |  

> * Etcd节点必须奇数2N+1，可以使用consul(替代方案)
> * Keepalived需要独立VIP：172.16.10.10
> * 172.16.10.11 ~ 172.16.10.15(2379~2380)
> * 172.16.10.14 ~ 172.16.10.15(5432~25432)
> * 172.16.10.10(25432)

### 2. 安装目录
所有节点上传安装包`cluster-postgre-install`到`gpt`目录(`bigdata4`安装在`/home/gpt/cluster-postgre-install`下)。

### 3. 系统参数配置
```
vim /etc/security/limits.conf
* soft noproc 11000
* hard noproc 11000
* soft nofile 65535
* hard nofile 65535
```

### 4. 环境变量配置
```
vim /etc/profile
```
在文件末尾追加:
```
export PATH=/gpt/cluster-postgre-install/:/gpt/cluster-postgre-install/pgsql/bin/:$PATH
export LD_LIBRARY_PATH=/gpt/cluster-postgre-install/pgsql/lib/:$LD_LIBRARY_PATH
export ETCDCTL_API=3
```
使文件生效:
```
source /etc/profile
mkdir /etc/etcd 
```
`Bigdata4`:磁盘分区不通，导致安装目录不同
```
export PATH=/home/gpt/cluster-postgre-install/:/home/gpt/cluster-postgre-install/pgsql/bin/:$PATH:$MAVEN_HOME/bin
export LD_LIBRARY_PATH=/home/gpt/cluster-postgre-install/pgsql/lib/:$LD_LIBRARY_PATH
export ETCDCTL_API=3
```
### 5. 配置自启动
自启动服务列表：
- -rw-r--r--. 1 root root 604 May 31 11:23 stolon-etcd.service
- -rw-r--r--. 1 root root 851 May 27 17:11 stolon-keeper.service
- -rw-r--r--. 1 root root 426 May 27 16:55 stolon-proxy.service
- -rw-r--r--. 1 root root 633 May 27 17:11 stolon-sentinel.service

#### `bigdata1`至`bigdata3`：
```
cp /gpt/cluster-postgre-install/stolon-etcd.service /usr/lib/systemd/system/
systemctl enable stolon-etcd.service
```
手动启动：
```
systemctl start stolon-etcd.service
```
#### `bigdata1`至`bigdata5`：
```
cp /gpt/cluster-postgre-install/stolon-*.service /usr/lib/systemd/system/
systemctl enable stolon-etcd.service
systemctl enable stolon-sentinel.service
systemctl enable stolon-keeper.service
systemctl enable stolon-proxy.service
```
手动启动：
```
systemctl start stolon-etcd.service
systemctl start stolon-sentinel.service
systemctl start stolon-keeper.service
systemctl start stolon-proxy.service
```

---
#### 手动旧启动方式：
`bigdata1`至`bigdata3`：
```
vim /etc/rc.local
```
在文件最后添加：
```
#startup etcd
etcd  --config-file=/etc/etcd/conf.yml >> /gpt/cluster-postgre-install/log-etcd/etcd.`date +%F`.log 2>&1 &
```
`bigdata5`：
```
vim /etc/rc.local
```
在文件最后添加：
```
#startup etcd node for postgre cluster
etcd  --config-file=/etc/etcd/conf.yml >> /home/gpt/cluster-postgre-install/log-etcd/etcd.`date +%F`.log 2>&1 &

#startup stolon sentinel for postgre ha
stolon-sentinel --cluster-name stolon-cluster --store-backend=etcdv3 >> /gpt/cluster-postgre-install/log-sentinel/stolon-sentinel.`date +%F`.log 2>&1 &

#startup stolon keeper for postgre ha
su postgres -l -c "stolon-keeper --cluster-name stolon-cluster --store-backend=etcdv3 --uid postgres1 --data-dir /gpt/cluster-postgre-install/data-postgres/ --pg-su-password=postgres123456.com --pg-repl-username=repluser --pg-repl-password=repluser123456.com --pg-listen-address=172.16.10.15 >> /gpt/cluster-postgre-install/log-postgres/stolon-keeper.`date +%F`.log 2>&1 &"

#startup stolon proxy for postgre ha
stolon-proxy --cluster-name stolon-cluster --store-backend=etcdv3 --port 25432 --listen-address=0.0.0.0 >> /gpt/cluster-postgre-install/log-proxy/stolon-proxy.`date +%F`.log 2>&1 &
```
`bigdata4`：磁盘分区不通，导致安装目录不同
```
vim /etc/rc.local
```
在文件最后添加：
```
#link  /home/gpt /gpt
ln  -s /home/gpt/ /gpt

#startup etcd node for postgre cluster
etcd  --config-file=/etc/etcd/conf.yml >> /home/gpt/cluster-postgre-install/log-etcd/etcd.`date +%F`.log 2>&1 &

#startup stolon sentinel for postgre ha
stolon-sentinel --cluster-name stolon-cluster --store-backend=etcdv3 >> /home/gpt/cluster-postgre-install/log-sentinel/sentinel.`date +%F`.log 2>&1 &

#startup stolon keeper for postgre ha
su postgres -l -c "stolon-keeper --cluster-name stolon-cluster --store-backend=etcdv3 --uid postgres0 --data-dir /home/gpt/cluster-postgre-install/data-postgres/ --pg-su-password=postgres123456.com --pg-repl-username=repluser --pg-repl-password=repluser123456.com --pg-listen-address=172.16.10.14 >> /home/gpt/cluster-postgre-install/log-postgres/stolon-keeper.`date +%F`.log 2>&1 &"

#startup stolon proxy for postgre ha
stolon-proxy --cluster-name stolon-cluster --store-backend=etcdv3 --port 25432 --listen-address=0.0.0.0 >> /home/gpt/cluster-postgre-install/log-proxy/stolon-proxy.`date +%F`.log 2>&1 &
```
---

### 6. `bigdata1`至`bigdata3`防火墙配置
```
firewall-cmd --zone=public --add-port=2379-2380/tcp --permanent
firewall-cmd --reload
```
### 7. `bigdata4`、`bigdata5`防火墙配置
```
firewall-cmd --zone=public --add-port=5432/tcp --permanent
firewall-cmd --zone=public --add-port=25432/tcp --permanent
firewall-cmd --reload
```
### 8. 时钟同步
`bigdata1`作为时钟同步服务器：
修改`/etc/ntp.conf`：
添加一行：`restrict 172.16.10.0`
注释掉外网，并添加一行本机时钟：
```
#server 0.centos.pool.ntp.org iburst
#server 1.centos.pool.ntp.org iburst
#server 2.centos.pool.ntp.org iburst
#server 3.centos.pool.ntp.org iburst
server 127.127.1.0 iburst
```
`bigdata2`至`bigdata5`修改`/etc/ntp.conf`：
```
#server 0.centos.pool.ntp.org iburst
#server 1.centos.pool.ntp.org iburst
#server 2.centos.pool.ntp.org iburst
#server 3.centos.pool.ntp.org iburst
server 172.16.10.11
```
`bigdata2`至`bigdata5`执行：
```
ntpdate 172.16.10.11
```
`bigdata1`至`bigdata5`执行：
```
systemctl enable ntpd
systemctl restart ntpd
```
`bigdata1`至`bigdata5`执行：
```
timedatectl set-ntp yes
```
### 9. 5节点Etcd安装
#### 创建`etcd`配置文件：
```
$ vi /etc/etcd/conf.yml
```
节点`bigdata1`，添加如下内容：
```
name: etcd-bigdata1
data-dir: /opt/cluster-postgre-install/data-etcd/
listen-client-urls: http://172.16.10.11:2379,http://127.0.0.1:2379
advertise-client-urls: http://172.16.10.11:2379,http://127.0.0.1:2379
listen-peer-urls: http://172.16.10.11:2380
initial-advertise-peer-urls: http://172.16.10.11:2380
initial-cluster: etcd-bigdata1=http://172.16.10.11:2380,etcd-bigdata2=http://172.16.10.12:2380,etcd-bigdata3=http://172.16.10.13:2380,etcd-bigdata4=http://172.16.10.14:2380,etcd-bigdata5=http://172.16.10.15:2380
initial-cluster-token: etcd-cluster-token
initial-cluster-state: new
```
节点`bigdata2`，添加如下内容：
```
name: etcd-bigdata2
data-dir: /opt/cluster-postgre-install/data-etcd/
listen-client-urls: http://172.16.10.12:2379,http://127.0.0.1:2379
advertise-client-urls: http://172.16.10.12:2379,http://127.0.0.1:2379
listen-peer-urls: http://172.16.10.12:2380
initial-advertise-peer-urls: http://172.16.10.12:2380
initial-cluster: etcd-bigdata1=http://172.16.10.11:2380,etcd-bigdata2=http://172.16.10.12:2380,etcd-bigdata3=http://172.16.10.13:2380,etcd-bigdata4=http://172.16.10.14:2380,etcd-bigdata5=http://172.16.10.15:2380
initial-cluster-token: etcd-cluster-token
initial-cluster-state: new
```
节点`bigdata3`，添加如下内容：
```
name: etcd-bigdata1
data-dir: /opt/cluster-postgre-install/data-etcd/
listen-client-urls: http://172.16.10.13:2379,http://127.0.0.1:2379
advertise-client-urls: http://172.16.10.13:2379,http://127.0.0.1:2379
listen-peer-urls: http://172.16.10.13:2380
initial-advertise-peer-urls: http://172.16.10.13:2380
initial-cluster: etcd-bigdata1=http://172.16.10.11:2380,etcd-bigdata2=http://172.16.10.12:2380,etcd-bigdata3=http://172.16.10.13:2380,etcd-bigdata4=http://172.16.10.14:2380,etcd-bigdata5=http://172.16.10.15:2380
initial-cluster-token: etcd-cluster-token
initial-cluster-state: new
```
节点`bigdata4`，添加如下内容：
```
name: etcd-bigdata1
data-dir: /opt/cluster-postgre-install/data-etcd/
listen-client-urls: http://172.16.10.14:2379,http://127.0.0.1:2379
advertise-client-urls: http://172.16.10.14:2379,http://127.0.0.1:2379
listen-peer-urls: http://172.16.10.14:2380
initial-advertise-peer-urls: http://172.16.10.14:2380
initial-cluster: etcd-bigdata1=http://172.16.10.11:2380,etcd-bigdata2=http://172.16.10.12:2380,etcd-bigdata3=http://172.16.10.13:2380,etcd-bigdata4=http://172.16.10.14:2380,etcd-bigdata5=http://172.16.10.15:2380
initial-cluster-token: etcd-cluster-token
initial-cluster-state: new
```

节点`bigdata5`，添加如下内容：
```
name: etcd-bigdata5
data-dir: /opt/cluster-postgre-install/data-etcd/
listen-client-urls: http://172.16.10.15:2379,http://127.0.0.1:2379
advertise-client-urls: http://172.16.10.15:2379,http://127.0.0.1:2379
listen-peer-urls: http://172.16.10.15:2380
initial-advertise-peer-urls: http://172.16.10.15:2380
initial-cluster: etcd-bigdata1=http://172.16.10.11:2380,etcd-bigdata2=http://172.16.10.12:2380,etcd-bigdata3=http://172.16.10.13:2380,etcd-bigdata4=http://172.16.10.14:2380,etcd-bigdata5=http://172.16.10.15:2380
initial-cluster-token: etcd-cluster-token
initial-cluster-state: new
```
#### 手动启动
在所有节点上执行
```
etcd  --config-file=/etc/etcd/conf.yml >> /gpt/cluster-postgre-install/log-etcd/etcd.`date +%F`.log 2>&1 &
```
#### 检查环境
查看版本信息：
```
$ etcdctl version
```
查看集群成员信息：
```
$ ./etcdctl member list
```
查看集群状态（`Leader`节点）：
```
$ ./etcdctl cluster-health
```
### 10. `Keepalived`安装配置（需要一个独立的VIP）
`keepalived`安装 
`Bigdata4`和`bigdata5`上执行：
```
yum install keepalived -y
```
#### 修改`bigdata4`上 `Keepalived` 配置文件`keepalived.conf` 
`keepalived.conf`位于(`/etc/keepalived/keepalived.conf`) 
```
mv keepalived.conf keepalived.conf.bak
```
修改`keepalived.conf`如下：
```
global_defs {
   notification_email {
      root@localhost
   }
   notification_email_from root@localhost
   smtp_server root
   smtp_connect_timeout 30
   router_id LVS_DEVEL
   vrrp_skip_check_adv_addr
 #  vrrp_strict
   vrrp_garp_interval 0
   vrrp_gna_interval 0
}
vrrp_instance VI_1 {
    state MASTER
    interface team0
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        172.16.10.10 
    }
}
```
#### 修改`bigdata5`上`keepalived.conf`
`keepalived.conf`位于(`/etc/keepalived/keepalived.conf`)
```
cp keepalived.conf keepalived.conf.bak
```
```
global_defs {
   notification_email {
      root@localhost
   }
   notification_email_from root@localhost
   smtp_server root
   smtp_connect_timeout 30
   router_id LVS_DEVEL
   vrrp_skip_check_adv_addr
 #  vrrp_strict
   vrrp_garp_interval 0
   vrrp_gna_interval 0
}
vrrp_instance VI_1 {
    state MASTER
    interface team0 #此处为自己的ip名
    virtual_router_id 51
    priority 90
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        172.16.10.10 
    }
}
```
#### 启动`keepalived`服务
`Bigdata4`和`bigdata5`上执行：
```
systemctl enable keepalived
systemctl restart keepalived
```
#### 检查vip
```
$ip a
```
### 11.2服务节点`Stolon`安装配置
#### 设置环境：
```
vi /etc/profile
```
在文件最后添加:
```
export PATH=/gpt/cluster-postgre-install/:/gpt/cluster-postgre-install/pgsql/bin/:$PATH
export LD_LIBRARY_PATH=/gpt/cluster-postgre-install/pgsql/lib/:$LD_LIBRARY_PATH
```
执行：
```
source /etc/profile
```
#### 添加目录和用户
两个节点上执行：
```
cd /opt/cluster-postgre-install/
mkdir data-postgres
mkdir log-postgres

useradd postgres
passwd postgres
#密码设置为123456
chown -R postgres:postgres data-postgres
chown -R postgres:postgres log-postgres
```
#### `Bigdata4`上执行初始化
```
stolonctl --cluster-name stolon-cluster --store-backend=etcdv3 init
stolonctl --cluster-name=stolon-cluster  --store-backend=etcdv3 update --patch '{ "pgParameters" : {"max_connections" : "1000" } }'
```
#### `Bigdata4`和`bigdata5`上启动`sentinel`
`Bigdata4`:
```
stolon-sentinel --cluster-name stolon-cluster --store-backend=etcdv3 >> /home/gpt/cluster-postgre-install/log-sentinel/sentinel.`date +%F`.log 2>&1 &
```
`bigdata5`:
```
stolon-sentinel --cluster-name stolon-cluster --store-backend=etcdv3 >> /gpt/cluster-postgre-install/log-sentinel/sentinel.`date +%F`.log 2>&1 &
```
#### `Bigdata4`和`bigdata5`上发布`keeper`必须以非root用户启动
启动：
`Bigdata4`上以`postgres`用户启动
```
su postgres -l -c "stolon-keeper --cluster-name stolon-cluster --store-backend=etcdv3 --uid postgres0 --data-dir /home/gpt/cluster-postgre-install/data-postgres/ --pg-su-password=postgres123456.com --pg-repl-username=repluser --pg-repl-password=repluser123456.com --pg-listen-address=172.16.10.14 >> /home/gpt/cluster-postgre-install/log-postgres/stolon-keeper.`date +%F`.log 2>&1 &"
```
`bigdata5`上以`postgres`用户启动
```
su postgres -l -c "stolon-keeper --cluster-name stolon-cluster --store-backend=etcdv3 --uid postgres1 --data-dir /gpt/cluster-postgre-install/data-postgres/ --pg-su-password=postgres123456.com --pg-repl-username=repluser --pg-repl-password=repluser123456.com --pg-listen-address=172.16.10.15 >> /gpt/cluster-postgre-install/log-postgres/stolon-keeper.`date +%F`.log 2>&1 &"
```

会启动一个`stolon keeper`，id为`postgres0`和`postgres1`，监听5432，将安装并初始化一个`postgres`实例到数据目录`data-postgres/postgres0/postgres/`和`data- postgres/postgres1/postgres/`

#### `bigdata4`和`bigdata5`上启动proxy
`bigdata4`:
```
stolon-proxy --cluster-name stolon-cluster --store-backend=etcdv3 --port 25432 --listen-address=0.0.0.0 >> /home/gpt/cluster-postgre-install/log-proxy/stolon-proxy.`date +%F`.log 2>&1 &
```
`bigdata5`:
```
stolon-proxy --cluster-name stolon-cluster --store-backend=etcdv3 --port 25432 --listen-address=0.0.0.0 >> /gpt/cluster-postgre-install/log-proxy/stolon-proxy.`date +%F`.log 2>&1 &
```
#### 连接VIP进行测试
```
psql --host 172.16.10.10 --port 25432 -U postgres
```
密码：`postgres123456.com`
```
postgres=#
Create a test table and insert a row
Connect to the db. Create a test table and do some inserts (we use the "postgres" database for these tests but usually this shouldn't be done).

postgres=# create table test (id int primary key not null, value text not null);
CREATE TABLE
postgres=# insert into test values (1, 'value1');
INSERT 0 1
postgres=# select * from test;
 id | value
----+--------
  1 | value1
(1 row)
```
可用性测试：(慎重)
Kill掉`bigdata4`上的`keeper`：(慎重)
```
postgres=# select * from test;

 id | value  
----+--------
  1 | value1
(1 row)

postgres=# select * from test;
server closed the connection unexpectedly
	This probably means the server terminated abnormally
	before or while processing the request.
The connection to the server was lost. Attempting reset: Succeeded.
postgres=# select * from test;
 id | value  
----+--------
  1 | value1
(1 row)
```
停止`bigdata4`上的`keepalived`:（慎重）
```
systemctl  stop keepalived
```
```
postgres=# select * from test;
server closed the connection unexpectedly
	This probably means the server terminated abnormally
	before or while processing the request.
The connection to the server was lost. Attempting reset: Succeeded.
postgres=# select * from test;
 id | value  
----+--------
  1 | value1
  2 | value2
(2 rows)
```
### 12.高可用`stolon`集群维护手册
#### `Bigdata4`和`bigdata5`上检查`keepalived`服务状态
检查服务状态：
```
systemctl status keepalived
```
vip地址检查：
```
ip addr | grep 10.10
```
#### 检查`etcd`集群
`bigdata5`进入集群安装目录：
```
cd /gpt/cluster-postgre-install/
./check_etcd.sh
```
打印`etcd`相关进程信息：

检查各节点`etcd`日志：各节点日期可能一样
```
tail -f /gpt/cluster-postgre-install/log-etcd/etcd.2019-06-26.log
```
#### 检查`bigdata4`和`bigdata5`上`stolon-proxy`进程及日志
检查配置好的`stolon-proxy`服务: （如果没有检查进程）
```
systemctl status stolon-proxy
```
检查进程：
```
ps -eaf |grep stolon-proxy
```

最重要的检查日志：（每个节点的日期可能不一致）
```
tail -f log-proxy/stolon-proxy.2019-06-28.log
```
#### 检查`bigdata4`和`bigdata5`上`stolon-sentinel`进程及日志
检查配置好的`stolon-sentinel`服务: （如果没有检查进程）
```
systemctl status stolon-sentinel
```
检查进程：
```
ps -eaf |grep stolon-sentinel
```

最重要的检查日志：（每个节点的日期可能不一致）
```
tail -f log-sentinel/stolon-sentinel.2019-05-21.log
```
#### 检查`bigdata4`和`bigdata5`上`stolon-keeper`（pg实例）进程及日志
检查配置好的`stolon-keeper`服务:（如果没有检查进程）
```
systemctl status stolon-keeper
```
检查进程：
```
ps -eaf |grep stolon-keeper
```
最重要的检查日志：（每个节点的日期可能不一致）
```
tail -f log-postgres/stolon-keeper.2019-05-23.log
```









