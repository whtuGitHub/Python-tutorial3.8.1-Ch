# `re` — 正则表达式操作 (Regular expression operations)

**源码:** [Lib/re.py](https://github.com/python/cpython/tree/3.8/Lib/re.py)

------

此模块提供与的正则表达式匹配操作Perl中的类似。

要匹配模式和要搜索的字符串既可以是Unicode字符串 ([`str`]())，也可以是8位字符串 ([字节 (`bytes`)]())。但是，Unicode字符串和8位字符串不能混合使用：即不能将Unicode字符串与字节模式匹配，反之亦然；类似地，在请求替换时，替换字符串必须与匹配模式和要搜索字符串具有相同的类型。

正则表达式使用反斜杠字符 (`\`) 来表示特殊的形式，或者允许使用特殊的字符而不调用它们的特殊含义。这与Python在字符串文本中出于相同目的而使用相同的字符是冲突的；例如，要匹配文字反斜杠，可能必须将`\\\\`作为模式字符串，因为正则表达式必须是`\\`，而且每个反斜杠必须在一个普通的Python字符串文字中表示为`\\`。另外，请注意，在Python中使用字符串文本中的反斜杠时，任何无效的转义序列现在都会生成一个[`DeprecationWarning`]()，将来这将成为一个[`SyntaxError`]()。即使是正则表达式的有效转义序列，也会发生这种行为。

解决方案是对==正则表达式模式使用==Python的**原始字符串表示法**，在以`r`为前缀的字符串文字中，不以任何特殊方式处理反斜杠。因此，`r"\n"`是一个包含`'\'`和`'n'`的双字符字符串，而`"\n"`是一个包含换行符的单字符字符串。通常，在Python代码中将使用这种原始字符串表示法表示模式。

需要注意的是，对于[已编译的正则表达式 (compiled regular expressions)]()，大多数正则表达式操作都可以作为模块级函数和方法使用。这些函数是不需要您首先编译正则表达式对象的快捷方式，但会丢失一些微调参数。

参见第三方[regex]()模块，它有一个与标准库 [`re`]() 模块兼容的API，但是提供了额外的功能和更全面的Unicode支持。

## 正则表达式语法 (Regular Expression Syntax)

正则表达式 (或RE) 指定一组与之匹配的字符串，这个模块中的函数允许您检查特定的字符串是否与给定的正则表达式匹配 (或者给定的正则表达式是否与特定的字符串匹配，其结果是相同的)。

可以将正则表达式连接形成新的正则表达式，如果A和B都是正则表达式，那么AB也是正则表达式。一般来说，如果一个字符串p匹配A，另一个字符串q匹配B，那么字符串pq将匹配AB。==除非A或B包含优先级较低的操作，或A与B之间有边界条件，或者A或B具有编号的组引用，这个规则都有效。==因此，复杂的表达式可以很容易地从简单的基本表达式构造出来，就像这里描述的那些。有关正则表达式的理论和实现的详细信息，请参阅Friedl书籍[Frie09]()，或者几乎所有关于编译器构造的教科书。

下面简要解释正则表达式的格式。有关更多信息和更温和的表示，请参阅[Regular Expression HOWTO]()。

正则表达式可以包含特殊字符和普通字符。大多数普通字符，如`'A'`、`a`或`O`，是最简单的正则表达式，他们只是匹配自己。您可以连接普通字符，因此`last`匹配字符串`'last'`。==(在本节的其余部分，我们将用这种`this special style`形式来编写RE，通常不带引号，来匹配字符串 `'in single quotes'`。)==

有些字符，如`'|'`或`'('`，是特殊的。特殊字符要么表示普通字符的类，要么影响它们周围的正则表达式的解释方式。

**重复限定符** (`*`、`+`、`?`、`{m、n}`等) ==不能直接嵌套==。这避免了非贪婪修饰符后缀`?`和其他实现中的其他修饰符产生歧义。要将第二次重复应用于内部重复，可以使用括号。例如，表达式`(?:a{6})*`匹配6个`'a'`字符的任意倍数。

特殊字符如下：

- `.`

  (句点`.`) 在默认模式下，它匹配除换行以外的任何字符。如果指定了[DOTALL]()标志，它将匹配包括换行符在内的所有字符。

- `^`

  (插入符号) 匹配字符串的开头，并且在[多行 (`MULTILINE`)]() 模式下，在==每个换行之后立即匹配==。

- `$`

  匹配字符串的末尾，或者==刚好在字符串末尾的换行== (即字符串的最后字符是换行符 `\n`，**译者注**) 之前，在 [多行 (MULTILINE)]() 模式下也匹配换行 (这个换行符 `\n` 可以在字符串的任何地方，**译者注**) 之前。`foo` 同时匹配 'foo' 和 'foobar'，而正则表达式`foo$`只匹配'foo'。更有趣的是，在 `'foo1\nfoo2\n'`中搜索`foo.$` ，通常匹配`'foo2'`，但在 [多行 (MULTILINE)]() 模式下匹配的是`'foo1'`；在`'foo\n'`中搜索单个`$`将发现两个(空的)匹配项：一个刚好在换行之前，另一个在字符串的末尾。

- `*`

  使产生的 RE 匹配==**0次**或多次==重复的其前一个 RE，而且是尽可能多地匹配重复次数。`ab*`将匹配'a'、'ab'或'a'后跟任意数量的'b'。

- `+`

  使产生的 RE 匹配==**1次**或更多次==重复的其前一个 RE。`ab+`将匹配 'a' 后面任意非零个数的'b'，那么它不会匹配 'a'。

- `?`

  使产生的 RE 匹配==**0次**或**1次**==其前一个 RE。`ab?`匹配 'a' 或 'ab'。

- `*?`,  `+?`,  `??`

  `*`、`+`和`?`限定符都是**贪婪**的，它们匹配尽可能多的文本，有时这种行为是不受欢迎的。如果正则表达式 `<.*>`与`'  b '`匹配，它将匹配整个字符串，而不仅仅是`''`。在限定符后添加`?`使其以**非贪婪或最小方式**执行匹配，匹配的字符越少越好。使用正则表达式 `<.*?>`将只匹配`''`。

- `{m}`

  指定要匹配前一个RE的 *m* 个副本，较少的匹配导致整个RE不匹配。例如，`a{6}`会精确匹配6个`'a'`字符，而不是5个。

- `{m,n}`

  导致产生的RE匹配前一个RE的m到n次重复，而且是**匹配尽可能多**的重复。例如，`a{3,5}`将匹配3到5个'a'字符 (含3和5)。省略m表示下边界为0的下界，省略n表示上边界为无穷大。例如，`a{4，}b`将匹配`'aaaab'`或一千个`'a'`字符后跟着一个`'b'`，但不匹配`'aaab'`。**逗号不能省略**，否则会与前面描述的形式混淆。

- `{m,n}?`

  导致产生的RE与前面的RE匹配m到n次，并且是匹配尽可能少的重复。这是上一个限定符 (即 `{m,n}`) 的非贪婪版本。例如，在6个字符的字符串`'aaaaaa'`上，`a{3,5}`将匹配5个`'a'`字符，而一个`a{3,5}?`将只匹配3个字符。

- `\`

  可以转义特殊字符 (允许你匹配`'*'`、`'?'`等字符)，也可以表示一个特殊的序列。下面来讨论特殊序列。

  如果你没有使用原始字符串来表示匹配模式，请记住Python也使用反斜杠作为字符串文本的转义序列。如果Python的解析器不能识别转义序列，则反斜杠和后续字符将包含在结果字符串中。但是，如果Python能够识别结果序列，则反斜杠应该重复两次。这很复杂，很难理解，所以强烈建议你对所有的表达式 (除了最简单的表达式) 使用原始字符串。

- `[]`

  用来表示一组字符。在一个组中：

  - 字符可单独列出，比如，`[amk]`会匹配`'a'`、`'m'`、或`'k'`。 
  - 字符的范围可以通过给出两个字符并用`'-'`分隔来表示，例如`[a-z]`将匹配任何小写的ASCII字母，`[0-5][0-9]`将匹配从`00`到`59`的所有两位数，`[0-9A-Fa-f]`将匹配任何十六进制数字。如果 `-` 被转义 (例如`[a\-z]`)，或者它被放在第一个或最后一个字符 (例如`[-a]`或`[a-]`)，它将匹配一个文字`'-'`。
  - 特殊字符在集合中失去了它们的特殊意义。例如，`[(+*)]`将匹配文字字符`'('`、`'+'`、`'*'`或`')'` 中的任何一个。
  - 像`\w`或`\S` (在下面定义) 这样的字符类也可以在一个集合中使用，尽管它们匹配的字符取决于 [`ASCII`]() 或[`LOCALE`]() 模式是否有效。
  - 不在一个范围内的字符可以通过求集合的补集来匹配。如果集合的第一个字符是`'^'`，则将匹配不在集合内的所有字符。例如，`[^5]`将匹配除`'5'`之外的任何字符，`[^^]`将匹配除`^`之外的任何字符。`^`如果不是集合中的第一个字符，则没有特殊意义。
  - 要在集合中匹配的文字`']'`，请在它前面加上反斜杠，或者将它放在集合的开头。例如，`[()[\]{}]`和`[]()[{}]`都将匹配==括号== (里面的圆括号、方括号和花括号均能匹配，**译者注**)。
  - 如[Unicode技术标准#18 (Unicode Technical Standard # 18)]() 中的嵌套集和集操作那样，将来可能会增加对嵌套集和集操作的支持。这将改变语法，因此为了促进这种改变，在目前遇到模糊情况会抛出[`FutureWarning`]() 警告。这包括以文字`['`或包含文字字符序列`'——'`、`' && '`、`'~~'`和`'||'`开头的集合。为了避免警告，请用用反斜杠转义它们。


在版本3.7进行了更改：如果一个字符集包含的结构在将来会在语义上发生变化，则会抛出 [`FutureWarning`]() 警告。

- `|`

  `A|B`，其中A和B可以是任意的正则表达式，它创建一个正则表达式来匹配A或B。通过这样的方式，可以通过`'|'`间隔来组合任意数量的正则表达式。这也可以在组中使用 (参见下面)。当扫描目标字符串时，从左到右尝试由`'|'`分隔的正则表达式。当一个模式完全匹配时，就接受这个分支。这意味着一旦A匹配，B将不会被进一步检查，即使它会产生一个更长的整体匹配。换句话说，`'|'`操作符从不贪婪。若要匹配文字`'|'`，请使用`\|`，或将其包含在==字符类==中，如`[|]`。

- `(...)`

  匹配括号内的任何正则表达式，并指示组的开始和结束；可以在执行匹配之后检索组的内容，并且可以稍后在字符串中使用下面描述的`\number`特殊序列进行匹配。要匹配文字`'('`或`')'`，请使用`\(`或`\)`，或将它们放在字符类中：`[(]`、`[)]`。

- `(?...)`

  这是一个扩展符号 ( `'?'`跟在`‘(’`后面是没有意义的)。`?`后面的第一个字符决定了结构的意义和进一步的语法。除了`(?P...)`外，它通常不会创建新组。以下是当前支持的扩展。

  - `(?aiLmsux)`
  (`'a'`、`'i'`、`'L'`、`'m'`、`'s'`、`'u'`和`'x'`中**一个**或多个字母)。组匹配空字符串；对于整个正则表达式，这些字母设置了相应的标志：[`re.A`]() (只匹配ASCII)、[`re.I`]() (忽略大小写)、[`re.L`]() (依赖于语言环境)、[`re.M`]() (多行匹配)、[`re.S`]() (点匹配所有)、[`re.U `]() (Unicode匹配) 和[`re.X`]() (==verbose 冗长匹配==？)。(这些标记在[模块内容 (Module Contents)]() 中进行介绍。) 如果希望将标志包含在正则表达式中，而不是将标志参数传递给 [`re.compile()`]() 函数，这是非常有用的。应该首先在表达式字符串中使用标记。
  - `(?:...)`
  正则括号的非捕获版本。匹配圆括号内的任何正则表达式，但是在执行匹配或在模式的后面引用之后，无法检索组匹配的子字符串。（==不解？？==）
  - `(?aiLmsux-imsx:...)`
  (`'a'`、`'i'`、`'L'`、`'m'`、`'s'`、`'u'`和`'x'`中**零个**或多个字母，然后可选地加上`'-'`，其后再加上 `'i'`, `'m'`, `'s'`, `'x'`中的一个或多个字母。) 作为整个正则表达式的一部分，设置或删除相应标志的字母：[`re.A`]() (只匹配ASCII)、[`re.I`]() (忽略大小写)、[`re.L`]() (依赖于语言环境)、[`re.M`]() (多行匹配)、[`re.S`]() (点匹配所有)、[`re.U `]() (Unicode匹配) 和[`re.X`]() (==verbose 冗长匹配==？)。(这些标记在[模块内容 (Module Contents)]() 中进行介绍。) 
  字母 `'a'`、 `'L'` 和 `'u'`在用作内联标志时是互斥的，因此它们不能联合使用或跟随在`'-'`之后。相反，当其中一个出现在内联组中时，它将覆盖封闭组中的匹配模式。在Unicode模式中`(?a:…)`切换到仅ASCII匹配，`(?u:…)`切换到Unicode匹配 (默认)。在字节模式中`(?L:…)`切换到取决于语言环境的匹配，而`(?a:…)`切换到仅ASCII匹配 (默认)。此覆盖仅对窄内联组有效，并且在组外恢复原始匹配模式。
  版本3.6中的新特性
  在版本3.7中发生改变：也可以在组中使用字母 `'a'`、`'L'` 和 `'u'`。
  
- `(?P...)`

  Similar to regular parentheses, but the substring matched by the group is  accessible via the symbolic group name *name*. Group names must be valid  Python identifiers, and each group name must be defined only once within a  regular expression. A symbolic group is also a numbered group, just as if the  group were not named. Named groups can be referenced in three contexts. If the pattern is `(?P['"]).*?(?P=quote)` (i.e. matching a  string quoted with either single or double quotes):    Context of reference to group “quote” Ways to reference it  in the same pattern itself  `(?P=quote)` (as shown) `\1`  when processing match object *m*  `m.group('quote')` `m.end('quote')` (etc.)  in a string passed to the *repl* argument of `re.sub()`  `\g` `\g<1>` `\1`

- `(?P=name)`

  A backreference to a named group; it matches whatever text was matched by the  earlier group named *name*.

- `(?#...)`

  A comment; the contents of the parentheses are simply ignored.

- `(?=...)`

  Matches if `...` matches next, but doesn’t consume any of the string.  This is called a *lookahead assertion*. For example, `Isaac (?=Asimov)` will match `'Isaac '` only if it’s followed by `'Asimov'`.

- `(?!...)`

  Matches if `...` doesn’t match next. This is a *negative  lookahead assertion*. For example, `Isaac (?!Asimov)` will match `'Isaac '` only if it’s *not* followed by `'Asimov'`.

- `(?<=...)`

  Matches if the current position in the string is preceded by a match for  `...`  that ends at the current position. This is called a *positive  lookbehind assertion*. `(?<=abc)def` will find a match in `'abcdef'`,  since the lookbehind will back up 3 characters and check if the contained  pattern matches. The contained pattern must only match strings of some fixed  length, meaning that `abc` or `a|b` are allowed, but `a*` and `a{3,4}` are  not. Note that patterns which start with positive lookbehind assertions will not  match at the beginning of the string being searched; you will most likely want  to use the [`search()`](#re.search) function rather than the [`match()`](#re.match) function: `>>> import re >>> m = re.search('(?<=abc)def', 'abcdef') >>> m.group(0) 'def' ` This example looks for a word following a hyphen: `>>> m = re.search(r'(?<=-)\w+', 'spam-egg') >>> m.group(0) 'egg' ` Changed in version 3.5: Added  support for group references of fixed length.

- `(?

  Matches if the current position in the string is not preceded by a match for  `...`.  This is called a *negative lookbehind assertion*. Similar to  positive lookbehind assertions, the contained pattern must only match strings of  some fixed length. Patterns which start with negative lookbehind assertions may  match at the beginning of the string being searched.

- `(?(id/name)yes-pattern|no-pattern)`

  Will try to match with `yes-pattern` if the group with given *id* or  *name* exists, and with `no-pattern` if it doesn’t. `no-pattern` is  optional and can be omitted. For example, `(<)?(\w+@\w+(?:\.\w+)+)(?(1)>|$)` is a poor email  matching pattern, which will match with `''` as well as `'user@host.com'`, but not with `' nor `'user@host.com>'`.

The special sequences consist of `'\'` and a  character from the list below. If the ordinary character is not an ASCII digit  or an ASCII letter, then the resulting RE will match the second character. For  example, `\$` matches the character `'$'`.

- `\number`

  Matches the contents of the group of the same number. Groups are numbered  starting from 1. For example, `(.+) \1` matches `'the the'` or `'55 55'`, but not `'thethe'`  (note the space after the group). This special sequence can only be used to  match one of the first 99 groups. If the first digit of *number* is 0, or  *number* is 3 octal digits long, it will not be interpreted as a group  match, but as the character with octal value *number*. Inside the `'['` and `']'` of a  character class, all numeric escapes are treated as characters.

- `\A`

  Matches only at the start of the string.

- `\b`

  Matches the empty string, but only at the beginning or end of a word. A word  is defined as a sequence of word characters. Note that formally, `\b` is defined  as the boundary between a `\w` and a `\W` character  (or vice versa), or between `\w` and the beginning/end of the string. This means that  `r'\bfoo\b'` matches `'foo'`, `'foo.'`, `'(foo)'`,  `'bar foo baz'` but not `'foobar'` or  `'foo3'`. By default Unicode alphanumerics are the ones used in Unicode patterns, but  this can be changed by using the [`ASCII`](#re.ASCII) flag. Word boundaries are determined by the  current locale if the [`LOCALE`](#re.LOCALE) flag is used. Inside a character range, `\b` represents  the backspace character, for compatibility with Python’s string  literals.

- `\B`

  Matches the empty string, but only when it is *not* at the beginning  or end of a word. This means that `r'py\B'`  matches `'python'`, `'py3'`, `'py2'`, but  not `'py'`, `'py.'`, or `'py!'`. `\B` is just  the opposite of `\b`, so word characters in Unicode patterns are Unicode  alphanumerics or the underscore, although this can be changed by using the [`ASCII`](#re.ASCII) flag. Word boundaries are determined by the  current locale if the [`LOCALE`](#re.LOCALE) flag is used.

- `\d`

  For Unicode (str) patterns: Matches any Unicode decimal digit (that is, any character in Unicode  character category [Nd]). This includes `[0-9]`, and  also many other digit characters. If the [`ASCII`](#re.ASCII) flag is used only `[0-9]` is  matched. For 8-bit (bytes) patterns: Matches any decimal digit; this is equivalent to `[0-9]`.

- `\D`

  Matches any character which is not a decimal digit. This is the opposite of  `\d`. If  the [`ASCII`](#re.ASCII) flag is used this becomes the equivalent of  `[^0-9]`.

- `\s`

  For Unicode (str) patterns: Matches Unicode whitespace characters (which includes `[ \t\n\r\f\v]`, and also many other characters, for example  the non-breaking spaces mandated by typography rules in many languages). If the  [`ASCII`](#re.ASCII) flag is used, only `[ \t\n\r\f\v]` is matched. For 8-bit (bytes) patterns: Matches characters considered whitespace in the ASCII character set; this is  equivalent to `[ \t\n\r\f\v]`.

- `\S`

  Matches any character which is not a whitespace character. This is the  opposite of `\s`. If the [`ASCII`](#re.ASCII) flag is used this becomes the equivalent of  `[^ \t\n\r\f\v]`.

- `\w`

  For Unicode (str) patterns: Matches Unicode word characters; this includes most characters that can be  part of a word in any language, as well as numbers and the underscore. If the [`ASCII`](#re.ASCII) flag is used, only `[a-zA-Z0-9_]`  is matched. For 8-bit (bytes) patterns: Matches characters considered alphanumeric in the ASCII character set; this  is equivalent to `[a-zA-Z0-9_]`. If the [`LOCALE`](#re.LOCALE) flag is used, matches characters considered  alphanumeric in the current locale and the underscore.

- `\W`

  Matches any character which is not a word character. This is the opposite of  `\w`. If  the [`ASCII`](#re.ASCII) flag is used this becomes the equivalent of  `[^a-zA-Z0-9_]`. If the [`LOCALE`](#re.LOCALE) flag is used, matches characters which are  neither alphanumeric in the current locale nor the underscore.

- `\Z`

  Matches only at the end of the string.

Most of the standard escapes supported by Python string literals  are also accepted by the regular expression parser:

```
\a      \b      \f      \n
\N      \r      \t      \u
\U      \v      \x      \\
```

(Note that `\b` is used to represent word boundaries, and means  “backspace” only inside character classes.)

`'\u'`, `'\U'`, and `'\N'` escape  sequences are only recognized in Unicode patterns. In bytes patterns they are  errors. Unknown escapes of ASCII letters are reserved for future use and treated  as errors.

Octal escapes are included in a limited form. If the first digit is a 0, or  if there are three octal digits, it is considered an octal escape. Otherwise, it  is a group reference. As for string literals, octal escapes are always at most  three digits in length.

Changed in version 3.3: The  `'\u'`  and `'\U'` escape sequences have been added.

Changed in version 3.6: Unknown  escapes consisting of `'\'` and an ASCII letter now are errors.

Changed in version 3.8: The  `'\N{name}'` escape sequence has been added. As in string  literals, it expands to the named Unicode character (e.g. `'\N{EM DASH}'`).

## Module Contents

The module defines several functions, constants, and an exception. Some of  the functions are simplified versions of the full featured methods for compiled  regular expressions. Most non-trivial applications always use the compiled  form.

Changed in version 3.6: Flag  constants are now instances of `RegexFlag`, which is a subclass of [`enum.IntFlag`](enum.html#enum.IntFlag).

- `re.``compile`(*pattern*, *flags=0*) 

  Compile a regular expression pattern into a [regular expression  object](#re-objects), which can be used for matching using its [`match()`](#re.Pattern.match), [`search()`](#re.Pattern.search) and other methods, described below. The expression’s behaviour can be modified by specifying a *flags*  value. Values can be any of the following variables, combined using bitwise OR  (the `|`  operator). The sequence `prog = re.compile(pattern) result = prog.match(string) ` is equivalent to `result = re.match(pattern, string) ` but using [`re.compile()`](#re.compile) and saving the resulting regular  expression object for reuse is more efficient when the expression will be used  several times in a single program. Note The compiled versions of the most recent patterns passed to [`re.compile()`](#re.compile) and the module-level matching functions  are cached, so programs that use only a few regular expressions at a time  needn’t worry about compiling regular expressions.

- `re.``A` 

- `re.``ASCII` 

  Make `\w`, `\W`, `\b`, `\B`, `\d`, `\D`, `\s` and `\S` perform ASCII-only matching instead of full Unicode  matching. This is only meaningful for Unicode patterns, and is ignored for byte  patterns. Corresponds to the inline flag `(?a)`. Note that for backward compatibility, the `re.U` flag still exists (as well as its synonym `re.UNICODE` and its embedded counterpart `(?u)`), but  these are redundant in Python 3 since matches are Unicode by default for strings  (and Unicode matching isn’t allowed for bytes).

- `re.``DEBUG` 

  Display debug information about compiled expression. No corresponding inline  flag.

- `re.``I` 

- `re.``IGNORECASE` 

  Perform case-insensitive matching; expressions like `[A-Z]` will  also match lowercase letters. Full Unicode matching (such as `Ü` matching  `ü`) also  works unless the [`re.ASCII`](#re.ASCII) flag is used to disable non-ASCII matches.  The current locale does not change the effect of this flag unless the [`re.LOCALE`](#re.LOCALE) flag is also used. Corresponds to the  inline flag `(?i)`. Note that when the Unicode patterns `[a-z]` or  `[A-Z]`  are used in combination with the [`IGNORECASE`](#re.IGNORECASE) flag, they will match the 52 ASCII  letters and 4 additional non-ASCII letters: ‘İ’ (U+0130, Latin capital letter I  with dot above), ‘ı’ (U+0131, Latin small letter dotless i), ‘ſ’ (U+017F, Latin  small letter long s) and ‘K’ (U+212A, Kelvin sign). If the [`ASCII`](#re.ASCII) flag is used, only letters ‘a’ to ‘z’ and ‘A’  to ‘Z’ are matched.

- `re.``L` 

- `re.``LOCALE` 

  Make `\w`, `\W`, `\b`, `\B` and case-insensitive matching dependent on the  current locale. This flag can be used only with bytes patterns. The use of this  flag is discouraged as the locale mechanism is very unreliable, it only handles  one “culture” at a time, and it only works with 8-bit locales. Unicode matching  is already enabled by default in Python 3 for Unicode (str) patterns, and it is  able to handle different locales/languages. Corresponds to the inline flag `(?L)`. Changed in version 3.6: [`re.LOCALE`](#re.LOCALE) can be used only with bytes patterns and  is not compatible with [`re.ASCII`](#re.ASCII). Changed in version 3.7: Compiled  regular expression objects with the [`re.LOCALE`](#re.LOCALE) flag no longer depend on the locale at  compile time. Only the locale at matching time affects the result of  matching.

- `re.``M` 

- `re.``MULTILINE` 

  When specified, the pattern character `'^'` matches  at the beginning of the string and at the beginning of each line (immediately  following each newline); and the pattern character `'$'` matches  at the end of the string and at the end of each line (immediately preceding each  newline). By default, `'^'` matches only at the beginning of the string, and  `'$'`  only at the end of the string and immediately before the newline (if any) at the  end of the string. Corresponds to the inline flag `(?m)`.

- `re.``S` 

- `re.``DOTALL` 

  Make the `'.'` special character match any character at all,  including a newline; without this flag, `'.'` will  match anything *except* a newline. Corresponds to the inline flag `(?s)`.

- `re.``X` 

- `re.``VERBOSE` 

  This flag allows you to write regular expressions that look nicer  and are more readable by allowing you to visually separate logical sections of  the pattern and add comments. Whitespace within the pattern is ignored, except  when in a character class, or when preceded by an unescaped backslash, or within  tokens like `*?`, `(?:` or `(?P<...>`. When a line contains a `#` that is not  in a character class and is not preceded by an unescaped backslash, all  characters from the leftmost such `#` through the  end of the line are ignored. This means that the two following regular expression objects that match a  decimal number are functionally equal: `a = re.compile(r"""\d +  # the integral part                   \.    # the decimal point                   \d *  # some fractional digits""", re.X) b = re.compile(r"\d+\.\d*") ` Corresponds to the inline flag `(?x)`.

- `re.``search`(*pattern*, *string*, *flags=0*) 

  Scan through *string* looking for the first location where the regular  expression *pattern* produces a match, and return a corresponding [match  object](#match-objects). Return `None` if no position in the string matches the pattern;  note that this is different from finding a zero-length match at some point in  the string.

- `re.``match`(*pattern*, *string*, *flags=0*) 

  If zero or more characters at the beginning of *string* match the  regular expression *pattern*, return a corresponding [match  object](#match-objects). Return `None` if the string does not match the pattern; note that  this is different from a zero-length match. Note that even in [`MULTILINE`](#re.MULTILINE) mode, [`re.match()`](#re.match) will only match at the beginning of the  string and not at the beginning of each line. If you want to locate a match anywhere in *string*, use [`search()`](#re.search) instead (see also [search() vs. match()](#search-vs-match)).

- `re.``fullmatch`(*pattern*, *string*, *flags=0*) 

  If the whole *string* matches the regular expression *pattern*,  return a corresponding [match object](#match-objects). Return `None` if the  string does not match the pattern; note that this is different from a  zero-length match. New in version  3.4.

- `re.``split`(*pattern*, *string*, *maxsplit=0*, *flags=0*) 

  Split *string* by the occurrences of *pattern*. If capturing  parentheses are used in *pattern*, then the text of all groups in the  pattern are also returned as part of the resulting list. If *maxsplit* is  nonzero, at most *maxsplit* splits occur, and the remainder of the string  is returned as the final element of the list. `>>> re.split(r'\W+', 'Words, words, words.') ['Words', 'words', 'words', ''] >>> re.split(r'(\W+)', 'Words, words, words.') ['Words', ', ', 'words', ', ', 'words', '.', ''] >>> re.split(r'\W+', 'Words, words, words.', 1) ['Words', 'words, words.'] >>> re.split('[a-f]+', '0a3B9', flags=re.IGNORECASE) ['0', '3', '9'] ` If there are capturing groups in the separator and it matches at the start of  the string, the result will start with an empty string. The same holds for the  end of the string: `>>> re.split(r'(\W+)', '...words, words...') ['', '...', 'words', ', ', 'words', '...', ''] ` That way, separator components are always found at the same relative indices  within the result list. Empty matches for the pattern split the string only when not adjacent to a  previous empty match. `>>> re.split(r'\b', 'Words, words, words.') ['', 'Words', ', ', 'words', ', ', 'words', '.'] >>> re.split(r'\W*', '...words...') ['', '', 'w', 'o', 'r', 'd', 's', '', ''] >>> re.split(r'(\W*)', '...words...') ['', '...', '', '', 'w', '', 'o', '', 'r', '', 'd', '', 's', '...', '', '', ''] ` Changed in version 3.1: Added  the optional flags argument. Changed in version 3.7: Added  support of splitting on a pattern that could match an empty  string.

- `re.``findall`(*pattern*, *string*, *flags=0*) 

  Return all non-overlapping matches of *pattern* in *string*, as  a list of strings. The *string* is scanned left-to-right, and matches are  returned in the order found. If one or more groups are present in the pattern,  return a list of groups; this will be a list of tuples if the pattern has more  than one group. Empty matches are included in the result. Changed in version 3.7:  Non-empty matches can now start just after a previous empty  match.

- `re.``finditer`(*pattern*, *string*, *flags=0*) 

  Return an [iterator](../glossary.html#term-iterator) yielding [match  objects](#match-objects) over all non-overlapping matches for the RE *pattern*  in *string*. The *string* is scanned left-to-right, and matches  are returned in the order found. Empty matches are included in the result. Changed in version 3.7:  Non-empty matches can now start just after a previous empty  match.

- `re.``sub`(*pattern*, *repl*, *string*, *count=0*, *flags=0*) 

  Return the string obtained by replacing the leftmost non-overlapping  occurrences of *pattern* in *string* by the replacement  *repl*. If the pattern isn’t found, *string* is returned  unchanged. *repl* can be a string or a function; if it is a string, any  backslash escapes in it are processed. That is, `\n` is  converted to a single newline character, `\r` is  converted to a carriage return, and so forth. Unknown escapes of ASCII letters  are reserved for future use and treated as errors. Other unknown escapes such as  `\&`  are left alone. Backreferences, such as `\6`, are  replaced with the substring matched by group 6 in the pattern. For example: `>>> re.sub(r'def\s+([a-zA-Z_][a-zA-Z_0-9]*)\s*\(\s*\):', ...        r'static PyObject*\npy_\1(void)\n{', ...        'def myfunc():') 'static PyObject*\npy_myfunc(void)\n{' ` If *repl* is a function, it is called for every non-overlapping  occurrence of *pattern*. The function takes a single [match  object](#match-objects) argument, and returns the replacement string. For example: `>>> def dashrepl(matchobj): ...     if matchobj.group(0) == '-': return ' ' ...     else: return '-' >>> re.sub('-{1,2}', dashrepl, 'pro----gram-files') 'pro--gram files' >>> re.sub(r'\sAND\s', ' & ', 'Baked Beans And Spam', flags=re.IGNORECASE) 'Baked Beans & Spam' ` The pattern may be a string or a [pattern object](#re-objects). The optional argument *count* is the maximum number of pattern  occurrences to be replaced; *count* must be a non-negative integer. If  omitted or zero, all occurrences will be replaced. Empty matches for the pattern  are replaced only when not adjacent to a previous empty match, so `sub('x*', '-', 'abxd')` returns `'-a-b--d-'`. In string-type *repl* arguments, in addition to the  character escapes and backreferences described above, `\g` will use the substring matched by the  group named `name`, as defined by the `(?P...)` syntax. `\g` uses the corresponding group number;  `\g<2>` is therefore equivalent to `\2`, but isn’t  ambiguous in a replacement such as `\g<2>0`.  `\20`  would be interpreted as a reference to group 20, not a reference to group 2  followed by the literal character `'0'`. The  backreference `\g<0>` substitutes in the entire substring matched  by the RE. Changed in version 3.1: Added  the optional flags argument. Changed in version 3.5:  Unmatched groups are replaced with an empty string. Changed in version 3.6: Unknown  escapes in *pattern* consisting of `'\'` and an  ASCII letter now are errors. Changed in version 3.7: Unknown  escapes in *repl* consisting of `'\'` and an  ASCII letter now are errors. Changed in version 3.7: Empty  matches for the pattern are replaced when adjacent to a previous non-empty  match.

- `re.``subn`(*pattern*, *repl*, *string*, *count=0*, *flags=0*) 

  Perform the same operation as [`sub()`](#re.sub), but return a tuple `(new_string, number_of_subs_made)`. Changed in version 3.1: Added  the optional flags argument. Changed in version 3.5:  Unmatched groups are replaced with an empty string.

- `re.``escape`(*pattern*) 

  Escape special characters in *pattern*. This is useful if you want to  match an arbitrary literal string that may have regular expression  metacharacters in it. For example: `>>> print(re.escape('http://www.python.org')) http://www\.python\.org >>> legal_chars = string.ascii_lowercase + string.digits + "!#$%&'*+-.^_`|~:" >>> print('[%s]+' % re.escape(legal_chars)) [abcdefghijklmnopqrstuvwxyz0123456789!\#\$%\&'\*\+\-\.\^_`\|\~:]+ >>> operators = ['+', '-', '*', '/', '**'] >>> print('|'.join(map(re.escape, sorted(operators, reverse=True)))) /|\-|\+|\*\*|\* ` This function must not be used for the replacement string in [`sub()`](#re.sub) and [`subn()`](#re.subn), only backslashes should be escaped. For  example: `>>> digits_re = r'\d+' >>> sample = '/usr/sbin/sendmail - 0 errors, 12 warnings' >>> print(re.sub(digits_re, digits_re.replace('\\', r'\\'), sample)) /usr/sbin/sendmail - \d+ errors, \d+ warnings ` Changed in version 3.3: The  `'_'`  character is no longer escaped. Changed in version 3.7: Only  characters that can have special meaning in a regular expression are escaped. As  a result, `'!'`, `'"'`, `'%'`, `"'"`, `','`, `'/'`, `':'`, `';'`, `'<'`, `'='`, `'>'`, `'@'`, and `"`"` are no  longer escaped.

- `re.``purge`() 

  Clear the regular expression cache.

- *exception* `re.``error`(*msg*, *pattern=None*, *pos=None*) 

  Exception raised when a string passed to one of the functions here is not a  valid regular expression (for example, it might contain unmatched parentheses)  or when some other error occurs during compilation or matching. It is never an  error if a string contains no match for a pattern. The error instance has the  following additional attributes: `msg`  The unformatted error message. `pattern`  The regular expression pattern. `pos`  The index in *pattern* where compilation failed (may be `None`). `lineno`  The line corresponding to *pos* (may be `None`). `colno`  The column corresponding to *pos* (may be `None`). Changed in version 3.5: Added  additional attributes.



## Regular Expression Objects

Compiled regular expression objects support the following methods and  attributes:

- `Pattern.``search`(*string*[, *pos*[, *endpos*]]) 

  Scan through *string* looking for the first location where this  regular expression produces a match, and return a corresponding [match  object](#match-objects). Return `None` if no position in the string matches the pattern;  note that this is different from finding a zero-length match at some point in  the string. The optional second parameter *pos* gives an index in the string where  the search is to start; it defaults to `0`. This is  not completely equivalent to slicing the string; the `'^'` pattern  character matches at the real beginning of the string and at positions just  after a newline, but not necessarily at the index where the search is to  start. The optional parameter *endpos* limits how far the string will be  searched; it will be as if the string is *endpos* characters long, so  only the characters from *pos* to `endpos - 1` will be searched for a match.  If *endpos* is less than *pos*, no match will be found; otherwise,  if *rx* is a compiled regular expression object, `rx.search(string,  0, 50)` is equivalent to  `rx.search(string[:50], 0)`. `>>> pattern = re.compile("d") >>> pattern.search("dog")     # Match at index 0 >>> pattern.search("dog", 1)  # No match; search doesn't include the "d" `

- `Pattern.``match`(*string*[, *pos*[, *endpos*]]) 

  If zero or more characters at the *beginning* of *string* match  this regular expression, return a corresponding [match object](#match-objects). Return  `None` if  the string does not match the pattern; note that this is different from a  zero-length match. The optional *pos* and *endpos* parameters have the same  meaning as for the [`search()`](#re.Pattern.search) method. `>>> pattern = re.compile("o") >>> pattern.match("dog")      # No match as "o" is not at the start of "dog". >>> pattern.match("dog", 1)   # Match as "o" is the 2nd character of "dog". ` If you want to locate a match anywhere in *string*, use [`search()`](#re.Pattern.search) instead (see also [search() vs. match()](#search-vs-match)).

- `Pattern.``fullmatch`(*string*[, *pos*[, *endpos*]]) 

  If the whole *string* matches this regular expression, return a  corresponding [match object](#match-objects). Return `None` if the  string does not match the pattern; note that this is different from a  zero-length match. The optional *pos* and *endpos* parameters have the same  meaning as for the [`search()`](#re.Pattern.search) method. `>>> pattern = re.compile("o[gh]") >>> pattern.fullmatch("dog")      # No match as "o" is not at the start of "dog". >>> pattern.fullmatch("ogre")     # No match as not the full string matches. >>> pattern.fullmatch("doggie", 1, 3)   # Matches within given limits. ` New in version  3.4.

- `Pattern.``split`(*string*, *maxsplit=0*) 

  Identical to the [`split()`](#re.split) function, using the compiled  pattern.

- `Pattern.``findall`(*string*[, *pos*[, *endpos*]]) 

  Similar to the [`findall()`](#re.findall) function, using the compiled pattern, but  also accepts optional *pos* and *endpos* parameters that limit the  search region like for [`search()`](#re.search).

- `Pattern.``finditer`(*string*[, *pos*[, *endpos*]]) 

  Similar to the [`finditer()`](#re.finditer) function, using the compiled pattern, but  also accepts optional *pos* and *endpos* parameters that limit the  search region like for [`search()`](#re.search).

- `Pattern.``sub`(*repl*, *string*, *count=0*) 

  Identical to the [`sub()`](#re.sub) function, using the compiled  pattern.

- `Pattern.``subn`(*repl*, *string*, *count=0*) 

  Identical to the [`subn()`](#re.subn) function, using the compiled  pattern.

- `Pattern.``flags` 

  The regex matching flags. This is a combination of the flags given to [`compile()`](#re.compile), any `(?...)` inline  flags in the pattern, and implicit flags such as `UNICODE` if the pattern is a Unicode  string.

- `Pattern.``groups` 

  The number of capturing groups in the pattern.

- `Pattern.``groupindex` 

  A dictionary mapping any symbolic group names defined by `(?P)` to group numbers. The dictionary is empty  if no symbolic groups were used in the pattern.

- `Pattern.``pattern` 

  The pattern string from which the pattern object was compiled.

Changed in version 3.7: Added  support of [`copy.copy()`](copy.html#copy.copy) and [`copy.deepcopy()`](copy.html#copy.deepcopy). Compiled regular expression objects  are considered atomic.



## Match Objects

Match objects always have a boolean value of `True`. Since  [`match()`](#re.Pattern.match) and [`search()`](#re.Pattern.search) return `None` when  there is no match, you can test whether there was a match with a simple `if`  statement:

```
match = re.search(pattern, string)
if match:
    process(match)
```

Match objects support the following methods and attributes:

- `Match.``expand`(*template*) 

  Return the string obtained by doing backslash substitution on the template  string *template*, as done by the [`sub()`](#re.Pattern.sub) method. Escapes such as `\n` are  converted to the appropriate characters, and numeric backreferences (`\1`, `\2`) and named  backreferences (`\g<1>`, `\g`) are replaced by the contents of the  corresponding group. Changed in version 3.5:  Unmatched groups are replaced with an empty string.

- `Match.``group`([*group1*, *...*]) 

  Returns one or more subgroups of the match. If there is a single argument,  the result is a single string; if there are multiple arguments, the result is a  tuple with one item per argument. Without arguments, *group1* defaults to  zero (the whole match is returned). If a *groupN* argument is zero, the  corresponding return value is the entire matching string; if it is in the  inclusive range [1..99], it is the string matching the corresponding  parenthesized group. If a group number is negative or larger than the number of  groups defined in the pattern, an [`IndexError`](exceptions.html#IndexError) exception is raised. If a group is  contained in a part of the pattern that did not match, the corresponding result  is `None`. If a group is contained in a part of the pattern  that matched multiple times, the last match is returned. `>>> m = re.match(r"(\w+) (\w+)", "Isaac Newton, physicist") >>> m.group(0)       # The entire match 'Isaac Newton' >>> m.group(1)       # The first parenthesized subgroup. 'Isaac' >>> m.group(2)       # The second parenthesized subgroup. 'Newton' >>> m.group(1, 2)    # Multiple arguments give us a tuple. ('Isaac', 'Newton') ` If the regular expression uses the `(?P...)` syntax, the *groupN*  arguments may also be strings identifying groups by their group name. If a  string argument is not used as a group name in the pattern, an [`IndexError`](exceptions.html#IndexError) exception is raised. A moderately complicated example: `>>> m = re.match(r"(?P\w+) (?P\w+)", "Malcolm Reynolds") >>> m.group('first_name') 'Malcolm' >>> m.group('last_name') 'Reynolds' ` Named groups can also be referred to by their index: `>>> m.group(1) 'Malcolm' >>> m.group(2) 'Reynolds' ` If a group matches multiple times, only the last match is accessible: `>>> m = re.match(r"(..)+", "a1b2c3")  # Matches 3 times. >>> m.group(1)                        # Returns only the last match. 'c3' `

- `Match.``__getitem__`(*g*) 

  This is identical to `m.group(g)`. This allows easier access to an individual  group from a match: `>>> m = re.match(r"(\w+) (\w+)", "Isaac Newton, physicist") >>> m[0]       # The entire match 'Isaac Newton' >>> m[1]       # The first parenthesized subgroup. 'Isaac' >>> m[2]       # The second parenthesized subgroup. 'Newton' ` New in version  3.6.

- `Match.``groups`(*default=None*) 

  Return a tuple containing all the subgroups of the match, from 1 up to  however many groups are in the pattern. The *default* argument is used  for groups that did not participate in the match; it defaults to `None`. For example: `>>> m = re.match(r"(\d+)\.(\d+)", "24.1632") >>> m.groups() ('24', '1632') ` If we make the decimal place and everything after it optional, not all groups  might participate in the match. These groups will default to `None` unless  the *default* argument is given: `>>> m = re.match(r"(\d+)\.?(\d+)?", "24") >>> m.groups()      # Second group defaults to None. ('24', None) >>> m.groups('0')   # Now, the second group defaults to '0'. ('24', '0') `

- `Match.``groupdict`(*default=None*) 

  Return a dictionary containing all the *named* subgroups of the match,  keyed by the subgroup name. The *default* argument is used for groups  that did not participate in the match; it defaults to `None`. For  example: `>>> m = re.match(r"(?P\w+) (?P\w+)", "Malcolm Reynolds") >>> m.groupdict() {'first_name': 'Malcolm', 'last_name': 'Reynolds'} `

- `Match.``start`([*group*]) 

- `Match.``end`([*group*]) 

  Return the indices of the start and end of the substring matched by  *group*; *group* defaults to zero (meaning the whole matched  substring). Return `-1` if *group* exists but did not contribute to  the match. For a match object *m*, and a group *g* that did  contribute to the match, the substring matched by group *g* (equivalent  to `m.group(g)`) is `m.string[m.start(g):m.end(g)] ` Note that `m.start(group)` will equal `m.end(group)`  if *group* matched a null string. For example, after `m = re.search('b(c?)', 'cba')`, `m.start(0)` is 1, `m.end(0)` is  2, `m.start(1)` and `m.end(1)` are  both 2, and `m.start(2)` raises an [`IndexError`](exceptions.html#IndexError) exception. An example that will remove *remove_this* from email addresses: `>>> email = "tony@tiremove_thisger.net" >>> m = re.search("remove_this", email) >>> email[:m.start()] + email[m.end():] 'tony@tiger.net' `

- `Match.``span`([*group*]) 

  For a match *m*, return the 2-tuple `(m.start(group),  m.end(group))`. Note that if *group* did not  contribute to the match, this is `(-1, -1)`. *group* defaults to zero, the entire  match.

- `Match.``pos` 

  The value of *pos* which was passed to the [`search()`](#re.Pattern.search) or [`match()`](#re.Pattern.match) method of a [regex object](#re-objects). This is  the index into the string at which the RE engine started looking for a  match.

- `Match.``endpos` 

  The value of *endpos* which was passed to the [`search()`](#re.Pattern.search) or [`match()`](#re.Pattern.match) method of a [regex object](#re-objects). This is  the index into the string beyond which the RE engine will not go.

- `Match.``lastindex` 

  The integer index of the last matched capturing group, or `None` if no  group was matched at all. For example, the expressions `(a)b`, `((a)(b))`, and  `((ab))`  will have `lastindex == 1` if applied to the string `'ab'`, while  the expression `(a)(b)` will have `lastindex == 2`, if applied to the same  string.

- `Match.``lastgroup` 

  The name of the last matched capturing group, or `None` if the  group didn’t have a name, or if no group was matched at all.

- `Match.``re` 

  The [regular expression object](#re-objects) whose [`match()`](#re.Pattern.match) or [`search()`](#re.Pattern.search) method produced this match  instance.

- `Match.``string` 

  The string passed to [`match()`](#re.Pattern.match) or [`search()`](#re.Pattern.search).

Changed in version 3.7: Added  support of [`copy.copy()`](copy.html#copy.copy) and [`copy.deepcopy()`](copy.html#copy.deepcopy). Match objects are considered  atomic.



## Regular Expression Examples

### Checking for a Pair

In this example, we’ll use the following helper function to display match  objects a little more gracefully:

```
def displaymatch(match):
    if match is None:
        return None
    return '<Match: %r, groups=%r>' % (match.group(), match.groups())
```

Suppose you are writing a poker program where a player’s hand is represented  as a 5-character string with each character representing a card, “a” for ace,  “k” for king, “q” for queen, “j” for jack, “t” for 10, and “2” through “9”  representing the card with that value.

To see if a given string is a valid hand, one could do the following:

```
>>> valid = re.compile(r"^[a2-9tjqk]{5}$")
>>> displaymatch(valid.match("akt5q"))  # Valid.
"<Match: 'akt5q', groups=()>"
>>> displaymatch(valid.match("akt5e"))  # Invalid.
>>> displaymatch(valid.match("akt"))    # Invalid.
>>> displaymatch(valid.match("727ak"))  # Valid.
"<Match: '727ak', groups=()>"
```

That last hand, `"727ak"`, contained a pair, or two of the same valued  cards. To match this with a regular expression, one could use backreferences as  such:

```
>>> pair = re.compile(r".*(.).*\1")
>>> displaymatch(pair.match("717ak"))     # Pair of 7s.
"<Match: '717', groups=('7',)>"
>>> displaymatch(pair.match("718ak"))     # No pairs.
>>> displaymatch(pair.match("354aa"))     # Pair of aces.
"<Match: '354aa', groups=('a',)>"
```

To find out what card the pair consists of, one could use the [`group()`](#re.Match.group) method of the match object in the following  manner:

```
>>> pair = re.compile(r".*(.).*\1")
>>> pair.match("717ak").group(1)
'7'

# Error because re.match() returns None, which doesn't have a group() method:
>>> pair.match("718ak").group(1)
Traceback (most recent call last):
  File "<pyshell#23>", line 1, in <module>
    re.match(r".*(.).*\1", "718ak").group(1)
AttributeError: 'NoneType' object has no attribute 'group'

>>> pair.match("354aa").group(1)
'a'
```

### Simulating scanf()

Python does not currently have an equivalent to `scanf()`. Regular expressions are generally more  powerful, though also more verbose, than `scanf()` format strings. The table below offers some  more-or-less equivalent mappings between `scanf()` format tokens and regular expressions.

| `scanf()` Token           | Regular Expression                        |
| ------------------------- | ----------------------------------------- |
| `%c`                      | `.`                                       |
| `%5c`                     | `.{5}`                                    |
| `%d`                      | `[-+]?\d+`                                |
| `%e`,  `%E`,  `%f`,  `%g` | `[-+]?(\d+(\.\d*)?|\.\d+)([eE][-+]?\d+)?` |
| `%i`                      | `[-+]?(0[xX][\dA-Fa-f]+|0[0-7]*|\d+)`     |
| `%o`                      | `[-+]?[0-7]+`                             |
| `%s`                      | `\S+`                                     |
| `%u`                      | `\d+`                                     |
| `%x`,  `%X`               | `[-+]?(0[xX])?[\dA-Fa-f]+`                |

To extract the filename and numbers from a string like

```
/usr/sbin/sendmail - 0 errors, 4 warnings
```

you would use a `scanf()` format like

```
%s - %d errors, %d warnings
```

The equivalent regular expression would be

```
(\S+) - (\d+) errors, (\d+) warnings
```



### search() vs. match()

Python offers two different primitive operations based on regular  expressions: [`re.match()`](#re.match) checks for a match only at the beginning  of the string, while [`re.search()`](#re.search) checks for a match anywhere in the  string (this is what Perl does by default).

For example:

```
>>> re.match("c", "abcdef")    # No match
>>> re.search("c", "abcdef")   # Match
<re.Match object; span=(2, 3), match='c'>
```

Regular expressions beginning with `'^'` can be  used with [`search()`](#re.search) to restrict the match at the beginning of  the string:

```
>>> re.match("c", "abcdef")    # No match
>>> re.search("^c", "abcdef")  # No match
>>> re.search("^a", "abcdef")  # Match
<re.Match object; span=(0, 1), match='a'>
```

Note however that in [`MULTILINE`](#re.MULTILINE) mode [`match()`](#re.match) only matches at the beginning of the string,  whereas using [`search()`](#re.search) with a regular expression beginning with  `'^'`  will match at the beginning of each line.

```
>>> re.match('X', 'A\nB\nX', re.MULTILINE)  # No match
>>> re.search('^X', 'A\nB\nX', re.MULTILINE)  # Match
<re.Match object; span=(4, 5), match='X'>
```

### Making a Phonebook

[`split()`](#re.split) splits a string into a list delimited by the  passed pattern. The method is invaluable for converting textual data into data  structures that can be easily read and modified by Python as demonstrated in the  following example that creates a phonebook.

First, here is the input. Normally it may come from a file, here we are using  triple-quoted string syntax

```
>>> text = """Ross McFluff: 834.345.1254 155 Elm Street
...
... Ronald Heathmore: 892.345.3428 436 Finley Avenue
... Frank Burger: 925.541.7625 662 South Dogwood Way
...
...
... Heather Albrecht: 548.326.4584 919 Park Place"""
```

The entries are separated by one or more newlines. Now we convert the string  into a list with each nonempty line having its own entry:

```
>>> entries = re.split("\n+", text)
>>> entries
['Ross McFluff: 834.345.1254 155 Elm Street',
'Ronald Heathmore: 892.345.3428 436 Finley Avenue',
'Frank Burger: 925.541.7625 662 South Dogwood Way',
'Heather Albrecht: 548.326.4584 919 Park Place']
```

Finally, split each entry into a list with first name, last name, telephone  number, and address. We use the `maxsplit` parameter of [`split()`](#re.split) because the address has spaces, our  splitting pattern, in it:

```
>>> [re.split(":? ", entry, 3) for entry in entries]
[['Ross', 'McFluff', '834.345.1254', '155 Elm Street'],
['Ronald', 'Heathmore', '892.345.3428', '436 Finley Avenue'],
['Frank', 'Burger', '925.541.7625', '662 South Dogwood Way'],
['Heather', 'Albrecht', '548.326.4584', '919 Park Place']]
```

The `:?` pattern matches the colon after the last name, so  that it does not occur in the result list. With a `maxsplit` of  `4`, we  could separate the house number from the street name:

```
>>> [re.split(":? ", entry, 4) for entry in entries]
[['Ross', 'McFluff', '834.345.1254', '155', 'Elm Street'],
['Ronald', 'Heathmore', '892.345.3428', '436', 'Finley Avenue'],
['Frank', 'Burger', '925.541.7625', '662', 'South Dogwood Way'],
['Heather', 'Albrecht', '548.326.4584', '919', 'Park Place']]
```

### Text Munging

[`sub()`](#re.sub) replaces every occurrence of a pattern with a  string or the result of a function. This example demonstrates using [`sub()`](#re.sub) with a function to “munge” text, or randomize  the order of all the characters in each word of a sentence except for the first  and last characters:

```
>>> def repl(m):
...     inner_word = list(m.group(2))
...     random.shuffle(inner_word)
...     return m.group(1) + "".join(inner_word) + m.group(3)
>>> text = "Professor Abdolmalek, please report your absences promptly."
>>> re.sub(r"(\w)(\w+)(\w)", repl, text)
'Poefsrosr Aealmlobdk, pslaee reorpt your abnseces plmrptoy.'
>>> re.sub(r"(\w)(\w+)(\w)", repl, text)
'Pofsroser Aodlambelk, plasee reoprt yuor asnebces potlmrpy.'
```

### Finding all Adverbs

[`findall()`](#re.findall) matches *all* occurrences of a  pattern, not just the first one as [`search()`](#re.search) does. For example, if a writer wanted to  find all of the adverbs in some text, they might use [`findall()`](#re.findall) in the following manner:

```
>>> text = "He was carefully disguised but captured quickly by police."
>>> re.findall(r"\w+ly", text)
['carefully', 'quickly']
```

### Finding all Adverbs and their Positions

If one wants more information about all matches of a pattern than the matched  text, [`finditer()`](#re.finditer) is useful as it provides [match  objects](#match-objects) instead of strings. Continuing with the previous example, if  a writer wanted to find all of the adverbs *and their positions* in some  text, they would use [`finditer()`](#re.finditer) in the following manner:

```
>>> text = "He was carefully disguised but captured quickly by police."
>>> for m in re.finditer(r"\w+ly", text):
...     print('%02d-%02d: %s' % (m.start(), m.end(), m.group(0)))
07-16: carefully
40-47: quickly
```

### Raw String Notation

Raw string notation (`r"text"`) keeps regular expressions sane. Without it,  every backslash (`'\'`) in a regular expression would have to be prefixed  with another one to escape it. For example, the two following lines of code are  functionally identical:

```
>>> re.match(r"\W(.)\1\W", " ff ")
<re.Match object; span=(0, 4), match=' ff '>
>>> re.match("\\W(.)\\1\\W", " ff ")
<re.Match object; span=(0, 4), match=' ff '>
```

When one wants to match a literal backslash, it must be escaped in the  regular expression. With raw string notation, this means `r"\\"`.  Without raw string notation, one must use `"\\\\"`,  making the following lines of code functionally identical:

```
>>> re.match(r"\\", r"\\")
<re.Match object; span=(0, 1), match='\\'>
>>> re.match("\\\\", r"\\")
<re.Match object; span=(0, 1), match='\\'>
```

### Writing a Tokenizer

A [tokenizer or  scanner](https://en.wikipedia.org/wiki/Lexical_analysis) analyzes a string to categorize groups of characters. This is a  useful first step in writing a compiler or interpreter.

The text categories are specified with regular expressions. The technique is  to combine those into a single master regular expression and to loop over  successive matches:

```
import collections
import re

Token = collections.namedtuple('Token', ['type', 'value', 'line', 'column'])

def tokenize(code):
    keywords = {'IF', 'THEN', 'ENDIF', 'FOR', 'NEXT', 'GOSUB', 'RETURN'}
    token_specification = [
        ('NUMBER',   r'\d+(\.\d*)?'),  # Integer or decimal number
        ('ASSIGN',   r':='),           # Assignment operator
        ('END',      r';'),            # Statement terminator
        ('ID',       r'[A-Za-z]+'),    # Identifiers
        ('OP',       r'[+\-*/]'),      # Arithmetic operators
        ('NEWLINE',  r'\n'),           # Line endings
        ('SKIP',     r'[ \t]+'),       # Skip over spaces and tabs
        ('MISMATCH', r'.'),            # Any other character
    ]
    tok_regex = '|'.join('(?P<%s>%s)' % pair for pair in token_specification)
    line_num = 1
    line_start = 0
    for mo in re.finditer(tok_regex, code):
        kind = mo.lastgroup
        value = mo.group()
        column = mo.start() - line_start
        if kind == 'NUMBER':
            value = float(value) if '.' in value else int(value)
        elif kind == 'ID' and value in keywords:
            kind = value
        elif kind == 'NEWLINE':
            line_start = mo.end()
            line_num += 1
            continue
        elif kind == 'SKIP':
            continue
        elif kind == 'MISMATCH':
            raise RuntimeError(f'{value!r} unexpected on line {line_num}')
        yield Token(kind, value, line_num, column)

statements = '''
    IF quantity THEN
        total := total + price * quantity;
        tax := price * 0.05;
    ENDIF;
'''

for token in tokenize(statements):
    print(token)
```

The tokenizer produces the following output:

```
Token(type='IF', value='IF', line=2, column=4)
Token(type='ID', value='quantity', line=2, column=7)
Token(type='THEN', value='THEN', line=2, column=16)
Token(type='ID', value='total', line=3, column=8)
Token(type='ASSIGN', value=':=', line=3, column=14)
Token(type='ID', value='total', line=3, column=17)
Token(type='OP', value='+', line=3, column=23)
Token(type='ID', value='price', line=3, column=25)
Token(type='OP', value='*', line=3, column=31)
Token(type='ID', value='quantity', line=3, column=33)
Token(type='END', value=';', line=3, column=41)
Token(type='ID', value='tax', line=4, column=8)
Token(type='ASSIGN', value=':=', line=4, column=12)
Token(type='ID', value='price', line=4, column=15)
Token(type='OP', value='*', line=4, column=21)
Token(type='NUMBER', value=0.05, line=4, column=23)
Token(type='END', value=';', line=4, column=27)
Token(type='ENDIF', value='ENDIF', line=5, column=4)
Token(type='END', value=';', line=5, column=9)
```

- [Frie09](#id1) 

  Friedl, Jeffrey. Mastering Regular Expressions. 3rd ed., O’Reilly Media,  2009. The third edition of the book no longer covers Python at all, but the  first edition covered writing good regular expression patterns in great  detail.