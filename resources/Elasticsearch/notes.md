1.fields属性指定一个字段的多个type,可以以搜索user.suggest自动补全,也可以搜索user进行普通查询

2.ik_max_word是中文分词器中最小粒度分词,ik_smart是比较大力度分词

3.用最蠢的办法建表,不要用嵌套结构

4.高版本not_analyzed不起作用?要整字段匹配必须使用term和keyword组合查询

5.function_score可以用来提高某字段匹配时的分数

6.分词器会搜不出纯数字,如果既要用模糊匹配又要精确匹配就在同一个查询两种都用上(冗余字段)

