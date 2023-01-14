---
title: Go 的编译 03 Token
description: 
date: 2021-12-16
author: zenk
draft: false
categories: Go 的编译
tags: [Go, 编译]
---

> 本文是 Go 源码（ `cmd/compile/internal/syntax/scanner.go` ）的阅读笔记。

利用上一篇文档中介绍的 `source` 数据结构，我们可以方便且高效地读取源文件中的字符，并且把可能需要的字符缓存下来。接下来编译器需要做的操作就是把字符流转换为 token 流了。

这个转换过程一般被称为“词法分析”，完成这一过程所使用的工具自然也就被称为词法分析器了。通常使用 lexer 或者 scanner 来表示词法分析器，在 Go 中使用了 scanner 这种说法。

``` go
type scanner struct {
    source
    mode   uint
    nlsemi bool // if set '\n' and EOF translate to ';'

    // current token, valid after calling next()
    line, col uint
    blank     bool // line is blank up to col
    tok       token
    lit       string   // valid if tok is _Name, _Literal, or _Semi ("semicolon", "newline", or "EOF"); may be malformed if bad is true
    bad       bool     // valid if tok is _Literal, true if a syntax error occurred, lit may be malformed
    kind      LitKind  // valid if tok is _Literal
    op        Operator // valid if tok is _Operator, _AssignOp, or _IncOp
    prec      int      // valid if tok is _Operator, _AssignOp, or _IncOp
}
```

以上就是 `scanner` 的数据结构了。与我的编程习惯不同的是，它并没有将 `source` 设计为一个成员变量，而是将其作为了一个匿名成员。这其实更符合 Go 的编程思想，学习了。

再看其他成员变量。 `mode` 是与注释相关的标志位，表示是否通过调用 error handler 报告注释。

``` go
// The mode flags below control which comments are reported
// by calling the error handler. If no flag is set, comments
// are ignored.
const (
    comments   uint = 1 << iota // call handler for all comments
    directives                  // call handler for directives only
)
```

如果 `mode = 0b01` ，那么 `scanner` 会报告所有注释；如果 `mode = 0b10` ，那么 `scanner` 仅会报告命令注释（ directives ）。所谓的命令注释，就是 Go 中写在注释中到一些命令，例如：

``` go
//go:generate stringer -type token -linecomment tokens.go

const (
    _    token = iota
    _EOF       // EOF

    // ...
)
```

这样就可以使用 `go generate` 命令来生成一些代码。这并非编译器的主要功能，因此不会详细分析命令是怎么被处理的。

`nlsemi` 则是与分号有关。在 Go 中，绝大部分情况下行末的分号可以省略不写。然而在 Go 的文法中定义了行末需要有分号，这是如何实现的呢？在词法分析阶段， scanner 会自动判断是否要在行末插入一个分号。假如某一行的最后一个 token 是 `+` ，那显然这个语句还没有结束，不应该插入分号。而如果最后一个 token 是 `2` ，那么可能就需要插入一个分号。如果以下几种 token 位于行末，那么 Go 会在它们后面插入一个分号：

- 标识符；
- 整形、浮点型、复数、字符（ `rune` ）或字符串字面量；
- 关键字 `break` 、 `continue` 、 `fallthrough` 或 `return` ；
- `++` 、 `--` 、 `)` 、 `]` 或 `}` 。

回到刚才的例子，由于 Go 必定会在行末的 `2` 之后插入一个分号，所以分行时需要注意：

``` go
// wrong
x := 2
   + 5

// correct
x := 2 +
     5
```

后面的信息都是与 token 相关了。接下来我们来看最重要的函数， `next()` 。本质上来说它就是一个大 `switch` ，分别处理各类情况。

``` go
func (s *scanner) next() {
    // 保存 s.nlsemi 到 nlsemi ，然后将 s.nlsemi 重置
    nlsemi := s.nlsemi
    s.nlsemi = false

redo:
    // 跳过空白符
    // skip white space
    // 停止录制 source.b = -1
    s.stop()
    // 保存起始点的位置
    startLine, startCol := s.pos()
    // 注意循环条件， 当 nlsemi 时遇到换行需要先插入分号，所以不能被跳过
    for s.ch == ' ' || s.ch == '\t' || s.ch == '\n' && !nlsemi || s.ch == '\r' {
        s.nextch()
    }

    // 正式开始分析 token
    // token start
    // 设置 token 的行列信息
    s.line, s.col = s.pos()
    // 前有空行
    s.blank = s.line > startLine || startCol == colbase
    // 开始录制
    s.start()
    // 快速判断是否是标识符，如果是就调用对应的函数 ident()
    if isLetter(s.ch) || s.ch >= utf8.RuneSelf && s.atIdentChar(true) {
        s.nextch()
        s.ident()
        return
    }
```

标识符在源代码中的出现频率非常高，所以 Go 把它单独拿出来，提到最前面做了一个优化。 Go 中的标识符需要由 letter 或下划线开头，并且这里所说的 letter 是指 Unicode 中的 letter 。根据 [Unicode 标准 8.0](https://www.unicode.org/versions/Unicode8.0.0/) 的分类， Go 中把 Lu 、 Ll 、 Lt 、 Lm 和 Lo 类下的字符都看作是 letter 。这之中，只有大小写英文字母和下划线的数值小于 `utf8.RuneSelf` ，也就是 128 。

``` go
func lower(ch rune) rune     { return ('a' - 'A') | ch } // returns lower-case ch iff ch is ASCII letter
func isLetter(ch rune) bool  { return 'a' <= lower(ch) && lower(ch) <= 'z' || ch == '_' }

func (s *scanner) atIdentChar(first bool) bool {
    switch {
    case unicode.IsLetter(s.ch) || s.ch == '_':
        // ok
    case unicode.IsDigit(s.ch):
        if first {
            s.errorf("identifier cannot begin with digit %#U", s.ch)
        }
    case s.ch >= utf8.RuneSelf:
        s.errorf("invalid character %#U in identifier", s.ch)
    default:
        return false
    }
    return true
}
```

标识符除了第一个字符之外，还可以使用 digit 字符。 Go 中对于 digit 的定义也不仅仅是阿拉伯数字 `0` 到 `9` ，而是 Unicode 分类中的 Nd 下的全部字符。理解了这些判断之后，我们再看看 `ident()` 函数，它可以从字符流中读出一个标识符来。

``` go
func (s *scanner) ident() {
    // 一般来说源代码都使用英文大小写字母来编写，所以在这里进行一个优化
    // accelerate common case (7bit ASCII)
    for isLetter(s.ch) || isDecimal(s.ch) {
        s.nextch()
    }

    // 通用情况， Unicode 中的广义 letter
    // general case
    if s.ch >= utf8.RuneSelf {
        for s.atIdentChar(false) {
            s.nextch()
        }
    }

    // 关键字都是标识符，标识符不一定是关键字，这里通过哈希表进行判断
    // possibly a keyword
    lit := s.segment()
    // 关键字的长度起码是 2
    if len(lit) >= 2 {
        if tok := keywordMap[hash(lit)]; tok != 0 && tokStrFast(tok) == string(lit) {
            // 部分关键字在行末需要插入分号，这里进行判断
            s.nlsemi = contains(1<<_Break|1<<_Continue|1<<_Fallthrough|1<<_Return, tok)
            s.tok = tok
            return
        }
    }

    // 标识符在行末需要插入分号，所以设置 s.nlsemi
    s.nlsemi = true
    s.lit = string(lit)
    s.tok = _Name
}
```

`ident()` 函数是非常直觉的，没有什么需要分析的地方。主要我想看看“哈希表”这一部分， Go 的优化可是无处不在的， `map` 的性能不能让人满意。现在问题是：我们需要一个哈希表，这个表构造好后，只会进行查询而不会进行修改；哈希表中存放的是 Go 中的关键字，而关键字的数量十分有限。到这里，你应该能想到优化思路了，那就是 [完美哈希](../../note/算法导论/11%20哈希表.md) ：手动设计一种哈希函数，能够让 Go 的关键字映射（散列）到一个非常有限的空间中。这样的哈希表能够提供“真正” $O(1)$ 的时间复杂度。

``` go
// hash is a perfect hash function for keywords.
// It assumes that s has at least length 2.
func hash(s []byte) uint {
    return (uint(s[0])<<4 ^ uint(s[1]) + uint(len(s))) & uint(len(keywordMap)-1)
}

var keywordMap [1 << 6]token // size must be power of two

func init() {
    // populate keywordMap
    for tok := _Break; tok <= _Var; tok++ {
        h := hash([]byte(tok.String()))
        if keywordMap[h] != 0 {
            panic("imperfect hash")
        }
        keywordMap[h] = tok
    }
}
```

Go 中使用的完美哈希函数真的是非常简单：第一个字符左移四位，与第二个字符进行异或，然后加上整个标识符的长度。最后，截取结果的后几位，得到哈希值。回到 `ident()` 函数中，继续研究判断关键字那一段。首先，它使用 `tok := keywordMap[hash(lit)]` 获取了 `lit` 的哈希值对应的关键字，接下来我们还需要逐字判断 `lit` 是否真的是关键字。

`tok != 0` 这一条件，直接能筛掉大部分标识符。如果通过这一测试，就继续判断 `tokStrFast(tok) == string(lit)` 。

``` go
// tokStrFast is a faster version of token.String, which assumes that tok
// is one of the valid tokens - and can thus skip bounds checks.
func tokStrFast(tok token) string {
    return _token_name[_token_index[tok-1]:_token_index[tok]]
}

// 下面的代码来自 cmd/compile/internal/syntax/token_string.go
const _token_name = "EOFnameliteralopop=opop=:=<-*([{)]},;:....breakcasechanconstcontinuedefaultdeferelsefallthroughforfuncgogotoifimportinterfacemappackagerangereturnselectstructswitchtypevar"

var _token_index = [...]uint8{0, 3, 7, 14, 16, 19, 23, 24, 26, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 42, 47, 51, 55, 60, 68, 75, 80, 84, 95, 98, 102, 104, 108, 110, 116, 125, 128, 135, 140, 146, 152, 158, 164, 168, 171, 171}
```

通过了上面两个测试，说明 `lit` 真的是一个关键字。下面我们面临另一个问题，有一部分关键字需要对 `s.nlsemi` 进行设置，还需要把它们甄别出来。 Go 中使用 `contains()` 函数完成这一操作。

``` go
// 下面的代码来自 cmd/compile/internal/syntax/token.go
// contains reports whether tok is in tokset.
func contains(tokset uint64, tok token) bool {
    return tokset&(1<<tok) != 0
}
```

`contains()` 并没有什么神奇操作，只是一个位操作。至此， `ident()` 函数就看完了。让我们回到 `next()` 函数，继续看下去。

``` go
    switch s.ch {
    case -1:
        if nlsemi {
            s.lit = "EOF"
            s.tok = _Semi
            break
        }
        s.tok = _EOF

    case '\n':
        s.nextch()
        s.lit = "newline"
        s.tok = _Semi
```

大 `switch` 来了。首先处理的，是文末和行末。当 `source.ch == -1` 时，表示它已经读到了源代码的末尾。如果满足插入分号的条件，那么设置 `lit` 和 `tok` ，否则就简单地设置 `tok` 。这里可以发现， token 流不是一定以 `_EOF` 结尾。再看 `\n` 这种换行的情况，它没有判断 `nlsemi` ，直接插入了分号。现在还看不懂这样做的原因，我们继续看下去。

``` go
    case '0', '1', '2', '3', '4', '5', '6', '7', '8', '9':
        s.number(false)
```

显然，接下来要处理数字了。 Go 中支持的数字格式是比较复杂的，不但有二进制、八进制、十进制、十六进制，还支持十进制和十六进制的浮点数及其科学计数法形式。不仅如此，数字之间还可以用 `_` 作为间隔来提升可读性，例如 `0b10_1101` 。支持了这么多功能，可以预见 `number()` 会是个非常麻烦的函数。

``` go
// 参数 seenPoint 表示是否遇到过小数点
func (s *scanner) number(seenPoint bool) {
    // 是否是合法的数字
    ok := true
    // token 类型默认为整形字面量
    kind := IntLit
    // 默认进制为十进制
    base := 10        // number base
    // 前缀，默认为十进制
    // 0 ：表示十进制，注意这个 0 是数值为 0 ；
    // 'b' ：表示二进制；
    // '0' ：表示（隐式）八进制，注意这个 0 是字符 0 ，
    //       在 Go 中，以 0 开头的数字是八进制，例如 077 ；
    // 'o' ：表示八进制；
    // 'x' ：表示十六进制
    prefix := rune(0) // one of 0 (decimal), '0' (0-octal), 'x', 'o', or 'b'
    // 第 0 位代表出现过数字，第 1 位代表出现过 '_'
    digsep := 0       // bit 0: digit present, bit 1: '_' present
    // 非法数字的下标，用于错误报告
    invalid := -1     // index of invalid digit in literal, or < 0

    // integer part
    if !seenPoint {
        // 判断前缀 '0' 后面的字符
        if s.ch == '0' {
            s.nextch()
            switch lower(s.ch) {
            case 'x':
                s.nextch()
                base, prefix = 16, 'x'
            case 'o':
                s.nextch()
                base, prefix = 8, 'o'
            case 'b':
                s.nextch()
                base, prefix = 2, 'b'
            default:
                base, prefix = 8, '0'
                digsep = 1 // leading 0
            }
        }
        // 获取后续的数字
        digsep |= s.digits(base, &invalid)
```

上面代码的主要作用就是初始化了一些变量和标志，然后判断了数字的进制，最后使用 `digits()` 函数读取后续的数字。接下来又要递归读代码了，查看 `digits()` 函数中如何读取剩余的数字。

``` go
// digits 读取数字和 '_' 构成的序列。如果进制 base 小于等于 10 ，那么
// digits 还会顺便检查是否出现了超范围的数字。例如 0b123 会在这里被识别出错误。
// 但是如果是十六进制，那么 digits 不会检查错误，例如 0xEFG ，digits 会扫描到 G 时停止。
// 即使遇到了不合法的数字， digits 也不会返回，它只是记录下不合法数字第一次出现的位置。
// digits accepts the sequence { digit | '_' }.
// If base <= 10, digits accepts any decimal digit but records
// the index (relative to the literal start) of a digit >= base
// in *invalid, if *invalid < 0.
// digits returns a bitset describing whether the sequence contained
// digits (bit 0 is set), or separators '_' (bit 1 is set).
func (s *scanner) digits(base int, invalid *int) (digsep int) {
    if base <= 10 {
        max := rune('0' + base)
        for isDecimal(s.ch) || s.ch == '_' {
            ds := 1
            if s.ch == '_' {
                ds = 2
            } else if s.ch >= max && *invalid < 0 {
                _, col := s.pos()
                *invalid = int(col - s.col) // record invalid rune index
            }
            digsep |= ds
            s.nextch()
        }
    } else {
        for isHex(s.ch) || s.ch == '_' {
            ds := 1
            if s.ch == '_' {
                ds = 2
            }
            digsep |= ds
            s.nextch()
        }
    }
    return
}

// 两个小函数
func isDecimal(ch rune) bool { return '0' <= ch && ch <= '9' }
func isHex(ch rune) bool     { return '0' <= ch && ch <= '9' || 'a' <= lower(ch) && lower(ch) <= 'f' }
```

`digits()` 函数比较简单，它会根据进制，向后读取数字。如果是二进制、八进制、十进制，那么 `digits()` 会读取后面所有的 `[0-9]` （哪怕当前是二进制），并且记录下第一个超进制的数字位置（如果有）。比如 `0b123` ，那么 `2` 的下标就会保存在 `invalid` 中。如果是十六进制，那么 `digits()` 会向后读取所有 `[0-9a-f]` 。接下来我们回到 `number()` 。

``` go
        // 判断是否遇到了小数点
        if s.ch == '.' {
            // 二进制和八进制的小数点是不支持的，要报错
            if prefix == 'o' || prefix == 'b' {
                s.errorf("invalid radix point in %s literal", baseName(base))
                ok = false
            }
            // 记录下已经遇到了小数点
            s.nextch()
            seenPoint = true
        }
    }
```

``` go
// 把进制变为对应的名称，用于错误信息
func baseName(base int) string {
    switch base {
    case 2:
        return "binary"
    case 8:
        return "octal"
    case 10:
        return "decimal"
    case 16:
        return "hexadecimal"
    }
    panic("invalid base")
}
```

小数点之前的部分就到此结束了，我们继续往下看 `number()` 。

``` go
    // 继续使用 digits() 读取小数点后面的数字（即使前面已经出现了错误）
    // 如果遇到了小数点，那么 token 类型要改成浮点型字面量
    // fractional part
    if seenPoint {
        kind = FloatLit
        digsep |= s.digits(base, &invalid)
    }

    // 特殊判断：仅由 '_' 组成的（部分）数字，例如 0b___ 、 0x___.aaa
    if digsep&1 == 0 && ok {
        s.errorf("%s literal has no digits", baseName(base))
        ok = false
    }

    // 指数部分
    // exponent
    if e := lower(s.ch); e == 'e' || e == 'p' {
        if ok {
            switch {
            // 'e' 可以用于十进制和八进制（此时隐式八进制失效，变为十进制）
            case e == 'e' && prefix != 0 &&a:
                s.errorf("%q exponent requires decimal mantissa", s.ch)
                ok = false
            // 'p' 可以用于十六进制
            case e == 'p' && prefix != 'x':
                s.errorf("%q exponent requires hexadecimal mantissa", s.ch)
                ok = false
            }
        }
        s.nextch()
        // 这里要再设置一次，也就是说只要出现了指数就是浮点数，例如 1e5
        kind = FloatLit
        // 指数可以加正负号
        if s.ch == '+' || s.ch == '-' {
            s.nextch()
        }
        // 读取指数部分
        // digsep 的第 0 位表示是否出现过数字
        // 经过下面的语句， digsep 的第 0 位仅能代表指数部分
        // 但是第 1 位还能代表整个数字
        digsep = s.digits(10, nil) | digsep&2 // don't lose sep bit
        // 指数部分全是 '_' ，没有数字
        if digsep&1 == 0 && ok {
            s.errorf("exponent has no digits")
            ok = false
        }
    } else if prefix == 'x' && kind == FloatLit && ok {
        // 十六进制遇到了非 `[a-fp]` 的字母时，会报这种错误，例如 0x123.4y8
        s.errorf("hexadecimal mantissa requires a 'p' exponent")
        ok = false
    }

    // 遇到字母 i ，代表这是虚数
    // suffix 'i'
    if s.ch == 'i' {
        kind = ImagLit
        s.nextch()
    }


    s.setLit(kind, ok) // do this now so we can use s.lit below
```

这一部分代码中， `number()` 又处理了小数点、指数、虚数。首先，如果遇到了小数点，那么再次调用 `digits()` 读取小数点后面的数字。然后，对 `digsep` 做了一次判断，全部由 `'_'` 构成的数字比如 `0x____` 会在这里产生一个错误。接下来， `number()` 又读取了指数部分。最后，则是虚数的判断。接下来我们递归，看看 `setLit()` 函数。

``` go
// 遇到了字面量类型的 token （整形、浮点型、虚数、字符、字符串），对 scanner 进行设置
// setLit sets the scanner state for a recognized _Literal token.
func (s *scanner) setLit(kind LitKind, ok bool) {
    // 字面量之后的换行和 EOF 之前要插入回车，因此要把 s.nlsemi 设置为 true
    s.nlsemi = true
    // 设置 token 类型为字面量
    s.tok = _Literal
    // 从 source 中获得“录制”的片段
    s.lit = string(s.segment())
    // bad 表示是否出现了错误
    s.bad = !ok
    // 设置字面量的具体类型
    s.kind = kind
}
```

数字是字面量类型之中的一种。设置好后，我们继续看 `number()` 。

``` go
    // 对整型遇到的错误进行报告
    // 报错需要使用 s.lit ，所以需要先 setLit()
    if kind == IntLit && invalid >= 0 && ok {
        s.errorAtf(invalid, "invalid digit %q in %s literal", s.lit[invalid], baseName(base))
        ok = false
    }

    // 检查 '_' 是否合法
    // 检查需要使用 s.lit ，所以需要先 setLit()
    if digsep&2 != 0 && ok {
        if i := invalidSep(s.lit); i >= 0 {
            s.errorAtf(i, "'_' must separate successive digits")
            ok = false
        }
    }

    // 由于上面又进行了两项检查，所以这里重新设置 s.bad
    s.bad = !ok // correct s.bad
}
```

到这里 `number()` 终于结束了，它最后还检查了处理整型时是否遇到了错误，检查了 `'_'` 的使用是否合法。这两项检查需要 `s.lit` ，所以先调用了 `setLit()` 。还是递归读代码，我们看看 `invalidSep()` 是怎么检查 `'_'` 的使用的。

``` go
// invalidSep 返回第一个不合法 _ 的下标；如果不存在这样的 _ ，返回 -1
// invalidSep returns the index of the first invalid separator in x, or -1.
func invalidSep(x string) int {
    // x1 是 x 的第二个字符（如果有）
    x1 := ' ' // prefix char, we only care if it's 'x'
    // 上一个字符的类型，下划线 '_' ，数字 '0' 或者其他 '.'
    d := '.'  // digit, one of '_', '0' (a digit), or '.' (anything else)
    // 下标
    i := 0

    // 前缀看作一个数字，也就是说 0x_2 是合法的
    // a prefix counts as a digit
    if len(x) >= 2 && x[0] == '0' {
        x1 = lower(rune(x[1]))
        if x1 == 'x' || x1 == 'o' || x1 == 'b' {
            d = '0'
            i = 2
        }
    }

    // mantissa and exponent
    for ; i < len(x); i++ {
        p := d // previous digit
        d = rune(x[i])
        switch {
        // 这个字符是 '_' ，上个字符不是数字，那么报错
        case d == '_':
            if p != '0' {
                return i
            }
        // 当前字符是数字，那么把 d 设置为 '0'
        case isDecimal(d) || x1 == 'x' && isHex(d):
            d = '0'
        default:
            // 如果 d 不是 '_' ，也不是数字
            // 那么 d 可能是小数点、 'e' 、 'p'
            // 那么此时上一个字符不可以是 '_'
            if p == '_' {
                return i - 1
            }
            d = '.'
        }
    }
    // 不可以以 '_' 结尾
    if d == '_' {
        return len(x) - 1
    }

    return -1
}
```

到这里，终于结束了 `number()` 的判断。那么让我考考你，请问 `0xabcdefghijk` 会在 `number()` 的哪一步报错呢？答案是不会在 `number()` 中报错。 `number` 会读取 `0xabcdef` 这个 token ，剩下的 `ghijk` 会被认为是下一个 token 。我们回到 `next()` 继续。

``` go
    case '"':
        s.stdString()
```

不用多说，接下来需要递归读代码了。遇到 `'"'` 时，显然是遇到了（标准）字符串。

``` go
func (s *scanner) stdString() {
    // 字符串是否合法
    ok := true
    // 读取一个字符，跳过 '"'
    s.nextch()

    for {
        // 再次遇到 '"' 时结束
        if s.ch == '"' {
            s.nextch()
            break
        }
        // 检查转义符后面的字符是否合法
        if s.ch == '\\' {
            s.nextch()
            if !s.escape('"') {
                ok = false
            }
            continue
        }
        // '"' 类型的字符中不允许有换行符
        if s.ch == '\n' {
            s.errorf("newline in string")
            ok = false
            break
        }
        // 还没找到另一个 '"' 就遇到了文件末尾
        if s.ch < 0 {
            s.errorAtf(0, "string not terminated")
            ok = false
            break
        }
        s.nextch()
    }

    // 设置字面量
    s.setLit(StringLit, ok)
}
```

哈哈，和数字一比，字符串真是简单得很。简单来说， `stdString()` 就是一直读取，直到遇到另一个 `'"'` 停止。读取时，还要顺便注意一下转义符、回车符、 EOF 。我们继续递归，看看 `escape()` 的内容。

``` go
func (s *scanner) escape(quote rune) bool {
    // 转义符后面字符的数量
    var n int
    // 进制和最大值，转义符后面可以写一个八进制三位数例如 \033 ，
    // 也可以写 Unicode 例如 \uab
    var base, max uint32

    switch s.ch {
    // 简单转义
    // 注意这里的 quote ， '"' 包裹的字符串中可以 \" 而不可以 \'
    // '\'' 包裹的字符中只能 \' 而不可以 \"
    case quote, 'a', 'b', 'f', 'n', 'r', 't', 'v', '\\':
        s.nextch()
        return true
    // 八进制三位数
    case '0', '1', '2', '3', '4', '5', '6', '7':
        n, base, max = 3, 8, 255
    // 十六进制两位数
    case 'x':
        s.nextch()
        n, base, max = 2, 16, 255
    // Unicode ，十六进制四位数
    case 'u':
        s.nextch()
        n, base, max = 4, 16, unicode.MaxRune
    // Unicode ，十六进制八位数
    case 'U':
        s.nextch()
        n, base, max = 8, 16, unicode.MaxRune
    default:
        // EOF 也可以转义
        // 不读取下一个字符（也没法读取）以便于 stdString()
        // 中报告一个“还没找到另一个 '"' 就遇到了文件末尾”的错误
        if s.ch < 0 {
            return true // complain in caller about EOF
        }
        // 非法的转义
        s.errorf("unknown escape")
        return false
    }

    // 读取 n 位 base 进制的数字
    var x uint32
    for i := n; i > 0; i-- {
        if s.ch < 0 {
            return true // complain in caller about EOF
        }
        d := base
        if isDecimal(s.ch) {
            d = uint32(s.ch) - '0'
        } else if 'a' <= lower(s.ch) && lower(s.ch) <= 'f' {
            d = uint32(lower(s.ch)) - 'a' + 10
        }
        if d >= base {
            s.errorf("invalid character %q in %s escape", s.ch, baseName(int(base)))
            return false
        }
        // d < base
        x = x*base + d
        s.nextch()
    }

    // 如果超出范围则要报错
    if x > max && base == 8 {
        s.errorf("octal escape value %d > 255", x)
        return false
    }

    // 某些 Unocode 字符似乎不能转义
    if x > max || 0xD800 <= x && x < 0xE000 /* surrogate range */ {
        s.errorf("escape is invalid Unicode code point %#U", x)
        return false
    }

    return true
}
```

好了，看完了 `escape()` ，我们要回到 `next()` 中了。

``` go
    case '`':
        s.rawString()
```

`` ` `` 代表我们遇到了另一种字符串，也就是不进行转义处理的原生字符串。

``` go
func (s *scanner) rawString() {
    ok := true
    s.nextch()

    for {
        // 持续读取字符，直到遇到下一个 '`'
        if s.ch == '`' {
            s.nextch()
            break
        }
        // '`' 字符串只可能遇到一种错误
        // 那就是意外遇到了 EOF
        if s.ch < 0 {
            s.errorAtf(0, "string not terminated")
            ok = false
            break
        }
        s.nextch()
    }
    // We leave CRs in the string since they are part of the
    // literal (even though they are not part of the literal
    // value).

    // 设置字面量
    s.setLit(StringLit, ok)
}
```

`` ` `` 字符串比 `"` 字符串还要简单。接下来继续回到 `next()` 。

``` go
    case '\'':
        s.rune()
```

这就是最后一种字面量了，字符字面量。

``` go
func (s *scanner) rune() {
    ok := true
    s.nextch()

    n := 0
    for ; ; n++ {
        // 又遇到了 '\''
        if s.ch == '\'' {
            if ok {
                if n == 0 {
                    // 空字符，字符串可以为空，但是字符是不能为空的
                    s.errorf("empty rune literal or unescaped '")
                    ok = false
                } else if n != 1 {
                    // 超过一个字符
                    s.errorAtf(0, "more than one character in rune literal")
                    ok = false
                }
            }
            s.nextch()
            break
        }
        if s.ch == '\\' {
            // 转义字符
            s.nextch()
            if !s.escape('\'') {
                ok = false
            }
            continue
        }
        if s.ch == '\n' {
            // 不可以在字符中换行
            if ok {
                s.errorf("newline in rune literal")
                ok = false
            }
            break
        }
        if s.ch < 0 {
            // 意外遇到 EOF
            if ok {
                s.errorAtf(0, "rune literal not terminated")
                ok = false
            }
            break
        }
        s.nextch()
    }

    // 设置字面量
    s.setLit(RuneLit, ok)
}
```

字符的复杂度也不高，我们继续回到 `next()` 。

``` go
    // 四个简单的单字符 token
    case '(':
        s.nextch()
        s.tok = _Lparen

    case '[':
        s.nextch()
        s.tok = _Lbrack

    case '{':
        s.nextch()
        s.tok = _Lbrace

    case ',':
        s.nextch()
        s.tok = _Comma

    case ';':
        // 遇到 ; 时特意在 lit 中注明了 semicolon
        // 我们已经知道 s.tok = _Semi 时
        // s.lit 还可能是 EOF 或 newline
        s.nextch()
        s.lit = "semicolon"
        s.tok = _Semi

    // 又是三个简单的单字符 token
    // 不过这三个后面的回车或 EOF 之前需要插入分号
    case ')':
        s.nextch()
        s.nlsemi = true
        s.tok = _Rparen

    case ']':
        s.nextch()
        s.nlsemi = true
        s.tok = _Rbrack

    case '}':
        s.nextch()
        s.nlsemi = true
        s.tok = _Rbrace

    case ':':
        // 冒号后面如果有等于号，那么就是 :=
        // 否则就只是一个冒号而已
        // （听君一席话，胜听一席话？）
        s.nextch()
        if s.ch == '=' {
            s.nextch()
            s.tok = _Define
            break
        }
        s.tok = _Colon

    case '.':
        // 浮点数可以以小数点开头，所以这里需要特殊处理
        // 以小数点开头的数字只能是十进制，但是后面可以有指数部分和虚数符号
        s.nextch()
        if isDecimal(s.ch) {
            s.number(true)
            break
        }
        // 或者，可能是 "..." 这个 token
        if s.ch == '.' {
            s.nextch()
            if s.ch == '.' {
                s.nextch()
                s.tok = _DotDotDot
                break
            }
            s.rewind() // now s.ch holds 1st '.'
            s.nextch() // consume 1st '.' again
        }
        // 最后是最普通的 "."
        s.tok = _Dot

    case '+':
        s.nextch()
        // 设置操作符和优先级
        s.op, s.prec = Add, precAdd
        // 如果不是 "++" ，那么跳往 assignop
        // 怎么能用 goto 呢，好孩子不要学
        if s.ch != '+' {
            goto assignop
        }
        s.nextch()
        s.nlsemi = true
        s.tok = _IncOp
```

上面是对一些简单 token 的处理。这里我们停下来，看看 `assignop` 那部分代码。这部分代码位于 `next()` 的最后，大概的模样是：

``` go
func next() {
    // ...
    return

assignop:
    // ...
}
```

也就是说，这部分代码写在了顺序执行不可能执行到的位置，写在了 `return` 的后面。

``` go
assignop:
    if s.ch == '=' {
        s.nextch()
        s.tok = _AssignOp
        return
    }
    s.tok = _Operator
}
```

这部分也很简单，就是区别 `+` 和 `+=` 而已。我们回到 `next()` 。

``` go
    case '-':
        // 与 + 完全相同的操作
        s.nextch()
        s.op, s.prec = Sub, precAdd
        if s.ch != '-' {
            goto assignop
        }
        s.nextch()
        s.nlsemi = true
        s.tok = _IncOp

    case '*':
        // 这次与 +- 不同了，单独判断了一下
        // 因为 assignop 那一段代码会给非赋值的 token 设为 _Operator
        // 而这里想要设为 _Star
        // 毕竟 * 功能比较多，后面还需要更多处理
        s.nextch()
        s.op, s.prec = Mul, precMul
        // don't goto assignop - want _Star token
        if s.ch == '=' {
            s.nextch()
            s.tok = _AssignOp
            break
        }
        s.tok = _Star

    case '/':
        s.nextch()
        if s.ch == '/' {
            // 行内注释，直接删掉
            s.nextch()
            s.lineComment()
            goto redo
        }
        if s.ch == '*' {
            // 多行注释，替换为一个换行
            s.nextch()
            s.fullComment()
            if line, _ := s.pos(); line > s.line && nlsemi {
                // A multi-line comment acts like a newline;
                // it translates to a ';' if nlsemi is set.
                s.lit = "newline"
                s.tok = _Semi
                break
            }
            goto redo
        }
        // 不是注释，判断是除法还是赋值
        s.op, s.prec = Div, precMul
        goto assignop
```

这一段也非常简单，但是因为要递归阅读 `lineComment()` 和 `fullComment()` ，所以就不往后看了。

``` go
func (s *scanner) lineComment() {
    // opening has already been consumed

    // mode = 0b01 ，报告所有注释
    // skipLine() 可以跳至行末， comment() 用于报告注释，这两个函数下面会解释
    if s.mode&comments != 0 {
        s.skipLine()
        s.comment(string(s.segment()))
        return
    }

    // 如果也没有要求报告命令注释，
    // 或者虽然要求报告，但是注释开头不是 'g' 或 'l' ，那么忽略这个注释
    // （因为命令注释的开头只会是 'g' 或 'l' ）
    // are we saving directives? or is this definitely not a directive?
    if s.mode&directives == 0 || (s.ch != 'g' && s.ch != 'l') {
        s.stop()
        s.skipLine()
        return
    }

    // 要求报告命令注释，并且当前注释非常可能是一条命令注释
    // 检查这条注释是不是真的是命令注释
    // 即是否以 "go:" 或 "line " 开头
    // recognize go: or line directives
    prefix := "go:"
    if s.ch == 'l' {
        prefix = "line "
    }
    for _, m := range prefix {
        if s.ch != m {
            // 不是命令注释，停止录制
            s.stop()
            s.skipLine()
            return
        }
        s.nextch()
    }

    // 确实是命令注释，报告
    // directive text
    s.skipLine()
    s.comment(string(s.segment()))
}
```

``` go
// 跳过这一行后面的所有字符
// 这种“跳过”仅仅是指不分析，录制还是录的
// 调用 skipLine() 之后可以通过 segment() 把注释读取出来
// 但是 \n 保留，留着 nlsemi 也许有用
func (s *scanner) skipLine() {
    // don't consume '\n' - needed for nlsemi logic
    for s.ch >= 0 && s.ch != '\n' {
        s.nextch()
    }
}
```

``` go
// 调用传进来的 error handler
func (s *scanner) comment(text string) {
    s.errorAtf(0, "%s", text)
}
```

看完了 `lineComment()` 和两个小函数，我们再看看 `fullComment()` 。

``` go
func (s *scanner) fullComment() {
    /* opening has already been consumed */

    // 要求报告所有注释
    if s.mode&comments != 0 {
        if s.skipComment() {
            s.comment(string(s.segment()))
        }
        return
    }

    // 似乎多行注释中只允许使用 "line " 命令注释
    // 下面的代码逻辑和 lineCommment() 一样了
    if s.mode&directives == 0 || s.ch != 'l' {
        s.stop()
        s.skipComment()
        return
    }

    // recognize line directive
    const prefix = "line "
    for _, m := range prefix {
        if s.ch != m {
            s.stop()
            s.skipComment()
            return
        }
        s.nextch()
    }

    // directive text
    if s.skipComment() {
        s.comment(string(s.segment()))
    }
}
```

``` go
// 向后读取，直到遇到 */
// 可能发生的错误：在找到 */ 之前遇到了 EOF
func (s *scanner) skipComment() bool {
    for s.ch >= 0 {
        for s.ch == '*' {
            s.nextch()
            if s.ch == '/' {
                s.nextch()
                return true
            }
        }
        s.nextch()
    }
    s.errorAtf(0, "comment not terminated")
    return false
}
```

在没有设置 `mode` 时， `lineComment()` 和 `fullComment()` 都会跳过注释。接下来看剩下的 `next()` 。

``` go
    case '%':
        // 还记得 assignop 吗？
        s.nextch()
        s.op, s.prec = Rem, precMul
        goto assignop

    case '&':
        // 这里要多判断一下是不是 &&
        s.nextch()
        if s.ch == '&' {
            s.nextch()
            s.op, s.prec = AndAnd, precAndAnd
            s.tok = _Operator
            break
        }
        // Go 中还有个诡异的运算符是 &^
        s.op, s.prec = And, precMul
        if s.ch == '^' {
            s.nextch()
            s.op = AndNot
        }
        goto assignop

    case '|':
        // 判断是否是 ||
        s.nextch()
        if s.ch == '|' {
            s.nextch()
            s.op, s.prec = OrOr, precOrOr
            s.tok = _Operator
            break
        }
        s.op, s.prec = Or, precAdd
        goto assignop

    case '^':
        // 异或
        s.nextch()
        s.op, s.prec = Xor, precAdd
        goto assignop

    case '<':
        s.nextch()
        // 是否是 <= ？
        if s.ch == '=' {
            s.nextch()
            s.op, s.prec = Leq, precCmp
            s.tok = _Operator
            break
        }
        // 是否是 << ？
        if s.ch == '<' {
            s.nextch()
            s.op, s.prec = Shl, precMul
            goto assignop
        }
        // 是否是 <- ？
        // 好家伙，打 LeetCode 根本用不着 channel
        // 看到 <- 我还愣了一下
        if s.ch == '-' {
            s.nextch()
            s.tok = _Arrow
            break
        }
        // 啥都不是，简单的 <
        s.op, s.prec = Lss, precCmp
        s.tok = _Operator

    case '>':
        // 与 > 类似，判断 >= 、 >> 、 >
        s.nextch()
        if s.ch == '=' {
            s.nextch()
            s.op, s.prec = Geq, precCmp
            s.tok = _Operator
            break
        }
        if s.ch == '>' {
            s.nextch()
            s.op, s.prec = Shr, precMul
            goto assignop
        }
        s.op, s.prec = Gtr, precCmp
        s.tok = _Operator

    case '=':
        // 区分 == 、 =
        s.nextch()
        if s.ch == '=' {
            s.nextch()
            s.op, s.prec = Eql, precCmp
            s.tok = _Operator
            break
        }
        s.tok = _Assign

    case '!':
        // 区分 != 、 !
        s.nextch()
        if s.ch == '=' {
            s.nextch()
            s.op, s.prec = Neq, precCmp
            s.tok = _Operator
            break
        }
        s.op, s.prec = Not, 0
        s.tok = _Operator

    case '~':
        // 这是啥，想了半天应该是按位取反
        s.nextch()
        s.op, s.prec = Tilde, 0
        s.tok = _Operator

    default:
        // 不认识的字符
        s.errorf("invalid character %#U", s.ch)
        s.nextch()
        goto redo
    }

    return

// 后面就是 assignop 了
```

终于看完了。
