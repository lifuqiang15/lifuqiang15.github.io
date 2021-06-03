# Elasticsearch 基本查询

`elasticsearch6.6`，需要对索引中的一个字段进行聚合查询，但是原来默认生成的mapping中该字段的type是`text`,是不能聚合操作的。然后通过api对该字段进行添加keyword属性。

### 相关代码

```
PUT /myindex/_mapping/doc
{
  "properties": {
      "myfield": {
      "type": "text",
      "fields":{
        "keyword":{
          "type":"keyword",
          "ignore_above":256
        }
      }
    }
  }
  
}
```

通过确认mapping文件，该操作是生效的。



```
Elasticsearch 中，text类型的字段是无法进行聚合，需要将字段fielddata设置为true才可以,进行修改：

PUT /event_news/_mapping
{"properties":{"news_source":{"type":"text","fielddata":true}}}

现在的mapping

  "news_source": {
     "type": "text",
     "fielddata": true
   },
   "related_freq": {
     "type": "integer"
}
```

