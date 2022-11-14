# Part 30：设计结构体、联合体和枚举

在我们的编译器编写之旅的这一部分，我将勾勒出我实现结构体（structs）、联合体（unions）和枚举（enums）的设计思路。与函数一样，要实现所有功能，需要采取以下几个步骤。

我还选择将符号表从单个数组重写为几个单链表。我已经提到了我这样做的意图：我关于如何实现复合类型的想法使得此时重写符号表实现非常重要。

在我们开始修改代码之前，让我们看看什么是复合类型。

## 复合类型、Enums 和 Typedefs

在 C 中，[结构体](https://en.wikipedia.org/wiki/Struct_(C_programming_language))和[联合体](https://en.wikipedia.org/wiki/Union_type#C/C++)被称为复合类型。结构体或联合体变量中可以包含许多成员。不同的是，在结构体中，保证成员在内存中不重叠，而在联合体中，我们希望所有成员共享相同的内存位置。

结构体类型的一个示例是：

```c
struct foo {
  int  a;
  int  b;
  char c;
};

struct foo fred;
```

变量`fred`的类型为`struct foo`，它有三个成员 `a`、`b` 和 `c`。我们现在可以对 fred 的三个成员赋值：

```c
  fred.a= 4;
  fred.b= 7;
  fred.c= 'x';
```

并且所有三个值都存储在 `fred` 中的相应成员中。

另一方面，这里是一个联合体类型的示例：

```c
union bar {
  int  a;
  int  b;
  char c;
};

union bar jane;
```

如果我们执行以下语句：

```c
  jane.a= 5;
  printf("%d\n", jane.b);
```

那么当`a`和`b`占据`jane`联合体中的相同内存位置时，将打印值 5。

### Enums

我将在这里讨论枚举，尽管它不像结构体和联合体那样定义复合类型。在 C 语言中，枚举本质上是为整数值命名的一种方法。枚举表示命名整数值的链表。

例如，我们可以定义这些新标识符：

```c
enum { apple=1, banana, carrot, pear=10, peach, mango, papaya };
```

我们现在有这些命名的整数值：

| Name   | Value |
| ------ | ----- |
| apple  | 1     |
| banana | 2     |
| carrot | 3     |
| pear   | 10    |
| peach  | 11    |
| mango  | 12    |
| papaya | 13    |

枚举有一些我不知道的有趣问题，我将在下面介绍。

### Typedefs

此时我还应该讨论 typedefs，尽管我不需要实现它们来让编译器编译自己。[typedef](https://en.wikipedia.org/wiki/Typedef) 是为现有类型提供另一个名称的方法。它通常用于使命名结构体和联合体更容易。

使用前面的示例，我们可以写：

```c
typedef struct foo Husk;
Husk kim;
```

`kim`是`Husk`类型，这与`kim`是`struct foo`类型的说法相同。

## 类型对符号？

那么，如果结构体、联合体和 typedefs 是新类型，它们与保存变量和函数定义的符号表有什么关系呢？枚举只是整数的名字，同样不是变量或函数。

事实是，所有这些东西都有名称：结构体或联合体的名称、成员的名称、成员的类型、枚举值的名称以及 typedefs 的名称。

我们需要将这些名称存储在某个地方，并且我们需要能够找到它们。对于结构体/联合体成员，我们需要找到它们的底层类型。对于枚举名称，我们需要查找它们的整数文字值。

这就是为什么我要用符号表来存储所有这些东西。但是，我们需要将表格分成几个特定的链表，这样我们就可以找到特定的内容，避免找到我们不想找到的内容。

## 重新设计符号表结构

首先，让我们来看看:

- 全局变量和函数的单链表
- 当前函数局部变量的单链表
- 当前函数局部参数的单链表

使用旧的基于数组的符号表，我们在搜索全局变量和函数时必须跳过函数形参。所以，让我们也有一个函数形参的单独方向的链表：

```c
struct symtable {
  char *name;                   // Name of a symbol
  int stype;                    // Structural type for the symbol
  ...
  struct symtable *next;        // Next symbol in one list
  struct symtable *member;      // First parameter of a function
};
```

让我们以图形方式看一下，下面的代码片段会是什么样子：

```c
  int a;
  char b;
  void func1(int x, int y);
  void main(int argc, char **argv) {
    int loc1;
    int loc2;
  }
```

这将存储在三个符号表链表中，如下所示：

![](./Figs/newsymlists.png)

注意，我们有三个链表“头”，它们指向这三个链表。我们现在可以遍历全局符号链表，而不必跳过参数，因为每个函数都将其参数保存在自己的链表中。

当需要解析函数体时，我们可以将参数链表指向函数的参数链表。然后，当局部变量被声明时，它们被简单地追加到局部变量链表中。

然后，一旦解析了函数体并生成了它的汇编代码，我们就可以将参数和局部链表重新设置为空，而不会扰乱全局可见函数中的参数链表。这就是我重写符号表的地方。但是它没有说明我们如何实现结构体、联合体和枚举。

## 有趣的问题和考虑

在我们了解如何扩充现有的符号表节点和单链表以支持结构体、联合体和枚举之前，我们首先必须考虑一些更有趣的问题。

### 联合体

我们从联合体开始。首先，我们可以将一个联合体放入一个结构体中。其次，联合体不需要名字。第三，不需要在结构体中声明变量来保存联合体。举个例子：

```c
#include <stdio.h>
struct fred {
  int x;
  union {
    int a;
    int b;
  };            // No need to declare a variable of this union type
};

int main() {
  struct fred foo;
  foo.x= 5;
  foo.a= 12;                            // a is treated like a struct member
  foo.b= 13;                            // b is treated like a struct member
  printf("%d %d\n", foo.x, foo.a);      // Print 5 and 13
}
```

我们需要能够支持这一点。匿名联合体（和结构体）将很容易：我们只需将符号表节点中的 `name` 设置为 NULL。但是这个联合体没有变量名：我认为我们可以通过将体的成员名也设置为 NULL 来实现这一点，即：

![](./Figs/structunion1.png)

### Enums

我以前使用过枚举，但我没有真正想过要实现它们。所以我编写了下面的 C 程序，看看我是否可以“打破”枚举：

```c
#include <stdio.h>

enum fred { bill, mary, dennis };
int fred;
int mary;
enum fred { chocolate, spinach, glue };
enum amy { garbage, dennis, flute, amy };
enum fred x;
enum { pie, piano, axe, glyph } y;

int main() {
  x= bill;
  y= pie;
  y= bill;
  x= axe;
  x= y;
  printf("%d %d %ld\n", x, y, sizeof(x));
}
```

这些问题是：

- 我们可以用不同的元素重新声明一个枚举链表吗，例如`enum fred`和`enum fred`？
- 我们可以声明一个和枚举链表同名的变量吗，比如 `fred`？
- 我们可以声明一个与枚举值同名的变量吗，比如 `mary`？
- 我们可以在另一个枚举链表中重用一个枚举值的名称吗，例如 `dennis` 和 `dennis`？
- 我们可以将一个枚举链表中的值赋给一个声明为不同枚举链表的变量吗？
- 我们可以在声明为不同枚举链表的变量之间赋值吗？

下面是`gcc`产生的错误和警告：

```c
z.c:4:5: error: ‘mary’ redeclared as different kind of symbol
 int mary;
     ^~~~
z.c:2:19: note: previous definition of ‘mary’ was here
 enum fred { bill, mary, dennis };
                   ^~~~
z.c:5:6: error: nested redefinition of ‘enum fred’
 enum fred { chocolate, spinach, glue };
      ^~~~
z.c:5:6: error: redeclaration of ‘enum fred’
z.c:2:6: note: originally defined here
 enum fred { bill, mary, dennis };
      ^~~~
z.c:6:21: error: redeclaration of enumerator ‘dennis’
 enum amy { garbage, dennis, flute, amy };
                     ^~~~~~
z.c:2:25: note: previous definition of ‘dennis’ was here
 enum fred { bill, mary, dennis };
                         ^~~~~~
```

将上述程序修改编译几次后，得到的答案是：

- 我们不能重新声明`enum fred`。这似乎是我们需要记住枚举链表名称的唯一地方。
- 我们可以重用枚举链表标识符`fred`作为变量名。
- 我们不能在另一个枚举链表中重用枚举值标识符`mary`，也不能作为变量名。
- 我们可以在任何地方分配枚举值：它们似乎被简单地视为文字整数值的名称。
- 似乎我们也可以用单词`int`替换`enum`和`enum X`作为一个类型。

## 设计考虑

好了，我想我们现在可以开始列出我们想要的了：

- 命名和未命名结构体的链表，其中包含每个结构体中成员的名称以及每个成员的类型详细信息。此外，我们还需要成员相对于结构体“base”的内存偏移量。
- 对于命名和未命名的结构体也是如此，尽管偏移量总是为零。
- 枚举链表名和实际枚举名及其相关值的链表。
- 在符号表中，我们需要非复合类型的现有类型信息，但是如果符号是结构体或联合体，我们还需要指向相关复合类型的指针。
- 假设一个结构体可以有一个指向自身的成员，我们需要能够将成员的类型指向同一个结构体。

## 对符号表节点结构的更改

下面粗体字是我对当前单链表符号表节点的修改：

```
struct symtable {
  char *name;                   // Name of a symbol
  int type;                     // Primitive type for the symbol
  struct symtable *ctype;       // If needed, pointer to the composite type
  int stype;                    // Structural type for the symbol
  int class;                    // Storage class for the symbol
  union {
    int size;                   // Number of elements in the symbol
    int endlabel;               // For functions, the end label
    int intvalue;               // For enum symbols, the associated value
  };
  union {
    int nelems;                 // For functions, # of params
    int posn;                   // For locals, the negative offset
                                // from the stack base pointer
  };
  struct symtable *next;        // Next symbol in one list
  struct symtable *member;      // First member of a function, struct,
};                              // union or enum
```

随着这个新的节点结构，我们将有六个链表：

- 全局变量和函数的单链表
- 当前函数局部变量的单链表
- 当前函数局部参数的单链表
- 已定义的结构体类型的单链表
- 已定义的联合体类型的单链表
- 已定义的枚举名称和枚举值的单链表

## 新符号表节点的用例

让我们看看上面的结构中的每个字段是如何被我上面列举的六个链表使用的。

### 新类型

我们将有两个新的类型，P_STRUCT 和 P_UNION，我将在下面描述。

### 全局变量和函数、参数变量、局部变量

- name：变量或函数的名称。
- type：变量的类型，或函数的返回值，加上 4 位间接级别。
- ctype：如果变量是 P_STRUCT 或 P_UNION，则此字段指向相关单链表中的关联结构体或联合体定义。
- stype：变量或函数的结构体类型：S_VARIABLE、S_FUNCTION 或 S_ARRAY。
- class：变量的存储类：C_GLOBAL、C_LOCAL 或 C_PARAM。
- size：对于变量，以字节为单位的总大小。对于数组，数组中的元素数。稍后我们将使用它来实现`sizeof()`。
- endlabel：对于函数，我们可以`return`的结束标签。
- nelems：对于函数，参数的数量。
- posn：对于局部变量和参数，变量距堆栈基指针的负偏移量。
- next：链表中的下一个符号。
- member：对于函数，指向第一个参数节点的指针。变量为 NULL。

## 结构体类型

- name：结构体类型的名称，如果是匿名的，则为 NULL。
- type：总是 P_STRUCT，不是真正需要的。
- ctype：未使用。
- stype：未使用。
- class：未使用。
- size：结构体的总大小（字节），稍后由 `sizeof()` 使用。
- nelems：结构体中的成员数。
- next：已定义的下一个结构体类型。
- member：指向第一个结构体成员节点的指针。

### 联合体类型

- name：联合类型的名称，如果是匿名的，则为NULL。
- type：总是 P_UNION，不是真正需要的。
- ctype：未使用。
- stype：未使用。
- class：未使用。
- size：联合体的总大小（字节），稍后由 `sizeof()` 使用。
- nelems：联合体中的成员数。
- next：已定义的下一个联合体类型。
- member：指向第一个联合体成员节点的指针。

###  结构体和联合体成员

每个成员本质上都是一个变量，因此与普通变量有很强的相似性。

- name：成员的名称。
- type：变量的类型加上 4 位间接级别。
- ctype：如果成员是 P_STRUCT 或 P_UNION，则此字段指向相关单链链表中的关联结构体或联合体定义。
- stype：成员的结构类型：S_VARIABLE 或 S_ARRAY。
- class：未使用。
- size：对于变量，以字节为单位的总大小。对于数组，数组中的元素数。稍后我们将使用它来实现`sizeof()`。
- posn：成员与结构体/联合体基址的正偏移量。
- next：结构体/联合体中的下一个成员。
- member：空。

### 枚举链表名称和值

我想存储以下所有符号和隐式值：

```c
  enum fred { chocolate, spinach, glue };
  enum amy  { garbage, dennis, flute, couch };
```

我们可以先链接 `fred`，然后链接 `amy`，然后使用 `fred` 中的成员字段作为 `chocolate`、`spinach`、`glue` 链表。`amy` 同上。

然而，我们实际上只关心`fred`和`amy`名称，以防止它们被重用为枚举链表名称。我们真正关心的是实际的枚举名称及其值。

因此，我提出了两个“dummy”类型值：P_ENUMLIST 和 P_ENUMVAL。然后我们只构建一个一维链表，如下所示：

```
     fred  -> chocolate-> spinach ->   glue  ->    amy  -> garbage -> dennis -> ...
  P_ENUMLIST  P_ENUMVAL  P_ENUMVAL  P_ENUMVAL  P_ENUMLIST  P_ENUMVAL  P_ENUMVAL
```

因此，当我们使用`glue`这个词时，我们只需要浏览一个链表。否则，我们必须找到`fred`，遍历`fred`的成员名单，然后对`amy`也是如此。我认为链表会更容易。

## 已更改的内容

在本文档的顶部，我提到了我已经将符号表从单个数组改写为多个单独链接的链表，在 `struct symtable` 节点中添加了以下新字段：

```c
  struct symtable *next;        // Next symbol in one list
  struct symtable *member;      // First parameter of a function
```

所以，让我们快速了解一下这些变化。首先，没有任何功能变化。

### 三个符号表链表

我们现在在`data.h`中有三个符号表链表：

```c
// Symbol table lists
struct symtable *Globhead, *Globtail;   // Global variables and functions
struct symtable *Loclhead, *Locltail;   // Local variables
struct symtable *Parmhead, *Parmtail;   // Local parameters
```

`sym.c`中的所有函数都被重写以使用它们。我已经编写了一个通用函数来添加到一个链表中：

```c
// Append a node to the singly-linked list pointed to by head or tail
void appendsym(struct symtable **head, struct symtable **tail,
               struct symtable *node) {

  // Check for valid pointers
  if (head == NULL || tail == NULL || node == NULL)
    fatal("Either head, tail or node is NULL in appendsym");

  // Append to the list
  if (*tail) {
    (*tail)->next = node; *tail = node;
  } else *head = *tail = node;
  node->next = NULL;
}
```

现在有一个函数`newsym()`，它被赋予符号表节点的所有字段值。它`malloc()`一个新节点，填充它并返回它。这里就不给出代码了。

对于每个链表，都有一个函数来构建一个节点并将其追加到链表中。一个例子是：

```c
// Add a symbol to the global symbol list
struct symtable *addglob(char *name, int type, int stype, int class, int size) {
  struct symtable *sym = newsym(name, type, stype, class, size, 0);
  appendsym(&Globhead, &Globtail, sym);
  return (sym);
}
```

有一个在链表中查找符号的通用函数，其中链表指针是链表的头：

```c
// Search for a symbol in a specific list.
// Return a pointer to the found node or NULL if not found.
static struct symtable *findsyminlist(char *s, struct symtable *list) {
  for (; list != NULL; list = list->next)
    if ((list->name != NULL) && !strcmp(s, list->name))
      return (list);
  return (NULL);
}
```

有三个特定于链表的`findXXX()`函数。

有一个函数`findsymbol()`，它首先尝试在函数的参数链表中查找符号，然后是函数的局部变量，最后是全局变量。

有一个函数`findlocl()`，它只搜索函数的参数链表和局部变量。当我们声明局部变量并且需要防止重声明时，我们使用这个。

最后还有一个函数`clear_symtable()`，将三个链表的头和尾都重置为 NULL，即清空三个链表。

### 参数和本地链表

只有在每个单独的源代码文件被解析后，全局符号链表才会被清除。但是我们需要 a)建立参数链表，b)清除本地符号链表，每次我们开始解析一个新函数的主体。

这就是它的工作原理。当我们在`expr.c`中解析`param_declaration()`中的参数链表时，我们为每个参数调用`var_declaration()`。这将创建一个符号表节点，并将其附加到参数链表，即`Parmhead`和`Parmtail`。当`param_declaration()`返回时，`Parmhead`指向参数链表。

回到正在解析整个函数（它的名称、参数链表和任何函数体）的`function_declaration()`，参数链表被复制到函数的符号表节点：

```c
    newfuncsym->nelems = paramcnt;
    newfuncsym->member = Parmhead;

    // Clear out the parameter list
    Parmhead = Parmtail = NULL;
```

我们通过清空`Parmhead`和`Parmtail`来清除参数链表，如上所示。这意味着所有这些都不再可以通过全局参数链表进行搜索。

解决方案是为函数的符号表条目设置一个全局变量`Functionid`：

```c
  Functionid = newfuncsym;
```

因此，当我们调用`compound_statement()`来解析函数体时，我们仍然可以通过`Functionid->member`使用参数链表来做类似这样的事情：

- 防止声明与参数名匹配的局部变量

- 使用一个参数的名字作为一个正常的局部变量等。

最后，`function_declaration()`将覆盖整个函数的 AST 树返回给`global_declarations()`，然后`global _ declarations()`将它传递给`gen.c`中的`genAST()`以生成汇编代码。而当`genAST()`返回时，`global_declarations()`调用`freeloclsyms()`清除局部和参数链表，并将`Functionid`重置回 NULL。

### 其他值得注意的变化

实际上，由于符号表的几个链表的改变，大量的代码不得不重写。我不打算讲解整个代码库。但是有些东西你很容易就能发现。例如，符号节点曾经用类似`Symtable[n->id]`的代码引用。这就是现在的`n->sym`。

另外，`cg.c`中的很多代码都引用了符号名，所以你现在可以看到这些符号名是`n->sym->name`。类似地，在`tree.c`中转储 AST 树的代码现在有很多`n->sym->name`。

## 总结

我们旅程的这一部分是部分设计和部分重新实现。我们花了很多时间来解决在实现结构体、联合体和枚举时会遇到的问题。然后我们重新设计了符号表来支持这些新概念。最后，我们将符号表重写为三个链表（暂时），为这些新概念的实现做准备。

在我们编译器编写旅程的下一部分，我可能会实现结构体类型的声明，但不会实际编写供它们使用的代码。我将在后面的部分中这样做。完成这两项工作后，我希望能够在第三部分实现联合。然后，在第四部分的枚举。我们拭目以待！