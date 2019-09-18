## 1.springboot中配置elasticSearch
1.1在工程中引入相关的jar包   
 1.1.1 在build.gradle中添加需要的jar包  

    gradle工程  

    // 添加  Spring Data Elasticsearch 的依赖
    compile('org.springframework.boot:spring-boot-starter-data-elasticsearch')
     
    // 添加  JNA 的依赖，java访问当前操作系统需要的包
    compile('net.java.dev.jna:jna:4.3.0')
 1.1.2 在pom.xml中添加需要的jar包  

    maven工程  

    // 添加  Spring Data Elasticsearch 的依赖  
    <dependency>  
        <groupId>org.springframework.boot</groupId>  
        <artifactId>spring-boot-starter-data-elasticsearch</artifactId>  
    </dependency>  

1.1.3在application.properties添加elasticsearch的配置    

    #es的默认名称,如果安装es时没有做特殊的操作名字都是此名称
    spring.data.elasticsearch.cluster-name=elasticsearch
    # Elasticsearch 集群节点服务地址，用逗号分隔，如果没有指定其他就启动一个客户端节点,默认java访问端口9300
    spring.data.elasticsearch.cluster-nodes=localhost:9300
    # 设置连接超时时间
    spring.data.elasticsearch.properties.transport.tcp.connect_timeout=120s

1.2创建文档实体对象    

    package site.wlss.blog.domain.es;
     
    import java.io.Serializable;
    import java.sql.Timestamp;
     
    import org.springframework.data.annotation.Id;
    import org.springframework.data.elasticsearch.annotations.Document;
    import org.springframework.data.elasticsearch.annotations.Field;
    import org.springframework.data.elasticsearch.annotations.FieldIndex;
     
    import site.wlss.blog.domain.Blog;  
     
     
    /**
     * EsBlog 文档类.
     * 
     * @since  2018年8月5日
     * @author wangli
     */
    /*@Document注解里面的几个属性，类比mysql的话是这样： 
        index –> DB 
        type –> Table 
        Document –> row 
    */
    @Document(indexName = "blog", type = "blog")
    public class EsBlog implements Serializable {
     
        private static final long serialVersionUID = 1L;
     
        @Id  // 主键,注意这个搜索是id类型是string，与我们常用的不同
        private String id;  //@Id注解加上后，在Elasticsearch里相应于该列就是主键了，在查询时就可以直接用主键查询
        @Field(index = FieldIndex.not_analyzed)  // 不做全文检索字段  
        private Long blogId; // Blog 实体的 id，这儿增加了一个blog的id属性
        private String title;
        private String summary;
        private String content;
        /**
         - 用户名，全匹配不做分词
         - */
        @Field(type = FieldType.Keyword)
        private String username;

        /**
         - 客户端UUID，全匹配不做分词
         - */
        @Field(type = FieldType.Keyword)
        private String clientUUID;

        /**
         - 历史检索关键字
         - 聚合开启："fielddata": true
         - 原因：5.x之后，Elasticsearch对排序、聚合所依据的字段用单独的数据结构(fielddata)缓存到内存里了，但是在text字段上默认是禁用的，如果有需要单独开启，这样做的目的是为了节省内存空间。
         - 官方文档地址：https://www.elastic.co/guide/en/elasticsearch/reference/current/fielddata.html
         *
         */
        @Field(type = FieldType.Text, fielddata = true, analyzer = "ik_max_word")
        private String hisKeywork;

        /**
         - 创建时间，不分词做排序
         */
        @Field(type = FieldType.Keyword)
        private String createTime;
    }  

上面是我的部分代码，注意要对实体对象有个@Document注解，对象的id也有个@id的注解，其中还有个@Field的注解，这是对该字段的说明，下面对这些注解给出详细解释  

解释一：@Document注解  

@Document注解里面的几个属性，类比mysql的话是这样：   
indexName –> 索引库的名称，建议以项目的名称命名，就相当于数据库DB  
type –> 类型，建议以实体的名称命名Table ，就相当于数据库中的表table  
Document –> row 就相当于某一个具体对象  

附上注解的内容：  

    String indexName();//索引库的名称，建议以项目的名称命名
     
    String type() default "";//类型，建议以实体的名称命名
     
    short shards() default 5;//默认分区数
     
    short replicas() default 1;//每个分区默认的备份数
     
    String refreshInterval() default "1s";//刷新间隔
     
    String indexStoreType() default "fs";//索引文件存储类型

解释二：@Id注解  

在Elasticsearch里相应于该列就是主键了，在查询时就可以直接用主键查询  

解释三：@Field注解  

     
    public @interface Field {
     
    FieldType type() default FieldType.Auto;#自动检测属性的类型
     
    FieldIndex index() default FieldIndex.analyzed;#默认情况下分词
     
    DateFormat format() default DateFormat.none;
     
    String pattern() default "";
     
    boolean store() default false;#默认情况下不存储原文
     
    String searchAnalyzer() default "";#指定字段搜索时使用的分词器
     
    String indexAnalyzer() default "";#指定字段建立索引时指定的分词器
     
    String[] ignoreFields() default {};#如果某个字段需要被忽略
     
    boolean includeInParent() default false;
    }

## 2.通过jpa创建文档库  

因为我们引入的是spring data的elasticsearch所以它遵循spring data的接口，也就是说操作elasticSearch与操作spring data jpa的方法是完全一样的，我们只将文档库继承ElasticsearchRepository即可。  

    package site.wlss.blog.repository.es;
     
    import org.springframework.data.domain.Page;
    import org.springframework.data.domain.Pageable;
    import org.springframework.data.elasticsearch.repository.ElasticsearchRepository;
     
    import site.wlss.blog.domain.es.EsBlog;
     
     
    /**
     * EsBlog Repository接口.
     * @author Wang Li
     * @date 2018年8月5日
     */
    public interface EsBlogRepository extends ElasticsearchRepository<EsBlog, String> {
        //下面是我们根据 spring data jpa 的命名规范额外创建的两个查询方法
        /**
         * 模糊查询(去重)，根据标题，简介，描述和标签查询（含有即可）Containing
         * @param title
         * @param Summary
         * @param content
         * @param tags
         * @param pageable
         * @return
         */
        Page<EsBlog> findDistinctEsBlogByTitleContainingOrSummaryContainingOrContentContainingOrTagsContaining(String title,String Summary,String content,String tags,Pageable pageable);
        
        /**
         * 根据 Blog 的id 查询 EsBlog
         * @param blogId
         * @return
         */
        EsBlog findByBlogId(Long blogId);
    }

里面的内容是我根据spring data jpa 额外创建的两个方法。  

## 3.根据reporitory查询文档  

这个方法和操作jpa中的普通的方法没什么区别，就是普通的增删改查。  
4.ElasticSearch的高级复杂查询：非聚合查询和聚合查询  

这儿才是我今天要讲的重点。  
4.1非聚合复杂查询(这儿展示了非聚合复杂查询的常用流程)  

    public List<EsBlog> elasticSerchTest() {  
        //1.创建QueryBuilder(即设置查询条件)这儿创建的是组合查询(也叫多条件查询),后面会介绍更多的查询方法  
         /*组合查询BoolQueryBuilder  
             * must(QueryBuilders)   :AND  
             * mustNot(QueryBuilders):NOT  
             * should:               :OR  
         */  
            BoolQueryBuilder builder = QueryBuilders.boolQuery();  
            //builder下有must、should以及mustNot 相当于sql中的and、or以及not  
            
            //设置模糊搜索,博客的简诉中有学习两个字  
            builder.must(QueryBuilders.fuzzyQuery("sumary", "学习"));  
            
            //设置要查询博客的标题中含有关键字  
            builder.must(new QueryStringQueryBuilder("man").field("springdemo"));  
     
            //按照博客的评论数的排序是依次降低  
            FieldSortBuilder sort = SortBuilders.fieldSort("commentSize").order(SortOrder.DESC);  
     
            //设置分页(从第一页开始，一页显示10条)  
            //注意开始是从0开始，有点类似sql中的方法limit 的查询  
            PageRequest page = new PageRequest(0, 10);  
     
            //2.构建查询  
            NativeSearchQueryBuilder nativeSearchQueryBuilder = new NativeSearchQueryBuilder();  
            //将搜索条件设置到构建中  
            nativeSearchQueryBuilder.withQuery(builder);  
            //将分页设置到构建中  
            nativeSearchQueryBuilder.withPageable(page);  
            //将排序设置到构建中  
            nativeSearchQueryBuilder.withSort(sort);  
            //生产NativeSearchQuery  
            NativeSearchQuery query = nativeSearchQueryBuilder.build();  
     
            //3.执行方法1  
            Page<EsBlog> page = esBlogRepository.search(query);  
            //执行方法2：注意，这儿执行的时候还有个方法那就是使用elasticsearchTemplate  
            //执行方法2的时候需要加上注解  
            //@Autowired  
            //private ElasticsearchTemplate elasticsearchTemplate;  
            List<EsBlog> blogList = elasticsearchTemplate.queryForList(query, EsBlog.class);  
           
            //4.获取总条数(用于前端分页)  
            int total = (int) page.getTotalElements();  
     
            //5.获取查询到的数据内容（返回给前端）  
            List<EsBlog> content = page.getContent();  
     
            return content;  
    }  

## 4.2查询条件QueryBuilder的构建方法举例  

在使用聚合查询之前我们有必要先来了解下创建查询条件QueryBuilder的几种常用方法  
4.2.1精确查询（必须完全匹配上）  

单个匹配termQuery  

    //不分词查询 参数1： 字段名，参数2：字段查询值，因为不分词，所以汉字只能查询一个字，英语是一个单词.  
    QueryBuilder queryBuilder=QueryBuilders.termQuery("fieldName", "fieldlValue");  
    //分词查询，采用默认的分词器  
    QueryBuilder queryBuilder2 = QueryBuilders.matchQuery("fieldName", "fieldlValue");  

多个匹配  

    //不分词查询，参数1： 字段名，参数2：多个字段查询值,因为不分词，所以汉字只能查询一个字，英语是一个单词.  
    QueryBuilder queryBuilder=QueryBuilders.termsQuery("fieldName", "fieldlValue1","fieldlValue2...");  
    //分词查询，采用默认的分词器  
    QueryBuilder queryBuilder= QueryBuilders.multiMatchQuery("fieldlValue", "fieldName1", "fieldName2", "fieldName3");  
    //匹配所有文件，相当于就没有设置查询条件  
    QueryBuilder queryBuilder=QueryBuilders.matchAllQuery();  

4.2.2模糊查询（只要包含即可）  

       //模糊查询常见的5个方法如下  
            //1.常用的字符串查询  
            QueryBuilders.queryStringQuery("fieldValue").field("fieldName");//左右模糊  
            //2.常用的用于推荐相似内容的查询  
            QueryBuilders.moreLikeThisQuery(new String[] {"fieldName"}).addLikeText("pipeidhua");//如果不指定filedName，则默认全部，常用在相似内容的推荐上  
            //3.前缀查询  如果字段没分词，就匹配整个字段前缀  
            QueryBuilders.prefixQuery("fieldName","fieldValue");  
            //4.fuzzy query:分词模糊查询，通过增加fuzziness模糊属性来查询,如能够匹配hotelName为tel前或后加一个字母的文档，fuzziness 的含义是检索的term 前后增加或减少n个单词的匹配查询  
            QueryBuilders.fuzzyQuery("hotelName", "tel").fuzziness(Fuzziness.ONE);  
            //5.wildcard query:通配符查询，支持* 任意字符串；？任意一个字符  
            QueryBuilders.wildcardQuery("fieldName","ctr*");//前面是fieldname，后面是带匹配字符的字符串  
            QueryBuilders.wildcardQuery("fieldName","c?r?");  

4.2.3范围查询  

            //闭区间查询  
            QueryBuilder queryBuilder0 = QueryBuilders.rangeQuery("fieldName").from("fieldValue1").to("fieldValue2");  
            //开区间查询  
            QueryBuilder queryBuilder1 = QueryBuilders.rangeQuery("fieldName").from("fieldValue1").to("fieldValue2").includeUpper(false).includeLower(false);//默认是true，也就是包含  
            //大于  
            QueryBuilder queryBuilder2 = QueryBuilders.rangeQuery("fieldName").gt("fieldValue");    
            //大于等于  
            QueryBuilder queryBuilder3 = QueryBuilders.rangeQuery("fieldName").gte("fieldValue");  
            //小于  
            QueryBuilder queryBuilder4 = QueryBuilders.rangeQuery("fieldName").lt("fieldValue");  
            //小于等于  
            QueryBuilder queryBuilder5 = QueryBuilders.rangeQuery("fieldName").lte("fieldValue");  

4.2.4组合查询/多条件查询/布尔查询  

    QueryBuilders.boolQuery()  
    QueryBuilders.boolQuery().must();//文档必须完全匹配条件，相当于and  
    QueryBuilders.boolQuery().mustNot();//文档必须不匹配条件，相当于not  
    QueryBuilders.boolQuery().should();//至少满足一个条件，这个文档就符合should，相当于or  

4.3聚合查询  

Elasticsearch有一个功能叫做 聚合(aggregations) ，它允许你在数据上生成复杂的分析统计。它很像SQL中的 GROUP BY 但是功能更强大。  

为了掌握聚合，你只需要了解两个主要概念：(参考https://blog.csdn.net/dm_vincent/article/details/42387161)  

Buckets(桶)：满足某个条件的文档集合。  

Metrics(指标)：为某个桶中的文档计算得到的统计信息。  

就是这样！每个聚合只是简单地由一个或者多个桶，零个或者多个指标组合而成。可以将它粗略地转换为SQL：  

    SELECT COUNT(color)   
    FROM table  
    GROUP BY color  

以上的COUNT(color)就相当于一个指标。GROUP BY color则相当于一个桶。  

桶和SQL中的组(Grouping)拥有相似的概念，而指标则与COUNT()，SUM()，MAX()等相似。  

让我们仔细看看这些概念。  
桶(Buckets)  
  
一个桶就是满足特定条件的一个文档集合：  

    一名员工要么属于男性桶，或者女性桶。  
    城市Albany属于New York州这个桶。  
    日期2014-10-28属于十月份这个桶。  

随着聚合被执行，每份文档中的值会被计算来决定它们是否匹配了桶的条件。如果匹配成功，那么该文档会被置入该桶中，同时聚合会继续执行。  

桶也能够嵌套在其它桶中，能让你完成层次或者条件划分这些需求。比如，Cincinnati可以被放置在Ohio州这个桶中，而整个Ohio州则能够被放置在美国这个桶中。  

ES中有很多类型的桶，让你可以将文档通过多种方式进行划分(按小时，按最流行的词条，按年龄区间，按地理位置，以及更多)。但是从根本上，它们都根据相同的原理运作：按照条件对文档进行划分。  
指标(Metrics)

桶能够让我们对文档进行有意义的划分，但是最终我们还是需要对每个桶中的文档进行某种指标计算。分桶是达到最终目的的手段：提供了对文档进行划分的方法，从而让你能够计算需要的指标。  

多数指标仅仅是简单的数学运算(比如，min，mean，max以及sum)，它们使用文档中的值进行计算。在实际应用中，指标能够让你计算例如平均薪资，最高出售价格，或者百分之95的查询延迟。  
将两者结合起来

一个聚合就是一些桶和指标的组合。一个聚合可以只有一个桶，或者一个指标，或者每样一个。在桶中甚至可以有多个嵌套的桶。比如，我们可以将文档按照其所属国家进行分桶，然后对每个桶计算其平均薪资(一个指标)。  

因为桶是可以嵌套的，我们能够实现一个更加复杂的聚合操作：  

    将文档按照国家进行分桶。(桶)  
    然后将每个国家的桶再按照性别分桶。(桶)  
    然后将每个性别的桶按照年龄区间进行分桶。(桶)  
    最后，为每个年龄区间计算平均薪资。(指标)  

聚合查询都是由AggregationBuilders创建的，一些常见的聚合查询如下  

（参考：http://blog.csdn.net/u010454030/article/details/63266035）  

    （1）统计某个字段的数量  
      ValueCountBuilder vcb=  AggregationBuilders.count("count_uid").field("uid");  
    （2）去重统计某个字段的数量（有少量误差）  
     CardinalityBuilder cb= AggregationBuilders.cardinality("distinct_count_uid").field("uid");  
    （3）聚合过滤  
    FilterAggregationBuilder fab= AggregationBuilders.filter("uid_filter").filter(QueryBuilders.queryStringQuery("uid:001"));  
    （4）按某个字段分组  
    TermsBuilder tb=  AggregationBuilders.terms("group_name").field("name");  
    （5）求和  
    SumBuilder  sumBuilder= AggregationBuilders.sum("sum_price").field("price");  
    （6）求平均  
    AvgBuilder ab= AggregationBuilders.avg("avg_price").field("price");  
    （7）求最大值  
    MaxBuilder mb= AggregationBuilders.max("max_price").field("price");   
    （8）求最小值  
    MinBuilder min= AggregationBuilders.min("min_price").field("price");  
    （9）按日期间隔分组  
    DateHistogramBuilder dhb= AggregationBuilders.dateHistogram("dh").field("date");  
    （10）获取聚合里面的结果      
    TopHitsBuilder thb=  AggregationBuilders.topHits("top_result");    
    （11）嵌套的聚合        
    NestedBuilder nb= AggregationBuilders.nested("negsted_path").path("quests");      
    （12）反转嵌套    
    AggregationBuilders.reverseNested("res_negsted").path("kps ");  

聚合查询的详细使用步骤如下：    

            public void test(){  
            //目标：搜索写博客写得最多的用户（一个博客对应一个用户），通过搜索博客中的用户名的频次来达到想要的结果  
            //首先新建一个用于存储数据的集合      
            List<String> ueserNameList=new ArrayList<>();    
            //1.创建查询条件，也就是QueryBuild  
            QueryBuilder matchAllQuery = QueryBuilders.matchAllQuery();//设置查询所有，相当于不设置查询条件  
            //2.构建查询  
            NativeSearchQueryBuilder nativeSearchQueryBuilder = new NativeSearchQueryBuilder();  
            //2.0 设置QueryBuilder    
            nativeSearchQueryBuilder.withQuery(matchAllQuery);  
            //2.1设置搜索类型，默认值就是QUERY_THEN_FETCH，参考https://blog.csdn.net/wulex/article/details/71081042  
            nativeSearchQueryBuilder.withSearchType(SearchType.QUERY_THEN_FETCH);//指定索引的类型，只先从各分片中查询匹配的文档，再重新排序和排名，取前size个文档  
            //2.2指定索引库和文档类型  
            nativeSearchQueryBuilder.withIndices("myBlog").withTypes("blog");//指定要查询的索引库的名称和类型，其实就是我们文档@Document中设置的indedName和type  
            //2.3重点来了！！！指定聚合函数,本例中以某个字段分组聚合为例（可根据你自己的聚合查询需求设置）  
            //该聚合函数解释：计算该字段(假设为username)在所有文档中的出现频次，并按照降序排名（常用于某个字段的热度排名）  
            TermsBuilder termsAggregation = AggregationBuilders.terms("给聚合查询取的名").field("username").order(Terms.Order.count(false));  
            nativeSearchQueryBuilder.addAggregation(termsAggregation);  
            //2.4构建查询对象  
            NativeSearchQuery nativeSearchQuery = nativeSearchQueryBuilder.build();  
            //3.执行查询  
            //3.1方法1,通过reporitory执行查询,获得有Page包装了的结果集  
            Page<EsBlog> search = esBlogRepository.search(nativeSearchQuery);  
            List<EsBlog> content = search.getContent();  
            for (EsBlog esBlog : content) {  
                ueserNameList.add(esBlog.getUsername());  
            }  
            //获得对应的文档之后我就可以获得该文档的作者，那么就可以查出最热门用户了  
            //3.2方法2,通过elasticSearch模板elasticsearchTemplate.queryForList方法查询  
            List<EsBlog> queryForList = elasticsearchTemplate.queryForList(nativeSearchQuery, EsBlog.class);  
            //3.3方法3,通过elasticSearch模板elasticsearchTemplate.query()方法查询,获得聚合(常用)  
            Aggregations aggregations = elasticsearchTemplate.query(nativeSearchQuery, new ResultsExtractor<Aggregations>() {  
                @Override  
                public Aggregations extract(SearchResponse response) {  
                    return response.getAggregations();  
                }  
            });  
            //转换成map集合  
            Map<String, Aggregation> aggregationMap = aggregations.asMap();  
            //获得对应的聚合函数的聚合子类，该聚合子类也是个map集合,里面的value就是桶Bucket，我们要获得Bucket  
            StringTerms stringTerms = (StringTerms) aggregationMap.get("给聚合查询取的名");  
            //获得所有的桶  
            List<Bucket> buckets = stringTerms.getBuckets();  
            //将集合转换成迭代器遍历桶,当然如果你不删除buckets中的元素，直接foreach遍历就可以了  
            Iterator<Bucket> iterator = buckets.iterator();  
            
            while(iterator.hasNext()) {  
                //bucket桶也是一个map对象，我们取它的key值就可以了  
                String username = iterator.next().getKeyAsString();//或者bucket.getKey().toString();  
                //根据username去结果中查询即可对应的文档，添加存储数据的集合  
                ueserNameList.add(username);  
            }  
            //最后根据ueserNameList搜索对应的结果集  
            List<User> listUsersByUsernames = userService.listUsersByUsernames(ueserNameList);  
    }  
