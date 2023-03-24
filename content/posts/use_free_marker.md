---
title: "Freemarker 的使用"
date: 2022-04-25T11:22:13+08:00
draft: false
categories: ["Tech"]
---

## Freemarker
freemarker 是一款开源的模版引擎，可以基于模版方便的生成结果。
https://freemarker.apache.org/

## Freemarker 的使用

### 编写 ftl 模版
以生成 postgres 查询的 sql 语句为例
编写 delete.ftl 文件，${} 中的字段是参数
``` sql
DROP TABLE IF EXISTS ${schema}.${name};
```
当然实际使用中的模版可能复杂的多，以部分创建表模版为例
我们可以在模版中使用 `import` 引入其它模版
使用 `assign` 设置变量
使用 `if`, `list` 等
使用 `??` 判断是否为空，使用 `!false` 如果为空，默认 false
```
<#import "../../macro/constraints.ftl" as CONSTRAINTS>
<#assign with_clause = false>
<#if fillfactor!false || parallel_workers!false || toast_tuple_target!false || (autovacuum_custom!false && add_vacuum_settings_in_sql!false) || autovacuum_enabled == 't' || autovacuum_enabled == 'f' || (toast_autovacuum!false && add_vacuum_settings_in_sql!false) || toast_autovacuum_enabled == 't' || toast_autovacuum_enabled == 'f' >
    <#assign with_clause = true>
</#if>
<#list columns as c >
```
其它模版使用可以参考 https://freemarker.apache.org/docs/dgui_template.html
### java 使用 freemarker
```java
@NoArgsConstructor(access = AccessLevel.PRIVATE)
public final class FreemarkerManager {
    
    private static final FreemarkerManager INSTANCE = new FreemarkerManager();
    
    private final Configuration templateConfig = createTemplateConfiguration();
    
    /**
     * Get freemarker manager instance.
     * 
     * @return freemarker manager instance
     */
    public static FreemarkerManager getInstance() {
        return INSTANCE;
    }
    
    @SneakyThrows
    private Configuration createTemplateConfiguration() {
        Configuration result = new Configuration(Configuration.VERSION_2_3_31);
        result.setDirectoryForTemplateLoading(new File(Objects.requireNonNull(FreemarkerManager.class.getClassLoader().getResource("template")).getFile()));
        result.setDefaultEncoding("UTF-8");
        return result;
    }
    
    /**
     * Get sql from template.
     * 
     * @param data data
     * @param path path
     * @return sql
     */
    @SneakyThrows
    public static String getSqlFromTemplate(final Map<String, Object> data, final String path) {
        Template template = FreemarkerManager.getInstance().templateConfig.getTemplate(path);
        try (StringWriter result = new StringWriter()) {
            template.process(data, result);
            return result.toString();
        }
    }
}
```