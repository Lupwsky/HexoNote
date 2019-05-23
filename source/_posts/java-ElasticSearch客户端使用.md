---
title: Java-ElasticSearch客户端使用
date: 2019-04-26 21:59:40
categories: Java
---

ElasticSearch Java 客户端在 SpringBoot 项目中的基础使用使用, 参考资料: [官方文档](), [ElasticSearch Java API](https://es.quanke.name/) 

# 创建客户端

```java
@Configurable
public class ElasticSearchConfig {
    @Bean
    public RestHighLevelClient elasticSearchClient() {
        return new RestHighLevelClient(RestClient.builder(
                        new HttpHost("localhost", 9200, "http")));
    }
}
```

<!-- more -->

# 添加文档

指定 ID 处添加文档, 如果索引库不存在就会创建, 单纯的创建索引库使用 CreateIndexRequest, 如果 ID 已经存在则会替换文档

参考资料:

* [官方文档](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/6.6/java-rest-high-document-index.html)
* [Index 和 Type 的区别](http://bayescafe.com/database/elasticsearch-using-index-or-type.html)

```java
@PostMapping("/elasticsearch/create/index/test")
public void createIndexTest() {
    Map<String, Object> jsonMap = Maps.newHashMap();
    jsonMap.put("name", "lpw");
    jsonMap.put("password", "test1111");

    IndexRequest indexRequest = new IndexRequest(ES_INDEX_1, ES_TYPE, "1");
    indexRequest.source(jsonMap);

    try {
        client.index(indexRequest, RequestOptions.DEFAULT);
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

# 查询文档

根据 ID 查询文档, 参考资料: [官方文档](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/6.6/java-rest-high-document-get.html)

```java
@PostMapping("/elasticsearch/get/request/test")
public void getRequestTest() {
    GetRequest getRequest = new GetRequest(ES_INDEX, ES_TYPE, "1");
    try {
        GetResponse getResponse = client.get(getRequest, RequestOptions.DEFAULT);
        log.info("index = {}", getResponse.getIndex());
        log.info("type = {}", getResponse.getType());
        log.info("id = {}", getResponse.getId());
        log.info("source = {}", getResponse.getSource());
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

# 检测文档

根据 ID 查询文档是否存在, 参考资料: [官方文档](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/6.6/java-rest-high-document-exist.html)

```java
@PostMapping("/elasticsearch/exist/request/test")
public void existRequestTest() {
    GetRequest getRequest = new GetRequest(ES_INDEX, ES_TYPE, "1");
    try {
        boolean isExist = client.exists(getRequest, RequestOptions.DEFAULT);
        log.info("isExist = {}", isExist);
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

# 删除文档

根据 ID 删除文档, 参考资料: [官方文档](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/6.6/java-rest-high-document-delete.html)

```java
@PostMapping("/elasticsearch/delete/request/test")
public void deleteRequestTest() {
    DeleteRequest deleteRequest = new DeleteRequest(ES_INDEX, ES_TYPE, "2");
    try {
        DeleteResponse deleteResponse = client.delete(deleteRequest, RequestOptions.DEFAULT);
        log.info("deleteStatus = {}", deleteResponse.status().getStatus());
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

# 更新文档

更新文档, 参考资料: [官方文档](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/6.6/java-rest-high-document-update.html)

```java
@PostMapping("/elasticsearch/update/request/test")
public void updateRequestTest() {
    UpdateRequest updateRequest = new UpdateRequest(ES_INDEX, ES_TYPE, "1");
    try {
        XContentBuilder builder = XContentFactory.jsonBuilder();
        builder.startObject();
        builder.field("name", "lpw123");
        builder.field("password", "lpw123");
        builder.endObject();

        UpdateResponse updateResponse = client.update(updateRequest, RequestOptions.DEFAULT);
        GetResult getResult = updateResponse.getGetResult();
        if (getResult.isExists()) {
            log.info("source = {}", getResult.getSource());
        }
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

# 查询统计信息

获取指定文档中指定字段的分词信息和统计信息, 可以设置指定返回哪些统计信息, 参考资料: [官方文档](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/6.6/java-rest-high-document-term-vectors.html)

```java
@PostMapping("/elasticsearch/term/request/test")
public void termVectorsRequestTest() {
    // 将数据存入 ES 的过程就是先将数据分词然后添加到倒排索引表的过程, 一个分词存放在一个 term 中
    TermVectorsRequest termVectorsRequest = new TermVectorsRequest(ES_INDEX, ES_TYPE, "1");
    termVectorsRequest.setFields("name", "password");
    try {
        TermVectorsResponse response = client.termvectors(termVectorsRequest, RequestOptions.DEFAULT);
        log.info("found 字段是否存在 = {}", response.getFound());

        TermVectorsResponse.TermVector termVector = response.getTermVectorsList().get(0);
        log.info("filedName 字段名称 = {}", termVector.getFieldName());

        TermVectorsResponse.TermVector.FieldStatistics fieldStatistics = termVector.getFieldStatistics();
        log.info("docCount 文档的数量 = {}", fieldStatistics.getDocCount());
        log.info("sumDocFreq 出现频率的总和 = {}", fieldStatistics.getSumDocFreq());
        log.info("sumTotalTermFreq 文档频率的总和 = {}", fieldStatistics.getSumTotalTermFreq());

        TermVectorsResponse.TermVector.Term term = termVector.getTerms().get(0);
        log.info("分词的值 = {}", term.getTerm());
        log.info("分词的频率 = {}", term.getTermFreq());
        log.info("分词在文档频率 = {}", term.getDocFreq());
        log.info("分词出现的总频率 = {}", term.getTotalTermFreq());
        log.info("分词的的分数 = {}", term.getScore());

        TermVectorsResponse.TermVector.Token token = term.getTokens().get(0);
        log.info("payload = {}", token.getPayload());
        log.info("position 分词在文档中的位置 = {}", token.getPosition());
        log.info("startOffset 分词的开始位置 = {}", token.getStartOffset());
        log.info("endOffset 分词的结束位置 = {}", token.getEndOffset());
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

# 批量操作

Bulk API = 可组合多个 Request, 一次执行增删改产的 Request, 参考资料: [官方文档](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/6.6/java-rest-high-document-bulk.html)
Multi API = 在一次请求中并行的执行多个 GetRequest, 参考资料: [官方文档](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/6.6/java-rest-high-document-multi-get.html)
Multi Term Vectors API = 批量操作获取分词的信息和统计信息, 参考资料: [官方文档](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/7.0/java-rest-high-document-multi-term-vectors.html)

```java
@PostMapping("/elasticsearch/bulk/request/test")
public void bulkRequestTest() {
    Map<String, Object> jsonMap = Maps.newHashMap();
    jsonMap.put("nameBulkTest", "lpw1");
    jsonMap.put("passwordBulkTest", "test222");

    BulkRequest bulkRequest = new BulkRequest();
    bulkRequest.add(new IndexRequest(ES_INDEX, ES_TYPE, "1").source(jsonMap));
    bulkRequest.add(new UpdateRequest(ES_INDEX, ES_TYPE, "1").doc(XContentType.JSON, jsonMap));
    try {
        BulkResponse bulkResponse = client.bulk(bulkRequest, RequestOptions.DEFAULT);
        BulkItemResponse[] items = bulkResponse.getItems();
        Arrays.stream(items).forEach(bulkItemResponse -> {
            if (!bulkItemResponse.isFailed()) {
                DocWriteRequest.OpType opType = bulkItemResponse.getOpType();
                switch (opType) {
                    case INDEX:
                        IndexResponse indexResponse = bulkItemResponse.getResponse();
                        log.info("indexResponse = {}", JSONObject.toJSONString(indexResponse));
                        break;
                    case CREATE:
                        IndexResponse createIndexResponse = bulkItemResponse.getResponse();
                        log.info("createIndexResponse = {}", JSONObject.toJSONString(createIndexResponse));
                        break;
                    case DELETE:
                        UpdateResponse updateResponse = bulkItemResponse.getResponse();
                        log.info("updateResponse = {}", JSONObject.toJSONString(updateResponse));
                        break;
                    case UPDATE:
                        DeleteResponse deleteResponse = bulkItemResponse.getResponse();
                        log.info("deleteResponse = {}", JSONObject.toJSONString(deleteResponse));
                        break;
                    default:
                        break;
                }
            } else {
                log.info("获取操作失败的信息 = {}", JSONObject.toJSONString(bulkItemResponse.getFailure()));
            }
        });
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

# 复制索引库

将文档从一个索引库或者多个索引库复制到指定的索引库, 新的接口和官方文档有不同, 参考资料: [官方文档](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/7.0/java-rest-high-document-reindex.html)

```java
@PostMapping("/elasticsearch/reindex/test")
public void reindexTest() {
    // 将 ES_INDEX 索引库中所有符合条件的文档复制到 ES_INDEX_1 索引库的 ES_TYPE 下
    SearchRequest searchRequest = new SearchRequest(ES_INDEX);
    SearchSourceBuilder searchSourceBuilder = SearchSourceBuilder.searchSource()
            .query(new TermQueryBuilder("name", "lpw"));
    ReindexRequest reindexRequest = new ReindexRequest(searchRequest, new IndexRequest(ES_INDEX_1, ES_TYPE));
}
```

# 更新和删除索引库

Update By Query API = 在不更改源的情况下更新索引库中的文档, 新的接口和官方文档有不同, 参考资料: [官方文档](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/7.0/java-rest-high-document-update-by-query.html)
DELETE BY QUERY API = 删除符合指定索引库中符合查询条件的文档, 新的接口和官方文档有不同, 参考资料: [官方文档](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/7.0/java-rest-high-document-delete-by-query.html)

```java
@PostMapping("/elasticsearch/update/by/query/api/test")
public void updateByQueryApi() {
    // 设置需要更新文档的索引库, 可设置多个
    SearchRequest searchRequest = new SearchRequest(ES_INDEX, ES_INDEX_1);
    // 设置条件, 例如这里 name = lpw 的文档事符合更改条件的文档, 并这设置处理文档的限制数量
    SearchSourceBuilder searchSourceBuilder = SearchSourceBuilder.searchSource()
            .query(new TermQueryBuilder("name", "lpw"))
            .size(100);
    searchRequest.source(searchSourceBuilder);

    UpdateByQueryRequest updateByQueryRequest = new UpdateByQueryRequest(searchRequest);
    try {
        BulkByScrollResponse bulkByScrollResponse = client.updateByQuery(updateByQueryRequest, RequestOptions.DEFAULT);
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

# 简单查询

查询, 搜索指定索引库中的所有文档, 参考资料: [官方文档](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/7.0/java-rest-high-search.html)

```java
@PostMapping("/elasticsearch/get/all/doc/test")
public void getAllDocTest() throws IOException {
    SearchRequest searchRequest = new SearchRequest(ES_INDEX);
    SearchSourceBuilder searchSourceBuilder = SearchSourceBuilder.searchSource();
    searchSourceBuilder.query(QueryBuilders.matchAllQuery());
    searchRequest.source(searchSourceBuilder);
    SearchResponse searchResponse = client.search(searchRequest, RequestOptions.DEFAULT);
    log.info("searchResponse = {}", JSONObject.toJSONString(searchResponse));

    // Clusters
    // 在集群情况下搜索时, 保存集群的信息, 总数量，成功的数量和跳过的数量
    SearchResponse.Clusters clusters = searchResponse.getClusters();
    log.info("集群中服务的总数量 = {}", clusters.getTotal());
    log.info("成功的数量 = {}", clusters.getSuccessful());
    log.info("跳过的数量 = {}", clusters.getSkipped());

    // 执行搜索时的使用的分片信息
    log.info("搜索执行失败的分片数 = {}", searchResponse.getFailedShards());
    log.info("执行搜索的分片总数 = {}", searchResponse.getTotalShards());
    log.info("执行搜索的跳过分片总数 = {}", searchResponse.getSkippedShards());
    log.info("执行搜索成功的分片数 = {}", searchResponse.getSuccessfulShards());
    log.info("执行搜索失败的分片原因集合 = {}", JSONObject.toJSONString(searchResponse.getShardFailures()));

    // 如果启用了性能分析, 则会返回包含每个分片的概要文件结果的对象, 如果未启用分析，则返回 null
    // 指向的是 $.hits.hits[0].fields 字段
    log.info("profileResults = {}", JSONObject.toJSONString(searchResponse.getProfileResults()));

    // 搜索是否超时
    log.info("搜索是否超时 = {}", searchResponse.isTimedOut());

    // 搜索使用的时间信息
    TimeValue took = searchResponse.getTook();
    log.info("搜索总耗时 = {}", took.getStringRep());

    // 命中的文档数据
    SearchHits responseHits = searchResponse.getHits();
    log.info("命中的文档总数 = {}", responseHits.getTotalHits());
    log.info("命中文档数据中的分数最大值 = {}", responseHits.getMaxScore());
    log.info("文档数据 = {}", responseHits.getAt(0));

    // 输出示例数据:
    //{
    //    "fields": {
    //        "$ref": "$.hits.hits[0].fields"
    //    },
    //    "fragment": false,
    //    "highlightFields": {
    //        "$ref": "$.hits.hits[0].fields"
    //    },
    //    "id": "poKZUmoBAPeAwT9XkOG4",
    //    "matchedQueries": [],
    //    "score": 1.0,
    //    "sortValues": [],
    //    "sourceAsMap": {
    //        "password": "lpw2",
    //        "username": "lpw2",
    //        "desc": "你好, 这是测试数据2",
    //        "timestamp": "2019-04-25 09:45:56"
    //    },
    //    "sourceAsString": "{\"username\":\"lpw2\",\"password\":\"lpw2\",\"desc\":\"你好, 这是测试数据2\",\"timestamp\":\"2019-04-25 09:45:56\"}",
    //    "sourceRef": {
    //        "childResources": [],
    //        "fragment": true
    //    },
    //    "type": "user_info",
    //    "version": -1
    //}
}
```

# 简单查询-添加查询条件

查询符合条件的文档, 参考资料: [官方文档](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/7.0/java-rest-high-search.html)

```java
@PostMapping("/elasticsearch/search/request/test")
public void searchRequestTest(String fieldName, String value) throws IOException {
    // Simple Query = 简单查询
    SearchRequest searchRequest = new SearchRequest(ES_INDEX, ES_INDEX_1);
    SearchSourceBuilder searchSourceBuilder = SearchSourceBuilder.searchSource();
    searchSourceBuilder.query(QueryBuilders.termQuery(fieldName, value));
    searchSourceBuilder.from(0);
    searchSourceBuilder.size(1);
    searchSourceBuilder.timeout(new TimeValue(60, TimeUnit.MINUTES));
    searchRequest.source(searchSourceBuilder);

    SearchResponse searchResponse = client.search(searchRequest, RequestOptions.DEFAULT);
    Arrays.stream(searchResponse.getHits().getHits()).forEach(searchHit -> {
        log.info("source = {}, highlighter = {}", searchHit.getSourceAsString(), searchHit.getHighlightFields());
    });
}
```

# DSL 查询

添加复杂的查询条件, 参考资料: [官方文档](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/7.0/java-rest-high-search.html), [前缀查询, 模糊查询, 正则查询](https://blog.csdn.net/SunnyYoona/article/details/52852483)

```java
@PostMapping("/elasticsearch/match/query/test")
public void matchQueryTest(String fieldName, String value) throws IOException {
    SearchRequest matchSearchRequest = new SearchRequest(ES_INDEX);
    SearchSourceBuilder matchSearchSourceBuilder = SearchSourceBuilder.searchSource();
    MatchQueryBuilder matchQueryBuilder = new MatchQueryBuilder(fieldName, value);
    // 启用 fuzziness 模糊查询, 5.0 已废弃, 不建议使用
    // ElasticSearch 5.4 中文文档 Fuzzy Query 查询 = http://cwiki.apachecn.org/pages/viewpage.action?pageId=4882439
    // matchQueryBuilder.fuzziness(Fuzziness.AUTO);
    // 模糊查询最少公共匹配前缀, 举几个例子:
    // filedName = username, value = lpw1, prefixLength = 0, 可匹配到 username 的值是以 lpw 开头的所有数据
    // filedName = username, value = pw1, prefixLength = 1, 可以匹配到
    // matchQueryBuilder.prefixLength(1);
    // 模糊查询最大编辑距离
    // matchQueryBuilder.maxExpansions(2);

    // 设定查询的 limit 和排序规则
    // 这里调试排序出现异常, 聚合操作异常 = https://blog.csdn.net/wwd0501/article/details/78490201
    // 排序 = matchSearchSourceBuilder.sort(new FieldSortBuilder("username").order(SortOrder.DESC))
    matchSearchSourceBuilder.query(matchQueryBuilder);
    matchSearchSourceBuilder.from(0);
    matchSearchSourceBuilder.size(10);

    // 设置需要高亮的字段
    HighlightBuilder highlightBuilder = new HighlightBuilder();
    highlightBuilder.field("username");
    matchSearchSourceBuilder.highlighter(highlightBuilder);

    matchSearchRequest.source(matchSearchSourceBuilder);

    // 开始查询
    SearchResponse searchResponse = client.search(matchSearchRequest, RequestOptions.DEFAULT);
    Arrays.stream(searchResponse.getHits().getHits()).forEach(searchHit -> {
        log.info("source = {}, highlighter = {}", searchHit.getSourceAsString(), searchHit.getHighlightFields());
    });
}
```

# 大量数据查询

使用 SearchScrollRequest 对大量数据进行搜索, 先使用普通 SearchRequest 获取 scrollId 后使用 SearchScrollRequest 请求进行查询, 再使用上次的 scrollId 进行下次查询, 直到查询数据完成, 参考资料: [官方文档](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/6.6/java-rest-high-search-scroll.html`), [中文文档](https://www.bookstack.cn/read/elasticsearch-java-rest/java-high-level-rest-client-supported-apis-search-scroll-api.md)

```java
@PostMapping("/elasticsearch/scroll/request/test")
public void searchScrollRequestTest(String fieldName, String value) throws IOException {
    // 普通 SearchRequest 获取 scrollId, scroll() 方法设置下一次查询的时间间隔
    SearchRequest searchRequest = new SearchRequest(ES_INDEX, ES_INDEX_1);
    SearchSourceBuilder searchSourceBuilder = SearchSourceBuilder.searchSource();
    searchSourceBuilder.query(QueryBuilders.termQuery(fieldName, value));
    searchSourceBuilder.size(1000);
    searchSourceBuilder.timeout(new TimeValue(60, TimeUnit.MINUTES));
    searchRequest.source(searchSourceBuilder);
    searchRequest.scroll(TimeValue.timeValueSeconds(30));

    // 获取 scrollId 并处理此次查询出来的数据
    SearchResponse searchResponse = client.search(searchRequest, RequestOptions.DEFAULT);
    String scrollId = searchResponse.getScrollId();
    log.info("scrollId = {}", scrollId);
    log.info("当前命中的数据数量 = {}", searchResponse.getHits().totalHits);

    // 使用此新的滚动标识符来搜索下一批数据
    SearchScrollRequest scrollRequest = new SearchScrollRequest(scrollId);
    scrollRequest.scroll(TimeValue.timeValueSeconds(30));
    SearchResponse searchScrollResponse = client.scroll(scrollRequest, RequestOptions.DEFAULT);
    scrollId = searchScrollResponse.getScrollId();
    log.info("scrollId = {}", scrollId);
    log.info("当前命中的数据数量 = {}", searchScrollResponse.getHits().totalHits);

    // 重复上面的过程, 直到数据检索完成, 然后调用 ClearScrollRequest 清除滚动上下文
    // 当滚动上下文到期时, 会自动清除, 但最建议是当滚动会话结束后尽快释放资源
    ClearScrollRequest clearScrollRequest = new ClearScrollRequest();
    clearScrollRequest.addScrollId(scrollId);
    ClearScrollResponse clearScrollResponse = client.clearScroll(clearScrollRequest, RequestOptions.DEFAULT);
    log.info("清除结果 = {}", clearScrollResponse.isSucceeded());
}
```

# 批量执行多个查询

批量执行多个查询请求, 参考资料: [官方文档](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/6.6/java-rest-high-multi-search.html)

```java
@PostMapping("/elasticsearch/multi/request/test")
public void multiSearchRequestTest(String fieldName, String value) throws IOException {
    SearchRequest searchRequest1 = new SearchRequest(ES_INDEX);
    SearchSourceBuilder searchSourceBuilder1 = new SearchSourceBuilder();
    searchSourceBuilder1.query(QueryBuilders.termQuery(fieldName, value));
    searchRequest1.source(searchSourceBuilder1);

    SearchRequest searchRequest2 = new SearchRequest();
    SearchSourceBuilder searchSourceBuilder2 = new SearchSourceBuilder();
    searchSourceBuilder2.query(QueryBuilders.termQuery(fieldName, value));
    searchRequest2.source(searchSourceBuilder2);

    MultiSearchRequest multiSearchRequest = new MultiSearchRequest();
    multiSearchRequest.add(searchRequest1);
    multiSearchRequest.add(searchRequest2);

    MultiSearchResponse multiSearchResponse = client.msearch(multiSearchRequest, RequestOptions.DEFAULT);
    MultiSearchResponse.Item[] responses = multiSearchResponse.getResponses();
    Arrays.stream(responses).forEach(response -> {
        log.info("hitCount = {}", response.getResponse().getHits().totalHits);
    });
}
```

# 跨索引库查询

跨越多个库对字段进行检索, 对于每个请求的字段, 返回的 FieldCapabilitiesResponse 包含其类型以及是否可以搜索或是否可以聚合等信息, 参考资料: [官方文档](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/6.6/java-rest-high-field-caps.html)

```java
@PostMapping("/elasticsearch/filed/capabilities/request/test")
public void filedCapabilitiesRequestTest() throws IOException {
    FieldCapabilitiesRequest fieldCapabilitiesRequest = new FieldCapabilitiesRequest();
    fieldCapabilitiesRequest.fields("username", "password");
    fieldCapabilitiesRequest.indices(ES_INDEX, ES_INDEX_1);
    FieldCapabilitiesResponse response = client.fieldCaps(fieldCapabilitiesRequest, RequestOptions.DEFAULT);
    Map<String, FieldCapabilities> fieldCapabilitiesMap = response.getField("username");
    fieldCapabilitiesMap.forEach((type, fieldCapabilities) -> {
        log.info("type = {}, fieldCapabilities = {}", type, JSONObject.toJSONString(fieldCapabilities));
    });
}
```

# 计数统计

计数统计, 获取匹配结果的数量, 参考资料: [官方文档](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/6.6/java-rest-high-count.html)

```java
@PostMapping("/elasticsearch/count.request/test")
public void countRequestTest() throws IOException {
    CountRequest countRequest = new CountRequest(ES_INDEX);
    SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
    searchSourceBuilder.query(QueryBuilders.matchQuery("username", "lpw1"));
    countRequest.source(searchSourceBuilder);
    CountResponse countResponse = client.count(countRequest, RequestOptions.DEFAULT);
    log.info("matchCount = {}", countResponse.getCount());
}
```