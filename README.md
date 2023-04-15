# Apple Pie 解释器

Apple Pie 解释器是一个用来指导 Python 解释器大作业入门的简单 `demo` 。

此项目为今年新加入的部分，所以可能有一些问题，有任何疑问可以提出 Issue。

联系助教：[Sirius](https://github.com/siriusneo)



## 0 简介

Python 解释器是你们接触的第一个需要用到框架的工程向大作业，因此存在着难入门的问题。借助 Apple Pie 解释器你会比较迅速地解决如下问题：

- 我要写什么？
- ctx 是什么，我怎么用它？
- 一些其它注意事项。

下面让我们开始吧。



## 1 文件结构与部署

```
ApplePie
├── README.md
├── generated
├── third_party
├── src
│   ├── main.cpp
│   ├── Scope.h
│   ├── Exception.h
│   ├── utils.h
│   ├── Evalvisitor.cpp
│   └── Evalvisitor.h
└── CMakeLists.txt
```

Apple Pie 的结构和 Python 解释器是差不多的。其中 `generated` 和 `third_party` 是 `antlr` 生成的文件，不要去修改它们（除非你要实现新的语法，你需要修改 `.g4` 文件）。



### 1.1 main.cpp

这里面大部分也是不需要修改的。你可以尝试修改的有：

-  `input(std::cin);`  

你可以将 `std::cin`  换成文件输入流等来改变程序的输入方式。

```c++
int main(int argc, const char* argv[]){
    //todo:please don't modify the code below the construction of ifs if you want to use visitor mode
    ANTLRInputStream input(std::cin);
    Python3Lexer lexer(&input);
    CommonTokenStream tokens(&lexer);
    tokens.fill();
    Python3Parser parser(&tokens);
    tree::ParseTree* tree=parser.file_input();
    EvalVisitor visitor;
    visitor.visit(tree);
    return 0;
}
```



### 1.2 Evalvisitor.h & **Evalvisitor.cpp**

这是你需要实现的类，它是语法树上的 `visitor`。

Apple Pie 已经完成了 `Apple Python` 的 `visitor` ，但你要写的远比这复杂。

其它剩余的文件都是为了实现它而定义的头文件。



### 1.3 部署

请使用 Linux 环境（or WSL），Windows 环境有很多未知问题。

在 ApplePie 目录下打开终端，执行：

```sh
cmake .
sudo make
```

来编译项目，第一次编译的比较长。

运行 ApplePie：

```sh
./code
```

运行测试样例：

```sh
./code < test.in
```



## 2 Apple Python 语法

Apple Pie 解释器所解释的语言，它的语法非常简单：没有函数定义、条件分支、循环等等。

注意，由于 Apple Pie 使用的语法文件和 Python 解释器一样，因此使用 Python 语法并不会导致 `Syntax Error`

但是会发生未实现错误（`UNIMPLEMENT`）或者未定义行为（Undefined Behaviour）



### 2.1 类型

只有整型。并且只有 `int` 类型，没有负数。因此也没有类型转换等问题。



### 2.2 数据 

只有两类数据：

- 非负整型常量。以数字形式出现。
- 整型变量。以变量名形式出现，变量名只允许大小写字母。



### 2.3 语句

它仅支持：

- 整型变量赋值。为方便，赋值语句返回 0 。

  ```python
  a = 1
  ```

- 加法运算符。返回两个整数相加的值。

  ```
  1 + 2
  ```

- 乘法运算符。返回两个整数相乘的值。

  ```
  a * 3
  ```

- 括号。改变算式的优先级。返回括号内部算式的值。

  ```
  (1 + 2) * 3
  ```

- 函数调用。为方便，函数调用结果返回 0。

  ```python
  print(a)
  ```

  

### 2.4 内置函数

你可能对 2.3 的最后一点有疑问：没有函数定义的语言怎么还能调用函数呢？

实际上，Apple Python 有一个内置函数 `print` ，这个函数只有一个参数，作用是输出该整型变量。

为了方便，该函数的返回值为整数 `0` 。

在 Python 解释器的大作业中，你需要实现比这更多的内置函数。



## 3 实现



### 3.1 语法树遍历

从一个最简单的 Apple Pie 语句讲起

```
1 + 2
```

阅读语法文件 `Python3.g4` 后，你会看到

```g4
atom: (NAME | NUMBER | STRING+| 'None' | 'True' | 'False' | ('(' test ')'));
```

语法 `atom` 是 Python 中最底层的语法结构，在 Apple Pie 中你可以忽略掉多余部分：

```g4
atom: (NAME | NUMBER);
```

显然 1 和 2 是个 `NUMBER` ，因此它们匹配 `atom` 语法。

对于每个语法单元，你的 `Evalvisitor` 都有一个 `visit` 函数来访问他们。为了解释数字常量，我们只要简单地返回它们的值就好。

```c++
virtual antlrcpp::Any visitAtom(Python3Parser::AtomContext *ctx) override {
    if (ctx->NUMBER()) {
        return stringToInt(ctx->NUMBER()->getText());
    }
    //...
}
```

 `ctx` 全称为 `Context` ，表示一个语法树上一个节点的所有上下文信息。`getText()` 可以返回一个 `TOKEN` 的字符串原文。

对于一些你暂时不想处理的节点，你可以直接访问子节点来跳过它们（Apple Pie 由于语法简单，所以跳过了大量语法节点）

```c++
virtual antlrcpp::Any visitFuncdef(Python3Parser::FuncdefContext *ctx) override {
    // no func def
    return visitChildren(ctx);
}
```



那么 `+` 怎么处理？继续看语法文件

```g4
arith_expr: term (addorsub_op term)*;
```

学过正则表达式的同学可能知道，这句话的意思就是一个 `term` 后面跟一个或多个 `addorsub_op term` 。

在这个例子，匹配结构为 `term addorsub_op term`

`term` 是一个上层语法单元，经过向下递归最后到 `atom` ，然后返回常量的值。

我们要对常量进行加法处理，实现如下：

```c++
virtual antlrcpp::Any visitArith_expr(Python3Parser::Arith_exprContext *ctx) override {
        auto termArray = ctx->term();
        if (termArray.size() == 1) return visitTerm(termArray[0]).as<int>();
        auto opArray = ctx->addorsub_op();
        int ret = visitTerm(termArray[0]).as<int>();
        for (int i = 1; i < termArray.size(); ++i) {
            std::string tmpOp = opArray[i-1]->getText();
            if (tmpOp == "+") ret += visitTerm(termArray[i]).as<int>();
            else throw Exception("", UNIMPLEMENTED);
        }
        return ret;
    }
```

注意一个语法规则中若有单元多次出现，将返回一个 `vector`，如代码中的 `ctx->term()` （因为你没法确定 `arith_expr` 里有几个 `term` ）

然后从左到右做加法即可。你可以尝试输出 `ret`，得到结果 `3` 则实现正确。（因为我们还未实现 `print` 函数）



### 3.2 变量管理

我们实现了常量的运算，接下来是变量。

如何管理变量名？我们用一个 `Scope` （可以理解为命名空间）。

在完整的 Python 中，`Scope` 应当是一个栈的结构：局部的 `Scope` 优先于全局的 `Scope`，这样就能达到局部变量覆盖全局变量的目的。

但是 Apple Python 只有一个 `Scope` —— 全局命名空间。



实现 `Scope` ，我们需要一个变量名 $\longmapsto$ 变量值的 `map`，并支持注册一个变量，查询一个变量。

```c++
#include <map>
#include <string>

class Scope {

    private:
        std::map<std::string, int> varTable;

    public:
        Scope(): varTable() {}
        void varRegister(const std::string& varName, int varData) {
            varTable[varName] = varData;
        }    

        std::pair<bool, int> varQuery(const std::string& varName) const {
            auto it = varTable.find(varName);
            if (it == varTable.end()) return std::make_pair(false, 0);
            return std::make_pair(true, it->second);
        }
};
```



什么时候会访问一个变量的值？由上文可以知道变量名 `NAME` 也是一个 `atom` 语法的匹配内容，因此它也在 `visitAtom` 处处理。

```c++
virtual antlrcpp::Any visitAtom(Python3Parser::AtomContext *ctx) override {
        if (ctx->NUMBER()) {
            return stringToInt(ctx->NUMBER()->getText());
        }
        else if (ctx->NAME()) {
            auto result = scope.varQuery(ctx->NAME()->getText());
            if (result.first) {
                return result.second;
            }
            else throw Exception(ctx->NAME()->getText(), UNDEFINED);
        }
    	//...
    }
```

我们直接向 `Scope` 查询变量即可。

什么时候定义变量呢？

```g4
expr_stmt: testlist ( (augassign testlist) |
                     ('=' testlist)*); // 连等 加等/减等/...
```

Apple Pie 不支持连等以及 `augassign` 的赋值方式（+= -= ...），因此我这里做了很严格的判定。

```c++
virtual antlrcpp::Any visitExpr_stmt(Python3Parser::Expr_stmtContext *ctx) override {
    if (ctx->augassign()) {
    throw Exception("", UNIMPLEMENTED);
    }
    auto testlistArray = ctx->testlist();
        if (testlistArray.size() == 1) {
        visitTestlist(testlistArray[0]);
        return 0;
    }
    else if (testlistArray.size() > 2) {
    	throw Exception("", UNIMPLEMENTED);
    }

    std::string varName = testlistArray[0]->getText();
    int varData = visitTestlist(testlistArray[1]);

    if (!validateVarName(varName)) {
    	throw Exception("", INVALID_VARNAME);
    }
    scope.varRegister(varName, varData);
    return 0;
}
```

思路是只支持两个变量赋值，并且左侧只能是一个合法变量名，这里我不选择 `visit` 左侧的 `testlist`，而选择直接获取它的文本，降低编程难度。



### 3.3 运算优先级

注意到 Apple Pie 支持乘法和括号。这种运算优先级是怎么实现的呢？

```g4
arith_expr: term (addorsub_op term)*;
addorsub_op: '+'|'-';
term: factor (muldivmod_op factor)*;
```

看如下定义，可以看出语法文件把只有乘、除的表达式规定为 `term` ，然后再将 `term` 用加减号连起来。

这样我们就会先计算 `term`，再计算 `arith_expr` 。

当然，这是老式的 `grammar` 设计方式，新风格的设计方式是用 `|` 相连，这在你们的编译器大作业会遇到。



那么括号呢？还记得上面提到的 `atom` 语法吗：

```g4
atom: (NAME | NUMBER | STRING+| 'None' | 'True' | 'False' | ('(' test ')'));
```



我们看到最右边的 `('(' test ')'));`  

可以分析出 `test` 是一个很顶层的表达式（基本上把什么都算完了），而这里采用一个巧妙的回指，实现括号内先计算。

```c++
virtual antlrcpp::Any visitAtom(Python3Parser::AtomContext *ctx) override {
    if (ctx->NUMBER()) {
        return stringToInt(ctx->NUMBER()->getText());
    }
    else if (ctx->NAME()) {
        auto result = scope.varQuery(ctx->NAME()->getText());
        if (result.first) {
            return result.second;
        }
        else throw Exception(ctx->NAME()->getText(), UNDEFINED);
    }
    else if (ctx->test()) return visitTest(ctx->test()).as<int>();
    throw Exception("", UNIMPLEMENTED);
}
```

实现并不难，直接 `visit` 它并返回就好了。



### 3.4 print 函数的实现

到此为止，你的程序应该能简单地定义变量，并进行一些加、乘运算了。

接下来我们希望你能将结果输出在屏幕上——这要求你要能解释一个函数的调用。

函数的调用在哪呢：

```g4
atom_expr: atom trailer?;
trailer: '(' (arglist)? ')' ;
arglist: argument (',' argument)*  (',')?;
argument: ( test |
            test '=' test );
```

Apple Pie 的 print 只支持单变量，因此我们对其它情况统统报错。

```c++
virtual antlrcpp::Any visitAtom_expr(Python3Parser::Atom_exprContext *ctx) override {
    if (!ctx->trailer()) return visitAtom(ctx->atom()).as<int>();
    auto functionName = ctx->atom()->getText();
    auto argsArray = visitTrailer(ctx->trailer()).as<std::vector<int>>();
    if (argsArray.size() != 1) {
        throw Exception(functionName, INVALID_FUNC_CALL);
    }
    if (functionName == "print") {
        std::cout << argsArray[0] << std::endl;
        return 0;
    }
    throw Exception("", UNIMPLEMENTED);
}
```

当没有括号 `trailer` 时候，它仅仅单个 `atom` 。

当有括号时，我们应当将其视作一个函数调用，我们获取它的函数名、参数表，并尝试调用它。

注意，当你写真正的 Python 解释器时候，你需要支持定义函数，因此你需要丰富你的 `Scope` 功能，使其可以注册一个函数并调用它。



## 4 异常处理

你可能会注意到 Apple Pie 里面已经有一部分异常处理的代码了。

事实上，一个合格的解释器不仅仅要能解释正确代码的运行结果，还要能对错误代码进行语法报错。

在本次作业中，这部分被归为 **Bonus**，但我推荐你们都尝试写一些（不必要做到 100% 报错成功，只是写到错误情况可以顺便处理一下）。

### 4.1 异常类

Apple Pie 中实现了一个简单的异常类，并且支持如下错误：

- `UNDEFINED`  变量未定义却被调用
- `UNIMPLEMENTED`  Apple Pie 未实现的功能
- `INVALID_VARNAME` 变量定义时，不合法的变量名
- `INVALID_FUNC_CALL` 不合法的函数调用，参数错误

设计简单的异常类

```c++
enum ExceptionType {UNDEFINED, UNIMPLEMENTED, INVALID_VARNAME, INVALID_FUNC_CALL};

class Exception {

    private:
        std::string message;

    public:
        Exception(const std::string& arg, ExceptionType type) {
            if (type == UNIMPLEMENTED) message = "Sorry, Apple Pie do not implement this.";
            else if (type == UNDEFINED) message = "Undefined Variable: " + arg;
            else if (type == INVALID_FUNC_CALL) message = "Invalid function call: " + arg;
        }    

        std::string what() {return message;}

};
```



### 4.2 异常捕获

在这个过程有两个环节可能抛出异常：Parse 以及 Visit。

其中 Visit 是我们自定义的部分，我们只要对这个环节捕获异常即可。

因此对 main 函数稍作修改：

```c++
int main(int argc, const char* argv[]){
    //todo:please don't modify the code below the construction of ifs if you want to use visitor mode
    ANTLRInputStream input(std::cin);
    Python3Lexer lexer(&input);
    CommonTokenStream tokens(&lexer);
    tokens.fill();
    Python3Parser parser(&tokens);
    tree::ParseTree* tree=parser.file_input();
    EvalVisitor visitor;
    try {
        visitor.visit(tree);
    } catch(Exception& e) {
        std::cout << e.what() << '\n';
        return 0;
    }
    return 0;
}
```



## 5 注意事项

- `bad_cast`

  这是由于 `Any` 类不能转换为你要求的类型，检查是否在某处返回类型错误。

- 返回 `vector`

  当你需要遍历一个 `vector` 时，请勿这样写：

  ```c++
  for (auto x : ctx->term()) ...
  ```

  因为每次 `vector` 都需要重新构造，会使得程序非常慢！

  ```c++
  auto termArray = ctx->term();
  for (auto x : termArray) ...
  ```

  

