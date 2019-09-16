## 初始化环境
（1）、删除rabbitmq: 
--卸载
--停止服务
rabbitmqctl app_stop
yum list | grep rabbitmq
yum -y remove rabbitmq-server.noarch

（2）删除erlang：
yum list | grep erlang
yum -y remove erlang-*
yum remove erlang.x86_64

（3） 删除lib文件：
rm -rf /usr/lib64/erlang 
rm -rf /var/lib/rabbitmq


## 一、安装erlang依赖环境
1、安装rabbitMQ依赖 erlang
注意：RabbitMQ 3.6.15 依赖的erlang版本
操作系统：CentOS7.X
RabbitMQ: 3.6.15
erlang/OTP 19.3 ~ 20.3.X

1.1 安装erlang依赖环境
安装依赖环境
yum -y install gcc gcc-c++ glibc-devel make ncurses-devel openssl-devel autoconf

1.2下载Erlang
wget http://erlang.org/download/otp_src_20.3.tar.gz

1.2.1解压
tar -xvzf otp_src_20.3.tar.gz

1.2.2进入解压后的Erlang目录
cd otp_src_20.3

1.2.3构建
./otp_build autoconf
如果出现 ./otp_build: line 319: autoconf: command not found ，需要yum install -y autoconf

1.2.4配置安装
./configure
make
make install

1.2.5配置环境变量
vim /etc/profile
export ERLANG_HOME=/usr/local/lib/erlang
export PATH=$PATH:$ERLANG_HOME/bin

1.2.6重新加载环境变量
source /etc/profile

1.2.7查看
erl

## 二、安装rabbitmq
2.1 安装rabbitmq    
下载官网rabbitmq 3.6.15
wget https://www.rabbitmq.com/releases/rabbitmq-server/v3.6.15/rabbitmq-server-3.6.15-1.el7.noarch.rpm

安装rabbitmq
yum install rabbitmq-server-3.6.15-1.el7.noarch.rpm  -y

2.2 配置rabbitmq
开机启动rabbitMQ服务
chkconfig rabbitmq-server on
开启rabbitmq插件
rabbitmq-plugins enable rabbitmq_management
开启、停止rabbitmq服务
systemctl enable rabbitmq-server.service
systemctl start rabbitmq-server.service
systemctl stop rabbitmq-server.service
开启权限
rabbitmqctl add_user admin admin
rabbitmqctl set_user_tags admin administrator
rabbitmqctl set_permissions -p / admin ".*" ".*" ".*"
删除访客用户
rabbitmqctl delete_user guest
访问地址
http://rabbitmq-server:15672/

## 三、安装rabbitmq延迟插件
下载 wget https://dl.bintray.com/rabbitmq/community-plugins/3.6.x/rabbitmq_delayed_message_exchange/rabbitmq_delayed_message_exchange-20171215-3.6.x.zip
解压 unzip rabbitmq_delayed_message_exchange-20171215-3.6.x.zip
移动到插件目录  mv rabbitmq_delayed_message_exchange-20171215-3.6.x.ez /usr/lib/rabbitmq/lib/rabbitmq_server-3.6.15/plugins/
启动 rabbitmq-plugins enable rabbitmq_delayed_message_exchange

