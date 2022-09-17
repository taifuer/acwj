# Part 1：词法扫描介绍

> 粗浅的中文翻译版本，如有不当之处，请指出，感谢：）

我们从一个简单的词法扫描器开始我们的编译器编写之旅。正如我在上一部分中提到的，扫描器的工作是识别输入语言中的词汇元素或标记（token）。

我们将从一种只有五个词汇元素的语言开始：

- 四个基本数学运算符：`*`、`/`、`+` 和 `-`
- 具有 `1` 个或多个数字 `0` .. `9` 的十进制整数

我们扫描的每个 token 都将存储在这个结构中（来自 `defs.h`）：

```c
// Token structure
struct token {
  int token;
  int intvalue;
};
```

其中，`token` 字段可以是以下值之一（来自 `defs.h`）：

```c
// Tokens
enum {
  T_PLUS, T_MINUS, T_STAR, T_SLASH, T_INTLIT
};
```

当 token 是 `T_INTLIT`（即整数）时，`intvalue` 字段将保存我们扫描的整数的值。

## `scan.c` 中的函数

`scan.c` 文件保存了词法扫描器的函数。我们将从输入文件中一次读入一个字符。然而，如果在输入流中读得太远，有时会需要“放回”一个字符。我们还希望跟踪当前所在的行，以便在调试消息中打印行号。所有这些都是由 `next()` 函数完成的：

```c
// Get the next character from the input file.
static int next(void) {
  int c;

  if (Putback) {                // Use the character put
    c = Putback;                // back if there is one
    Putback = 0;
    return c;
  }

  c = fgetc(Infile);            // Read from input file
  if ('\n' == c)
    Line++;                     // Increment line count
  return c;
}
```

 `Putback` 和 `Line` 变量，与我们的输入文件指针一起，都被定义在 `data.h` 文件中：

```c
extern_ int     Line;
extern_ int     Putback;
extern_ FILE    *Infile;
```

所有的 C 文件都将包含这个，其中 `extern_` 被替换为 `extern`。但是 `main.c` 会去掉 `extern_`；因此，这些变量将“属于” `main.c`。

最后，我们如何将字符放回输入流中？因此：

```c
// Put back an unwanted character
static void putback(int c) {
  Putback = c;
}
```

## 忽略空格

我们需要一个函数来读取并跳过空白字符，直到它得到一个非空白字符并返回它。因此：

```c
// Skip past input that we don't need to deal with, 
// i.e. whitespace, newlines. Return the first
// character we do need to deal with.
static int skip(void) {
  int c;

  c = next();
  while (' ' == c || '\t' == c || '\n' == c || '\r' == c || '\f' == c) {
    c = next();
  }
  return (c);
}
```

## 扫描 token：scan()

所以现在我们可以在跳过空白的同时读入字符；如果我们读取的字符提前了一个，我们也可以把它放回去。我们现在可以编写第一个词汇扫描器：

```c
// Scan and return the next token found in the input.
// Return 1 if token valid, 0 if no tokens left.
int scan(struct token *t) {
  int c;

  // Skip whitespace
  c = skip();

  // Determine the token based on
  // the input character
  switch (c) {
  case EOF:
    return (0);
  case '+':
    t->token = T_PLUS;
    break;
  case '-':
    t->token = T_MINUS;
    break;
  case '*':
    t->token = T_STAR;
    break;
  case '/':
    t->token = T_SLASH;
    break;
  default:
    // More here soon
  }

  // We found a token
  return (1);
}
```

简单的单字符 token 就是这样：对于每个识别的字符，将其转换为一个 token。你可能会问：为什么不将识别的字符放入 `struct token` 中？答案是，稍后我们将需要识别多字符 token，如 `== ` 和关键字，如 `if` 和 `while`。因此，使用 token 值的枚举列表会更容易。

## 整型字面值

事实上，我们已经不得不面对这种情况，因为我们还需要识别像 `3827` 和 `87731` 这样的整型字面值。下面是 `switch` 语句中缺少的 `default` 代码：

```c
  default:

    // If it's a digit, scan the
    // literal integer value in
    if (isdigit(c)) {
      t->intvalue = scanint(c);
      t->token = T_INTLIT;
      break;
    }

    printf("Unrecognised character %c on line %d\n", c, Line);
    exit(1);
```

一旦我们碰到一个十进制数字字符，我们就从该字符开始调用辅助函数 `scanint()`。它将返回扫描的整数值。要做到这一点，它必须依次读取每个字符，检查它是否是合法数字，并构建最终的数字。下面是代码：

```c
// Scan and return an integer literal
// value from the input file. Store
// the value as a string in Text.
static int scanint(int c) {
  int k, val = 0;

  // Convert each character into an int value
  while ((k = chrpos("0123456789", c)) >= 0) {
    val = val * 10 + k;
    c = next();
  }

  // We hit a non-integer character, put it back.
  putback(c);
  return val;
}
```

例如，如果我们有字符 `3`，`2`，`8`，我们会：

- `val= 0 * 10 + 3`，即 3
- `val= 3 * 10 + 2`，即 32
- `val= 32 * 10 + 8`，即 328

就在最后，你是否注意到了 `putback(c)` 的调用？此时我们发现了一个不是十进制数字的字符。我们不能简单地丢弃它，但幸运的是，我们可以将它放回输入流中，以便稍后使用。

此时您可能会问：为什么直接将 `c` 中减去ASCII值 `0` 使其成为整数？答案是，稍后我们将能够使用`chrpos("0123456789abcdef")` 来转换十六进制数字。

以下是 `chrpos()` 的代码：

```c
// Return the position of character c
// in string s, or -1 if c not found
static int chrpos(char *s, int c) {
  char *p;

  p = strchr(s, c);
  return (p ? p - s : -1);
}
```

这就是现在 `scan.c` 中的词法扫描器代码。

## 让扫描器工作

`main.c` 中的代码让上述扫描器工作。`main()` 函数打开一个文件，然后扫描其中的 token ：

```cpp
void main(int argc, char *argv[]) {
  ...
  init();
  ...
  Infile = fopen(argv[1], "r");
  ...
  scanfile();
  exit(0);
}
```

`scanfile()` 在有新 token 时循环，并打印出 token 的详细信息:

```c
// List of printable tokens
char *tokstr[] = { "+", "-", "*", "/", "intlit" };

// Loop scanning in all the tokens in the input file.
// Print out details of each token found.
static void scanfile() {
  struct token T;

  while (scan(&T)) {
    printf("Token %s", tokstr[T.token]);
    if (T.token == T_INTLIT)
      printf(", value %d", T.intvalue);
    printf("\n");
  }
}
```

## 一些输入文件示例

我提供了一些示例输入文件，这样你就可以看到扫描程序在每个文件中找到了哪些标记，以及扫描程序拒绝了哪些输入文件。

```
$ make
cc -o scanner -g main.c scan.c

$ cat input01
2 + 3 * 5 - 8 / 3

$ ./scanner input01
Token intlit, value 2
Token +
Token intlit, value 3
Token *
Token intlit, value 5
Token -
Token intlit, value 8
Token /
Token intlit, value 3

$ cat input04
23 +
18 -
45.6 * 2
/ 18

$ ./scanner input04
Token intlit, value 23
Token +
Token intlit, value 18
Token -
Token intlit, value 45
Unrecognised character . on line 3
```

## 结论和下一步

我们从小的开始，现在已经有了一个简单的词法扫描器，它可以识别四个主要的数学运算符和整型字面值。我们看到，我们需要跳过空白，并且如果读取的字符超前，那么需要放回字符。

单字符标记很容易扫描，但多字符标记就有点难了。但在最后，`scan()` 函数会在 `struct token` 变量中返回输入文件中的下一个 token：

```c
struct token {
  int token;
  int intvalue;
};
```

在编译器编写过程的下一部分中，我们将构建一个递归下降解析器来解释输入文件的语法，并计算+打印出每个文件的最终值。

