## CUE 模板规则

### 字段名中的连字符

在 CUE 模板文件中，当字段名包含字符 `-` 时，需要区分该连字符的用途：

- 如果 `-` 是 CUE 模板引擎需要处理的特殊字符（例如用于表达式或运算符），按 CUE 语法正常使用。
- 如果 `-` 只是普通字段名称的一部分（例如 Redis 配置项 `protected-mode`、`tcp-backlog`），则必须将整个字段名用双引号括起来。

### 示例

正确写法：

```cue
#RedisParameter: {
    "protected-mode"?: string
    "tcp-backlog"?: string
    timeout?: string
}
```

错误写法：

```cue
#RedisParameter: {
    protected-mode?: string  // 非法：未加双引号，会被 CUE 解析为减法表达式
    tcp-backlog?: string     // 非法：同上
}
```
