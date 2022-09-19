# Part 5：语句

是时候在我们语言的语法中添加一些“适当的”语句了。我希望能够写出这样的代码行:

```
print 2 + 3 * 5;
print 18 - 6/3 + 4*2;
```

当然，由于我们忽略了空格，所以没有必要将一条语句的所有标记都放在同一行上。每个语句以关键字  `print`  开始，以分号结束。所以这些将成为我们语言中的新符号。

## BNF 语法描述

我们已经看到了表达式的 BNF 表示法。现在，让我们为上述类型的语句定义 BNF 语法：

```
statements: statement
     | statement statements
     ;

statement: 'print' expression ';'
     ;
```

输入文件由几个语句组成。它们要么是一个语句，要么是一个语句后面跟着多个语句。每个语句都以关键字 `print` 开始，然后是一个表达式，然后是一个分号。

## 词法扫描器的更改

在开始解析上述语法的代码之前，我们需要向现有的代码中添加一些东西。让我们从词汇扫描器开始。

添加分号标记很容易。现在是 `print` 关键字。稍后，我们将在语言中添加许多关键字，以及变量的标识符，因此我们需要添加一些代码来帮助我们处理它们。

在 `scan.c` 中，我添加了从 SubC 编译器中借来的代码。它将字母数字字符读入缓冲区，直到碰到非字母数字字符为止。

```c
// Scan an identifier from the input file and
// store it in buf[]. Return the identifier's length
static int scanident(int c, char *buf, int lim) {
  int i = 0;

  // Allow digits, alpha and underscores
  while (isalpha(c) || isdigit(c) || '_' == c) {
    // Error if we hit the identifier length limit,
    // else append to buf[] and get next character
    if (lim - 1 == i) {
      printf("identifier too long on line %d\n", Line);
      exit(1);
    } else if (i < lim - 1) {
      buf[i++] = c;
    }
    c = next();
  }
  // We hit a non-valid character, put it back.
  // NUL-terminate the buf[] and return the length
  putback(c);
  buf[i] = '\0';
  return (i);
}
```

我们还需要一个函数来识别语言中的关键字。一种方法是有一个关键字列表，遍历列表并使用 `strcmp()` 将每个关键字都与  `scanident()` 的缓冲区 `buf` 进行比较 。来自 `SubC` 的代码有一个优化：在执行 `strcmp()` 之前匹配第一个字母。这加快了对数十个关键字的比较。现在我们不需要这种优化，但我把它放在后面：

```c
// Given a word from the input, return the matching
// keyword token number or 0 if it's not a keyword.
// Switch on the first letter so that we don't have
// to waste time strcmp()ing against all the keywords.
static int keyword(char *s) {
  switch (*s) {
    case 'p':
      if (!strcmp(s, "print"))
        return (T_PRINT);
      break;
  }
  return (0);
}
```

现在，在 `scan()` 中的 switch 语句的底部，我们添加了这段代码来识别分号和关键字：

```c
    case ';':
      t->token = T_SEMI;
      break;
    default:

      // If it's a digit, scan the
      // literal integer value in
      if (isdigit(c)) {
        t->intvalue = scanint(c);
        t->token = T_INTLIT;
        break;
      } else if (isalpha(c) || '_' == c) {
        // Read in a keyword or identifier
        scanident(c, Text, TEXTLEN);

        // If it's a recognised keyword, return that token
        if (tokentype = keyword(Text)) {
          t->token = tokentype;
          break;
        }
        // Not a recognised keyword, so an error for now
        printf("Unrecognised symbol %s on line %d\n", Text, Line);
        exit(1);
      }
      // The character isn't part of any recognised token, error
      printf("Unrecognised character %c on line %d\n", c, Line);
      exit(1);
```

我还添加了一个全局 `Text` 缓冲区来存储关键字和标识符：

```c
#define TEXTLEN         512             // Length of symbols in input
extern_ char Text[TEXTLEN + 1];         // Last identifier scanned
```

## 表达式解析器的更改

到目前为止，我们的输入文件只包含一个表达式；因此，在 `binexpr()` （在 `expr.c`）中的 Pratt 解析器代码中，我们使用以下代码退出解析器：

```c
  // If no tokens left, return just the left node
  tokentype = Token.token;
  if (tokentype == T_EOF)
    return (left);
```

在我们的新语法中，每个表达式都以分号结束。因此，我们需要更改表达式解析器中的代码，以发现`T_SEMI` 标记并退出表达式解析:

```c
// Return an AST tree whose root is a binary operator.
// Parameter ptp is the previous token's precedence.
struct ASTnode *binexpr(int ptp) {
  struct ASTnode *left, *right;
  int tokentype;

  // Get the integer literal on the left.
  // Fetch the next token at the same time.
  left = primary();

  // If we hit a semicolon, return just the left node
  tokentype = Token.token;
  if (tokentype == T_SEMI)
    return (left);

    while (op_precedence(tokentype) > ptp) {
      ...

          // Update the details of the current token.
    // If we hit a semicolon, return just the left node
    tokentype = Token.token;
    if (tokentype == T_SEMI)
      return (left);
    }
}
```

## 代码生成器的更改

我想让 `gen.c` 中的通用代码生成器与 `cg.c` 中的 CPU 特定代码分开。这也意味着编译器的其余部分应该只调用 `gen.c` 的函数，只有 `gen.c` 应该调用 `cg.c` 的代码。

为此，我在 `gen.c` 中定义了一些新的“前端”功能：

```c
void genpreamble()        { cgpreamble(); }
void genpostamble()       { cgpostamble(); }
void genfreeregs()        { freeall_registers(); }
void genprintint(int reg) { cgprintint(reg); }
```

## 添加语句解析器

我们有一个新文件 `stmt.c`。它将保存我们语言中所有主要语句的解析代码。现在，我们需要解析BNF 语法以获取我在上面放弃的语句。这是用这个函数完成的。我已将递归定义转换为循环：

```c
// Parse one or more statements
void statements(void) {
  struct ASTnode *tree;
  int reg;

  while (1) {
    // Match a 'print' as the first token
    match(T_PRINT, "print");

    // Parse the following expression and
    // generate the assembly code
    tree = binexpr(0);
    reg = genAST(tree);
    genprintint(reg);
    genfreeregs();

    // Match the following semicolon
    // and stop if we are at EOF
    semi();
    if (Token.token == T_EOF)
      return;
  }
}
```

在每个循环中，代码都会找到一个 `T_PRINT` 标记。然后它调用  `binexpr()` 来解析表达式。最后，它找到 `T_SEMI` 标记。如果后面跟着一个 `T_EOF` 标记，则跳出循环。

在每个表达式树之后，调用 `gen.c` 中的代码将树转换为汇编代码，并调用汇编 `printint()` 函数打印出最终值。

## 一些辅助函数

上面的代码中有几个新的辅助函数，我已经把它们放到了一个新文件 `misc.c`  中：

```c
// Ensure that the current token is t,
// and fetch the next token. Otherwise
// throw an error 
void match(int t, char *what) {
  if (Token.token == t) {
    scan(&Token);
  } else {
    printf("%s expected on line %d\n", what, Line);
    exit(1);
  }
}

// Match a semicon and fetch the next token
void semi(void) {
  match(T_SEMI, ";");
}
```

这些构成了解析器中语法检查的一部分。稍后，我将添加更多的短函数来调用 `match()`，以使我们的语法检查更容易。

## 修改 main ()

`main()` 用于直接调用 `binexpr()` 来解析旧输入文件中的单个表达式。现在它这样做：

```  scan(&Token);                 // Get the first token from the input
  genpreamble();                // Output the preamble
  statements();                 // Parse the statements in the input
  genpostamble();               // Output the postamble
  fclose(Outfile);              // Close the output file and exit
  exit(0);
```

## 尝试一下

以上就是新代码和更改代码的全部内容。让我们试一下新代码。下面是新的输入文件 `input01`：

```c
print 12 * 3;
print 
   18 - 2
      * 4; print
1 + 2 +
  9 - 5/2 + 3*5;
```

是的，我决定检查一下我们是否有分散在多行上的 token。要编译并运行输入文件，执行 `make test`:

```
$ make test
cc -o comp1 -g cg.c expr.c gen.c main.c misc.c scan.c stmt.c tree.c
./comp1 input01
cc -o out out.s
./out
36
10
25
```

它有效！

## 总结与下一步

我们在语言中添加了第一个“真正的”语句语法。我用 BNF 表示法定义了它，但用循环实现它更容易，而不是递归。不用担心，我们很快就会回到递归解析。

在此过程中，我们必须修改扫描程序，添加对关键字和标识符的支持，并更清晰地分离通用代码生成器和 CPU 特定生成器。

在编译器编写旅程的下一部分，我们将向语言中添加变量。这将需要大量的工作。