---
layout: post
title:  "ElasticSearch笔记-keyword、text、numeric"
date:   2020-05-06 17:28 +0800
categories: es

---

### 分词、倒排

ES 中有**精确值**（Exact Values）与**全文本**（Full Text）之分：

- 精确值：包括数字，日期，一个具体字符串（例如"Hello World”）。
  - 在 ES 中用 [**keyword**](https://www.elastic.co/guide/en/elasticsearch/reference/current/keyword.html) 、numeric数据类型表示。
  - 精确值**不需要**做分词处理。
- 全文本：非结构化的文本数据。
  - 在 ES 中用 **[text](https://www.elastic.co/guide/en/elasticsearch/reference/current/text.html)** 数据类型表示。
  - 全文本**需要**做分词处理。

**分词**主要用于建立**单词**（Term / Token）与**倒排索引**对应关系，比如将 `hello world` 拆分为 `hello` 和 `world`

当用户向ES发送一个搜索请求的时候，搜索引擎经过了以下步骤：

1. 分词器对搜索字符串进行分词处理。
2. 在倒排索引表中查到匹配的文档。
3. 对每个匹配的文档进行相关性评分。
4. 根据相关性评分对文档进行排序。
5. 将排好序的文档返回给用户。

### **[numeric](https://www.elastic.co/guide/en/elasticsearch/reference/current/number.html)**存储

因为numeric支持等值精确查询，所以字段类型到底使用keyword还是numeric？答案是keyword。

numeric类型为了能有效的支持范围查询，它的存储结构并不是倒排索引。numeric类型从lucene6.0开始，使用了一种名为`block KD tree`的存储结构。[Block KD tree介绍在这里](https://jelinet.com/es/2020/04/23/ES%E7%AC%94%E8%AE%B0-Lucene-BKD%E6%A0%91.html){:target="_blank"}，不过多阐述。

具体的ES内部（其实是Lucene）目前的版本是基于所谓的PointValues，比如整型在Lucene内部是IntPoint类表示，还有DoublePoint等，完整的对应关系是：

| Java type | Lucene class |
| --------- | ------------ |
| int       | IntPoint     |
| long      | LongPoint    |
| float     | DoublePoint  |
| byte[]    | BinaryPoint  |


而这些PointValues是基于kd-tree存储的，根据官方文档的介绍，lucene把叶子节点在磁盘是顺序存储的，这样搜索的效率就会非常高。

#### 为啥numeric对于term精确匹配的查询性能没有keyword好

前面我们提到了IntPoint类，这个类有三个查询方法：

```java
//构造精确查询，内部还是调用newRangeQuery
Query newExactQuery(String field, int value) 

//构造一维查询，内部是调用多维查询的方法
Query newRangeQuery(String field, int lowerValue, int upperValue)
//构造多维查询
Query newRangeQuery(String field, int[] lowerValue, int[] upperValue)
```


比如有这样一个索引：

```json
PUT blogs 
{
  "mappings": {
    "properties": { 
      "title":    { "type": "keyword"  }, 
      "content":    { "type": "text"  }, 
      "status":     { "type": "integer"  }
    }
  }
}
```

基于status查询

```json
{
  "query": {
    "term": {
      "title": {
        "status": 2
      }
    }
  }
}
```


在lucene内部其实还是进行了一个2~2的范围查询。即便kd-tree的性能也很高，但是对于这种精确查询还是要到树上走一遭，而倒排索引相当于是直接在内存里就定位到了结果集的文档id。如果是bool组合查询的话，term还可以利用跳表，这点numeric字段也是做不到的。



