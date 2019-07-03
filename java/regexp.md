# 正则表达式（Regular Expression） #

正则表达式用于指定字符串模式，可以使用正则表达式检测字符串是否匹配某个模式（提取字符串中和模式匹配的子字符串）、替换字符串中匹配指定模式的子字符串、根据指定模式对字符串进行分组。

## 语法 ##

### 字符 ###

| 结构 | 描述 |
|:---:|:---:|
|`c`（除了`.`、`*`、`+`、`?`、`{`、`|`、`(`、`)`、`[`、`\`、`^`、`$`）| 匹配字符c |
| `\\` | 匹配反斜线 |
| `\t` | 匹配制表符 |
| `\n` | 匹配换行符 |
| `\r` | 匹配回车符 |
| `\f` | 匹配换页符 |
| `\a` | 匹配警告符 |
| `\e` | 匹配esc符 |
| `\cx` | x为`[A-Z@\[\\\]^_?]`匹配对应于x的控制符 |
| `\c`（c不在`[A-Za-z0-9]`中）| 匹配字符c |
| `\0n` | 匹配Unicode代码点为八进制值0n的字符（0<=n<=7）|
| `\0nn` | 匹配Unicode代码点为八进制值0nn的字符（0<=n<=7）|
| `\0mnn` | 匹配Unicode代码点为八进制值0mnn的字符（0<=m<=3，0<=n<=7）|
| `\xhh` | 匹配Unicode代码点为十六进制值0xhh的字符 |
| `\uhhhh` | 匹配Unicode代码点为十六进制值0xhhhh的字符 |
| `\x{h...h}` | 匹配Unicode代码点为十六进制值0xh...h的字符（Character.MIN_CODE_POINT<=0xh...h<=Character.MAX_CODE_POINT）|

### 字符类 ###

字符类是方括号包围的可选择的字符集合。

字符类可以出现在其他字符类中，可以通过并集（隐式）操作符或交集（`&&`）操作符结合。

字符类操作符的优先级由高到低如下所示：

1. 转义序列（`\x`）
2. 方括号（`[...]`）
3. 范围（`a-z`）
4. 并集（`[a-e][i-u]`）
5. 交集（`[a-z&&[aeiou]]`）

字符类内的元字符和字符类外的不同：

+ `.`仅表示字符`.`。
+ `-`表示范围，指所有Unicode值介于上下界之间的字符。
+ `^`为首个字符时表示取补，指除这些字符之外的所有字符。

字符类中包含`-`时，可以将其作为第一个元素或最后一个元素。字符类中包含`]`时，可以将其作为第一个元素。字符类中包含`^`时，将其放在除首位外的任意位置。在字符类中仅需要转义`[`和`\`。

| 结构 | 描述 |
| :---: | :--- |
| `[C1C2...]`（Ci是字符、范围表达式或者字符类）| 匹配C1、C2表示的字符 |
| `[abc]` | 匹配字符a, b或者c |
| `[^abc]` | 匹配除了a，b和c外的任何字符 |
| `[a-zA-Z]` | 范围，匹配a到z和A-Z中的任何字符（包含a、z、A、Z）|
| `[a-d[m-p]]` | 并集，匹配a到z和m到p中的任何字符，等价于[a-dm-p]|
| `[a-z&&[def]]` | 交集，匹配字符d, e或f |
| `[a-z]&&[^bc]]` | 求差，匹配a到z中除b和c外的任何字符，等价于`[ad-z]` |
| `[a-z&&[^m-p]]` | 求差，匹配a到z中除m到p外的任何字符，等价于`[a-lq-z]` |

### 预定义字符类 ###

还有一些预定义的字符类

| 结构 | 描述 |
| :---: | :--- |
| `.` | 匹配除换行符外的任何字符，设置DOTALL时，匹配所有字符 |
| `\d` | 匹配十进制数字，等价于`[0-9]` |
| `\D` | 匹配十进制数字外的任何字符，等价于`[^0-9]` |
| `\h` | 匹配任何水平空白字符，等价于`[ \t\xA0\u1680\u180e\u2000-\u200a\u202f\u205f\u3000]` |
| `\H` | 匹配除水平空白字符外的任何字符，等价于`[^\h]` |
| `\s` | 匹配空白字符，等价于`[ \t\n\x0B\f\r]` |
| `\S` | 匹配非空白字符，等价于`[^\s]` |
| `\v` | 匹配垂直空白字符，等价于`[\n\x0B\f\r\x85\u2028\u2029]` |
| `\V` | 匹配除垂直空白字符之外的任何字符，等价于`[^\v]` |
| `\w` | 匹配单词字符，等价于`[a-zA-Z_0-9]` |
| `\W` | 匹配非单词字符，等价于`[^\w]` |
|POSIX字符类（仅US-ASCII字符）| |
| `\p{Lower}` | 匹配小写字母字符，等价于`[a-z]` |
| `\p{Upper}` | 匹配大写字母字符，等价于`[A-Z]` |
| `\p{ASCII}` | 匹配所有ASCII字符，等价于`[\x00-\x7F]` |
| `\p{Alpha}` | 匹配字母字符，等价于`[\p{Lower}\p{Upper}]` |
| `\p{Digit}` | 匹配十进制数字，等价于`[0-9]` |
| `\p{Alnum}` | 匹配数字字母字符，等价于`[\p{Alpha}\p{Digit}]` |
| `\p{Punct}` | 匹配标点符号，`!"#$%&'()*+,-./:;<=>?@[\]^_`{|}~`其中之一 |
| `\p{Graph}` | 匹配可视字符，等价于`[\p{Alnum}\p{Punct}]` |
| `\p{Print}` | 匹配可打印字符，等价于`[\p{Graph}\x20]` |
| `\p{Blank}` | 匹配空格或制表符，等价于`[ \t]` |
| `\p{Cntrl}` | 匹配控制字符，等价于`[\x00-\x1F\x7F]` |
| `\p{XDigit}` | 匹配十六进制数字，等价于`[0-9a-fA-F]` |
| `\p{Space}` | 匹配空白字符`[ \t\n\x0B\f\r]` |
| `java.lang.Character`类 | |
| `\p{javaLowerCase}` | 等价于`java.lang.Character.isLowerCase()` |
| `\p{javaUpperCase}` | 等价于`java.lang.Character.isUpperCase()` |
| `\p{javaWhitespace}` |  等价于`java.lang.Character.isWhitespace()` |
| `\p{javaMirrored}` 等价于`java.lang.Character.isMirrored()`
| Unicode脚本、块、分类、二进制属性类 |
| `\p{IsLatin}` | 匹配Latin脚本字符 |
| `\p{InGreek}` | 匹配Greek block中的字符 |
| `\p{Lu}` | 匹配大写字母 |
| `\p{IsAlphabetic}` | 匹配字母字符 |
| `\p{Sc}` | 匹配货币字符 |
| `\P{InGreek}` | 除Greek block中的任何字符 |
| `[\p{L}&&[^\p{Lu}]]` | 除大写字母外的任何字符 |

### 边界匹配器 ###

| 结构 | 描述 |
| :---: | :--- |
| `^` | 作为正则表达式中第一个字符时，匹配输入字符串开始，multiline模式中匹配行首 |
| $ | 作为正则表达式中最后一个字符时，匹配输入字符串结尾，multiline模式中匹配行尾 |
| `\A` | 匹配输入字符串的开始位置 |
| `\Z` | 匹配输入字符串的结束位置，但是输入的结尾可以有也可以没有终止符 |
| `\z` | 匹配输入的结束位置 |
| `\b` | 匹配单词边界 |
| `\B` | 匹配非单词边界 |
| `\R` | 任何Unicode行结束符, 等价于`\u000D\u000A|[\u000A\u000B\u000C\u000D\u0085\u2028\u2029]` |
| `\G` | 匹配上一次匹配的结尾 |

行结束符是标志输入字符串行结束的包含一个或两个字符的字符串，下列被识别为行结束符：

+ 换行符（`\n`）
+ 连续的回车符和换行符（`\r\n`）
+ 回车符（`\r`）
+ 下一行符（`\u0085`）
+ 行分隔符（`\u2028`）
+ 段分隔符（`\u2029`）

UNIX_LINES模式中，仅换行符是行结束符。

默认，`^`和`$`忽略行结束符，仅匹配输入字符串的开始和结束，在MULTILINE模式中，`^`匹配输入字符串的开始和行结束符后的位置（除了输入结尾），`$`匹配行结束符前的位置和输入字符串的结尾。

### 量词 ###

量词（`?`、`*`、`+`、`{n}`、`{n,}`、`{n,m}`）的匹配模式：

+ greedy（贪婪）：默认模式，在整个表达式匹配成功的前提下，匹配尽可能多的重复次数。
+ reluctant（勉强）：量词后加上`?`，匹配最少重复次数。
+ possessive（独占）：量词后加上`+`，匹配最大重复次数，即使整个表达式匹配失败。

默认为贪婪模式。

| 结构 | 描述 |
| :---: | :--- |
| `X?` | 匹配0个或1个字符X |
| `X*` | 匹配0个或多个字符X |
| `X+` | 匹配1个或多个字符X |
| `X{n}` | 匹配n个字符X |
| `X{n,}` | 至少匹配n个字符X |
| `X{n,m}` | 匹配n到m个字符X |

#### 勉强量词 ####

形式为`Q?`，其中Q为量词表达式。

| 结构 | 描述 |
| :---: | :--- |
| `X??` | 匹配0个或1个字符X |
| `X*?` | 匹配0个或多个字符X |
| `X+?` | 匹配1个或多个字符X |
| `X{n}?` | 匹配n个字符X |
| `X{n,}?` | 至少匹配n个字符X |
| `X{n,m}?` | 匹配n到m个字符X |

#### 独占量词 ####

形式为`Q+`，其中Q为量词表达式。

| 结构 | 描述 |
| :---: | :--- |
| `X?+` | 匹配0个或1个字符X |
| `X*+` | 匹配0个或多个字符X |
| `X++` | 匹配1个或多个字符X |
| `X{n}+` | 匹配n个字符X |
| `X{n,}+` | 至少匹配n个字符X |
| `X{n,m}+` | 匹配n到m个字符X |

### 逻辑操作符 ###

| 结构 | 描述 |
| :---: | :--- |
| `XY` | X后跟着Y |
| `X|Y` | X或Y |

### 捕获分组与反向索引 ###

可以使用组来定义子表达式，组用`()`包围，可以通过模式匹配器获取每组匹配到的字符串，也可以在替换字符串中或正则表达式组定义后通过`\n`引用指定组匹配到的字符串，`n`表示组编号，从1开始。0表示整个正则表达式匹配到的字符串，嵌套分组按左括号出现的顺序排序。

| 结构 | 描述 |
| :---: | :--- |
| `(X)` | 捕获X匹配到的子字符串作为分组 |
| `(?<name>X)` | 捕获X匹配到的子字符串作为命名分组 |
| `(?:X)` | 匹配X但是不捕获X匹配到的子字符串 |
| `(?>X)` | 将X作为一个单独的非捕获分组 |
| `(?f1f2...)` （fi为`[dimsuUx]`） | 不匹配任何字符，但是打开或关闭匹配标志i、d、m、s、u、x、U |
| `(?f1f2...:X)` （fi为`[dimsuUx]`） | 匹配X，但是不捕获X匹配到的子字符串，打开或关闭匹配标志i、d、m、s、u、x、U |
| `\n` | 第n个捕获分组匹配到的内容 |
| `\k<name>` | 命名捕获分组name匹配到的内容 |

### 引用 ###

反斜线（`\`）用于构造转义序列，或者引用其他将被解释为非转义序列的字符。

不能在任何不表示转义序列的字母字符前使用反斜线，但是可以在非字母字符前使用反斜线，无论这个字符是否是非转义序列的一部分。

Java语言规范指定Java源代码里字符串字面值中的反斜线将被解释为Unicode转义符或者其他字符转义，所以需要加倍表示正则表达式的字符串字面值中的反斜线，以避免他们被Java字节码编译器解释。比如，字符串`"\b"`解释为正则表达式时仅表示匹配单个退格符，而`"\\b"`才表示匹配单词边界。字符串字面值`"\(hello\)"`是非法字符串并且导致编译错误，为了匹配字符串"(hello)"，必须使用`"\\(hello\\)"`。

| 结构 | 描述 |
| :---: | :--- |
| `\` | 不匹配任何字符，但是转义其后的字符 |
| `\Q` | 不匹配任何字符，但是转义其后到`\E`之间的所有字符 |
| `\E` | 不匹配任何字符，停止`\Q`开始的转义 |
| `(?=X)` | 零宽正向先行 |
| `(?!X)` | 零宽负向先行 |
| `(?<=X)` | 零宽正向后行 |
| `(?<!X)` | 零宽负向后行 |

### 标志 ###

+ `Pattern.CASE_INSENSITIVE`或`i`：忽略字母大小写，默认仅考虑US ASCII字符
+ `Pattern.UNICODE_CASE`或`u`：和`CASE_INSENSITIVE`一起使用时，使用UNICODE字母大小写
+ `Pattern.UNICODE_CHATACTER_CLASS`或`U`：使用Unicode字符类而不是POSIX，隐式设置`UNICODE_CASE`
+ `Pattern.MULTILINE`或`m`：使`^`和`$`匹配行首和行尾，而不是整个输入的开始和结束
+ `Pattern.UNIX_LINES`或`d`：在multiline模式中匹配`^`和`$`时仅使用`\n`作为行结束符
+ `Pattern.DOTALL`或`s`：使`.`匹配所有字符，包括行结束符
+ `Pattern.COMMENTS`或`x`：忽略空白和注释（从`#`开始到行尾）
+ `Pattern.LITERAL`：字符串必须和模式字面值完全匹配，除了大小写
+ `Pattern.CANON_EQ`：考虑Unicode等价性

可以在正则表达式里设置或在编译正则表达式时设置标志。

在正则表达式中通过`(?f1f2...:expression)`设置，fi为d、i、m、s、u、U、x中的一个。

## Java类 ##

### Pattern ###

正则表达式的编译表示，线程安全。

以字符串表示的正则表达式必须先编译成Patter对象，然后用Pattern对象创建Matcher对象，Matcher对象负责把任意字符串和正则表达式匹配。匹配过程中使用的所有状态都存储在Matcher对象中，多个Matcher对象可以共享相同的模式。

包含方法如下：

+ `Predicate<String> asPredicate()`：创建一个用于匹配字符串的断言。
+ `static Pattern compile(String regex)`
+ `static Pattern compile(String regex, int flags)`
    : 编译指定的正则表达式，可以指定标志。

+ `Matcher matcher(CharSequence input)`：创建一个将字符串和正则表达式匹配的Matcher。
+ `static boolean matches(String regex, CharSequence input)`：检测字符串是符合正则表达式匹配。
+ `String[] split(CharSequence input)`
+ `String[] split(CharSequence input, int limit)`
    : 将正则表达式匹配到的字符串作为分隔符将字符串分组。
+ `Stream<String> splitAsStream(CharSequence input)`：将输入字符串根据正则表达式匹配到的字符串分组，将结果作为流返回。
+ `int flags()`：返回模式的匹配标志。

### Matcher ###

Matcher负责解释Pattern对输入字符串进行匹配操作，线程不安全。

Matcher对象由调用Pattern对象的matcher方法创建，被用于执行以下三种匹配操作：

+ matches方法尝试将整个输入字符串和模式匹配，当且仅当整个输入字符串和模式匹配时返回true。
+ lookingAt方法尝试从字符串开始处和模式进行匹配，当且仅当输入字符串的前缀和模式匹配时返回true。
+ find方法扫描输入字符串查找模式匹配到的下个子字符串，当且仅当输入字符串的子字符串匹配模式时返回true。

这些方法都返回一个布尔值表示匹配是否成功。

包含方法如下：

+ `boolean matches()`：尝试将整个输入字符串和模式匹配，如果成功，可以通过`start`、`end`、`group`方法获取匹配到的字符串的信息。
+ `boolean lookingAt()`：尝试从字符串开始处和模式进行匹配，如果匹配成功，可以通过`start`、`end`、`group`方法获取匹配到的字符串的信息。
+ `boolean find()`
+ `boolean find(int start)`
    ：尝试查找输入字符串中匹配模式的下个子字符串，从输入字符串的开始处，或者上个匹配的结束处开始匹配，匹配成功，可以通过`start`、`end`、`group`方法获取匹配到的字符串的信息。
      + start：指定匹配开始处的索引。start小于0或者大于输入字符串长度时，抛出IndexOutOfBoundsException异常。
+ `int start()`：返回上次匹配到的子字符串首字符的索引。
+ `int start(int group)`
+ `int start(String name)`
    ：返回上次匹配时正则表达式中指定组匹配到的子字符串首字符的索引。
+ `int end()`：返回上次匹配到的子字符串最后一个字符后的偏移量。
+ `int end(int group)`
+ `int end(String name)`
    ：返回上次匹配时正则表达式中指定组匹配到的子字符串最后一个字符后的偏移量。
+ `String group()`：返回上次匹配时匹配到的输入字符串中的子字符串，等价于`s.subString(m.start(), m.end())`。
+ `String group(int group)`
+ `String group(String group)`
    ：返回上次匹配时正则表达式中指定组匹配到的子字符串，等价于`s.subString(m.start(group), m.end(group))`。
+ `int groupCount()`：返回捕获分组的数目。
+ `String replaceAll(String replacement)`：使用指定的字符串替换正则表达式在输入字符串中匹配到的所有子字符串。
+ `String replaceFirst(String replacement)`：使用指定的字符串替换正则表达式在输入字符串第一次匹配到的子字符串。

## 应用 ##

### 匹配 ###

检查字符串是否和模式匹配，获取匹配的字符串：

```Java
Patter pattern = Pattern.compile(patternString);
Matcher matcher = pattern.matcher(input);
if (matcher.matches()) {
    int start = matcher.start();
    int end = matcher.end();
    String match = matcher.group();
}

boolean b = Pattern.matches("a*b", "aaaaab");
```

在输入字符串中获取一个或多个和模式匹配的子字符串：

```Java
while (matcher.find())
{
    int start = matcher.start();
    int end = matcher.end();
    String match = matcher.group();
}
```

正则表达式里包含分组时：

+ `groupCount()`：获取分组数目
+ `int start(int groupIndex)`
+ `int start(String groupName)`
：获取分组匹配到的字符串开始位置的索引
+ `int end(int groupIndex)`
+ `int end(String groupName)`
：获取分组匹配到的字符串结束位置的索引
+ `String group(int groupIndex)`
+ `String group(String groupName)`
：获取分组匹配到的字符串

匹配集合或流中的元素时，可以将模式转化为断言：

```java
Stream<String> strings = ...;
Stream<String> result = strings.filter(pattern.asPredicate());
```

### 替换 ###

使用函数replaceAll或replaceFirst使用指定字符串替换输入字符串中匹配模式的子字符串。

```Java
Pattern pattern = Pattern.compile(patternString);
Matcher matcher = pattern.matcher(input);
// 替换所有和模式匹配的子字符串
String output = matcher.replaceAll(targetString);
// 仅替换第一个和模式匹配的子字符串
String output = matcher.replaceFirst(targetString);
```

替换的字符串中可以包含模式中的分组，`$n`被替换为第n组，`${name}`被替换为指定名称的分组。使用`\$`表示`$`字面值。使用`Matcher.quoteReplacement(str))`来保留替换字符串中的`$`和`\`。

### 分组 ###

使用函数split、splitAsStream或String.split，以正则表达式匹配到的字符串作为分界符，将输入字符串分割成组。

```scala
Pattern pattern = Pattern.compile(patternString);
String[] strings = pattern.split(input)
```

字符串类中包含的正则表达式方法：

+ `boolean matches​(String regex)`：检测字符串是否匹配指定的正则表达式。
+ `String replaceAll​(String regex, String replacement)`：使用指定字符串替换正则表达式匹配到的所有子字符串。
+ `String replaceFirst​(String regex, String replacement)`：使用指定字符串替换正则表达式匹配到的第一个子字符串。
+ `String[] split​(String regex)`：使用正则表达式匹配到的子字符串作为分隔符对字符串进行分组，等价于`split(regex, 0)`，删除尾部的空字符串。
+ `String[] split​(String regex, int limit)`：使用正则表达式匹配到的子字符串作为分隔符对字符串进行分组。参数limit控制应用模式匹配的次数，大于0时，最多应用模式匹配n-1次，输入字符串中第n-1个模式匹配到的子字符串后的部分将作为结果数组的最后一个元素，为非正数时，尽可能多的应用模式匹配，另外为0时，将会忽略尾部的空字符串。
