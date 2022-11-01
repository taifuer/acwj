# Part 26：函数原型

在编译器编写过程的这一部分中，我增加了编写函数原型的能力。在这个过程中，我不得不重写我在前面部分中刚刚编写的一些代码；很抱歉。我看得不够远!

我们想用函数原型做什么:

- 能够声明一个没有主体的函数原型
- 以后声明完整函数的能力
- 将原型保存在全局符号表部分，将参数作为局部变量保存在局部符号表部分
- 根据以前的函数原型检查参数的数量和类型的错误

下面是我不会做的事情，至少现在不会:

- `function(void)`：这将与 `function()` 声明相同
- 函数的声明只有类型，例如`function(int,char, long);`，因为这会增加解析的难度。我们可以以后再做。

## 哪些函数需要重写

在我们旅程的最近一部分中，我添加了带有参数和完整函数体的函数声明。当我们解析每个参数时，我立即将其添加到全局符号表（形成原型）和局部符号表（作为函数的参数）。

既然我们想要实现函数原型，那么参数列表不一定会成为实际函数的参数。考虑这个函数原型:

```c
int fred(char a, int foo, long bar);
```

我们只能将`fred`定义为一个函数，将`a`、`foo`和`bar`定义为全局符号表中的三个参数。我们必须等到完整的函数声明之后才能向局部符号表中添加`a`、`foo`和`bar`。

我需要在全局符号表和局部符号表上分离 C_PARAM 项的定义。

## 新的解析机制的设计

下面是我对新函数解析机制的快速设计，它也处理原型。

```
Get the identifier and '('.
Search for the identifier in the symbol table.
If it exists, there is already a prototype: get the id position of
the function and its parammeter count.

While parsing parameters:
  - if a previous prototype, compare this param's type against the existing
    one. Update the symbol's name in case this is a full function
  - if no previous prototype, add the parameter to the symbol table

Ensure # of params matches any existing prototype.
Parse the ')'. If ';' is next, done.

If '{' is next, copy the parameter list from the global symtable to the
local sym table. Copy them in a loop so that they are put in reverse order
in the local sym table.
```

这是我前几个小时完成的，所以这里是代码更改。

## `sym.c `的更改

我更改了`sym.c`中几个函数的参数列表：

```c
int addglob(char *name, int type, int stype, int class, int endlabel, int size);
int addlocl(char *name, int type, int stype, int class, int size);
```

以前，我们让`addlocl()`也调用`addglob()`向两个符号表添加 C_PARAM 符号。现在我们分离了这个函数，将符号的实际类传递给两个函数是有意义的。

在`main.c`和`decl.c`中有对这些函数的调用。稍后我将介绍`decl.c`中的那些。`main.c`中的变化是微不足道的。

一旦击中实函数的声明，就需要将其形参列表从全局复制到局部符号表。由于这是特定于符号表的，所以我将这个函数添加到`sym.c`中：

```c
// Given a function's slot number, copy the global parameters
// from its prototype to be local parameters
void copyfuncparams(int slot) {
  int i, id = slot + 1;

  for (i = 0; i < Symtable[slot].nelems; i++, id++) {
    addlocl(Symtable[id].name, Symtable[id].type, Symtable[id].stype,
            Symtable[id].class, Symtable[id].size);
  }
}
```

## 对`decl.c`的更改

几乎所有对编译器的更改都局限于`decl.c`。我们先从小的开始，再到大的。

### `var_declaration()`

我已经将形参列表更改为`var_declaration()`，方法与我对`symm.c`函数所做的相同：

```c
void var_declaration(int type, int class) {
  ...
  addglob(Text, pointer_to(type), S_ARRAY, class, 0, Token.intvalue);
  ...
  if (addlocl(Text, type, S_VARIABLE, class, 1) == -1)
  ...
  addglob(Text, type, S_VARIABLE, class, 0, 1);
}
```

我们将在其他`decl.c`函数中使用传递类的功能。

### `param_declaration()`

这里有很大的变化，因为我们可能已经在全局符号表中有一个参数列表作为现有的原型。如果我们这样做了，我们需要对照原型检查新列表中的数量和类型。

```c
// Parse the parameters in parentheses after the function name.
// Add them as symbols to the symbol table and return the number
// of parameters. If id is not -1, there is an existing function
// prototype, and the function has this symbol slot number.
static int param_declaration(int id) {
  int type, param_id;
  int orig_paramcnt;
  int paramcnt = 0;

  // Add 1 to id so that it's either zero (no prototype), or
  // it's the position of the zeroth existing parameter in
  // the symbol table
  param_id = id + 1;

  // Get any existing prototype parameter count
  if (param_id)
    orig_paramcnt = Symtable[id].nelems;

  // Loop until the final right parentheses
  while (Token.token != T_RPAREN) {
    // Get the type and identifier
    // and add it to the symbol table
    type = parse_type();
    ident();

    // We have an existing prototype.
    // Check that this type matches the prototype.
    if (param_id) {
      if (type != Symtable[id].type)
        fatald("Type doesn't match prototype for parameter", paramcnt + 1);
      param_id++;
    } else {
      // Add a new parameter to the new prototype
      var_declaration(type, C_PARAM);
    }
    paramcnt++;

    // Must have a ',' or ')' at this point
    switch (Token.token) {
    case T_COMMA:
      scan(&Token);
      break;
    case T_RPAREN:
      break;
    default:
      fatald("Unexpected token in parameter list", Token.token);
    }
  }

  // Check that the number of parameters in this list matches
  // any existing prototype
  if ((id != -1) && (paramcnt != orig_paramcnt))
    fatals("Parameter count mismatch for function", Symtable[id].name);

  // Return the count of parameters
  return (paramcnt);
}
```

记住，第一个形参的全局符号表槽位紧挨着函数名符号的槽位。我们将通过现有原型的槽位，如果没有原型，则为 -1。

我们可以在此基础上添加 1 以获得第一个参数的槽号，或者 0 表示不存在现有的原型，这是一个令人愉快的巧合。

我们仍然循环解析每个新参数，但是现在有了新的代码来与现有的原型进行比较，或者将参数添加到全局符号表中。

一旦我们退出循环，我们可以将这个列表中的参数数量与任何现有原型中的参数数量进行比较。

现在，代码感觉有点丑，我相信如果我把它放一段时间，我将能够看到一个重构它的方法。

### `function_declaration()`

以前，这是一个相当简单的函数：获取类型和名称、添加全局符号、读入参数、获取函数体并为函数代码生成 AST 树。

现在，我们要处理的事实是这可能只是一个原型，也可能是一个完整的函数。在解析';'（用于原型）或'{'（用于完整函数）之前，我们不会知道答案。因此，让我们分阶段阐述代码。

```c
// Parse the declaration of function.
// The identifier has been scanned & we have the type.
struct ASTnode *function_declaration(int type) {
  struct ASTnode *tree, *finalstmt;
  int id;
  int nameslot, endlabel, paramcnt;

  // Text has the identifier's name. If this exists and is a
  // function, get the id. Otherwise, set id to -1
  if ((id = findsymbol(Text)) != -1)
    if (Symtable[id].stype != S_FUNCTION)
      id = -1;

  // If this is a new function declaration, get a
  // label-id for the end label, and add the function
  // to the symbol table,
  if (id == -1) {
    endlabel = genlabel();
    nameslot = addglob(Text, type, S_FUNCTION, C_GLOBAL, endlabel, 0);
  }
  // Scan in the '(', any parameters and the ')'.
  // Pass in any existing function prototype symbol slot number
  lparen();
  paramcnt = param_declaration(id);
  rparen();
```

这与之前版本的代码几乎相同，除了当没有之前的原型时，`id`被设置为-1，当有之前的原型时，`id`被设置为正数。只有在全局符号表中还没有函数名的情况下，才将函数名添加到全局符号表中。

```c
  // If this is a new function declaration, update the
  // function symbol entry with the number of parameters
  if (id == -1)
    Symtable[nameslot].nelems = paramcnt;

  // Declaration ends in a semicolon, only a prototype.
  if (Token.token == T_SEMI) {
    scan(&Token);
    return (NULL);
  }
```

我们得到了参数的计数。如果没有以前的原型，请使用此计数更新此原型。现在我们可以在参数列表结束后查看标记。如果是分号，这只是一个原型。我们现在没有要返回的 AST 树，所以跳过标记并返回 NULL。我不得不稍微修改`global_declarations()`中的代码来处理这个 NULL 值：没有什么大的变化。

如果我们继续下去，我们现在正在处理一个带有主体的完整函数声明。

```c
  // This is not just a prototype.
  // Copy the global parameters to be local parameters
  if (id == -1)
    id = nameslot;
  copyfuncparams(id);
```

我们现在需要将参数从原型复制到本局部符号表。当我们自己刚刚添加了全局符号并且没有以前的原型时，`id＝nameslot` 代码就在那里。

`function_declaration()`中的其余代码与之前相同，我将省略它。它检查非 void 函数是否返回值，并生成具有 A_function 根节点的 AST 树。

## 测试

 `tests/runtests` 脚本的一个缺点是，它假定编译器一定会产生一个可以装配和运行的汇编输出文件`out.s`。这使得我们无法测试编译器是否检测到语法和语义错误。

对`decl.c`的快速 grep 显示这些新的错误被检测到。

```c
fatald("Type doesn't match prototype for parameter", paramcnt + 1);
fatals("Parameter count mismatch for function", Symtable[id].name);
```

因此，我最好重写 `tests/runtests` ，以验证编译器是否在错误输入时检测到这些错误。

我们有两个新的工作测试程序，`input29.c`和`input30.c`。第一个与`input28.c`相同，只是我把所有函数的原型放在了程序的顶部：

```c
int param8(int a, int b, int c, int d, int e, int f, int g, int h);
int fred(int a, int b, int c);
int main();
```

这和所有以前的测试程序仍然有效。然而，`input30.c可`能是我们编译器给出的第一个非平凡程序。它打开自己的源文件并将其打印到标准输出：

```c
int open(char *pathname, int flags);
int read(int fd, char *buf, int count);
int write(int fd, void *buf, int count);
int close(int fd);

char *buf;

int main() {
  int zin;
  int cnt;

  buf= "                                                             ";
  zin = open("input30.c", 0);
  if (zin == -1) {
    return (1);
  }
  while ((cnt = read(zin, buf, 60)) > 0) {
    write(1, buf, cnt);
  }
  close(zin);
  return (0);
}
```

我们还不能调用预处理器，所以我们手动放入`open()`、`read()`、`write()`和`close()`函数的原型。我们还必须在`open()`调用中使用 0 而不是 O_RDONLY。

现在，编译器允许我们声明`char buf[60];`但我们不能用`buf`本身作为 char 指针。所以我选择给 char 指针赋值一个 60 个字符的字符串作为缓冲区。

我们仍然需要用 '{' ... '}' 来包装 IF 和 WHILE 体使它们成为复合语句：我还没有处理悬空 else 问题。最后，我们还不能接受`char *argv[]`作为 main 的参数声明，所以我不得不硬编码输入文件名。

尽管如此，我们现在有一个非常原始的 cat(1) 程序，我们的编译器可以编译它！这就是进步。

## 总结与展望

在编译器编写过程的下一部分中，我将继续上面的注释并改进编译器功能的测试。