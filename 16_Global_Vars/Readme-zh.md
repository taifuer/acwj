# Part 16：正确声明全局变量

我确实承诺过要考虑给指针添加偏移量的问题，但是我需要先考虑一下这个问题。所以我决定将全局变量声明移出函数声明。实际上，我还保留了函数内部变量声明的解析，因为稍后我们将把它们更改为局部变量声明。

我还想扩展我们的语法，这样我们可以同时声明多个相同类型的变量，例如：

```c
  int x, y, z;
```

## 新 BNF 语法

以下是用于全局声明（函数和变量）的新 BNF 语法：

```
 global_declarations : global_declarations 
      | global_declaration global_declarations
      ;

 global_declaration: function_declaration | var_declaration ;

 function_declaration: type identifier '(' ')' compound_statement   ;

 var_declaration: type identifier_list ';'  ;

 type: type_keyword opt_pointer  ;
 
 type_keyword: 'void' | 'char' | 'int' | 'long'  ;
 
 opt_pointer: <empty> | '*' opt_pointer  ;
 
 identifier_list: identifier | identifier ',' identifier_list ;
```

`function_declaration`  和  `global_declaration`  都以 `type` 开始。这现在是一个`type_keyword `后跟 `opt_pointer,` `opt_pointer` 是零个或多个 '*' 标记。在此之后，`function_declaration` 和 `global_declaration` 后面都必须跟一个标识符。

但是，在 `type` 之后，`var_declaration` 后面跟着一个 `identifier_list`，它是一个或多个由 ',' 标记分隔的标识符。同样，`var_declaration` 必须以 ';' 标记结束，而 `function_declaration` 则以 `compound_statement` 结束，没有 ';' 标记。

## 新标记

现在我们有了 `scan.c` 中 ',' 字符的 T_COMMA 标记。

##  `decl.c` 的修改

现在，我们将上面的 BNF 语法转换为一组递归下降函数，但是，由于我们可以进行循环，我们可以将一些递归转换为内部循环。

### `global_declarations()`

由于有一个或多个全局声明，我们可以循环解析每个声明。当标记用完时，我们可以退出循环。

```c
// Parse one or more global declarations, either
// variables or functions
void global_declarations(void) {
  struct ASTnode *tree;
  int type;

  while (1) {

    // We have to read past the type and identifier
    // to see either a '(' for a function declaration
    // or a ',' or ';' for a variable declaration.
    // Text is filled in by the ident() call.
    type = parse_type();
    ident();
    if (Token.token == T_LPAREN) {

      // Parse the function declaration and
      // generate the assembly code for it
      tree = function_declaration(type);
      genAST(tree, NOREG, 0);
    } else {

      // Parse the global variable declaration
      var_declaration(type);
    }

    // Stop when we have reached EOF
    if (Token.token == T_EOF)
      break;
  }
}
```

现在我们只有全局变量和函数，我们可以扫描这里的类型和第一个标识符。然后，我们看下一个标记。如果是 '('，则调用 `function_declaration()`。如果不是，我们可以假设它是 `var_declaration()`。我们将 `type` 传递给两个函数。

既然我们在这里从 `function_declaration()` 接收了 AST 树，我们可以立即从 AST 树生成代码。这段代码在 `main()` 中，但现在被移到了这里。`main()` 现在只需要调用 `global_declarations()`：

```c
  scan(&Token);                 // Get the first token from the input
  genpreamble();                // Output the preamble
  global_declarations();        // Parse the global declarations
  genpostamble();               // Output the postamble
```

### `var_declaration()`

函数的解析与以前几乎相同，只是扫描类型和标识符的代码在别处完成，我们将 `type` 作为参数接收。

对变量的解析也会丢失类型和标识符扫描代码。我们可以将标识符添加到全局符号中，并为它生成汇编代码。但现在我们需要添加一个循环。如果后面有一个 ','，则返回以获得下一个具有相同类型的标识符。如果后面有一个';'，那就是变量声明的结束。

```c
// Parse the declaration of a list of variables.
// The identifier has been scanned & we have the type
void var_declaration(int type) {
  int id;

  while (1) {
    // Text now has the identifier's name.
    // Add it as a known identifier
    // and generate its space in assembly
    id = addglob(Text, type, S_VARIABLE, 0);
    genglobsym(id);

    // If the next token is a semicolon,
    // skip it and return.
    if (Token.token == T_SEMI) {
      scan(&Token);
      return;
    }
    // If the next token is a comma, skip it,
    // get the identifier and loop back
    if (Token.token == T_COMMA) {
      scan(&Token);
      ident();
      continue;
    }
    fatal("Missing , or ; after identifier");
  }
}
```

## 不完全的局部变量

`var_declaration()` 现在可以解析变量声明列表，但它需要预先扫描类型和第一个标识符。

因此，我在 `stmt.c` 的 `single_statement()` 中保留了对 `var_declaration()` 的调用。稍后，我们将对此进行修改，以声明局部变量。但是现在，这个示例程序中的所有变量都是全局变量：

```c
int   d, f;
int  *e;

int main() {
  int a, b, c;
  b= 3; c= 5; a= b + c * 10;
  printint(a);

  d= 12; printint(d);
  e= &d; f= *e; printint(f);
  return(0);
}
```

## 测试

上面的代码是我们的 `tests/input16.c`。和往常一样，我们可以测试它：

```
$ make test16
cc -o comp1 -g -Wall cg.c decl.c expr.c gen.c main.c misc.c scan.c
      stmt.c sym.c tree.c types.c
./comp1 tests/input16.c
cc -o out out.s lib/printint.c
./out
53
12
12
```

## 总结与展望

在编译器编写过程的下一部分中，我承诺将解决向指针添加偏移量的问题。