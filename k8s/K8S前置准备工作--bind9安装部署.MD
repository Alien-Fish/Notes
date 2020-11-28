
## 基础依赖
### 关闭防火墙
systemctl stop firewalld

### 安装基础依赖工具
yum install wget net-tools telnet tree nmap sysstat lrzsz dosunix bind-utils -y

## bind安装和配置
### 10.4.7.11上安装DNS解析工具bind
yum install bind -y

### bind配置
vi /etc/named.conf
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
        forwarders      {10.4.7.254; }

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
## 监听地址
listen-on port 53 { 10.4.7.11; };
## 白名单
allow-query     { any; };
## 网关
forwarders      {10.4.7.254; }
## 关闭DNS
dnssec-enable no;
dnssec-validation no;