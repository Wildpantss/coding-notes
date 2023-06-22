# RESP

[*RESP(REdis Serialization Protocol)*](https://redis.io/docs/reference/protocol-spec) 是 [*Redis*](https://redis.io) 使用的通讯协议

> 这里介绍的是 *RESPv2*, 从 *Redis 6.0* 以上的版本可以使用语义更加丰富的 *RESP3*。(但对 *RESPv2* 保持向下兼容)

除了 **管道(pipelining)** 和 **发布/订阅(pub/sub)** 之外, *RESP* 都是一个非常简单的 *Request-Response* 协议:

| RESP Message                                | Description      |
| ------------------------------------------- | ---------------- |
| [RESP Simple Strings](#resp-simple-strings) | 单行简单字符串   |
| [RESP Errors](#resp-errors)                 | 单行错误字符串   |
| [RESP Integers](#resp-integers)             | 64-bit 整数      |
| [RESP Bulk Strings](#resp-bulk-strings)     | 二进制安全字符串 |
| [RESP Arrays](#resp-arrays)                 | 数组             |

## RESP-Simple-Strings

**简单字符串(simple-string)** 是一种 **二进制不安全** 的 **单行** 字符串。

它遵循如下的格式: `b"+{content}\r\n"`

> 以 `'+'` 作为起始字符, 后跟随不包含 `'\r'` 和 `'\n'` 的字符串内容, 最后以 `"\r\n"` 标志结束。

*Examples*:

```text
"+OK\r\n"
```

## RESP-Errors

**Redis** 为 **错误字符串** 提供了单独的编码, 它和 [**简单字符串**](#resp-simple-strings) 规则一样, 仅仅是开始的 **逃逸字符(escape-character)** 不一样。

> 区分普通字符串和错误字符串对于客户端侧十分有用。

它遵循如下的格式: `b"-{content}\r\n"`

> 以 `'-'` 作为起始字符, 后跟随不包含 `'\r'` 和 `'\n'` 的字符串内容, 最后以 `"\r\n"` 标志结束。

*Examples*:

```text
"-ERR\r\n"
```

## RESP-Integers

**整型数字(integer)** 是一个仅包含合法数字的 **单行** 字符串。

它遵循如下的格式: `b":{digits}\r\n"`

> 以 `':'` 作为起始字符, 后跟随数位, 最后以 `"\r\n"` 标志结束。

*Examples*:

```text
":1000\r\n"
```

> 注意 **RESP** 中的 **integer** 是 **64-bit** 宽的, 在有些语言中对应类型为 `long`。

## RESP-Bulk-Strings

**长字符串(bulk-string)** 是一种 **二进制安全** 的, **允许多行** 的字符串。

它遵循如下的格式: `b"${len}\r\n{content}\r\n"`

> 以 `'$'` 作为起始字符, 后跟随一个表示长度的 integer, 下一行为最后以 `"\r\n"` 标志结束的真实内容。

*Examples*:

```text
"$5\r\nhello\r\n"     // "hello"
"$0\r\n\r\n"          // empty string
"$-1\r\n"             // this is null
```

> 注意长度为 `-1` 的 **bulk-string** 表示 `null` 值。

## RESP-Arrays

**数组(array)** 是一种包含了多个 **RESP对象** 的 **有序** 集合。(数组 **支持嵌套**)

它遵循如下的格式 `b"*{len}\r\n{item1}\r\n...{itemN}\r\n"`

> 以 `'*'` 作为起始字符, 后跟随一个表示长度的 integer, 后接数个其他 RESP对象 的编码。

*Examples*:

```text
"*2\r\n$5\r\nhello\r\n$5\r\nworld\r\n"                        // ["hello", "world"]
"*3\r\n:1\r\n:2\r\n:3\r\n"                                    // [1, 2, 3]
"*2\r\n*3\r\n:1\r\n:2\r\n:3\r\n*2\r\n+Hello\r\n-World\r\n"    // [[1, 2, 3], ["hello", "world"]]
"*-1\r\n"                                                     // null
"*0\r\n"                                                      // []
```

> 注意长度为 `-1` 的 **array** 表示 `null` 值。
