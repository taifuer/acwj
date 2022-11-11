# Part 28：添加更多运行时标志

我们编译器编写旅程的这一部分与扫描、解析、语义分析或代码生成没有任何关系。在这一部分中，我向编译器添加了`-c`、`-S`和`-o`运行时标志，以便它的行为更像传统的 Unix C 编译器。

因此，如果这不有趣，请随意跳到旅程的下一部分。

## 编译步骤

到目前为止，我们的编译器只输出汇编文件。但是将高级语言的源代码文件转换为可执行文件还需要更多的步骤：

- 扫描并解析源代码文件以生成汇编输出
- 将汇编输出汇编成[目标文件](https://en.wikipedia.org/wiki/Object_file)
- [链接](https://en.wikipedia.org/wiki/Linker_(computing))一个或多个目标文件以生成可执行文件

我们已经手动或使用 Makefile 完成了最后两步，但是我将修改编译器以调用外部汇编程序和链接程序来执行最后两步。

为此，我将重新组织 `main.c` 中的一些代码，并在 `main.c` 中编写更多的函数来进行汇编和链接。这些代码的大部分是用 C 语言编写的典型的字符串和文件处理代码，所以我将仔细阅读这些代码，但如果你从未见过这类代码，可能会觉得有趣。

## 解析命令行标志

我将编译器重命名为`cwj`，以反映项目的名称。当你在没有命令行参数的情况下运行它时，它现在会给出以下用法消息：

```
$ ./cwj 
Usage: ./cwj [-vcST] [-o outfile] file [file ...]
       -v give verbose output of the compilation stages
       -c generate object files but don't link them
       -S generate assembly files but don't link them
       -T dump the AST trees for each input file
       -o outfile, produce the outfile executable file
```

我们现在允许多个源代码文件作为输入。我们有四个布尔标志，`-v`，`-c`，`-S` 和 `-T`，现在我们可以命名输出的可执行文件。

`main() `中的 `argv[]` 解析代码现在被修改来处理这个问题，并且有几个选项变量来保存结果。

```c
  // Initialise our variables
  O_dumpAST = 0;        // If true, dump the AST trees
  O_keepasm = 0;        // If true, keep any assembly files
  O_assemble = 0;       // If true, assemble the assembly files
  O_dolink = 1;         // If true, link the object files
  O_verbose = 0;        // If true, print info on compilation stages

  // Scan for command-line options
  for (i = 1; i < argc; i++) {
    // No leading '-', stop scanning for options
    if (*argv[i] != '-')
      break;

    // For each option in this argument
    for (int j = 1; (*argv[i] == '-') && argv[i][j]; j++) {
      switch (argv[i][j]) {
      case 'o':
        outfilename = argv[++i]; break;         // Save & skip to next argument
      case 'T':
        O_dumpAST = 1; break;
      case 'c':
        O_assemble = 1; O_keepasm = 0; O_dolink = 0; break;
      case 'S':
        O_keepasm = 1; O_assemble = 0; O_dolink = 0; break;
      case 'v':
        O_verbose = 1; break;
      default:
        usage(argv[0]);
      }
    }
  }
```

请注意，有些选项是互斥的，例如，如果我们只想用`-S`得到汇编输出，那么我们就不想链接或创建目标文件。

## 执行编译阶段

解析完命令行标志后，我们现在可以运行编译阶段了。我们可以很容易地编译和汇编每个输入文件，但是可能有许多目标文件需要在最后链接在一起。所以我们在`main()`中有一些局部变量来存储对象文件名：

```c
#define MAXOBJ 100
  char *objlist[MAXOBJ];        // List of object file names
  int objcnt = 0;               // Position to insert next name
```

我们首先依次处理所有输入源文件：

```c
  // Work on each input file in turn
  while (i < argc) {
    asmfile = do_compile(argv[i]);      // Compile the source file

    if (O_dolink || O_assemble) {
      objfile = do_assemble(asmfile);   // Assemble it to object format
      if (objcnt == (MAXOBJ - 2)) {
        fprintf(stderr, "Too many object files for the compiler to handle\n");
        exit(1);
      }
      objlist[objcnt++] = objfile;      // Add the object file's name
      objlist[objcnt] = NULL;           // to the list of object files
    }

    if (!O_keepasm)                     // Remove the assembly file if
      unlink(asmfile);                  // we don't need to keep it
    i++;
  } 
```

`do_compile()`有以前在`main()`中打开文件、自己解析并生成汇编文件的代码。但是我们不能像以前那样打开硬编码的文件名。我们现在需要将`filename.c`转换为`filename.s`。

## 更改输入文件名

我们有一个辅助函数来改变文件名。

```c
// Given a string with a '.' and at least a 1-character suffix
// after the '.', change the suffix to be the given character.
// Return the new string or NULL if the original string could
// not be modified
char *alter_suffix(char *str, char suffix) {
  char *posn;
  char *newstr;

  // Clone the string
  if ((newstr = strdup(str)) == NULL) return (NULL);

  // Find the '.'
  if ((posn = strrchr(newstr, '.')) == NULL) return (NULL);

  // Ensure there is a suffix
  posn++;
  if (*posn == '\0') return (NULL);

  // Change the suffix and NUL-terminate the string
  *posn++ = suffix; *posn = '\0';
  return (newstr);
}
```

只有`strdup()`、`strrchr()`和最后两行做了真正的工作；剩下的就是错误检查了。

## 进行编译

这是我们曾经拥有的代码，现在被重新打包成一个新的函数。

```c
// Given an input filename, compile that file
// down to assembly code. Return the new file's name
static char *do_compile(char *filename) {
  Outfilename = alter_suffix(filename, 's');
  if (Outfilename == NULL) {
    fprintf(stderr, "Error: %s has no suffix, try .c on the end\n", filename);
    exit(1);
  }
  // Open up the input file
  if ((Infile = fopen(filename, "r")) == NULL) {
    fprintf(stderr, "Unable to open %s: %s\n", filename, strerror(errno));
    exit(1);
  }
  // Create the output file
  if ((Outfile = fopen(Outfilename, "w")) == NULL) {
    fprintf(stderr, "Unable to create %s: %s\n", Outfilename,
            strerror(errno));
    exit(1);
  }

  Line = 1;                     // Reset the scanner
  Putback = '\n';
  clear_symtable();             // Clear the symbol table
  if (O_verbose)
    printf("compiling %s\n", filename);
  scan(&Token);                 // Get the first token from the input
  genpreamble();                // Output the preamble
  global_declarations();        // Parse the global declarations
  genpostamble();               // Output the postamble
  fclose(Outfile);              // Close the output file
  return (Outfilename);
}
```

这里只有很少的新代码，只是调用`alter_suffix()`来获得正确的输出文件名。

有一个重要的变化：汇编输出文件现在是一个名为`Outfilename`的全局变量。这允许`fatal()`函数和`misc.c`中的朋友删除汇编文件，如果我们从来没有完全生成它们，例如：

```c
// Print out fatal messages
void fatal(char *s) {
  fprintf(stderr, "%s on line %d\n", s, Line);
  fclose(Outfile);
  unlink(Outfilename);
  exit(1);
}
```

## 汇编上述输出

现在我们有了汇编输出文件，我们现在可以调用一个外部汇编程序来实现这一点。这在`defs.h`中定义为 ASCMD：

```c
#define ASCMD "as -o "
// Given an input filename, assemble that file
// down to object code. Return the object filename
char *do_assemble(char *filename) {
  char cmd[TEXTLEN];
  int err;

  char *outfilename = alter_suffix(filename, 'o');
  if (outfilename == NULL) {
    fprintf(stderr, "Error: %s has no suffix, try .s on the end\n", filename);
    exit(1);
  }
  // Build the assembly command and run it
  snprintf(cmd, TEXTLEN, "%s %s %s", ASCMD, outfilename, filename);
  if (O_verbose) printf("%s\n", cmd);
  err = system(cmd);
  if (err != 0) { fprintf(stderr, "Assembly of %s failed\n", filename); exit(1); }
  return (outfilename);
}
```

我使用`snprintf()`来构建我们将运行的汇编命令。如果用户使用了`-v`命令行标志，将向他们显示该命令。然后我们使用`system()`来执行这个 Linux 命令。示例：

```
$ ./cwj -v -c tests/input54.c 
compiling tests/input54.c
as -o  tests/input54.o tests/input54.s
```

## 链接目标文件

在`main()`中，我们构建了一个`do_assemble()`返回给我们的目标文件列表：

```c
      objlist[objcnt++] = objfile;      // Add the object file's name
      objlist[objcnt] = NULL;           // to the list of object files
```

因此，当我们需要将它们链接在一起时，我们需要将这个列表传递给`do_link()`函数。代码类似于`do_assemble()`，因为它使用`snprintf()`和`system()`。不同之处在于，我们必须跟踪我们在命令缓冲区中的位置，以及还剩下多少空间来进行更多的`snprintf()`操作。

```c
#define LDCMD "cc -o "
// Given a list of object files and an output filename,
// link all of the object filenames together.
void do_link(char *outfilename, char *objlist[]) {
  int cnt, size = TEXTLEN;
  char cmd[TEXTLEN], *cptr;
  int err;

  // Start with the linker command and the output file
  cptr = cmd;
  cnt = snprintf(cptr, size, "%s %s ", LDCMD, outfilename);
  cptr += cnt; size -= cnt;

  // Now append each object file
  while (*objlist != NULL) {
    cnt = snprintf(cptr, size, "%s ", *objlist);
    cptr += cnt; size -= cnt; objlist++;
  }

  if (O_verbose) printf("%s\n", cmd);
  err = system(cmd);
  if (err != 0) { fprintf(stderr, "Linking failed\n"); exit(1); }
}
```

一个烦恼是我仍然调用外部 C 编译器 cc 来做链接。我们真的应该能够打破对另一个编译器的依赖。

很久以前，手动链接一组目标文件是可能的，例如：

```
  $ ln -o out /lib/crt0.o file1.o file.o /usr/lib/libc.a
```

我认为在当前的 Linux 上执行类似的命令应该是可能的，但是到目前为止我的 Google-fu 还不足以解决这个问题。如果你读到这里，知道答案，请告诉我！

## 丢失`printint()`和`printchar()`

既然我们能够在可以编译的程序中直接调用`printf()`，我们就不再需要手写的`printint()`和`printchar()`函数了。我已经删除了`lib/printint.c`，并更新了`tests/`目录中的所有测试，以使用`printf()`。

我还更新了`tests/mktests`和`tests/runtests`脚本，以便它们使用新的编译器命令行参数，并对顶层 Makefile 进行了同样的操作。所以 `make test` 仍然可以运行我们的回归测试。

## 总结与展望

这就是我们旅程的这一部分。我们的编译器现在感觉像我习惯的传统 Unix 编译器。

我确实承诺在这一步中添加对外部预处理器的支持，但我决定不这么做。主要原因是我需要解析预处理器嵌入到输出中的文件名和行号，例如：

```c
# 1 "tests/input54.c"
# 1 "<built-in>"
# 1 "<command-line>"
# 31 "<command-line>"
# 1 "/usr/include/stdc-predef.h" 1 3 4
# 32 "<command-line>" 2
# 1 "tests/input54.c"
int printf(char *fmt);

int main()
{
  int i;
  for (i=0; i < 20; i++) {
    printf("Hello world, %d\n", i);
  }
  return(0);
}
```

在我们编译器编写旅程的下一部分，我们将着眼于为我们的编译器添加对 structs 的支持。我认为，在我们开始实施更改之前，我们可能必须先进行另一个设计步骤。