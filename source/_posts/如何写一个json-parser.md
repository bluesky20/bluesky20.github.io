---
title: 如何写一个json parser
date: 2018-04-16 16:43:43
tags: json, parser, python
---

---

### 1  将json string 处理成token list

- 1 .1 定义一个class 用于存每个token 的值和类型, 类型有`string`, `keyword（true，false，null）`, `number`，`[`, `]` , `{`, `}`， 冒号和逗号
```python
class Token(object):
    def __init__(self, token_type=None, value=None):
        d = {
            ':': Type.colon,
            ',': Type.comma,
            '{': Type.braceLeft,
            '}': Type.braceRight,
            '[': Type.bracketLeft,
            ']': Type.bracketRight,
        }
        if token_type == Type.auto:
            self.type = d[value]
        else:
            self.type = token_type
        self.value = value
```

- 1.2 遍历json string，如果是空格（包括\b \f \n \r \t), 跳过，其它类型则存一个对应的token。注意，对于string要处理转义字符，对于number要处理小数和负数。还要判断出number和string的结束下标。
```python
def loads(code):
    i = 0
    length = len(code)
    tokens = []
    spaces = ' \b\f\n\r\t'
    digits = '0123456789'

    is_open = True
    while i < length:
        c = code[i]
        i += 1
        if c in spaces:
            continue
        elif c in ':,[]{}':
            t = Token(Type.auto, c)
            tokens.append(t)
        elif c == '"' and is_open:
            result, index = string_end(code, i)
            if index != -1:
                t = Token(Type.string)
                t.value = result
                i = index
                tokens.append(t)
                is_open = not is_open
            else:
                return
        elif c == '"' and not is_open:
            is_open = not is_open
            continue
        elif c in digits:
            offset = number_end(code, i)
            t = Token(Type.number)
            s = code[i-1:i+offset]
            # 判断是否 float
            if '.' in s:
                t.value = float(s)
            else:
                t.value = int(s)
            i += offset
            tokens.append(t)
        elif c in 'tfn':
            # true false null
            kvs = dict(
                t='true',
                f='false',
                n='null',
            )
            t = Token(Type.keyword)
            t.value = kvs[c]
            tokens.append(t)
            i += len(kvs[c])
        else:
            print("*** 错误", c, code[i:i+10])
            return
    return tokens
```

### 2 将token list 处理成 dict 和 list。

- 2.1 取第一位token，删除token list里的第一位token
- 2.2 如果token的类型不是大括号，也不是中括号，返回token的值
- 2.3 如果token类型是左大括号，新建一个dict，遍历token list，取后2位元素后（key和冒号）将剩余token list的递归处理（用于处理嵌套的情况）拿到value ，然后判断token list的第一位元素是否逗号，如果是，删除逗号。继续下一个循环。循环结束后删除token list里结尾的右大括号, 返回dict。
- 2.4 如果token类型是左中括号，新建一个list，遍历token list , 递归获取value，然后判断token list的第一位元素是否逗号，如果是，删除逗号。继续下一个循环。循环结束后删除token list里结尾的右中括号， 返回list。
```python
def parse(ts):
    t = ts[0]
    del ts[0]
    if t.type == Type.braceLeft:
        obj = {}
        while ts[0].type != Type.braceRight:
            k = ts[0]
            _colon = ts[1]
            # 确保 k.type 必须是 string
            assert k.type == Type.string
            # 确保 _colon 必须是 colon
            assert _colon.type == Type.colon
            del ts[0]
            del ts[0]
            v = parse(ts)
            obj[k] = v
            _comma = ts[0]
            # 吃一个 逗号
            if _comma.type == Type.comma:
                del ts[0]
        # 结束 删除末尾的 '}'
        del ts[0]
        return obj
    elif t.type == Type.bracketLeft:
        l = []
        while ts[0].type != Type.bracketRight:
            v = parse(ts)
            # 吃一个 逗号
            _comma = ts[0]
            if _comma.type == Type.comma:
                del ts[0]
            l.append(v)
        # 删除末尾的 ']'
        del ts[0]
        return l
    else:
        return t.value
```

完整代码:  [https://github.com/bluesky20/json-parser](https://github.com/bluesky20/json-parser)
