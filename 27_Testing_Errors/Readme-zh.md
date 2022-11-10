# Part 27：回归测试和惊喜

我们最近在编译器编写过程中经历了一些重大的修改，所以我们可以在这一部分稍作休息，放慢脚步，回顾我们迄今为止的进展。 

在上一步中，我注意到我们无法确认语法和语义错误检查是否正常工作。因此，我刚刚重写了`tests/`文件夹中的脚本。

我从 20 世纪 80 年代末就开始使用 Unix，所以我的自动化工具是 shell 脚本和 Makefiles，或者，如果我需要更复杂的工具，用 Python 或 Perl 编写的脚本（是的，我很老了）。

因此，让我们快速查看 `tests/` 目录中的 `runtest` 脚本。尽管我说过我一直在使用 Unix 脚本，但我绝对不是一个超级脚本作家。

## `runtest` 脚本

这个脚本的工作是获取一组输入程序，让我们的编译器编译它们，运行可执行文件，并将其输出与已知正确的输出进行比较。如果它们匹配，测试就成功了。如果没有，那就是失败。

我只是扩展了它，这样，如果有一个“错误”文件与一个输入相关联，我们就可以运行我们的编译器并捕获它的错误输出。如果这个错误输出与预期的错误输出相匹配，测试就成功了，因为编译器正确地检测到了错误的输入。

因此，让我们分阶段来看看`runtest`脚本的各个部分。

```bash
# Build our compiler if needed
if [ ! -f ../comp1 ]
then (cd ..; make)
fi
```

我在这里使用 '( ... )' 语法来创建子shell。这可以在不影响原始 shell 的情况下更改其工作目录，因此我们可以向上移动一个目录并重新构建编译器。

```bash
# Try to use each input source file
for i in input*
# We can't do anything if there's no file to test against
do if [ ! -f "out.$i" -a ! -f "err.$i" ]
   then echo "Can't run test on $i, no output file!"
```

'[' 实际上是外部 Unix 工具 test(1)。哦，如果你以前从未见过这种语法，test(1) 意味着 test 的手册页在手册页的第 1 部分，你可以这样做：

```
$ man 1 test
```

阅读手册页第 1 部分的 test 手册。`/usr/bin/[`可执行文件通常链接到`/usr/bin/test`，因此当你在 shell 脚本中使用'['时，它与运行 test 命令相同。

我们可以将 `[ ! -f "out.$i" -a ! -f "err.$i" ]`读作：测试是否没有文件"out.\$1"和文件"
err.\$i"。如果两者都不存在，我们可以给出错误消息。

```bash
   # Output file: compile the source, run it and
   # capture the output, and compare it against
   # the known-good output
   else if [ -f "out.$i" ]
        then
          # Print the test name, compile it
          # with our compiler
          echo -n $i
          ../comp1 $i

          # Assemble the output, run it
          # and get the output in trial.$i
          cc -o out out.s ../lib/printint.c
          ./out > trial.$i

          # Compare this agains the correct output
          cmp -s "out.$i" "trial.$i"

          # If different, announce failure
          # and print out the difference
          if [ "$?" -eq "1" ]
          then echo ": failed"
            diff -c "out.$i" "trial.$i"
            echo

          # No failure, so announce success
          else echo ": OK"
          fi
```

这是脚本的主要部分。我认为这些注释解释了发生的事情，但也许还有一些微妙之处需要补充。`cmp-s`比较两个文本文件；`-s`标志表示不生成输出，但将`cmp`退出时给出的退出值设置为：

> 如果输入相同，则为 0；如果输入不同，则为 1；如果故障，则为 2。（来自手册页）

 `if [ "$?" -eq "1" ]` 表示：是否最后一个命令的输出值等于数字 1。因此，如果编译器的输出与已知的正确输出不同，我们将指出这一点，并使用`diff`工具显示两个文件之间的差异。

```bash
   # Error file: compile the source and
   # capture the error messages. Compare
   # against the known-bad output. Same
   # mechanism as before
   else if [ -f "err.$i" ]
        then
          echo -n $i
          ../comp1 $i 2> "trial.$i"
          cmp -s "err.$i" "trial.$i"
          ...
```

当存在错误文档"err.\$i"时，将执行此部分。这一次，我们使用 shell 语法`2>`将编译器的标准错误输出捕获到文件"trial.\$i"中，并将其与正确的错误输出进行比较。之后的逻辑与前面相同。

## 我们在做什么：回归测试

我以前没怎么谈论过测试，但现在是时候了。我过去教过软件开发，所以在某个时候不涉及测试是我的失职。

我们在这里做的是[回归测试](https://en.wikipedia.org/wiki/Regression_testing)。维基百科给出了这个定义：

> 回归测试是重新运行功能性和非功能性测试的行为，以确保先前开发和测试的软件在发生变化后仍能运行。
>
> / 修改了旧代码后，重新进行测试以确认修改没有引入新的错误或导致其他代码产生错误。

由于编译器在每一步都在变化，我们必须确保每一个新的变化都不会破坏前面步骤的功能（以及错误检查）。因此，每次引入更改时，我都会添加一个或多个测试，以 a）证明它有效，b）在将来的更改中重新运行此测试。只要所有的测试都通过了，我确信新代码没有破坏旧代码。

### 功能性测试

`runtests`脚本查找带有`out`前缀的文件来进行功能性测试。目前，我们有：

```
tests/out.input01.c  tests/out.input12.c   tests/out.input22.c
tests/out.input02.c  tests/out.input13.c   tests/out.input23.c
tests/out.input03.c  tests/out.input14.c   tests/out.input24.c
tests/out.input04.c  tests/out.input15.c   tests/out.input25.c
tests/out.input05.c  tests/out.input16.c   tests/out.input26.c
tests/out.input06.c  tests/out.input17.c   tests/out.input27.c
tests/out.input07.c  tests/out.input18a.c  tests/out.input28.c
tests/out.input08.c  tests/out.input18.c   tests/out.input29.c
tests/out.input09.c  tests/out.input19.c   tests/out.input30.c
tests/out.input10.c  tests/out.input20.c   tests/out.input53.c
tests/out.input11.c  tests/out.input21.c   tests/out.input54.c
```

这是对编译器功能的 33 次独立测试。现在，我知道一个事实，我们的编译器有点脆弱。这些测试没有一个真正以任何方式给编译器带来压力：它们只是简单的测试，每个测试只有几行代码。稍后，我们将开始添加一些令人讨厌的压力测试来帮助增强编译器，使它更有弹性。

### 非功能性测试

`runtests`脚本寻找带有`err`前缀的文件来进行非功能性测试。目前，我们有：

```
tests/err.input31.c  tests/err.input39.c  tests/err.input47.c
tests/err.input32.c  tests/err.input40.c  tests/err.input48.c
tests/err.input33.c  tests/err.input41.c  tests/err.input49.c
tests/err.input34.c  tests/err.input42.c  tests/err.input50.c
tests/err.input35.c  tests/err.input43.c  tests/err.input51.c
tests/err.input36.c  tests/err.input44.c  tests/err.input52.c
tests/err.input37.c  tests/err.input45.c
tests/err.input38.c  tests/err.input46.c
```

在我们旅程的这一步，我通过在编译器中寻找 `fatal()` 调用，创建了这 22 个编译器错误检查测试。对于每一个，我都试图编写一个小的输入文件来触发它。阅读匹配的源文件，看看是否能找出每一个引发的语法或语义错误。

## 其他形式的测试

这不是一门关于软件开发方法的课程，所以我不会给出太多关于测试的内容。但是我会给你一些链接，我强烈建议你看看:

- [单元测试](https://en.wikipedia.org/wiki/Unit_testing)
- [测试驱动开发](https://en.wikipedia.org/wiki/Test-driven_development)
- [持续集成](https://en.wikipedia.org/wiki/Continuous_integration)
- [版本控制](https://en.wikipedia.org/wiki/Version_control)

我没有用我们的编译器做过任何单元测试。这里的主要原因是代码在函数的 API 方面非常灵活。我没有使用传统的开发瀑布模型，所以我会花太多时间重写我的单元测试来匹配所有函数的最新 API。所以，从某种意义上说，我生活在这里是危险的：代码中会有一些我们还没有发现的潜在错误。

然而，肯定会有更多的错误，编译器看起来像是接受了 C 语言，但当然这不是真的。编译器违反了[最少意外原则](https://en.wikipedia.org/wiki/Principle_of_least_astonishment)。我们需要花一些时间来添加一个“普通”C程序员希望看到的功能。

## 还有一个惊喜

最后，我们对编译器的功能有了一个惊喜。不久前，我有意省略了测试函数调用的参数数量和类型与函数原型匹配的代码（在`expr.c`中）：

```
  // XXX Check type of each argument against the function's prototype
```

我省略了这一点，因为我不想在其中一个步骤中添加太多新代码。

现在我们有了原型，我想最终添加对`printf()`的支持，这样我们就可以抛弃我们自己开发的`printint()`和`printchar()`函数。但是我们现在还不能这样做，因为`printf()`是一个可变函数：它可以接受可变数量的参数。而且，现在，我们的编译器只允许函数声明有固定数量的参数。

然而（这是一个很好的惊喜），因为我们不检查函数调用中参数的数量，我们可以向`printf()`传递任意数量的参数，只要我们已经给了它一个现有的原型。所以，目前这段代码（`tests/input53.c`）是有效的：

```c
int printf(char *fmt);

int main()
{
  printf("Hello world, %d\n", 23);
  return(0);
}
```

这是一件好事！

有一个问题。对于给定的`printf()`原型，当函数返回时，`cgcall()`中的清理代码不会调整堆栈指针，因为原型中的参数少于 6 个。但是我们可以用十个参数调用`printf()`：我们将其中的四个放入堆栈，但是当`printf()`返回时，`cgcall()`不会清除这四个参数。

## 总结与展望

在这一步中没有新的编译器代码，但我们现在正在测试编译器的错误检查能力，我们现在有 54 个回归测试来帮助确保我们在添加新功能时不会破坏编译器。幸运的是，我们现在可以使用`printf()`以及其他外部固定参数计数函数。

在我们编译器编写旅程的下一部分，我想我会尝试:

- 添加对外部预处理器的支持
- 允许编译器编译命令行上命名的多个文件
- 将`-o`、`-c`和`-S`标志添加到编译器中，使它看起来更像一个“正常的” C 编译器