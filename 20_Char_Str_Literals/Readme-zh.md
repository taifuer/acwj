# Part 20：字符和字符串字面量

我一直想用我们的编译器打印“Hello world”，所以，现在我们有了指针和数组，是时候在这段旅程中添加字符和字符串字面量了。

当然，这些是字面值。字符字面量的定义是用单引号括起来的单个字符。字符串字面量有一个由双引号包围的字符序列。

现在，说真的，C 中的字符和字符串字面量是完全疯狂的。我只想实现最明显的反斜杠转义字符。我还将借用 SubC 的字符和字符串字面量扫描代码，让我的工作更轻松。

这段旅程将很短，但它将以“Hello world”结束。

## 一个新的标记

我们的语言需要一个新的标记：T_STRLIT。这与 T_IDENT 非常相似，因为与标记关联的文本存储在全局文本中，而不是存储在标记结构本身中。

## 扫描字符字面量

字符字面量以单引号开始，后跟单个字符的定义，并以另一个单引号结束。解释这个字符的代码很复杂，所以让我们修改`scan.c`中的`scan()`来调用它：

```c
      case '\'':
      // If it's a quote, scan in the
      // literal character value and
      // the trailing quote
      t->intvalue = scanch();
      t->token = T_INTLIT;
      if (next() != '\'')
        fatal("Expected '\\'' at end of char literal");
      break;
```

我们可以将字符字面量视为`char`类型的整型字面量；也就是说，假设我们把自己局限于 ASCII，不去尝试处理 Unicode。这就是我在这里做的。

## `scanch()`的代码

`scanch()`函数的代码来自 SubC，经过了一些简化：

```c
// Return the next character from a character
// or string literal
static int scanch(void) {
  int c;

  // Get the next input character and interpret
  // metacharacters that start with a backslash
  c = next();
  if (c == '\\') {
    switch (c = next()) {
      case 'a':  return '\a';
      case 'b':  return '\b';
      case 'f':  return '\f';
      case 'n':  return '\n';
      case 'r':  return '\r';
      case 't':  return '\t';
      case 'v':  return '\v';
      case '\\': return '\\';
      case '"':  return '"' ;
      case '\'': return '\'';
      default:
        fatalc("unknown escape sequence", c);
    }
  }
  return (c);                   // Just an ordinary old character!
}
```

该代码识别大多数转义字符序列，但它不会尝试识别八进制字符编码或其他困难的序列。

## 扫描字符串字面量

字符串以双引号开始，后跟零个或多个字符，并以另一个双引号结束。与字符文字一样，我们需要在`scan()`中调用一个单独的函数：

```c
case '"':
  // Scan in a literal string
  scanstr(Text);
  t->token= T_STRLIT;
  break;
```

我们创建一个新的 T_STRLIT 并将字符串扫描到文本缓冲区中。下面是`scanstr()`的代码:

```c
// Scan in a string literal from the input file,
// and store it in buf[]. Return the length of
// the string. 
static int scanstr(char *buf) {
  int i, c;

  // Loop while we have enough buffer space
  for (i=0; i<TEXTLEN-1; i++) {
    // Get the next char and append to buf
    // Return when we hit the ending double quote
    if ((c = scanch()) == '"') {
      buf[i] = 0;
      return(i);
    }
    buf[i] = c;
  }
  // Ran out of buf[] space
  fatal("String literal too long");
  return(0);
}
```

我认为代码很简单。它终止被扫描的字符串，并确保它不会溢出文本缓冲区。注意，我们使用`scanch()`函数扫描单个字符。

## 解析字符串字面量

正如我提到的，字符字面量被视为整型字面量，我们已经处理过了。哪里可以有字符串字面量？回到 Jeff Lee 在 1985 年写的 C 语言的 BNF 语法，我们看到：

```c
primary_expression
        : IDENTIFIER
        | CONSTANT
        | STRING_LITERAL
        | '(' expression ')'
        ;
```

因此我们知道应该修改`expr.c`中的`primary()`：

```c
// Parse a primary factor and return an
// AST node representing it.
static struct ASTnode *primary(void) {
  struct ASTnode *n;
  int id;


  switch (Token.token) {
  case T_STRLIT:
    // For a STRLIT token, generate the assembly for it.
    // Then make a leaf AST node for it. id is the string's label.
    id= genglobstr(Text);
    n= mkastleaf(A_STRLIT, P_CHARPTR, id);
    break;
```

现在，我要创建一个匿名全局字符串。它需要将字符串中的所有字符存储在内存中，我们还需要一种引用它的方法。我不想用这个新字符串污染符号表，所以我选择为字符串分配一个标签，并将标签的编号存储在这个字符串文本的 AST 节点中。我们还需要一个新的 AST 节点类型：A_STRLIT。标签实际上是字符串中字符数组的基础，因此它应该是 P_CHARPTR 类型。

我很快会回到由`genglobstr()`完成的汇编输出的生成。

### AST 树示例

目前，字符串文字被视为匿名指针。下面是该语句的 AST 树：

```
  char *s;
  s= "Hello world";

  A_STRLIT rval label L2
  A_IDENT s
A_ASSIGN
```

它们都是同一类型，所以没有必要缩放或加宽任何内容。

## 生成汇编输出

在通用代码生成器中，变化很少。我们需要一个函数来生成新字符串的存储。我们需要为它分配一个标签，然后输出字符串的内容（在`gen.c`中）：

```c
int genglobstr(char *strvalue) {
  int l= genlabel();
  cgglobstr(l, strvalue);
  return(l);
}
```

我们需要识别 A_STRLIT AST 节点类型，并为其生成汇编代码。在 `genAST()` 中，

```c
    case A_STRLIT:
        return (cgloadglobstr(n->v.id));
```

## 生成 x86-64 汇编输出

我们最终得到了新的汇编输出函数。有两个：一个生成字符串的存储，另一个加载字符串的基址。

```c
// Generate a global string and its start label
void cgglobstr(int l, char *strvalue) {
  char *cptr;
  cglabel(l);
  for (cptr= strvalue; *cptr; cptr++) {
    fprintf(Outfile, "\t.byte\t%d\n", *cptr);
  }
  fprintf(Outfile, "\t.byte\t0\n");
}

// Given the label number of a global string,
// load its address into a new register
int cgloadglobstr(int id) {
  // Get a new register
  int r = alloc_register();
  fprintf(Outfile, "\tleaq\tL%d(\%%rip), %s\n", id, reglist[r]);
  return (r);
}
```

回到我们的例子：

```c
  char *s;
  s= "Hello world";
```

对应的汇编输出是：

```assembly
L2:     .byte   72              # Anonymous string
        .byte   101
        .byte   108
        .byte   108
        .byte   111
        .byte   32
        .byte   119
        .byte   111
        .byte   114
        .byte   108
        .byte   100
        .byte   0
        ...
        leaq    L2(%rip), %r8   # Load L2's address
        movq    %r8, s(%rip)    # and store in s
```

## 其他修改

在为这段旅程编写测试程序时，我发现了现有代码中的另一个 bug。当缩放一个整数值以匹配指针所指向的类型大小时，我忘记了当比例为 1 时什么也不做。`types.c`中的`modify_type()`中的代码现在是：

```c
    // Left is int type, right is pointer type and the size
    // of the original type is >1: scale the left
    if (inttype(ltype) && ptrtype(rtype)) {
      rsize = genprimsize(value_at(rtype));
      if (rsize > 1)
        return (mkastunary(A_SCALE, rtype, tree, rsize));
      else
        return (tree);          // Size 1, no need to scale
    }
```

我省略了`return(tree)`，因此在尝试缩放`char *`指针时返回一个空树。

## 总结与展望

我很高兴我们现在可以输出文本：

```
$ make test
./comp1 tests/input21.c
cc -o out out.s lib/printint.c
./out
10
Hello world
```

这次的大部分工作是扩展我们的词法扫描器，以处理字符和字符串字面量分隔符以及其中字符的转义。但是在代码生成器上也做了一些工作。

在编译器编写旅程的下一部分，我们将向编译器识别的语言中添加更多的二元运算符。