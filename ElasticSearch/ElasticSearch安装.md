## 安装方式、版本
本文使用docker方式安装，ElasticSearch版本为5.3.3。

## 拉取镜像
docker pull docker.elastic.co/elasticsearch/elasticsearch:5.3.3

## 下载中文IK分词插件(对应ES版本)
https://github.com/medcl/elasticsearch-analysis-ik/releases

## 上传IK中文插件elasticsearch-analysis-ik-5.6.0.zip，解压
cd /home/tongzhongbiao/elasticsearch/plugins/
unzip elasticsearch-analysis-ik-5.6.0.zip

## 修改插件目录名称
mv ./elasticsearch ./plugins/ik

## 启动方式1：映射配置、数据、分词目录
docker run -d --name es1 -p 9200:9200  -p 9300:9300 -e "http.host=0.0.0.0" -e "transport.host=0.0.0.0" -v /home/tongzhongbiao/elasticsearch/config:/usr/share/elasticsearch/config -v /home/tongzhongbiao/elasticsearch/data:/usr/share/elasticsearch/data  -v /home/tongzhongbiao/elasticsearch/logs:/usr/share/elasticsearch/logs -v
/home/tongzhongbiao/elasticsearch/plugins:/usr/share/elasticsearch/plugins  docker.elastic.co/elasticsearch/elasticsearch:5.3.3

## 启动方式2：不映射目录
docker run -d --name es1 -p 9200:9200  -p 9300:9300 -e "http.host=0.0.0.0" -e "transport.host=0.0.0.0" docker.elastic.co/elasticsearch/elasticsearch:5.3.3

## 启动方式3：映射分词插件目录
docker run -d --name es-ik -p 9200:9200  -p 9300:9300 -e "http.host=0.0.0.0" -e "transport.host=0.0.0.0" -v /home/tongzhongbiao/elasticsearch/plugins/ik:/usr/share/elasticsearch/plugins/ik docker.elastic.co/elasticsearch/elasticsearch:5.3.3

## 允许远程连接
transport.host=0.0.0.0
## 禁用插件
xpack.security.enabled: false