### 一、安装rabbitmq
环境：CenterOS7 Docker JDK1.8  

1.1.下载镜像  

1.2.运行docker镜像命令：  
docker run -d --name rabbitmq3.7.7 -p 5672:5672 -p 15672:15672 -v `pwd`/data:/var/lib/rabbitmq --hostname myRabbit -e RABBITMQ_DEFAULT_VHOST=my_vhost  -e   RABBITMQ_DEFAULT_USER=admin -e RABBITMQ_DEFAULT_PASS=admin df80af9ca0c9  

1.3.命令参数解析：  
-d 后台运行容器；  
--name 指定容器名；  
-p 指定服务运行的端口（5672：应用访问端口；15672：控制台Web端口号）；  
-v 映射目录或文件；  
--hostname  主机名（RabbitMQ的一个重要注意事项是它根据所谓的 “节点名称” 存储数据，默认为主机名）；  
-e 指定环境变量；（RABBITMQ_DEFAULT_VHOST：默认虚拟机名；RABBITMQ_DEFAULT_USER：默认的用户名；RABBITMQ_DEFAULT_PASS：默认用户名的密码）  

------------------------------------------------------------------------------------------------------------------

### 二、安装延迟队列插件  
2.1.制作Dockerfile  
进入文件夹后vim Dockerfile  

Dockerfile内容：  
FROM rabbitmq:3.7-management  
COPY rabbitmq_delayed_message_exchange-20171201-3.7.x.ez /plugins  
RUN rabbitmq-plugins enable --offline rabbitmq_mqtt rabbitmq_federation_management rabbitmq_stomp rabbitmq_delayed_message_exchange  

2.1.编译  
下面开始build  
docker build -t rabbitmq:3.7 .  

2.2.运行  
完成后使用如下命令进行运行  
docker run -d --hostname my-rabbit --name some-rabbit -p 15672:15672 -p 4369:4369 -p 5671:5671 -p 5672:5672 -p 15671:15671 -p 25672:25672  rabbitmq:3.7  
或者  
docker run -d --hostname my-rabbit --name some-rabbit -e RABBITMQ_DEFAULT_USER=jmall_test -e RABBITMQ_DEFAULT_PASS=123456 -p 15672:15672 -p 4369:4369 -p 5671:5671 -p 5672:5672 -p 15671:15671 -p 25672:25672  rabbitmq:3.7  


### 三、容器外查看日志  
查看日志  
docker logs 容器ID或名字  