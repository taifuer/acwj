# Part 23：局部变量

我刚刚在堆栈上实现了局部变量，遵循了我在编译器编写过程的前一部分中描述的设计思想，一切都很顺利。下面，我将介绍实际的代码更改。

## 符号表更改

我们从符号表的更改开始，因为这些更改是具有两个变量作用域的核心：全局和局部。符号表条目的结构现在是（`defs.h`）：

```c
// Storage classes
enum {
        C_GLOBAL = 1,           // Globally visible symbol
        C_LOCAL                 // Locally visible symbol
};

// Symbol table structure
struct symtable {
  char *name;                   // Name of a symbol
  int type;                     // Primitive type for the symbol
  int stype;                    // Structural type for the symbol
  int class;                    // Storage class for the symbol
  int endlabel;                 // For functions, the end label
  int size;                     // Number of elements in the symbol
  int posn;                     // For locals,the negative offset 
                                // from the stack base pointer
};
```

添加了`class`和`posn`字段。如上一部分所述，`posn`为负，并保持与堆栈基指针的偏移，即局部变量存储在堆栈上。在这一部分中，我只实现了局部变量，没有参数。还要注意，我们现在有标记为 C_GLOBAL 或 C_LOCAL 的符号。

符号表的名称也已更改，其中的索引也已更改（在`data.h`中）：

```c
extern_ struct symtable Symtable[NSYMBOLS];     // Global symbol table
extern_ int Globs;                              // Position of next free global symbol slot
extern_ int Locls;                              // Position of next free local symbol slot
```

从视觉上看，全局符号存储在符号表的左侧，`Globs` 指向下一个可用的全局符号槽，`Locls` 指向下个可用的局部符号槽。

```c
0xxxx......................................xxxxxxxxxxxxNSYMBOLS-1
     ^                                    ^
     |                                    |
   Globs                                Locls
```

在`sym.c`以及现有的`findglob()`和`newglob()`函数中查找或分配一个全局符号，我们现在有了`findlocl()`和`newlocl()`。他们有代码来检测 `Globs` 和 `Locls` 之间的冲突：

```c
// Get the position of a new global symbol slot, or die
// if we've run out of positions.
static int newglob(void) {
  int p;

  if ((p = Globs++) >= Locls)
    fatal("Too many global symbols");
  return (p);
}

// Get the position of a new local symbol slot, or die
// if we've run out of positions.
static int newlocl(void) {
  int p;

  if ((p = Locls--) <= Globs)
    fatal("Too many local symbols");
  return (p);
}
```

现在有一个通用函数`updatesym()`来设置符号表条目中的所有字段。我不会给出代码，因为它只是一次设置一个字段。

`updatesym()`函数由`addglobl()`和`addlocl()`调用。这些函数首先试图找到一个现有的符号，如果找不到就分配一个新的符号，然后调用`updatesym()`来设置这个符号的值。最后，有一个新函数`findsymbol()`，它在符号表的局部和全局部分中搜索符号：

```c
// Determine if the symbol s is in the symbol table.
// Return its slot position or -1 if not found.
int findsymbol(char *s) {
  int slot;

  slot = findlocl(s);
  if (slot == -1)
    slot = findglob(s);
  return (slot);
}
```

在余下的代码中，对`findglob()`的旧调用已被`findsymbol()`替换。

## 对声明解析的更改

我们需要能够解析全局和局部变量的声明。解析它们的代码（目前）是相同的，所以我给函数添加了一个标志：

```c
void var_declaration(int type, int islocal) {
    ...
      // Add this as a known array
      if (islocal) {
        addlocl(Text, pointer_to(type), S_ARRAY, 0, Token.intvalue);
      } else {
        addglob(Text, pointer_to(type), S_ARRAY, 0, Token.intvalue);
      }
    ...
    // Add this as a known scalar
    if (islocal) {
      addlocl(Text, type, S_VARIABLE, 0, 1);
    } else {
      addglob(Text, type, S_VARIABLE, 0, 1);
    }
    ...
}
```

目前我们的编译器中有两个对`var_declaration()`的调用。`decl.c`中`global_declarations()`中的这个解析全局变量声明：

```c
void global_declarations(void) {
      ...
      // Parse the global variable declaration
      var_declaration(type, 0);
      ...
}
```

`stmt.c`中`single_statement()`中的这个解析局部变量声明：

```c
static struct ASTnode *single_statement(void) {
  int type;

  switch (Token.token) {
    case T_CHAR:
    case T_INT:
    case T_LONG:

      // The beginning of a variable declaration.
      // Parse the type and get the identifier.
      // Then parse the rest of the declaration.
      type = parse_type();
      ident();
      var_declaration(type, 1);
   ...
  }
  ...
}
```

## x86-64 代码生成器的更改

与往常一样，`cg.c`中平台特定代码中的许多`cgXX()`函数作为`gen.c`中的`genXX()`函数暴露给编译器的其余部分。因此，虽然我只提到了`cgXX()`函数，但不要忘记通常还有匹配的`genXX()`函数。

对于每个局部变量，我们需要为它分配一个位置，并将其记录在符号表的`posn`字段中。我们是这样做的。在`cg.c`中，我们有一个新的静态变量和两个函数来操作它：

```c
// Position of next local variable relative to stack base pointer.
// We store the offset as positive to make aligning the stack pointer easier
static int localOffset;
static int stackOffset;

// Reset the position of new local variables when parsing a new function
void cgresetlocals(void) {
  localOffset = 0;
}

// Get the position of the next local variable.
// Use the isparam flag to allocate a parameter (not yet XXX).
int cggetlocaloffset(int type, int isparam) {
  // Decrement the offset by a minimum of 4 bytes
  // and allocate on the stack
  localOffset += (cgprimsize(type) > 4) ? cgprimsize(type) : 4;
  return (-localOffset);
}
```

现在，我们在栈上分配所有的局部变量。它们之间至少有 4 个字节对齐。对于 64 位整数和指针，每个变量是 8 字节。

> 我知道，在过去，多字节数据项必须在内存中正确对齐，否则 CPU 就会出错。看起来，至少对于 x86-64 来说，[不需要对齐数据项](https://lemire.me/blog/2012/05/31/data-alignment-for-speed-myth-or-reality/)。

> 但是，在函数调用之前，x86-64 上的堆栈指针必须正确对齐。在 Agner Fog 的《[汇编语言中的优化子例程](https://www.agner.org/optimize/optimizing_assembly.pdf)》第 30 页中，他指出“在任何调用指令之前，堆栈指针必须对齐 16，因此在函数入口，RSP 的值是 8 模 16。”

> 这意味着，作为函数前导的一部分，我们需要将`%rsp`设置为正确对齐的值。

一旦我们将函数名添加到符号表中，但在开始解析局部变量声明之前，就会在`function_declaration()`中调用`cgresetlocals()`。这会将`localOffset`设置回零。

我们看到用新的局部标量调用了`addlocl()`，或者解析了局部数组。`addlocl()`用新变量的类型调用`cggetlocaloffset()`。这将从堆栈基址指针递减一个适当的偏移量，该偏移量存储在符号的`posn`字段中。

现在我们有了符号从堆栈基址指针的偏移量，我们现在需要修改代码生成器，这样，当我们访问局部变量而不是全局变量时，我们输出一个到`%rbp`的偏移量，而不是命名一个全局位置。

因此，我们现在有了一个`cgloadlocal()`函数，它与`cgloadglob()`几乎相同，只是所有`%s(%%rip)`格式字符串都用来打印`Symtable[id]`。名称被替换为`%d(%%rbp)`格式字符串以打印`Symtable[id].posn`。在`cg.c`中，你会发现所有这些新的局部变量引用。

### 更新堆栈指针

既然我们使用了堆栈上的位置，我们最好把堆栈指针移到保存局部变量的区域下面。因此，我们需要修改函数前同步码和后同步码中的堆栈指针：

```c
// Print out a function preamble
void cgfuncpreamble(int id) {
  char *name = Symtable[id].name;
  cgtextseg();

  // Align the stack pointer to be a multiple of 16
  // less than its previous value
  stackOffset= (localOffset+15) & ~15;
  
  fprintf(Outfile,
          "\t.globl\t%s\n"
          "\t.type\t%s, @function\n"
          "%s:\n" "\tpushq\t%%rbp\n"
          "\tmovq\t%%rsp, %%rbp\n"
          "\taddq\t$%d,%%rsp\n", name, name, name, -stackOffset);
}

// Print out a function postamble
void cgfuncpostamble(int id) {
  cglabel(Symtable[id].endlabel);
  fprintf(Outfile, "\taddq\t$%d,%%rsp\n", stackOffset);
  fputs("\tpopq %rbp\n" "\tret\n", Outfile);
}
```

记住`localOffset`是负数。所以我们在函数前置中加一个负值，在函数后置中加一个负负值。

## 测试

我认为这是向编译器中添加局部变量的主要更改。测试程序`tests/input25.c`演示了局部变量在堆栈上的存储：

```c
int a; int b; int c;

int main()
{
  char z; int y; int x;
  x= 10;  y= 20; z= 30;
  a= 5;   b= 15; c= 25;
}
```

下面是带汇编的汇编输出：

```c
        .data
        .globl  a
a:      .long   0                       # Three global variables
        .globl  b
b:      .long   0
        .globl  c
c:      .long   0

        .text
        .globl  main
        .type   main, @function
main:
        pushq   %rbp
        movq    %rsp, %rbp
        addq    $-16,%rsp               # Lower stack pointer by 16
        movq    $10, %r8
        movl    %r8d, -12(%rbp)         # z is at offset -12
        movq    $20, %r8
        movl    %r8d, -8(%rbp)          # y is at offset -8
        movq    $30, %r8
        movb    %r8b, -4(%rbp)          # x is at offfset -4
        movq    $5, %r8
        movl    %r8d, a(%rip)           # a has a global label
        movq    $15, %r8
        movl    %r8d, b(%rip)           # b has a global label
        movq    $25, %r8
        movl    %r8d, c(%rip)           # c has a global label
        jmp     L1
L1:
        addq    $16,%rsp                # Raise stack pointer by 16
        popq    %rbp
        ret
```

最后，`$make test`证明编译器通过了所有以前的测试。

## 总结与展望

我认为实现局部变量会很棘手，但在对解决方案的设计进行了一些思考之后，结果比我预期的要容易。不知何故，我怀疑下一步将是棘手的一步。

在编译器编写之旅的下一部分，我将尝试向编译器添加函数实参和形参。祝我好运！