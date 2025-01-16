---
title: "Arthas watch ognl map 过滤"
date: 2025-01-06T18:09:11+08:00
draft: false
---

今天想通过 arthas watch 一个方法，并过滤入参，死活写不出来。折腾了半天终于搞出来了

例如我想监控这个方法
```java
    public <T> List<T> find(Query query, Class<T> entityClass) {
        return this.<T>find(query, entityClass, this.determineCollectionName(entityClass));
    }
```

    toString 输出的结构体如下
```
@Query[Query: { "code" : "A1b4a_6437_task_0006", "tenantId" : { "$in" : ["athenaPaaSDesigner", "SYSTEM"] }, "version" : "2.0" }, Fields: { }, Sort: { }]
```

```java
    public Document getQueryObject() {
        Document document = new Document();

        for(CriteriaDefinition definition : this.criteria.values()) {
            document.putAll(definition.getCriteriaObject());
        }

        if (!this.restrictedTypes.isEmpty()) {
            document.put("_$RESTRICTED_TYPES", this.getRestrictedTypes());
        }

        return document;
    }
```
最终的方式如下，getQueryObject() 返回的是一个 map，然后通过 get 方法过滤并判空
```
watch org.springframework.data.mongodb.core.MongoTemplate find "{params, returnObj}" "params[0].getQueryObject().get('code') != null && params[0].getQueryObject().get('code').equals('A1b4a_6437_task_0006')" -s -x 2
```

