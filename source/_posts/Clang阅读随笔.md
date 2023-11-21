---
title: "Clang 阅读随笔"
category: CS&Maths
#id: 57
date: 2023-11-20 09:00:00
tags: 
  - LLVM
  - Clang
  - Compiler
toc: true
#sticky: 1 # 数字越大置顶优先级越高。数字都需要大于 0。
#cover: /images/about.jpg # 指定封面图片的 URL
timeline: article  # 展示在时间线列表中
---
这是在阅读 Clang 时遇到不会内容时的随笔，看到哪写到哪，可能不会有任何的结构（Who knows）。

😘 在此先疯狂鸣谢一波 ChatGPT 和 Claude 😍。

<!--more-->

## CompilerInstance 类
CompilerInstance 是 LLVM/Clang 编译器框架中的一个类，用于管理整个编译过程中的状态和组件。它负责协调和组织编译器的不同部分，包括预处理器、语法分析器、语义分析器等。在编译的不同阶段，CompilerInstance 提供了访问和控制这些组件的方法。

### CompilerInstance 的结构
CompilerInstance 的具体结构可能会有一些复杂，因为它需要协调各个编译器组件的工作。一般而言，它包含了以下关键成员：

- 语法树（AST）： 用于表示源代码结构的树形数据结构。AST 在编译器的语法分析阶段构建，以便更容易进行语义分析和代码生成。
- 预处理器（Preprocessor）： 负责对源代码进行预处理，包括宏展开、条件编译、去注释等。预处理器生成预处理后的源代码，供后续的编译阶段使用。
- 语义分析器（Sema）： 在 AST 的基础上进行语义分析，执行类型检查、符号解析等操作。语义分析是编译器的重要阶段，它确保程序语义的正确性。
- 代码生成器（Code Generator）： 将语义分析阶段得到的中间表示（IR）转换为目标代码。这是编译器的最后阶段。
- 源码管理器（SourceManager）： 负责跟踪源代码的位置信息，以便在编译过程中进行错误报告和调试信息生成。
- 前端选项（FrontendOptions）： 包含与编译过程相关的各种选项，如编译目标、优化级别等。

这只是 CompilerInstance 结构的一般概述，具体实现可能更为复杂。

## Static Analyzer

Static Analyzer 的源代码入口主要位于 Clang 代码库的 `lib/StaticAnalyzer` 目录中。以下是几个关键文件和目录：

- **`lib/StaticAnalyzer/Core`：** 包含 Static Analyzer 的核心实现，如路径敏感性分析器、检查器管理器等。
- **`lib/StaticAnalyzer/Checkers`：** 包含各种 Checker 的实现，每个 Checker 都有一个对应的目录，其中包含 Checker 的具体实现和规则定义。
- **`lib/StaticAnalyzer/Frontend`：** 包含与前端集成相关的代码，处理命令行参数，设置分析配置等。
- **`lib/StaticAnalyzer/PathSensitive`：** 包含路径敏感性分析的相关实现。
- **`lib/StaticAnalyzer/Frontend/AnalysisConsumer.cpp`：** 这个文件定义了 `AnalysisConsumer` 类，它是 Static Analyzer 在编译过程中的 AST 消费者，负责驱动整个静态分析过程。
- **`lib/StaticAnalyzer/Core/CheckerManager.cpp`：** 这个文件包含 `CheckerManager` 类的实现，负责管理所有的 Checker。
- **`tools/scan-build`：** 该工具用于通过 Clang Static Analyzer 进行静态分析。
  
调用情况如图所示：
![Clang Static Analyzer 调用情况](/Clang阅读随笔/image1.png)

总体而言，`AnalysisConsumer.cpp` 和 `CheckerManager.cpp` 可以看作是 Static Analyzer 的源代码入口，它们定义了整个分析过程的驱动和管理逻辑。其他目录则包含了各个组件的具体实现。

### Clang 静态分析模式
在 Clang 的静态分析器（`/llvm/tools/clang/lib/StaticAnalyzer/Frontend/AnalysisConsumer.cpp: HandleTranslationUnit()`）中，`AM_Syntax` 和 `AM_Path` 分别代表不同的分析模式，用于控制静态分析的行为。以下是它们的区别：

1. **`AM_Syntax`（语法分析模式）：**
   - **含义：** `AM_Syntax` 表示语法分析模式，指的是在分析中仅关注语法层面的结构，而不考虑程序的具体路径执行信息。
   - **行为：** 在 `AM_Syntax` 模式下，静态分析器主要关注程序的语法结构，执行基本的语法检查和分析，例如识别语法错误、检查变量的声明和使用情况等。这种模式下的分析通常更快，但可能会错过一些路径敏感性的问题。

2. **`AM_Path`（路径敏感分析模式）：**
   - **含义：** `AM_Path` 表示路径敏感分析模式，指的是在分析中考虑程序的具体路径执行信息，以检测可能的路径相关问题。
   - **行为：** 在 `AM_Path` 模式下，静态分析器会模拟程序的不同执行路径，考虑程序在不同条件下的行为。这种模式下的分析可以发现更多的潜在问题，例如路径上的条件分支错误、空指针解引用等。然而，路径敏感分析通常会增加分析的复杂性和执行时间。

总的来说，区别在于分析是否关注程序的路径执行信息。`AM_Syntax` 主要关注语法结构，而 `AM_Path` 则在此基础上考虑了路径敏感性，以更全面地发现潜在问题。

[Cross-checking Semantic Correctness: The Case of Finding File System Bugs](/2023/08/31/Cross-checking_Semantic_Correctness:The_Case_of_Finding_File_System_Bugs阅读/) 这篇文章中使用的是`AM_Syntax`。

## Control Flow Graph
控制流图（control-flow graph）简称CFG，是计算机科学中的表示法，利用数学中图的表示方式，标示计算机程序执行过程中所经过的所有路径。

节点表示基本块（basic block），边表示控制流的流向。Basic Block是CFG的主体。

> Basic Block：一个最长的语句序列，并保证入口只能在最开始指令且出口只能在最后一个指令。

### 构造Basic Blocks

- Input：程序P的三地址码序列
- Output：程序P的basic blocks
- 算法
  - 确定leaders（每个basic block的头）
    - 序列中的第一个指令
    - 跳转指令的目标指令
    - 跳转指令的下一条指令
    - return指令
  - 每个Basic Block包含其leader至下一个leader前的所有语句
- 算法的另一种阐释
  - 初始语句作为第一个基本块的入口
  - 遇到标号类语句，结束当前基本块，标号作为新基本块的入口（标号不在当前基本块中，而是划到下一个基本块）
  - 遇到转移类语句时，结束当前当前基本块，转移语句作为当前基本块的结尾
  - 当给引用类型变量赋值时，结束基本块，作为出口
- 基本块有以下特点：
  - 单一入口点，其他程式中，没有任何一个分支指令的目标在这段程式基本块之内（基本块的第一行除外）。
  - 单一结束点，这段程式一定要执行完最后一行才会执行其他基本块的程式。
  - 因为上述特点，基本块中的程式，只要执行了第一行，后面的程式码就会依序执行，每一行程式都会执行一次。

### 构造CFG

添加边，在以下两种情况下：

- 无条件跳转：一条无条件跳转语句会创建一个有向边，将当前基本块的出口指向目标基本块的入口。
- 条件跳转：创建两条有向边，分别表示条件为真和条件为假时的目标基本块。
  

在构建控制流图（CFG）时，通常会引入两个特殊的节点，即entry节点和exit节点，以更好地表示程序的整体控制流。这两个节点不对应具体的代码块，而是代表程序的入口和出口。

- entry节点： entry节点表示程序的起始点。它没有对应的代码，而是作为整个控制流图的入口。从entry节点开始，程序的执行流程进入其他基本块。
- exit节点： exit节点表示程序的结束点。类似地，它也没有对应的代码，而是作为整个控制流图的出口。程序的执行流程可能从不同的基本块经过，最终都会汇聚到exit节点。


**示例：**

考虑以下伪代码：

```
1.  if condition
2.    statement1
3.  else
4.    statement2
5.  endif
6.  statement3
```


1. 划分基本块：

   - 基本块1: Line. 1
   - 基本块2: Line. 2 - 3
   - 基本块3: Line. 4
   - 基本块4: Line. 5 - 6
  
    `else` 为转移类语句，`endif` 为标号类语句。
2. 构建基本块间的控制流关系：

   - 从基本块1到基本块2，因为有条件跳转语句。
   - 从基本块1到基本块3，因为有条件跳转语句。
   - 从基本块2到基本块4，因为是顺序执行。
   - 从基本块3到基本块4，因为是顺序执行。

3. 建立CFG：

  ```mermaid
  graph TD
  
    entry --> 1[<div style="text-align:left;">1.  if condition</div>]
    1 --> 2[<div style="text-align:left;">2.    statement1<br>3.  else</div>]
    1 --> 3[<div style="text-align:left;">4.    statement2</div>]
    2 --> 4[<div style="text-align:left;">5.  endif<br>6.  statement3</div>]
    3 --> 4
    4 --> exit

    style entry fill:#98FB98,stroke:#4CAF50,stroke-width:2px;
    style exit fill:#98FB98,stroke:#4CAF50,stroke-width:2px;

  ```
