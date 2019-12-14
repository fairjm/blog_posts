---
title: "ES踩坑笔记"
date: 2018/11/13 15:42:00
---
现在开始在业务上使用ES,记录一些踩坑经历,做点笔记.  

---  

2018-11-13  
# source不返回问题  
使用了角色校验,客户端插入成功之后获取数据没有source,和查询参数无关.  
检查mapping,发现获取mapping也是空...  
如下:  
```
{'test_index': {'mappings': {'test_doc': {'properties': {}}}}}  
```  
排查了一会儿..找不出原因.  
后来要到了一个高权限的账号去kibana看了眼...发现  

![](/images/1244488-20181113154038997-784426282.png)  
能获取的fields为空... ...  
emmmmm....   
设置为`*`后解决  

参考链接:  
https://www.elastic.co/guide/en/elastic-stack-overview/6.4/field-level-security.html  

---

2018-11-16  

# termQuery不返回结果  

emmmmm 一个id是用了dynamic mapping插入,默认使用的是standard分词器.  
一些id有中划线,分词结果:
```
GET /_analyze
{
  "analyzer": "standard",
  "text": "1111-2222-4444-3333-55555"
}
```  
```
{
  "tokens": [
    {
      "token": "1111",
      "start_offset": 0,
      "end_offset": 4,
      "type": "<NUM>",
      "position": 0
    },
    {
      "token": "2222",
      "start_offset": 5,
      "end_offset": 9,
      "type": "<NUM>",
      "position": 1
    },
    {
      "token": "4444",
      "start_offset": 10,
      "end_offset": 14,
      "type": "<NUM>",
      "position": 2
    },
    {
      "token": "3333",
      "start_offset": 15,
      "end_offset": 19,
      "type": "<NUM>",
      "position": 3
    },
    {
      "token": "55555",
      "start_offset": 20,
      "end_offset": 25,
      "type": "<NUM>",
      "position": 4
    }
  ]
}
```  
然后在代码里使用的是termQuery:  
```java  
SearchRequest searchRequest = new SearchRequest(INDEX_NAME);
searchRequest.types(TYPE_NAME);
SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
sourceBuilder.query(QueryBuilders.termQuery(ID, id));
```  
termQuery是不对搜索词分词的...   
于是就什么都查不出来..  
解决方式是用dynamic mapping自动生成的keyword field.  

参考:  
https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-analyzers.html  

---  

2018-12-14

# 处理深分页  

默认的`from`+`size`限制是10000,大于这个会报错:  
```  
{
  "error": {
    "root_cause": [
      {
        "type": "query_phase_execution_exception",
        "reason": "Result window is too large, from + size must be less than or equal to: [1000000] but was [1000001]. See the scroll api for a more efficient way to request large data sets. This limit can be set by changing the [index.max_result_window] index level setting."
      }
    ],
```  
深分页的操作对coordinator会造成很大的影响,会占用大量heap存放数据并进行排序操作.  
业务上没有注意,结果就超了... ...  
可以设定`index.max_result_window`来提高上限.  
但如果真的要有这么深,还是使用search after比较可靠.又因为是实时业务查询,所以用scroll是不合适的.  

参考:  
https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules.html  
https://www.elastic.co/guide/en/elasticsearch/reference/current/search-request-search-after.html
