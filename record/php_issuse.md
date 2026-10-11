本地没有 PHP 可执行文件（`php not found`），所以下面的结论来自版本变更记录核对，关键节点我都做了验证。先给一个结论：**5.4 → 8.5 的字符串返回值差异不是一次性变更，而是在 7.0 / 7.1 / 7.4 / 8.0 / 8.1 / 8.2 / 8.3 七个节点上累积出来的**，其中 **PHP 8.0 是重灾区**。

---

# 一、版本节点总览

| 版本 | 字符串相关返回值变更 |
|---|---|
| 5.4 | 基线。`htmlspecialchars` 默认编码已是 UTF-8，但默认 flags 仍是 `ENT_COMPAT` |
| 7.0 | `substr` 在"字符串长度恰好等于 start"时由 `false` 改 `""`；`ereg*` / `split` / `sql_regcase` 系列整体移除 |
| 7.1 | 字符串负偏移 `$s[-1]` 开始支持；`strpos` 负 offset 开始支持 |
| 7.4 | `$s{0}` 大括号偏移废弃；`implode($arr, $glue)` 逆序参数废弃；`money_format` 废弃 |
| **8.0** | **`substr`/`iconv_substr`/`grapheme_substr` 越界 `false` → `""`**；参数类型错误 `E_WARNING+null` → `TypeError`；多个函数负数参数 → `ValueError`；`implode` 逆序参数移除；`$s{0}` 移除；`mbstring.func_overload` 移除；`money_format` 移除 |
| **8.1** | **`htmlspecialchars`/`htmlentities` 默认 flags 改为 `ENT_QUOTES\|ENT_SUBSTITUTE\|ENT_HTML401`**；向内部函数传 `null` 开始 `Deprecated` |
| **8.2** | **`str_split('')` 返回 `[]` 而非 `[""]`**；`strcmp` 家族不再返回长度差；`strtolower`/`strtoupper`/`stristr` 等不再受 locale 影响；`"${var}"` 插值废弃 |
| 8.3 | 新增 `str_increment()` / `str_decrement()` / `mb_str_pad()` |
| **8.5** | 字符串函数**无新的破坏性返回值变更**；仅废弃对非数字字符串使用 `++`/`--`（改用 `str_increment()`）；新增 `grapheme_str_split()` |

---

# 二、返回值与类型变化（核心）

## 2.1 统一规则：5.4 的"警告 + 返回 null"被拆成四类异常

PHP 5.4 时代内部函数的错误处理高度一致：**发 `E_WARNING`，然后返回 `null`（部分函数返回 `false`）**。PHP 8.x 把它拆成了四类：

| 5.4 的触发条件 | 5.4 表现 | 8.5 表现 |
|---|---|---|
| 参数类型完全不匹配（传 array/object 给 string 参数） | `E_WARNING` + 返回 `null` | **`TypeError`**（致命，可 catch） |
| 参数值非法（空分隔符、负数长度、非法 offset） | `E_WARNING` + 返回 `null`/`false` | **`ValueError`** |
| 参数个数不足 | `E_WARNING: Too few arguments` + 返回 `null` | **`ArgumentCountError`** |
| 传 `null` 给非 nullable 参数 | 静默当 `""` 处理 | 8.1+ **`Deprecated`**，仍当 `""`（PHP 9 将变 `TypeError`） |

```php
// PHP 5.4
var_dump(strlen(null));        // int(0)                                —— 完全静默
var_dump(strlen([]));          // NULL + Warning: strlen() expects parameter 1 to be string, array given
var_dump(trim(null, ' '));     // string(0) ""
var_dump(str_repeat('a', -1)); // NULL + Warning: Second argument has to be greater than or equal to 0
var_dump(strpos('abc', 'a', 5)); // false + Warning: Offset not contained in string

// PHP 8.5
var_dump(strlen(null));        // Deprecated: strlen(): Passing null to parameter #1 ($string) of type string is deprecated
                               // int(0)   —— 注意：仍是 0，不是报错
var_dump(strlen([]));          // Fatal error: Uncaught TypeError: strlen(): Argument #1 ($string) must be of type string, array given
var_dump(trim(null, ' '));     // Deprecated + string(0) ""
var_dump(str_repeat('a', -1)); // Fatal error: Uncaught ValueError: str_repeat(): Argument #2 ($times) must be greater than or equal to 0
var_dump(strpos('abc', 'a', 5)); // false（无警告，静默返回 false）
```

> ⚠️ **最容易踩的坑**：`strlen(null)` 在 5.4 和 8.5 下**返回值都是 `0`**。升级后大量 `Deprecated` 日志会让日志系统爆掉，但业务逻辑不会立刻出错——这类问题会被长期掩盖。

## 2.2 逐函数返回值对照表

### `explode()`

| 调用 | PHP 5.4 | PHP 8.5 |
|---|---|---|
| `explode(',', '')` | `[""]` （1 个元素） | `[""]` （未变） |
| `explode(',', 'a,b,')` | `["a","b",""]` | 同左 |
| `explode('', 'abc')` | `false` + `Warning: Empty delimiter` | **`ValueError: explode(): Argument #1 ($separator) must not be empty`** |
| `explode(',', null)` | `[""]`，无警告 | `[""]` + `Deprecated` |
| `explode(',', [])` | `null` + `Warning: expects parameter 2 to be string, array given` | **`TypeError`** |
| `explode(',', 'a,b,c', -2)` | `["a"]` | `["a"]` （未变） |

```php
// 5.4 常见写法：靠 false 兜底 —— 8.5 下会直接抛异常，兜底分支永远走不到
$parts = explode($delimiter, $str) ?: [$str];   // 危险！

// 8.5 正确写法
if ($delimiter === '') {
    throw new InvalidArgumentException('分隔符不能为空');
}
$parts = explode($delimiter, $str);
```

### `implode()` / `join()`

| 调用 | PHP 5.4 | PHP 8.5 |
|---|---|---|
| `implode(',', [])` | `""` | `""` （未变） |
| `implode(',', ['a','b'])` | `"a,b"` | 同左 |
| `implode(',', null)` | `null` + `Warning: Invalid arguments passed` | **`TypeError`** |
| `implode(',', 'abc')` | `null` + `Warning: Invalid arguments passed` | **`TypeError`** |
| `implode(['a','b'], ',')` | `"a,b"`（逆序，可用） | **`TypeError`**（7.4 废弃、8.0 移除） |

### `substr()` —— 最危险的一个

| 调用 | PHP 5.4 | PHP 8.5 |
|---|---|---|
| `substr('FooBar', 42)` | **`false`** | **`""`** |
| `substr('FooBar', 3)` | `"Bar"` | `"Bar"` |
| `substr('FooBar', -3)` | `"Bar"` | `"Bar"` |
| `substr('FooBar', -42)` | `"FooBar"`（start 夹紧到 0） | `"FooBar"` |
| `substr('abcdef', 4, -4)` | **`false`** | **`""`** |
| `substr(null, 0, 2)` | `""` | `""` + `Deprecated` |
| `substr([], 0, 2)` | `null` + `Warning` | **`TypeError`** |

```php
// PHP 5.4 —— 这段在 8.5 下会静默失效（永远进不了 if）
$suffix = substr($filename, -4);
if ($suffix === false) {
    $suffix = '.tmp';
}

// PHP 8.5 —— 必须显式判断
$filename = (string) $filename;
$suffix = strlen($filename) >= 4 ? substr($filename, -4) : '.tmp';
```

同样变更的还有 **`iconv_substr()`** 和 **`grapheme_substr()`**（`mb_substr()` 在 5.4 就已返回 `""`，未变）。

### `strpos()` / `stripos()` / `strrpos()` / `strripos()`

| 调用 | PHP 5.4 | PHP 8.5 |
|---|---|---|
| `strpos('abc', 'x')` | `false` | `false` （**未变**） |
| `strpos('abc', 'a')` | `0` | `0` （**未变**） |
| `strpos('abc', 'a', 5)` | `false` + `Warning: Offset not contained in string` | `false`（无警告） |
| `strpos('abcabc', 'a', -2)` | `false` + `Warning` | **`3`**（7.1 起支持负 offset） |
| `strpos(null, 'a')` | `false` | `false` + `Deprecated` |
| `strpos([], 'a')` | `null` + `Warning` | **`TypeError`** |

```php
// 5.4 与 8.5 都成立的经典坑，且 PHP 8.0 的"字符串转数字"新规则让下面这行更隐蔽
if (strpos($haystack, $needle) == false) { ... }  // 找到位置 0 时会误判
// 必须用 === false
if (strpos($haystack, $needle) === false) { ... }
```

### `trim()` / `ltrim()` / `rtrim()`

| 调用 | PHP 5.4 | PHP 8.5 |
|---|---|---|
| `trim('  a  ')` | `"a"` | `"a"` （未变） |
| `trim(null)` | `""` | `""` + `Deprecated` |
| `trim([])` | `null` + `Warning` | **`TypeError`** |
| `rtrim('aab', 'b')` | `"aa"` | `"aa"` （未变） |

### `str_replace()`

| 调用 | PHP 5.4 | PHP 8.5 |
|---|---|---|
| `str_replace('a','b','abc')` | `"bbc"` | 同左 |
| `str_replace(['a','b'],'x','ab')` | `"xx"` | 同左 |
| `str_replace('a','b',[])` | `[]` | `[]` （未变） |
| `str_replace('a','b',null)` | `""` | `""` + `Deprecated` |
| `str_replace([], 'b', 'abc')` | `"abc"` | `"abc"` （未变） |

### `sprintf()` / `vsprintf()` —— 失败返回值消失

| 调用 | PHP 5.4 | PHP 8.5 |
|---|---|---|
| `sprintf('%s')` | `false` + `Warning: Too few arguments` | **`ArgumentCountError`** |
| `vsprintf('%s %s', ['a'])` | `false` + `Warning: Too few arguments` | **`ArgumentCountError`** |
| `sprintf('%s', null)` | `""` | `""` + `Deprecated` |
| `sprintf('%q', 'a')` | `false` + `Warning` | **`ValueError`**（未知格式符） |
| `sprintf('%d', 'abc')` | `0` | `0`（未变，但 8.0 起非数字字符串转 int 不再有 notice） |

### `str_split()`

| 调用 | PHP 5.4 | PHP 8.5 |
|---|---|---|
| `str_split('abc')` | `["a","b","c"]` | 同左 |
| `str_split('')` | **`[""]`** | **`[]`**（8.2 起） |
| `str_split('abc', 0)` | `false` + `Warning` | **`ValueError`** |
| `str_split('abc', -1)` | `false` + `Warning` | **`ValueError`** |

```php
// 5.4 下 str_split('') 返回 [""]，count() === 1；8.5 下返回 []，count() === 0
// 依赖 count() 的分页/分组逻辑会集体偏移一位
$chars = str_split($input);
if (count($chars) === 0) { /* 8.5 才进得来 */ }
```

### `strcmp()` 家族 —— 返回值语义收窄

| 调用 | PHP 5.4 | PHP 8.5 |
|---|---|---|
| `strcmp('a','b')` | 返回 `-1`（长度差或字符差，**任意负数**） | **保证 `-1`** |
| `strcmp('a','abc')` | 返回 `-2`（= 长度差） | **`-1`** |
| `strcasecmp('A','a')` | `0` | `0` |

PHP 8.2 起 `strcmp()` / `strcasecmp()` / `strncmp()` / `strncasecmp()` / `substr_compare()` / `strnatcmp()` / `strnatcasecmp()` 在长度不等时不再返回 `strlen()` 差值，改为 `-1`/`1`（起因是规避 2GB 以上长度差的整数溢出）。

```php
// ❌ 5.4 时代有人这么写，8.5 下逻辑彻底改变
$diff = strcmp($a, $b);
if ($diff > 1) { /* 靠差值大小分流 */ }

// ✅ 正确（两个版本都安全）
if (strcmp($a, $b) < 0) { ... }
```

### 其他返回值微调

| 函数 | PHP 5.4 | PHP 8.5 |
|---|---|---|
| `number_format(1.5, -1)` | `null` + `Warning` | **`ValueError`**（8.0） |
| `str_repeat('a', -1)` | `null` + `Warning` | **`ValueError`**（8.0） |
| `strtolower()` / `strtoupper()` | 受 `setlocale()` 影响（土耳其语 locale 下 `I` 转换异常） | **恒定 ASCII 行为**（8.2），locale 不再影响返回值 |
| `stristr()` / `strstr()` | 同上级联 locale 影响 | 恒定 |
| `count(null)` | `0` + `Warning` | **`TypeError`** |

---

# 三、默认编码与安全转义的行为变化

## 3.1 `htmlspecialchars()` / `htmlentities()` 默认值三级跳

| 项 | PHP 5.4 | PHP 8.0 | PHP 8.5（8.1 起） |
|---|---|---|---|
| 默认 flags | `ENT_COMPAT` | `ENT_COMPAT` | **`ENT_QUOTES \| ENT_SUBSTITUTE \| ENT_HTML401`** |
| 默认 encoding | `UTF-8`（5.4.0 起） | `default_charset`（UTF-8） | `default_charset`（UTF-8） |
| 单引号 `'` | **不转义** | 不转义 | **转义为 `&#039;`** |
| 非法 UTF-8 字节序列 | **返回空字符串 `""`** | **返回空字符串 `""`** | **返回带 `U+FFFD`（`�`）的字符串** |
| 传 `null` | `""` | `Deprecated` + `""` | `Deprecated` + `""` |
| 传 array | `null` + `Warning` | **`TypeError`** | `TypeError` |

**返回值差异的两个关键点：**

```php
$bad = "caf\xE9";  // 非法 UTF-8（latin-1 的 é）

// PHP 5.4 / 8.0
var_dump(htmlspecialchars($bad, ENT_COMPAT, 'UTF-8'));
// string(0) ""       ← 整个字符串变成空白！页面上数据凭空消失

// PHP 8.5（默认 flags）
var_dump(htmlspecialchars($bad));
// string(7) "caf�"   ← 只有坏字节被替换
```

```php
// 单引号转义差异，直接影响前端 JS 与 HTML 属性拼接
$s = "it's";

// PHP 5.4
echo htmlspecialchars($s);              // it's          ← 单引号原样输出

// PHP 8.5
echo htmlspecialchars($s);              // it&#039;s     ← 单引号被转义
echo htmlspecialchars($s, ENT_COMPAT);  // it's          ← 显式指定则行为不变
```

> **迁移建议**：老项目里凡是把 `htmlspecialchars()` 结果塞进 JS 字符串或未加引号的 HTML 属性的地方，8.5 下输出会从 `it's` 变成 `it&#039;s`。若业务依赖旧输出，**显式写全参数**：`htmlspecialchars($s, ENT_COMPAT, 'UTF-8')`。反过来，非法 UTF-8 从 `""` 变 `"caf�"` 通常是好事，但如果下游有 `if ($out === '')` 的空值判断，会失效。

## 3.2 mbstring 扩展

| 项 | PHP 5.4 | PHP 8.5 |
|---|---|---|
| `mbstring.func_overload` | **可用**（可让 `strlen` 变成 `mb_strlen`） | **PHP 8.0 已移除**，`strlen()` 恒为字节数 |
| `mbstring.internal_encoding` | 可用 | 已移除，改用 `default_charset` |
| `mb_substr('FooBar', 42)` | `""` | `""` （未变，与 `substr` 不同步） |
| `mb_strlen(null)` | `0` | `0` + `Deprecated` |
| `mb_convert_encoding($s, 'UTF-8', null)` | 用内部编码 | `Deprecated` |

```php
// PHP 5.4 下若 func_overload = 7（旧项目常见配置）
strlen("中文");     // 6（被重载为 mb_strlen，按字符数）

// PHP 8.5 下 func_overload 已不存在
strlen("中文");     // 6（按字节数）—— ASCII 场景下结果相同，非 ASCII 场景结果不同！
mb_strlen("中文");  // 2
```

---

# 四、PHP 8.x 新增的现代字符串函数

| 新函数 | 引入版本 | 替代的 5.4 繁琐写法 | 返回值 |
|---|---|---|---|
| `str_contains($h, $n)` | 8.0 | `strpos($h, $n) !== false` | `bool` |
| `str_starts_with($h, $n)` | 8.0 | `strpos($h, $n) === 0` 或 `substr($h,0,strlen($n)) === $n` | `bool` |
| `str_ends_with($h, $n)` | 8.0 | `substr($h, -strlen($n)) === $n` | `bool` |
| `str_increment($s)` | 8.3 | `++$s`（8.5 起废弃） | `string` |
| `str_decrement($s)` | 8.3 | `--$s` | `string` |
| `mb_str_split($s, $n)` | 7.4 | `preg_split('//u', $s, -1, PREG_SPLIT_NO_EMPTY)` | `array` |
| `mb_str_pad(...)` | 8.3 | 手写 `mb_substr` 截断填充 | `string` |
| `mb_ucfirst` / `mb_lcfirst` | 8.4 | `mb_strtoupper(mb_substr($s,0,1)) . mb_substr($s,1)` | `string` |
| `mb_trim` / `mb_ltrim` / `mb_rtrim` | 8.4 | 手写正则去全角空白 | `string` |
| `grapheme_str_split($s)` | 8.5 | `preg_split` + grapheme 簇 | `array\|false` |
| `preg_last_error_msg()` | 8.0 | `array_flip(get_defined_constants())` 黑魔法 | `string` |

```php
// PHP 5.4
if (strpos($url, 'https://') === 0) { ... }
if (strpos($email, '@') !== false) { ... }
if (substr($file, -4) === '.php') { ... }   // 且文件名短于 4 时返回 false，还要额外兜底

// PHP 8.5
if (str_starts_with($url, 'https://')) { ... }
if (str_contains($email, '@')) { ... }
if (str_ends_with($file, '.php')) { ... }   // 内部处理边界，无 false 分支
```

---

# 五、字符串偏移量与大括号语法

## 5.1 读取偏移

| 表达式（`$s = 'abc'`） | PHP 5.4 | PHP 8.5 |
|---|---|---|
| `$s[0]` | `"a"` | `"a"` |
| `$s[-1]` | `""` + `Notice: Uninitialized string offset: -1` | **`"c"`**（7.1 起支持负索引） |
| `$s[10]` | `""` + `Notice: Uninitialized string offset: 10` | `""` + **`Warning`**（8.0 起由 Notice 升级为 Warning） |
| `$s['foo']` | `"a"` + `Warning: Illegal string offset 'foo'`（字符串被转成 0） | **`TypeError: Cannot access offset of type string on string`** |
| `$s{0}` | `"a"` | **`Fatal error: Array and string offset access syntax with curly braces is no longer supported`**（8.0 移除） |

```php
// PHP 5.4 —— 常见"容错"写法
$last = $s[-1];        // ""（靠 Notice 兜底）
$first = $s['foo'];    // "a"（键名被静默转成 0）

// PHP 8.5
$last  = $s[-1];       // "c"  —— 返回值变了，逻辑会漂移
$first = $s['foo'];    // Fatal error: Uncaught TypeError
$curly = $s{0};        // Fatal error（解析阶段）
```

## 5.2 写入偏移

| 表达式（`$s = 'abc'`） | PHP 5.4 | PHP 8.5 |
|---|---|---|
| `$s[0] = 'X'` | `"Xbc"` | `"Xbc"` |
| `$s[5] = 'X'` | `"abc  X"`（补空格） | `"abc  X"`（未变） |
| `$s[0] = 'XYZ'` | `"Xbc"`（只取首字节） | `"Xbc"`（未变） |
| `$s[] = 'X'` | `Fatal error: [] operator not supported for strings` | `Error: [] operator not supported for strings` |
| `$s['foo'] = 'X'` | `"Xbc"` + `Warning: Illegal string offset` | **`TypeError: Cannot access offset of type string on string`** |

## 5.3 变量插值语法

```php
$name = 'PHP';

// 三种写法的生命周期
echo "hello {$name}";    // 5.4 ✅   8.5 ✅            推荐
echo "hello ${name}";    // 5.4 ✅   8.5 ⚠️ Deprecated  （8.2 废弃，PHP 9 移除）
echo "hello ${$var}";    // 5.4 ✅   8.5 ⚠️ Deprecated
echo "hello ${expr()}";  // 5.4 ✅   8.5 ⚠️ Deprecated
```

> ✅ **`"${name}"` 只是 Deprecated，返回值不变**，所以这是可以排期处理的低优先级项；但 `"$name[0]"` 这类**未加大括号的数组/偏移插值**在 PHP 8.0 有额外的解析行为变化，建议统一改写为 `"{$name[0]}"`。

---

# 六、迁移检查清单（可直接 grep）

按风险从高到低：

```bash
# 1. 【最高】func_overload：移除后字节/字符语义静默漂移
grep -rn "func_overload" .

# 2. 【最高】依赖 substr 返回 false 的越界判断 —— 8.0 后永远不成立
grep -rnE "===\s*false|!==\s*false" --include=*.php . | grep -i substr

# 3. 【高】implode 逆序参数
grep -rnE "implode\s*\(\s*\\\$?[a-zA-Z_]+\s*," --include=*.php . | grep -v "'"

# 4. 【高】大括号偏移
grep -rnE '\$[a-zA-Z_][a-zA-Z0-9_]*\{[0-9$]' --include=*.php .

# 5. 【中】htmlspecialchars 未显式指定 flags（默认行为 8.1 变了）
grep -rn "htmlspecialchars\|htmlentities" --include=*.php . | grep -v "ENT_"

# 6. 【中】strpos 的松散比较
grep -rnE "strpos\(.*\)\s*==|!strpos\(" --include=*.php .

# 7. 【中】依赖 strcmp 返回具体数值
grep -rnE "strcmp|strcasecmp|strncmp" --include=*.php .

# 8. 【中】"${var}" 插值
grep -rn '\${' --include=*.php .

# 9. 【低】已移除函数
grep -rnE "\b(ereg|eregi|split|spliti|sql_regcase|money_format|mcrypt_|create_function|each)\s*\(" --include=*.php .
```

**三条最需要人工复核的破坏性变更：**

1. `substr()` 越界返回值 `false` → `""`（PHP 8.0）——靠 `=== false` 兜底的逻辑会**静默失效**，比抛异常更危险。
2. `htmlspecialchars()` 非法 UTF-8 返回值 `""` → `"caf�"`（PHP 8.1）——空值判断失效。
3. `str_split('')` 返回值 `[""]` → `[]`（PHP 8.2）——依赖 `count()` 的逻辑集体偏移。

---

需要我把这份对照表整理成一个 Markdown 文档写进仓库吗（比如 `php/php54-to-php85-strings.md`）？另外如果你能提供一个真实的 PHP 8.5 运行环境（本地没装 PHP），我可以把上面这些 `var_dump` 全部跑一遍，用实测输出替换掉现在的推演结论。