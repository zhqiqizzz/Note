在 PostgreSQL 里查 jsonb 具体字段，主要用这几个操作符：

```sql
->    取 JSON 对象/数组，返回 json/jsonb
->>   取 JSON 字段的文本值
#>    按路径取 JSON，返回 json/jsonb
#>>   按路径取文本值
@>    判断 JSON 是否包含某个结构
```

假设表是：

```sql
CREATE TABLE tool_calls (
  id SERIAL PRIMARY KEY,
  tool_name VARCHAR(100),
  input JSONB,
  output JSONB
);
```

数据是：

```json
{
  "query": "PostgreSQL jsonb",
  "limit": 5,
  "options": {
    "language": "zh-CN",
    "safe": true
  },
  "tags": ["db", "json", "agent"]
}
```

**查一级字段**

查 input.query：

```sql
SELECT input->>'query' AS query
FROM tool_calls;
```

结果是文本：

```text
PostgreSQL jsonb
```

查 input.limit：

```sql
SELECT input->>'limit' AS limit
FROM tool_calls;
```

注意这里拿到的是文本，如果要当数字用，要转类型：

```sql
SELECT (input->>'limit')::int AS limit
FROM tool_calls;
```

**按字段过滤**

查 query = 'PostgreSQL jsonb' 的记录：

```sql
SELECT *
FROM tool_calls
WHERE input->>'query' = 'PostgreSQL jsonb';
```

查 limit > 3：

```sql
SELECT *
FROM tool_calls
WHERE (input->>'limit')::int > 3;
```

**查嵌套字段**

查 input.options.language：

```sql
SELECT input->'options'->>'language' AS language
FROM tool_calls;
```

也可以用路径写法：

```sql
SELECT input#>>'{options,language}' AS language
FROM tool_calls;
```

查布尔值 safe：

```sql
SELECT (input#>>'{options,safe}')::boolean AS safe
FROM tool_calls;
```

过滤：

```sql
SELECT *
FROM tool_calls
WHERE input#>>'{options,language}' = 'zh-CN';
```

**查数组字段**

查第一个 tag：

```sql
SELECT input->'tags'->>0 AS first_tag
FROM tool_calls;
```

判断 tags 里是否包含 json，常用 @>：

```sql
SELECT *
FROM tool_calls
WHERE input->'tags' @> '["json"]'::jsonb;
```

**判断 JSON 是否包含某些字段和值**

查 input 是否包含：

```json
{"query": "PostgreSQL jsonb"}
```

SQL：

```sql
SELECT *
FROM tool_calls
WHERE input @> '{"query": "PostgreSQL jsonb"}'::jsonb;
```

查嵌套对象：

```sql
SELECT *
FROM tool_calls
WHERE input @> '{"options": {"language": "zh-CN"}}'::jsonb;
```

**判断字段是否存在**

判断 query 字段存在：

```sql
SELECT *
FROM tool_calls
WHERE input ? 'query';
```

判断多个字段任意存在：

```sql
SELECT *
FROM tool_calls
WHERE input ?| array['query', 'keyword'];
```

判断多个字段都存在：

```sql
SELECT *
FROM tool_calls
WHERE input ?& array['query', 'limit'];
```

**常用总结**

```sql
-- 取一级字段文本
input->>'query'

-- 取嵌套字段文本
input#>>'{options,language}'

-- 数字比较
(input->>'limit')::int > 3

-- 布尔比较
(input#>>'{options,safe}')::boolean = true

-- 判断包含结构
input @> '{"query": "PostgreSQL jsonb"}'::jsonb

-- 判断数组包含值
input->'tags' @> '["json"]'::jsonb

-- 判断字段存在
input ? 'query'
```

实际开发里记住最常用的三个就够先用了：

```sql
->>
#>>
@>
```

->> 查普通字段，#>> 查嵌套字段，@> 做 JSON 包含过滤。