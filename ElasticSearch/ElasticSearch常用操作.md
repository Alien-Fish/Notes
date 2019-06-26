##1.列出所有文档
http://127.0.0.1:9200/_cat/indices?v

##2.查看文档
http://127.0.0.1:9200/index_name?pretty

##3.查询索引数据
http://127.0.0.1:9200/index_name/_search?pretty

##4.测试分词(ik_smart、ik_max_word)
http://127.0.0.1:9200/index_name/_analyze?analyzer=ik_smart&pretty=true&text=我是中国人
http://127.0.0.1:9200/index_name/_analyze?analyzer=ik_max_word&pretty=true&text=我是中国人


##Linux命令
curl -XPOST '127.0.0.1:9200/index_name/_search?pretty' -d '{"query": { "match_all": {} },"from": 10, "size": 10}'