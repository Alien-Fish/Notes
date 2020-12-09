## 1.基础依赖
### 关闭防火墙
[root@hdss7-11 ~]# systemctl stop firewalld

### 2.安装基础依赖工具
[root@hdss7-11 ~]# yum install wget net-tools telnet tree nmap sysstat lrzsz dosunix bind-utils -y

## 3.bind安装和配置
### 10.4.7.11上安装DNS解析工具bind
[root@hdss7-11 ~]# yum install bind -y

### bind配置
[root@hdss7-11 ~]# vi /etc/named.conf
```base
options {
        listen-on port 53 { 10.4.7.11; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        recursing-file  "/var/named/data/named.recursing";
        secroots-file   "/var/named/data/named.secroots";
        allow-query     { any; };
        forwarders      {10.4.7.254; };

        /*
         - If you are building an AUTHORITATIVE DNS server, do NOT enable recursion.
         - If you are building a RECURSIVE (caching) DNS server, you need to enable
           recursion.
         - If your recursive DNS server has a public IP address, you MUST enable access
           control to limit queries to your legitimate users. Failing to do so will
           cause your server to become part of large scale DNS amplification
           attacks. Implementing BCP38 within your network would greatly
           reduce such attack surface
        */
        recursion yes;

        dnssec-enable no;
        dnssec-validation no;

        /* Path to ISC DLV key */
        bindkeys-file "/etc/named.root.key";

        managed-keys-directory "/var/named/dynamic";

        pid-file "/run/named/named.pid";
        session-keyfile "/run/named/session.key";
};
```
更新内容:  
##监听地址  
listen-on port 53 { 10.4.7.11; };  
##白名单  
allow-query     { any; };  
##网关  
forwarders      {10.4.7.254; };  
##关闭DNS  
dnssec-enable no;  
dnssec-validation no;  

### named-checkconf
[root@hdss7-11 ~]# named-checkconf  
执行检查命令named-checkconf，无报错则正常

### bind区域配置
[root@hdss7-11 ~]# vi /etc/named.rfc1912.zones
```base
zone "host.com" IN {
        type master;
        file "host.com.zone";
        allow-update { 10.4.7.11; };
};

zone "lean.com" IN {
        type master;
        file "lean.com.zone";
        allow-update { 10.4.7.11; };
};
```
新增主机域host.com和业务域lean.com

### bind区域数据文件配置
#### 主机域配置
[root@hdss7-11 ~]# vi /var/named/host.com.zone
```base
$ORIGIN host.com.
$TTL 600        ;  10 minutes
@       IN SOA  dns.host.com. dnsadmin.host.com. (
                                2020112801 ; serial
                                10800      ; refresh (3 hours)
                                900        ; retry (1 week)
                                604800     ; expire (1 week) 
                                86400      ; minimun (1 day)
                                )
                         NS dns.host.com.
$TTL 60 ; 1minute
dns               A      10.4.7.11
HDSS7-11          A      10.4.7.11
HDSS7-12          A      10.4.7.12
HDSS7-21          A      10.4.7.21
HDSS7-22          A      10.4.7.22
HDSS7-200         A      10.4.7.200
```
#### 业务域配置
[root@hdss7-11 ~]# vi /var/named/lean.com.zone
```base
$ORIGIN lean.com.
$TTL 600        ;  10 minutes
@               IN SOA  dns.lean.com. dnsadmin.lean.com. (
                                2020112801 ; serial
                                10800      ; refresh (3 hours)
                                900        ; retry (1 week)
                                604800     ; expire (1 week)
                                86400      ; minimun (1 day)
                                )
                         NS dns.lean.com.
$TTL 60 ; 1minute
dns               A      10.4.7.11
```

### named-checkconf
[root@hdss7-11 ~]# named-checkconf  
执行检查命令named-checkconf，无报错则正常

### 禁用ip6，让bind仅支持ipv4
[root@hdss7-11 ~]# vi /etc/sysconfig/named  
OPTIONS="-4"  

### 启动named
[root@hdss7-11 ~]# systemctl start named
[root@hdss7-11 ~]# systemctl enable named    
[root@hdss7-11 ~]# netstat -luntp|grep 53  
tcp        0      0 127.0.0.1:953           0.0.0.0:*               LISTEN      1829/named          
tcp        0      0 10.4.7.11:53            0.0.0.0:*               LISTEN      1829/named          
tcp6       0      0 ::1:953                 :::*                    LISTEN      1829/named          
udp        0      0 10.4.7.11:53            0.0.0.0:*                           1829/named  
[root@hdss7-11 ~]# systemctl status named.service   
● named.service - Berkeley Internet Name Domain (DNS)  
   Loaded: loaded (/usr/lib/systemd/system/named.service; enabled; vendor preset: disabled)  
   Active: active (running) since Wed 2020-12-09 22:29:06 CST; 7min ago  
  Process: 1159 ExecStart=/usr/sbin/named -u named -c ${NAMEDCONF} $OPTIONS (code=exited, status=0/SUCCESS)  
  Process: 1119 ExecStartPre=/bin/bash -c if [ ! "$DISABLE_ZONE_CHECKING" == "yes" ]; then /usr/sbin/named-checkconf -z "$NAMEDCONF"; else echo "Checking of zone files  is disabled"; fi (code=exited, status=0/SUCCESS)  
 Main PID: 1178 (named)  
   CGroup: /system.slice/named.service  
           └─1178 /usr/sbin/named -u named -c /etc/named.conf -4  


### 防火墙上开放DNS服务
[root@hdss7-200 ~]# firewall-cmd --zone=public --permanent --add-service=dns  
success  
[root@hdss7-200 ~]# firewall-cmd --zone=public --permanent --list-services  
dhcpv6-client dns ssh   
备注：其他vm机器也执行  

### 解析DNS
[root@hdss7-11 ~]# dig -t A hdss7-21.host.com @10.4.7.11 +short  
10.4.7.21  
[root@hdss7-11 ~]# dig -t A hdss7-20.host.com @10.4.7.11 +short  
[root@hdss7-11 ~]# dig -t A hdss7-200.host.com @10.4.7.11 +short  
10.4.7.200  

### 配置客户端使用DNS服务
[root@hdss7-11 ~]# vi /etc/sysconfig/network-scripts/ifcfg-ens33
```base
DNS1=10.4.7.11
```
DNS1=10.4.7.254 改成 DNS1=10.4.7.11  

[root@hdss7-11 ~]# systemctl restart network  
[root@hdss7-11 ~]# cat /etc/resolv.conf  

nameserver 10.4.7.11  

[root@hdss7-11 ~]# ping badu.com  
PING badu.com (47.254.33.193) 56(84) bytes of data.  
64 bytes from 47.254.33.193 (47.254.33.193): icmp_seq=1 ttl=128 time=176 ms  
64 bytes from 47.254.33.193 (47.254.33.193): icmp_seq=2 ttl=128 time=176 ms  
64 bytes from 47.254.33.193 (47.254.33.193): icmp_seq=3 ttl=128 time=173 ms  

[root@hdss7-11 ~]# ping hdss7-21.host.com  
PING HDSS7-21.host.com (10.4.7.21) 56(84) bytes of data.  
64 bytes from 10.4.7.21 (10.4.7.21): icmp_seq=1 ttl=64 time=0.844 ms  
64 bytes from 10.4.7.21 (10.4.7.21): icmp_seq=2 ttl=64 time=0.331 ms  

备注：其他vm机器也执行

### 匹配域名后缀，主机域使用短域名
[root@hdss7-11 ~]# vi /etc/resolv.conf
```base
search host.com
```
[root@hdss7-11 ~]# ping hdss7-21  
PING HDSS7-21.host.com (10.4.7.21) 56(84) bytes of data.  
64 bytes from 10.4.7.21 (10.4.7.21): icmp_seq=1 ttl=64 time=0.532 ms  
64 bytes from 10.4.7.21 (10.4.7.21): icmp_seq=2 ttl=64 time=0.365 ms  

备注：其他vm机器也执行  

### 使用nslookup测试DNS服务
[root@hdss7-11 ~]# nslookup hdss7-200.host.com  
Server:		10.4.7.11  
Address:	10.4.7.11#53  

Name:	HDSS7-200.host.com  
Address: 10.4.7.200  


## 4.配置windows客户端
控制面板\网络和 Internet\网络连接\VMware Virtual Ethernet Adapter for VMnet8\TCP/IP4属性  
首选DNS服务器改成10.4.7.11  

C:\Users\10744>ping hdss7-200.host.com  

正在 Ping HDSS7-200.host.com [10.4.7.200] 具有 32 字节的数据:  
来自 10.4.7.200 的回复: 字节=32 时间<1ms TTL=64  
来自 10.4.7.200 的回复: 字节=32 时间<1ms TTL=64  

备注：如果ping不同，修改vm8和本地网的自动跃点为10  

至此，局域网DNS配置就成功了！